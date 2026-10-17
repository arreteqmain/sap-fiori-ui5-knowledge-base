# Testing Strategy for SAP Fiori and UI5

## Scope and prerequisites

This chapter describes a layered testing strategy for application-owned UI5 behavior: business rules, controller coordination, control bindings, navigation, service integration, and critical user journeys. It applies directly to freestyle applications and to supported extensions in Fiori elements. Tests should verify extension contracts rather than the internal implementation of generated pages.

Assume a maintained UI5 1.x project with an existing browser test runner, QUnit 2, asynchronous module loading, and a project-specific test bootstrap. Examples use ES2015 syntax. The deployed SDK, browser support, and approved tooling determine which APIs are actually available. UI5's bundled test-library versions vary across runtime releases; do not copy a modern Sinon example into an older bundled version without checking its API.

This document contains test-module examples and illustrative application code, not an installed harness. No test runner, coverage middleware, package, or CI action is introduced. Commands refer only to tools already installed in the consuming project. Read package scripts and lockfiles before running them, and use the smallest existing selector that covers a change.

## Translate risk into a test portfolio

Begin with business consequences rather than a desired test count. An incorrect approval decision, duplicate financial posting, or lost draft deserves stronger evidence than an optional visual preference. For every important action, identify its inputs, observable result, service boundary, failure modes, and supported recovery.

Use unit tests for deterministic rules and transformations, integration tests for view/controller/model collaboration, and a small number of deployed end-to-end checks for authentication, routing, service connectivity, and platform integration. These layers overlap intentionally but should not duplicate every assertion.

For an order submission feature, unit-test eligibility boundaries and request normalization. Use OPA5 to verify that invalid input blocks submission, a valid interaction produces a visible result, and a rejection preserves editable values. Use a controlled real-service test to verify authorization, concurrency, and response contracts. A browser mock cannot prove backend enforcement.

Write this mapping in the feature's acceptance criteria. Avoid an abstract pyramid that never identifies which layer owns which risk. Reviewers should know exactly where a regression in rounding, navigation, or service rejection would be detected.

## Make assertions describe behavior

Assert outcomes that matter to a consumer: a normalized object, an enabled action, an accessible error message, a persisted state, or a route change. Avoid asserting private method calls merely because they are easy to spy on. Private call-count assertions often fail harmless refactoring while missing incorrect user behavior.

Use precise assertions. An array being truthy does not establish that it contains the expected item. Check business keys, required fields, or exact calculated values. For dates, specify time zone and representation. For monetary values, compare the domain representation, such as integer minor units or validated decimal strings, rather than a locale-dependent display string.

Give each assertion a diagnostic message that states the intended rule. Failure output should identify the relevant case without printing the entire service response. Do not include customer names, credentials, or confidential identifiers in test names.

## Build small QUnit units

Extract deterministic business rules into modules with explicit inputs. Do not instantiate an application merely to test a pure predicate. UI5 officially supports QUnit, including asynchronous testing, and links its coverage and test-double integration guidance [1].

The following complete illustrative production module would live at `webapp/model/orderRules.js` in an application with namespace `example.orders`. It accepts already-normalized records and deliberately treats numeric strings as invalid.

```javascript
sap.ui.define([], function () {
    "use strict";

    return {
        canSubmit: function (order) {
            return Boolean(
                order &&
                order.status === "Draft" &&
                order.supplierId &&
                Number.isSafeInteger(order.totalMinor) &&
                order.totalMinor > 0
            );
        }
    };
});
```

The next module is an actual QUnit 2 test definition, conditional on that production module existing and the existing harness loading QUnit before this module. The bootstrap must map `example.orders` to the application's `webapp` root. It is not a self-starting test page, and no undefined test helper is implied.

```javascript
/* global QUnit */
sap.ui.define(["example/orders/model/orderRules"], function (rules) {
    "use strict";

    QUnit.module("Submission eligibility");

    QUnit.test("requires a positive integral amount and supplier", function (assert) {
        const base = {
            status: "Draft",
            supplierId: "SUPPLIER-TEST",
            totalMinor: 1
        };

        assert.strictEqual(rules.canSubmit(base), true, "One minor unit is valid");
        assert.strictEqual(
            rules.canSubmit(Object.assign({}, base, { totalMinor: 0 })),
            false,
            "Zero is not eligible"
        );
        assert.strictEqual(
            rules.canSubmit(Object.assign({}, base, { totalMinor: "1" })),
            false,
            "Transport strings require explicit normalization"
        );
        assert.strictEqual(
            rules.canSubmit(Object.assign({}, base, { supplierId: "" })),
            false,
            "A supplier is required"
        );
        assert.strictEqual(rules.canSubmit(null), false, "Missing data is safe");
    });
});
```

