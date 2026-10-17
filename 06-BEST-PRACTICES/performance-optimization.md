# Performance Optimization for SAP Fiori and UI5

## Scope and prerequisites

This guide addresses application startup, navigation, data access, rendering, memory, and delivery caching for UI5 applications. It distinguishes browser work from network and backend work so optimization targets the actual bottleneck. It applies to freestyle applications and supported Fiori elements extension points; framework-generated content should be optimized through supported configuration and service design before custom overrides.

Assume a maintained UI5 1.x runtime, asynchronous loading, an application descriptor, and an existing UI5 CLI build process. Examples use ES2015 syntax and must match the deployed browser baseline. Builder details cited here use UI5 CLI v4 documentation, not the development version of the next major release. Verify task names and options against the version installed in the consuming project.

No packages or configuration files are introduced. Commands are examples for project-installed tools only. Measure the production-like artifact with realistic service latency and representative synthetic data before deciding that a source-level change improves user experience.

## Define the user-visible budget

Start with named scenarios: opening the application from a shell, following an object deep link, filtering a list, opening a value help, submitting a document, and returning to a previously visited route. Separate cold startup from warm navigation.

Define what “ready” means for each scenario. A page shell that appears quickly but leaves the main action blocked for ten seconds is not operationally ready. Record the time until useful content and the time until the intended interaction becomes available.

Set budgets using actual device, network, and backend constraints. As an illustrative team policy, require a documented improvement in the slowest critical journey without worsening the common warm journey. Do not copy arbitrary millisecond thresholds from another product.

Include request count, transferred bytes, main-thread work, backend duration, and memory trend alongside elapsed time. A single aggregate number cannot explain whether a change improves caching while increasing parsing cost or shifts work to the first interaction.

## Establish a repeatable baseline

Use the same application build, framework version, browser version, data volume, viewport, and route for comparisons. Record whether the browser cache is enabled and whether the shell is already initialized. An empty-cache direct launch is not equivalent to a warm shell navigation.

Capture several runs and report a distribution rather than the best result. Separate cold-cache runs from repeat visits. Record throttling settings, device class, and service environment so a later developer can reproduce the observation.

Use browser performance and network tools already available in the development environment. UI5 also provides measurement facilities, but confirm availability and production policy before enabling detailed interaction tracing. Avoid making instrumentation itself the source of the slowdown.

Sanitize exported traces. URLs, request bodies, authorization headers, response data, screenshots, and user-entered values can contain secrets or business information. Prefer synthetic scenarios and restricted artifact storage with short retention.

## Diagnose the critical path

Read a startup waterfall from the first request to usable content. Identify serialized dependencies: bootstrap, framework libraries, component resources, metadata, initial data, and rendering. A slow request matters most when it blocks all following work.

Distinguish server waiting from connection setup, transfer, JavaScript execution, and layout. Reducing a small XML file cannot solve a multi-second backend query. Likewise, faster backend execution may not help if thousands of controls monopolize the main thread afterward.

Trace causality rather than merely sorting requests by size. One accidentally synchronous dependency can serialize many otherwise parallel loads. One unnecessary initial binding can request a large collection before the user supplies mandatory filters.

Make one coherent change at a time and repeat the same scenarios. Retain a short record of the hypothesis, evidence, result, and tradeoff. Remove unsuccessful experimental changes instead of accumulating speculative optimizations.

## Build application preloads correctly

Deploy built application resources rather than relying on development-source loading. UI5 Builder's application tasks normally include component preload generation and minification, while library and self-contained builds have different task behavior [1]. Confirm the actual build configuration instead of assuming a filename proves correct bundling.

`Component-preload.js` reduces separate resource requests by packaging component modules and related resources. It does not automatically make every dependency smaller or remove unused code. UI5 module bundling is not a promise of modern tree shaking.

Inspect the production-like network trace to confirm the preload is requested successfully and individual resources are not fetched again unexpectedly. A missing preload, wrong namespace, excluded resource, or resource-root mismatch can turn a built application back into many small requests.

