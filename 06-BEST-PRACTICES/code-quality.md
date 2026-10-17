# Code Quality for SAP Fiori and UI5

## Scope and prerequisites

This guide concerns application-owned JavaScript, XML views, fragments, styles, and supporting documentation in freestyle UI5 applications. Apply the same principles at supported extension points in SAP Fiori elements; do not rewrite framework-generated behavior merely to satisfy an application convention. The goal is predictable maintenance: another developer should be able to identify ownership, change a business rule, and verify the result without understanding every screen.

Assume a maintained UI5 1.x runtime, asynchronous module loading, an application descriptor, and project-installed tooling recorded in a lockfile. Examples use ES2015 syntax and must match the application's browser support or existing transpilation process. They are documentation examples, not files added to an application. Check each API against the exact deployed SDK, especially when supporting an older on-premise landscape. UI5 CLI and UI5 runtime versions are separate compatibility decisions.

Before changing conventions, inspect existing package scripts, lint configuration, formatting configuration, browser targets, framework dependencies, and deployment constraints. Do not silently introduce a newer ESLint configuration format into a project whose installed ESLint does not support it. This chapter recommends policy without installing packages or prescribing unreviewed versions.

## Establish a quality contract

Agree on a short contract that identifies mandatory checks, responsible owners, and exception handling. Separate correctness rules from layout rules. A missing dependency declaration can prevent application startup; a trailing comma normally cannot. Treat these differently in review discussions and migration plans.

The contract should cover application behavior, public module interfaces, lifecycle cleanup, stable identifiers, error handling, accessible interaction, and deployment compatibility. Define “done” as an observable outcome rather than merely a green style check. For example, completing an approval change includes a rejected service response, duplicate submission prevention, and a recovery path that preserves user input.

Record exceptions near the relevant rule or decision. Give each exception a reason, scope, owner, and review date. An undocumented global suppression is not an exception policy. Periodically remove obsolete exceptions after runtime upgrades or component retirement.

## Divide ESLint and Prettier responsibilities

Use ESLint for detectable programming mistakes and project-specific restrictions. Useful categories include undefined variables, unreachable code, accidental reassignment, suspicious equality, unused dependencies, and unsupported API patterns. Use Prettier for consistent whitespace, line wrapping, quotation layout, and other formatting choices supported by its installed version.

Avoid assigning both tools competing formatting rules. Review the project's existing integration before enabling format-related ESLint rules. XML formatting support depends on the formatter and plugins already approved in that project; do not assume the JavaScript formatter handles UI5 XML correctly.

Run checks through existing scripts, such as `npm run lint` and `npm run format:check`, only when those scripts exist. Inspect their definitions first. Never interpret an absent script as permission to download an arbitrary tool through an auto-installing command. If calling an installed binary directly is necessary, use the repository-local executable rather than a globally mutable installation.

Start adoption by measuring current findings. Fix a coherent category, then make that category blocking for changed code. Mixing thousands of formatting changes with a business change hides defects and makes rollback harder. A dedicated formatting commit or pull request is easier to review and can be excluded from blame analysis where repository tooling supports that workflow.

## Configure runtime boundaries accurately

Lint browser modules, test modules, and Node-based build helpers with different environments. Browser code can legitimately refer to framework bootstrap facilities; build helpers must not assume `window`. QUnit globals belong only in test contexts. Narrow file globs prevent a permissive test environment from accidentally allowing undefined production dependencies.

Allow `sap` as the framework entry point when needed, but do not use that allowance to justify undeclared `sap.m` constructors or mutable application globals. Import a control or service through its documented module path. The official module guidance requires one unnamed top-level `sap.ui.define` per module and case-sensitive dependency names [1].

Use UI5-aware rules only when the project's reviewed toolchain already includes them. Record the rule package version and intended runtime baseline. A rule based on a new SDK can incorrectly reject valid compatibility code or miss APIs unavailable on an older target. Automated findings need context, but a false positive should be demonstrated rather than asserted.

## Name business concepts consistently

Choose names that describe purpose before implementation. Prefer `approvalLimit`, `selectedOrder`, and `isSubmissionPending` over `value`, `object`, and `flag`. A boolean should read naturally in a condition. Distinguish a document identifier from a display number and distinguish a formatted amount from a numeric quantity.