Add independent cases for non-draft states, negative amounts, unsafe integers, and fractional amounts in the consuming application. Do not confuse this client rule with authorization. Backend permissions and state transitions require separate service-side enforcement and verification.

## Test asynchronous completion honestly

QUnit 2 can wait for a returned promise. Return the promise representing the assertion path; otherwise a test may finish before the work completes. Use `assert.async()` only when callback-style APIs require it, and ensure every path completes exactly once.

The following self-contained QUnit test module demonstrates rejected asynchronous work and guaranteed busy-state cleanup. Its local `submit` function is illustrative application logic, not a framework helper. A real application should import its own implementation and use this pattern to test that implementation.

```javascript
/* global QUnit */
sap.ui.define([], function () {
    "use strict";

    function submit(state, save) {
        state.busy = true;
        return Promise.resolve().then(save).then(function (result) {
            state.busy = false;
            return result;
        }, function (error) {
            state.busy = false;
            throw error;
        });
    }

    QUnit.test("rejection clears busy state and preserves the error", function (assert) {
        const state = { busy: false };
        const failure = new Error("Synthetic service failure");
        assert.expect(3);

        const completion = submit(state, function () {
            return Promise.reject(failure);
        });
        assert.strictEqual(state.busy, true, "Submission starts busy");

        return completion.then(function () {
            assert.ok(false, "Submission should reject");
        }, function (error) {
            assert.strictEqual(error, failure, "Original failure reaches the owner");
            assert.strictEqual(state.busy, false, "Failure removes busy state");
        });
    });
});
```

Test overlapping requests and navigation separately. A test that resolves one promise successfully cannot detect a stale result overwriting a newer route context. Control response order deliberately through injected service functions or the project's existing stub facilities.

## Own setup and teardown

Construct fresh mutable objects for each test. Reset models, fixtures, router state, and stubbed methods in module hooks supported by the installed QUnit version. Destroy controls and views after use. Remove custom timers and event subscriptions even if an assertion fails.

Prefer a sandbox when the approved Sinon version provides one, and restore it in teardown. Do not globally replace UI5 internals for convenience. A leaked stub can make an unrelated test pass or fail depending on execution order. If a suite passes alone but fails after another suite, investigate shared state before adding retries.

For control tests, create controls inside the harness's owned fixture area and wait for rendering using the supported mechanism for that runtime. Do not assume a particular rendering utility exists on every supported UI5 patch. Rendering assertions should inspect public properties or stable semantic output, not undocumented generated suffixes.

## Separate test doubles by purpose

A stub supplies a controlled response, a spy records calls, and a mock generally includes expectations about interactions. Choose the least powerful double that answers the question. A simple injected function is often sufficient for a pure service wrapper.

Stub at an application-owned boundary where possible. For controller logic, replacing a service interface is usually more stable than patching low-level XHR behavior. For OData binding behavior, however, a service-shaped mock may be necessary because binding resolution, metadata, messages, and request sequencing are part of the integration under test.

Avoid reconstructing the production implementation inside a fake. A mock that uses the same faulty calculation as production produces reassuring agreement without independent evidence. Use fixed, reviewed expected values for business boundaries, and test normalization separately from transport.

## Keep fixtures deterministic and safe

Use synthetic records with obvious test identifiers and no personal or commercially sensitive data. Do not export production responses into source control, even after removing one visible name. Attachments, free-text notes, nested addresses, and metadata can retain confidential information.

Store minimal fixtures for specific scenarios: empty list, single result, multiple currencies, long translated text, missing optional fields, expired authorization, concurrent update, and invalid decimal input. Name fixtures by behavior rather than by an arbitrary screenshot date.

Make time explicit. A fixture whose validity depends on today's date eventually breaks. Inject a clock into application rules or use fake timers carefully when the installed test library supports them. Do not freeze all browser timers during OPA5 execution without understanding the effect on polling, animations, and framework scheduling.

Treat fixtures as contracts. When service metadata changes, update fixtures with the corresponding contract review. Random generation can explore additional combinations, but use a recorded seed and preserve any failing example as a deterministic regression case.