Never edit the generated preload by hand. Correct the source namespace, manifest, or approved build definition. Keep debug resources available only according to deployment policy, and verify that performance measurements are not accidentally forcing debug variants.

If the project already defines `npm run build`, inspect and use that script. Direct invocation such as `./node_modules/.bin/ui5 build --all` is illustrative and appropriate only when that local CLI and intended build mode already exist. Do not install or select a new CLI merely to run the example.

## Separate application and library delivery

Declare required UI5 libraries accurately in the application descriptor. Avoid loading a large library just to use a utility available in a smaller documented module. Conversely, do not omit legitimate dependencies because a development shell happens to preload them.

Framework library resources may be delivered by the host platform, a controlled CDN, or a packaged runtime. These choices have different cache, patching, network, and security implications. Use the organization's approved resource origin and supported framework version.

Do not bundle the entire framework into every application without evidence. A self-contained build can suit a particular offline or standalone target, but it can duplicate resources across applications and alter upgrade responsibilities. Evaluate total landscape cost, not only one direct-launch benchmark.

Keep framework and application resource versions consistent. Mixing cached application modules with an incompatible runtime can cause failures that resemble random loading regressions. Record the full deployment and cache policy in release evidence.

## Load optional work on demand

Keep startup dependencies limited to the initial task. Rarely used dialogs, export code, secondary charts, and administrative routes are candidates for deferred loading. Use documented asynchronous module loading rather than synchronous factories or global constructors [2].

The following illustrative module is complete: it returns a function that asynchronously loads the standard message dialog on first use. The caller owns error presentation and should invoke it only from an appropriate interaction. The displayed text is supplied by the caller's resource bundle, not hardcoded inside this module.

```javascript
sap.ui.define([], function () {
    "use strict";

    return function showInformation(text) {
        return new Promise(function (resolve, reject) {
            sap.ui.require(["sap/m/MessageBox"], function (MessageBox) {
                MessageBox.information(text);
                resolve();
            }, reject);
        });
    };
});
```

The returned promise resolves after opening the dialog, not after the user closes it. That distinction matters to callers and tests. Use the documented close callback when completion must represent dismissal.

Lazy loading defers cost; it does not remove cost. Measure first use of the deferred feature, provide appropriate feedback, and consider carefully controlled prefetching only when usage data justifies it. Do not prefetch confidential datasets or expensive reports merely because the browser is idle.

## Manage fragment reuse and lifecycle

For an often-reused dialog, cache its loading promise at the owning view or controller scope using the fragment API supported by the runtime. A promise cache prevents double creation when the user activates the action repeatedly before loading completes.

Attach the resulting dialog as a dependent where appropriate and destroy it with its owner. Clear a failed loading promise if retry is intended; otherwise one transient failure can permanently block the feature for that view instance.

Consider destruction during asynchronous creation. A dialog that finishes loading after its view exits must not remain orphaned. Use supported controller helpers or an explicit ownership check rather than allowing late callbacks to attach resources to destroyed objects.

Do not cache dialogs globally across unrelated component instances. Their models, binding contexts, event handlers, and user data can leak between navigations or sessions.

## Design cache layers separately

Distinguish static resource caches, model-held data, application memoization, browser storage, shared proxies, and backend caches. Each has its own key, lifetime, invalidation rule, and security boundary.

For static application resources, use the platform's supported versioned URLs or application cache-buster mechanism. Official optimization guidance distinguishes framework resource cache busting from application resource cache busting [3]. Arbitrary timestamp query strings defeat useful caching and can conceal an incomplete deployment process.

Cache-buster metadata does not itself define HTTP cache headers or make a deployment atomic. Verify server and proxy behavior as well as resource URLs. Long-lived immutable caching is safe only when the URL genuinely identifies immutable content.