Many UI5 JavaScript codebases use type prefixes such as `oOrder`, `aItems`, and `sStatus`; others use descriptive unprefixed names or TypeScript types. Follow the local convention consistently instead of mixing styles within a module. Prefixes must not replace meaningful domain names or imply guarantees that the value does not satisfy.

Use PascalCase for exported classes, established UI5 filenames such as `Orders.controller.js`, and deliberate module namespaces. Match filename case exactly on disk and in imports. A reference that works on a case-insensitive workstation can fail after a Linux build or production deployment.

Name event handlers after the user action or event, such as `onSubmitOrder`, not the current widget color or screen position. Give important controls stable local IDs such as `submitOrder` and `supplierInput`. Stable IDs support tests, adaptation, and diagnostics; they should survive layout changes.

## Organize around clear ownership

A practical application keeps views and controllers close to their navigation responsibilities, pure transformations in small model or utility modules, service access behind explicit boundaries, and test fixtures under test-owned directories. Keep `Component.js` focused on component initialization and ownership rather than making it a general business-service registry.

Use the descriptor for declared models, dependencies, and routing where supported. Avoid duplicating those declarations imperatively in controllers. Multiple independently configured instances of the same service model lead to inconsistent batching, message handling, and cache behavior.

For larger applications, introduce feature folders only when they reduce navigation cost. A directory per tiny helper increases indirection. Conversely, a single `util.js` containing authorization checks, date formatting, downloads, and navigation hides unrelated responsibilities. Extract by capability, not by arbitrary file size.

Keep generated build output separate from authored sources according to repository policy. Never manually repair a generated preload file. Fix its source or build definition, then regenerate it through the approved pipeline. UI5 Builder tasks depend on project type and configuration; do not assume every build produces the same resources [2].

## Declare dependencies without side effects

Use dependency arrays as the module's explicit interface to the loader. Keep callback parameter order aligned with dependency order. Remove imports that are genuinely unused, while retaining documented side-effect imports only where their purpose is unavoidable and clearly explained.

Do not execute application work at module evaluation time. Constructing controls, issuing network requests, or registering persistent listeners during import makes startup order and tests unpredictable. Export a function or class and let an owning component initiate the work.

Prevent cycles by moving shared concepts into lower-level modules. Two controllers should not import each other to coordinate state. Use a component-owned model or a narrow orchestration service instead. If introducing events, define payloads and subscription lifetimes; an event bus without ownership can be harder to reason about than the original cycle.

The following is a complete illustrative pure module for `webapp/model/orderRules.js`. Its input is an already-normalized application object, not an OData entity with implicit type conversion. Amounts are integer minor units in a single currency, avoiding floating-point price calculations.

```javascript
sap.ui.define([], function () {
    "use strict";

    function canSubmit(order) {
        return Boolean(
            order &&
            order.status === "Draft" &&
            order.supplierId &&
            Number.isSafeInteger(order.totalMinor) &&
            order.totalMinor > 0
        );
    }

    return { canSubmit: canSubmit };
});
```

This rule is a client-side usability check, not authorization. The backend must independently validate permission, state transitions, currency rules, and concurrent changes. Documenting that boundary prevents later developers from treating a disabled button as a security control.

## Keep controllers as coordinators

A controller should translate user intent into model or service operations and update presentation state. It should not implement transport protocols, currency arithmetic, and every business rule inline. Extract logic when it has an independent contract or repeats for a substantive reason.

Do not extract every two-line expression into a remote abstraction. Excessive delegation obscures a simple interaction. A reviewer should be able to follow the main path of a submission without opening ten files. Keep the happy path readable and move complex validation or response normalization into named functions.

Use guard clauses for invalid or unavailable contexts, but distinguish expected absence from broken invariants. An empty optional selection can be normal. A missing required named model indicates configuration damage and should not be silently ignored. Fail with an actionable, sanitized diagnostic rather than allowing a later unrelated exception.

## Model asynchronous state explicitly

Represent pending, success, empty, and failure states deliberately. A single boolean called `loaded` cannot explain whether a request has not started or failed. Keep transient view state separate from persisted business data so that a busy flag is never submitted accidentally.