## Distinguish OData V2 and V4 mocks

The framework's `sap.ui.core.util.MockServer` simulates OData V2; it is not a generic OData V4 implementation. Official documentation explicitly separates the V2 mock server from V4 mocking through CAP Node.js [2]. Do not claim V4 fidelity by changing a URL or a metadata version string.

For a project already using the V2 mock server, start it before creating the OData model so metadata and initial reads are intercepted. Ensure its root URI exactly matches the model's service root, including path normalization and trailing slash requirements. Stop and destroy it during teardown.

Use the project's approved V4 mock or service test environment for V4 applications. Selecting a new mock product is a dependency and security decision outside this document. In either protocol, list unsupported behavior explicitly: authorization, ETags, batching details, server filtering, draft semantics, side effects, or transaction rollback may not match production.

Keep real-service contract tests small and controlled. Use an isolated tenant or test dataset, short-lived credentials, and cleanup that does not delete another developer's records. Never run destructive business operations against production to obtain convenient integration evidence.

## Design OPA5 around user intent

OPA5 tests coordinate asynchronous UI interactions through `waitFor`, matchers, and actions. Prefer public control properties, stable local IDs, view names, binding contexts, and documented actions. Avoid raw coordinate clicks or CSS selectors tied to generated markup.

Page objects should express business tasks such as selecting a supplier or submitting an order. They should not expose every button as a public method. Keep technical selection details inside the page object so a layout change does not require rewriting every journey.

The following is a real OPA5 test-module pattern under explicit assumptions: the existing integration harness maps namespace `example.orders`, its component starts with view `example.orders.view.Orders`, a stable local button `submitOrder` exists, and the synthetic startup state intentionally makes it disabled. No page-object helper is required.

```javascript
sap.ui.define([
    "sap/ui/test/Opa5",
    "sap/ui/test/opaQunit"
], function (Opa5, opaTest) {
    "use strict";

    Opa5.extendConfig({
        autoWait: true,
        viewNamespace: "example.orders.view.",
        arrangements: new Opa5(),
        actions: new Opa5(),
        assertions: new Opa5()
    });

    opaTest("Incomplete startup data cannot be submitted", function (Given, When, Then) {
        Given.iStartMyUIComponent({
            componentConfig: { name: "example.orders" }
        });

        Then.waitFor({
            id: "submitOrder",
            viewName: "Orders",
            success: function (button) {
                Opa5.assert.strictEqual(
                    button.getEnabled(),
                    false,
                    "Submission remains unavailable"
                );
            },
            errorMessage: "The submission action was not found"
        });

        Then.iTeardownMyApp();
    });
});
```

The component and startup fixture are application prerequisites, not supplied by this document. Without them this snippet is not runnable. A disabled-action assertion does not prove the valid submission journey; add that journey using the application's real page objects and service fixture.

## Wait for conditions, not arbitrary delays

Enable `autoWait` where supported and appropriate. Wait for a specific visible outcome or binding state, not a fixed two-second sleep. A delay can both waste time on fast machines and fail on slower CI agents.

The official OPA5 cookbook explains exceptions such as observing busy controls or transient message toasts [3]. Restrict `autoWait: false` to the particular assertion that requires it. Globally disabling synchronization to make one toast visible destabilizes the rest of the suite.

A message toast is not the best sole proof of a business transaction. Assert a durable state change, updated row, or explicit confirmation as well. If testing busy state, recognize that automatic waiting may intentionally wait until it disappears; configure that narrow observation deliberately.

Set timeouts based on expected operations and diagnose the first missing condition. Increasing the global timeout after every intermittent failure hides broken selectors, leaked pending timers, or unresolved requests. Collect sanitized diagnostics rather than normalizing long hangs.

## Cover navigation and recovery

Verify direct deep links, back navigation, unknown identifiers, and navigation away from dirty forms. A journey that always starts from the home page cannot establish that a bookmarked object route works.

Test failure recovery at the level where the user experiences it. After a rejected save, input should remain available, the busy state should clear, and an accessible message should explain the next step. After an authentication expiry, avoid displaying a generic “saved” state merely because a shell redirect started.

Check concurrent updates with the supported service contract. A mock response can exercise the UI's conflict message, while a real-service test verifies the actual ETag or version behavior. Separate evidence about presentation from evidence about backend concurrency enforcement.

## Measure coverage without mistaking it for assurance