For personalized responses, review server cache directives and shared-proxy configuration. Do not place authorization-dependent data into a public cache. A cache key that contains only the entity ID can return another user's representation if tenant, permissions, language, or other relevant context is omitted.

Avoid persistent browser storage for confidential data unless an approved design requires it and addresses retention, logout, shared devices, and incident response. Encryption in browser code is not a substitute for a sound data-storage boundary.

## Invalidate business data deliberately

A successful mutation must update or invalidate the relevant data views. Prefer documented model refresh and binding mechanisms over ad hoc copies of service entities in several JSON models. Multiple caches create disagreements that are expensive to diagnose.

Refresh the smallest meaningful scope. A full application model refresh after every field edit can trigger unnecessary requests and discard useful state. Conversely, updating only one displayed label can leave totals and related lists stale.

Account for concurrency and authorization changes. Data considered valid at initial load may no longer be safe to reuse after role changes, tenant switches, or a long idle session. Define when session-sensitive caches are cleared.

Cache failures cautiously. Memoizing a rejected request indefinitely can make a temporary outage permanent from the user's perspective. Give retries and recovery explicit behavior rather than layering another refresh button over inconsistent cache state.

## Reduce service payloads

Request the fields needed for the current interaction, using supported OData query options and model APIs. Apply `$select` and `$expand` according to the protocol and model version. OData V2 and V4 configuration are not interchangeable, and V4 automatic expand/select behavior has prerequisites.

Avoid loading an entire collection into a JSONModel just to filter it in the browser when server-side filtering is available and appropriate. Large client collections increase transfer, parsing, memory, and control creation together.

Use server-side paging for large datasets. Align growing lists or table virtualization with the control's actual binding capabilities; enabling a visual “growing” property does not guarantee an efficient backend request pattern.

Request counts only when the user needs them. A count query can be expensive on a complex service and may delay an otherwise useful list. Likewise, a deeply expanded object graph can overfetch far more than a few targeted reads.

Inspect actual requests after every query change. A descriptor setting that looks efficient may be ignored at the wrong binding level or unsupported by the target service.

## Tune request coordination carefully

Batching can reduce transport overhead and group related operations, but it does not eliminate backend processing cost. One slow operation may delay a whole batch. Understand the model's request groups, submission semantics, and error reporting before changing them.

Do not debounce explicit save or approval operations as if they were search suggestions. For free-text search, a short debounce may reduce redundant requests, but the implementation must handle cancellation, latest-result ordering, and accessibility feedback.

Parallelize independent reads when the service and model permit it. Preserve dependencies where one result determines another request. Unbounded parallel reads can overload the service and worsen tail latency for every user.

Handle busy state at the correct scope. Blocking the entire page for a secondary panel makes the application feel slower than necessary. Keep essential actions safe while allowing independent content to remain usable.

## Reduce rendering and layout cost

Measure control count and rendering work before introducing custom renderers. Thousands of nested controls can consume far more time than a modest network payload. Choose responsive tables, grid tables, or specialized controls based on data volume and interaction requirements.

Keep row templates lightweight. Avoid expensive formatters, repeated resource lookups, and network calls during formatting. A formatter may run many times and should normally be pure, cheap, and free of model mutation.

Update model properties in coherent groups where the model supports it. Repeated imperative setter calls can cause excess invalidation and rendering. Do not bypass invalidation through private APIs to obtain a benchmark improvement.

UI5 owns control DOM. Manual insertion, attribute mutation, or direct style changes can be overwritten and can break semantics. Use public properties and supported styling hooks. Never hide required focus targets with CSS or remove visible focus indicators as a rendering shortcut.

## Keep interaction responsive

Long synchronous calculations block input, rendering, and assistive technology updates. Move appropriate computation away from the critical interaction path or split it using the project's supported scheduling approach. A busy indicator cannot animate while the main thread remains blocked.

Avoid repeated layout reads interleaved with writes in custom code. Measuring geometry after every style change can force layout repeatedly. Prefer framework layout controls, responsive behavior, and batched updates over manual pixel positioning.