Disable repeat actions while a mutation is pending when appropriate, but also design for server-side idempotency or conflict detection. Browser controls cannot prevent retries from another tab or a disrupted connection. Preserve input after validation failure and avoid refreshing away unsaved edits.

When navigation can change during a request, associate the result with the initiating context. A late response for order A must not overwrite order B after the user navigates. Prefer documented cancellation where available, or compare a request generation/context before applying results. Do not assume every UI5 model operation returns a native promise; OData V2 and V4 APIs differ.

Use a guaranteed cleanup path appropriate to the supported JavaScript baseline to reset busy state. Cleanup must not replace the original failure with an unrelated exception. Test both successful and rejected operations, including teardown while work remains pending.

## Respect lifecycle ownership

The object that creates a listener, timer, dialog, or auxiliary model must define who removes or destroys it. Attach router listeners once at an appropriate lifecycle boundary and detach them when their owner exits. Repeated attachment during rendering commonly causes multiple requests from one user action.

Use view dependents for dialogs where appropriate so model propagation and destruction follow the view. Check the exact fragment-loading API supported by the runtime. Do not mix old synchronous fragment factories with newer asynchronous patterns without an explicit compatibility reason.

Keep `onAfterRendering` for work that genuinely requires rendered output. It is not a substitute for initialization. UI5 owns control DOM: direct changes to generated elements can disappear after rerendering and bypass accessibility, state, or invalidation logic. Prefer public control properties, aggregations, associations, and events.

## Control complexity with observable rules

Use complexity metrics as prompts for review, not universal correctness scores. A nested decision tree handling approval states might need a transition table; a long declarative field mapping may remain easy to inspect. Assess branching depth, state combinations, side effects, and the number of concepts a function requires.

For state transitions, write down allowed source and target states before coding. Separate “may this action be offered?” from “how is the action executed?” Keep a single authoritative mapping instead of duplicating status strings across formatters, controllers, and XML expressions.

Avoid dense expression bindings that mix localization, arithmetic, and business eligibility. Simple visibility bindings are useful; complex domain decisions belong in testable code or backend-derived capabilities. Watch for expression differences involving `null`, empty strings, zero, and undefined model paths.

When refactoring complex behavior, capture existing behavior with focused tests first. Change structure separately from business semantics whenever possible. A refactor with new rules and new transport behavior is harder to validate and backport.

## Document contracts rather than syntax

Document exported functions, non-obvious model shapes, units, time zones, and rejection behavior. Explain whether a function mutates its input, whether it accepts missing data, and whether its result is localized. These facts are more valuable than comments restating an assignment.

Use JSDoc supported by the existing project where it improves editor feedback. Keep private implementation notes short. A public utility that accepts money should specify currency and representation; a date helper should distinguish an instant from a date-only business field.

Record architectural decisions when alternatives have real consequences: shared model versus feature model, optimistic versus confirmed updates, or a special compatibility workaround. Include why the rejected option was unsuitable and what condition would permit revisiting the decision.

User-facing text belongs in resource bundles rather than source comments or hardcoded labels. Give translators context for ambiguous words such as “Post,” “Release,” or “Plant.” Test long translations instead of truncating essential action names to satisfy a fixed-width design.

## Handle errors and logs safely

Map service errors into actionable messages without exposing internal stack traces, tokens, headers, or entire response bodies. A correlation identifier can help support teams, provided it is safe to display and does not encode sensitive business data.

Log a stable event category and minimal diagnostic context using the project's approved UI5 logging approach. Do not serialize whole models, user records, authorization claims, or attachment content. Lower log verbosity in production and enforce retention and access policies for centralized collectors.

Never catch an error solely to keep the console quiet. Either recover deliberately, present a meaningful failure, or rethrow to an owning boundary. Conversely, avoid showing a modal error for expected user cancellation. Classify failures so telemetry distinguishes business validation, authentication expiry, transport interruption, and application defects.

## Review XML, styles, and semantics

Review XML as executable application behavior. Check binding paths, named models, formatter references, stable IDs, and event handler names. Escaping rules differ between XML attributes and JavaScript strings; copying an expression between them can change its meaning.