Track statements, branches, functions, and lines for application-owned code. Branch coverage is particularly useful for eligibility rules and error handling. A high statement percentage can still miss every rejected promise and every boundary condition.

Use the existing project's coverage integration. Official UI5 documentation describes Istanbul and identifies Blanket as legacy; its guidance recommends Istanbul from UI5 1.113 onward [4]. That recommendation does not mean every old project can enable coverage by adding one script tag. Server instrumentation, middleware, source maps, and test bootstrap must be compatible.

Define the denominator intentionally. Exclude generated resources, framework code, and test fixtures, but include unexecuted application modules through the runner's supported instrumentation configuration. Otherwise a report that only counts imported files can misleadingly show excellent coverage while entire features are absent.

Set thresholds according to risk and current evidence, then ratchet them upward. As an illustrative policy, require changed business-rule branches to be exercised while tracking a lower legacy baseline separately. Never add assertions with no behavioral value solely to reach a percentage.

Verify source-map accuracy for transpiled code and avoid double-counting source and generated JavaScript. Archive machine-readable reports using the established CI mechanism, and inspect a deliberately uncovered branch once when configuring the system to confirm that it appears as a gap.

## Build a secure CI sequence

Use the repository's locked dependency restoration process on a supported runner. Validate cheap static checks before expensive browser journeys. Run unit tests, the relevant OPA5 subset, then the critical deployed smoke tests when the trust boundary permits them.

Commands such as `npm run test:unit` or `npm run test:integration` are examples of script names, not UI5 defaults. Execute them only if the consuming project defines them. UI5 CLI builds application resources; it does not imply an installed universal `ui5 test` command.

Keep untrusted pull-request jobs separate from deployment jobs. Do not provide service credentials, write tokens, or privileged caches to code from an untrusted contribution. Pin and review any actions already used by the repository according to its supply-chain policy; this guide intentionally selects none.

Make browser and framework versions visible in test metadata. Cache dependencies only with trustworthy keys and appropriate isolation. Do not cache authenticated browser profiles, cookies, tokens, captured service payloads, or test-generated confidential data.

Publish concise failure summaries and restricted diagnostic artifacts. Screenshots and network traces can contain user names, order details, or session material. Redact where feasible, restrict retention, and avoid dumping full request headers into logs.

## Manage flakiness as a defect

Classify intermittent failures by cause: timing, external dependency, selector ambiguity, shared state, browser issue, or genuine application race. Record the first failing assertion, runtime versions, reproduction conditions, and owner.

A single retry may distinguish a transient infrastructure problem, but it does not erase the original failure. Track retry rates separately from final pass rates. A suite that passes only after repeated reruns is not a reliable release gate.

Quarantine narrowly when necessary, with an issue, owner, expiry date, and replacement coverage for critical behavior. Never silently skip the sole test for a financial or security-sensitive path. Fix deterministic selectors and fixture isolation before tuning execution speed.

## Review test maintainability

Review test code with the same care as production code. Look for undefined helpers, hidden global state, accidental real network calls, and assertions that cannot fail meaningfully. Run a negative sanity check during test development by temporarily violating the expected behavior locally, then restore the implementation before review.

Do not encode an entire application workflow in one enormous test. Smaller independent scenarios provide better diagnostics and allow targeted execution. Share business-level arrangements, but avoid chaining scenarios so that the second requires data created by the first.

When a framework upgrade changes rendering, first determine whether the public interaction contract changed. Update tests that depended on internals rather than weakening meaningful assertions. Keep runtime compatibility evidence separate from new feature acceptance.

## Select a compatibility matrix

Choose a small, explicit matrix rather than running every scenario in every possible browser. Run the fastest stable browser configuration for frequent feedback, then exercise representative critical journeys across the browser and UI5 combinations the product actually supports. Include the oldest maintained runtime when a shared application must serve several landscapes.

Allocate special cases deliberately. Date-only fields need time-zone boundary cases; localized amount entry needs decimal and grouping variations; responsive tables need narrow layouts and keyboard interaction. These cases often reveal defects that repeating the default desktop journey cannot detect.

Separate environmental failures from assertions about application behavior. If a required browser cannot start, report an infrastructure failure instead of producing an empty successful suite. Likewise, verify that the runner discovered a plausible number of tests. A misconfigured module glob can make a job finish quickly because it executed nothing.

Review matrix cost periodically. Remove combinations that are no longer supported only through the product's compatibility policy, not because a job is inconvenient. Add a focused reproducer when a defect is specific to one combination, while keeping broad duplicated journeys limited enough that the team can diagnose failures promptly.