Do not disable all animation globally without checking supported user preferences and product requirements. Reduce unnecessary movement and honor reduced-motion needs where applicable. Performance should improve access, not sacrifice usable feedback.

Test slow devices and zoomed layouts. A layout that performs well on a desktop at default zoom can become expensive when text wraps and more content changes size.

## Detect memory growth

Repeat a realistic navigation cycle: open a list, open an object, launch and close a dialog, return, and repeat. Observe whether retained controls, listeners, timers, and data collections stabilize after normal collection opportunities.

Investigate monotonic growth rather than treating every temporary allocation as a leak. Compare retained ownership chains in browser tools. Common causes include event handlers attached on every render, global references to destroyed views, and dialogs cached beyond their owner.

Large model snapshots in debug logs can retain data in developer tools and distort memory observations. Remove unnecessary logging before drawing conclusions.

Test logout or component destruction where the host platform supports it. A warm shell can keep the browser alive for many application sessions, magnifying leaks that a full page reload would conceal.

## Verify optimization without regressions

Repeat baseline scenarios with the same measurement conditions and artifact type. Compare cold startup, warm revisit, first deferred interaction, and error recovery. An optimization that saves initial requests but makes every common action slower is a tradeoff, not an automatic improvement.

Run existing targeted unit and integration checks in the consuming project. Validate keyboard navigation, screen-reader announcements, stale-request handling, and service errors. A faster screen that drops a confirmation message or loses focus is a functional regression.

Review operational behavior: cache invalidation after release, resource fallback, partial network failure, and rollback. Confirm that source maps or debug variants follow the intended publication policy and that telemetry does not collect sensitive payloads.

## Verification checklist

- [ ] Scenarios, readiness criteria, data volume, and environment are recorded.
- [ ] Cold and warm measurements use repeatable production-like artifacts.
- [ ] The critical path is identified before changing code.
- [ ] Application preloads, library dependencies, and resource roots are verified.
- [ ] Deferred modules and fragments have loading, failure, and ownership rules.
- [ ] Cache keys, lifetimes, invalidation, and security boundaries are explicit.
- [ ] Requests use appropriate selection, paging, filtering, and grouping.
- [ ] Rendering changes preserve public APIs, keyboard operation, and semantics.
- [ ] Repeated navigation shows stable retained resources.
- [ ] Measured gains and functional checks support the final decision.

## Related paths

The [README index](../README.md) lists these exact companion paths.

- [Code quality](./code-quality.md): `06-BEST-PRACTICES/code-quality.md`.
- [Testing strategy](./testing-strategy.md): `06-BEST-PRACTICES/testing-strategy.md`.
- [Git workflow](./git-workflow.md): `06-BEST-PRACTICES/git-workflow.md`.
- [Accessibility](./accessibility.md): `06-BEST-PRACTICES/accessibility.md`.

## Official references

Research access: the official GitHub documentation below was retrieved successfully. Direct hosted UI5 documentation access was unavailable; confirm runtime and CLI behavior against the versions installed in the consuming project.

1. [UI5 CLI v4: Builder](https://github.com/UI5/cli/blob/v4/docs/pages/Builder.md), consulted 2026-10-17. Documents task defaults, preload generation, self-contained builds, minification, debug variants, and source maps.
2. [OpenUI5: Best Practices for Loading Modules](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/best-practices-for-loading-modules-00737d6.md), consulted 2026-10-17. Describes asynchronous loading, explicit imports, lazy require, and acyclic dependencies.
3. [OpenUI5: Optimizing Applications](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/optimizing-applications-2f492c4.md), consulted 2026-10-17. Distinguishes framework and application resource cache-buster concepts.
4. [UI5 CLI repository and version guidance](https://github.com/UI5/cli/blob/main/README.md), consulted 2026-10-17. Identifies stable v4 separately from upcoming major-version development.