Prefer standard controls and theme-aware classes over deeply nested selectors targeting framework internals. A selector tied to a generated suffix can break in a patch update. Never remove visible focus styling to make a screenshot cleaner, and do not hide a required focus target with CSS while leaving keyboard navigation dependent on it.

Assess custom controls more strictly than application views. Their public properties, invalidation behavior, keyboard model, renderer semantics, and cleanup affect every consumer. If a standard control covers the requirement, composition usually carries less maintenance risk than a new rendering implementation.

## Integrate quality into everyday work

Before implementation, identify acceptance criteria, affected models, runtime constraints, and existing tests. During implementation, keep the diff focused and update contracts alongside behavior. Before review, inspect the diff as a reviewer would: remove debugging output, accidental generated artifacts, and unrelated formatting.

In CI, run existing formatting and lint scripts before longer browser suites so cheap failures return quickly. Give jobs minimal permissions and avoid exposing deployment credentials to untrusted contributions. Tool output can include file paths or snippets; treat artifacts according to repository confidentiality requirements.

Quality gates should explain failures and remediation. Do not let new warnings accumulate indefinitely under a “temporary” nonblocking setting. Track debt separately from feature acceptance, but block regressions in touched areas where reasonable. Exceptions should never allow a known production safety defect merely because a baseline already contains warnings.

## Practical integration pitfalls

A formatter can alter snapshots or long XML bindings without fixing their semantics. Review changed bindings explicitly. A module rename can update JavaScript imports while leaving descriptor routes and test selectors stale. Search all references, including case-sensitive resource names.

A shared helper can acquire view dependencies and turn a low-level module into a startup bottleneck. Keep pure modules free from controls and global model access. A seemingly harmless formatter that mutates a model can trigger repeated binding updates and make rendering order affect results.

Finally, code quality is not synonymous with passing lint. A perfectly formatted request that fetches every record, stores confidential responses in a shared cache, or excludes keyboard users remains defective. Use the companion guides to verify those dimensions instead of inventing one universal numeric score.

## Verification checklist

- [ ] Runtime, language baseline, and installed tooling assumptions are recorded.
- [ ] Dependencies use documented modules, exact case, and no accidental cycles.
- [ ] Controllers coordinate work; independent business rules have explicit contracts.
- [ ] Listeners, timers, models, and dialogs have lifecycle owners.
- [ ] Pending, failure, retry, navigation, and teardown behavior are covered.
- [ ] XML bindings, translations, stable IDs, and accessible focus remain correct.
- [ ] Logs and artifacts exclude credentials and unnecessary business data.
- [ ] Existing lint and formatting scripts pass in the consuming project.
- [ ] Exceptions are narrow, justified, owned, and scheduled for review.

## Related paths

The repository README lists these exact chapter paths; this guide does not require changing that index.

- [Testing strategy](./testing-strategy.md): `06-BEST-PRACTICES/testing-strategy.md`.
- [Git workflow](./git-workflow.md): `06-BEST-PRACTICES/git-workflow.md`.
- [Performance optimization](./performance-optimization.md): `06-BEST-PRACTICES/performance-optimization.md`.
- [Accessibility](./accessibility.md): `06-BEST-PRACTICES/accessibility.md`.
- [Repository index](../README.md).

## Official references

Research access: the official GitHub documentation below was retrieved successfully. Direct access to hosted UI5 documentation was unavailable; use the SDK matching the deployed runtime to confirm API support.

1. [OpenUI5: Best Practices for Loading Modules](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/best-practices-for-loading-modules-00737d6.md), consulted 2026-10-17. Establishes unnamed modules, case-sensitive paths, explicit imports, and acyclic dependencies.
2. [UI5 CLI v4: Builder](https://github.com/UI5/cli/blob/v4/docs/pages/Builder.md), consulted 2026-10-17. Documents project-specific tasks, preload generation, minification, and source-map considerations.
3. [OpenUI5: Unit Testing with QUnit](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/unit-testing-with-qunit-09d145c.md), consulted 2026-10-17. Supports testing asynchronous behavior rather than relying on static checks alone.
4. [OpenUI5: Screen Reader Support](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/screen-reader-support-for-openui5-controls-656e825.md), consulted 2026-10-17. Explains framework and application accessibility responsibilities.