## Verify batching and partial failure

Treat a transport-level success and a successful business operation as different observations. A batch envelope can arrive successfully while an individual operation reports a validation or concurrency failure. Build fixtures using the protocol representation expected by the chosen model, rather than returning a convenient plain object that bypasses response processing.

For a multi-item action, decide whether the supported service contract is atomic or permits partial completion. Test the corresponding user experience explicitly: completed items must not be silently resubmitted, failed items need actionable feedback, and totals must reflect the confirmed state. Do not infer transaction semantics from the fact that several requests share one network envelope.

Exercise an ambiguous interruption separately from an explicit rejection. If connectivity disappears after the server commits but before the browser receives confirmation, automatically repeating a mutation can create duplicate business effects. Verify the application's documented reconciliation or idempotency behavior at the appropriate service boundary. A unit test can establish that the UI does not falsely announce success; controlled service evidence is needed to establish that duplicates are prevented.

## Maintain a regression evidence trail

When fixing a defect, preserve a minimal failing scenario that expresses the original user impact. Link the issue and test through the repository's existing conventions. A regression test should fail for the broken behavior and pass for the intended repair, rather than merely execute the newly added line.

Record any gap that the available harness cannot cover. For example, a mock-based journey can establish that an authorization rejection is presented correctly but cannot prove the deployed identity provider supplies the correct roles. Assign that missing evidence to an environment-level check with an owner instead of describing the mocked test as end-to-end assurance.

Keep execution evidence interpretable after the immediate change. Include the commit, artifact identity when relevant, runtime versions, selected suite, test count, and result. Distinguish skipped cases from passing cases and distinguish quarantined scenarios from unsupported configurations. Store only sanitized diagnostics and retain them according to policy.

Review regression tests when a feature is retired. Obsolete scenarios, fixtures, and page objects increase maintenance cost and can obscure the coverage that remains. Remove them with the owning capability, while preserving tests for shared behavior still required elsewhere.

## Verification checklist

- [ ] Every critical risk has an assigned test layer and observable assertion.
- [ ] QUnit asynchronous tests return or complete their actual work.
- [ ] Fixtures are synthetic, deterministic, minimal, and metadata-compatible.
- [ ] Doubles, controls, listeners, timers, and mock servers are cleaned up.
- [ ] V2 and V4 mocking capabilities are not conflated.
- [ ] OPA5 uses stable selectors, explicit prerequisites, and condition-based waits.
- [ ] Error, retry, concurrency, deep-link, and navigation paths are covered.
- [ ] Coverage includes unexecuted application code and meaningful branch gaps.
- [ ] CI trust boundaries, logs, caches, and artifacts protect sensitive data.
- [ ] Flaky tests have owners rather than unlimited retries.

## Related paths

These are the exact chapter paths listed in the [repository README](../README.md).

- [Code quality](./code-quality.md): `06-BEST-PRACTICES/code-quality.md`.
- [Git workflow](./git-workflow.md): `06-BEST-PRACTICES/git-workflow.md`.
- [Performance optimization](./performance-optimization.md): `06-BEST-PRACTICES/performance-optimization.md`.
- [Accessibility](./accessibility.md): `06-BEST-PRACTICES/accessibility.md`.

## Official references

Research access: the official GitHub documentation and source files below were retrieved successfully. Hosted SDK and UI5 documentation sites were not directly accessible during research; consult the version-matched SDK before implementation.

1. [OpenUI5: Unit Testing with QUnit](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/unit-testing-with-qunit-09d145c.md), consulted 2026-10-17.
2. [OpenUI5: Mock Server](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/mock-server-69d3cbd.md), consulted 2026-10-17. Separates built-in V2 mocking from CAP-based V4 mocking.
3. [OpenUI5: Cookbook for OPA5](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/cookbook-for-opa5-ce4b180.md), consulted 2026-10-17. Covers synchronization exceptions, busy controls, application parameters, and transient messages.
4. [OpenUI5: Code Coverage Measurement](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/code-coverage-measurement-7ef3242.md), consulted 2026-10-17. Documents Istanbul integration and legacy Blanket limitations.
5. [OpenUI5: OPA QUnit adapter implementation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/test/opaQunit.js), consulted 2026-10-17. Confirms the adapter's exported function and QUnit integration; use the matching runtime documentation for application support commitments.
