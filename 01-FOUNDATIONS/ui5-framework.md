# UI5 Framework Foundations

**Scope:** practical architecture for component-based freestyle UI5 browser applications, with boundaries to SAP Fiori elements. **Prerequisites:** JavaScript, promises, HTTP, HTML, and basic XML. **Runtime assumptions:** a deliberately selected, supported UI5 release, asynchronous loading, and an HTTP development server. Examples use public APIs common to SAPUI5 and OpenUI5 unless identified otherwise. They are instructional and have not been executed in this repository.

## 1. Know which product you are using

UI5 is a client-side application framework, not an ABAP runtime or a backend service. It supplies a managed object system, rendering infrastructure, accessible controls, model binding, navigation, localization, and application composition. A browser runs the application; services enforce business rules and authorization. UI5 can consume an SAP service, but it does not require every service to originate in SAP software.

OpenUI5 is the Apache-licensed open-source delivery. SAPUI5 shares the core and many libraries with OpenUI5, but adds libraries and is distributed under SAP product terms. Neither a public download nor a successful technical request establishes your license entitlement. Check the target product agreement and the library list for the exact version. The official [delivery comparison](https://github.com/SAP/openui5-docs/blob/main/docs/02_Read-Me-First/sapui5-vs-openui5-5982a97.md) explains this boundary.

For example, `sap.m`, `sap.ui.core`, and `sap.ui.layout` support many conventional enterprise screens in both deliveries. SAP Fiori elements and some analytical or smart-control scenarios require SAPUI5 libraries. Do not infer availability from a namespace that begins with `sap`; inspect the relevant distribution's API reference. A working OpenUI5 prototype is not proof that every SAPUI5 template can run on OpenUI5.

## 2. Separate design, framework, and application architecture

SAP Fiori is a design system. UI5 is one implementation technology. SAP Fiori elements is a metadata-driven application approach using SAPUI5, service annotations, and supported extension contracts. Freestyle UI5 gives application developers direct responsibility for views, controllers, navigation, and interaction design. These categories overlap without being interchangeable.

Choose Fiori elements when supported floorplans and service semantics cover the task. Choose freestyle when interactions genuinely require custom composition. Do not choose freestyle simply to avoid understanding annotations, or choose generated pages expecting arbitrary DOM manipulation to be supported. Either approach still needs a task model, authorization design, accessible content, and operational support.

Keep application architecture layered. Views describe controls and bindings. Controllers coordinate interactions. Models expose application state or remote data. Pure modules hold calculations and transformations. Service adapters isolate transport details when useful. The component supplies application-wide ownership, models, and routing. This division prevents a controller from becoming a second backend or an untestable collection of network calls.

## 3. Understand the managed object hierarchy

The important conceptual inheritance chain is `sap.ui.base.Object`, `EventProvider`, `ManagedObject`, `sap.ui.core.Element`, and `sap.ui.core.Control`. Each layer adds capabilities. Managed objects understand metadata, properties, aggregations, associations, binding, and lifecycle. Elements add UI-oriented identity and participation in the element tree. Controls add rendering and browser interaction.

Not every object is a control. Layout data and list items may be elements without independently rendered root DOM nodes. Models are not controls. Controllers also do not inherit from `Control`. Knowing this avoids inappropriate calls such as expecting every framework object to have a rendered DOM reference.

Control inheritance is different from visual containment. A `Button` inherits behavior from `Control`; a `VBox` contains buttons through an aggregation. Prefer composing existing controls over subclassing an unrelated control to borrow its appearance. Private methods, underscore-prefixed members, and internal DOM classes are implementation details, not extension contracts.

When reading API documentation, include inherited properties and events. A property apparently missing from a concrete class may be declared by a base class. Conversely, do not assume that similarly named controls support identical properties. Their metadata and documented aggregation types determine valid combinations.

## 4. Properties, aggregations, associations, and events

A property represents a value such as text, visibility, or enabled state. Use generated setters or model binding to update it. Directly assigning `oButton.text` bypasses managed state and normally does not update the UI.

An aggregation owns child objects. A page owns its content; a list owns its items. Destroying an owning parent normally destroys aggregated children. A child cannot simultaneously occupy two ordinary ownership positions. Attempting to reuse one control instance in two containers moves or otherwise conflicts with ownership rather than duplicating its appearance.

An association references another element, usually through its ID, without owning it. `ariaLabelledBy` and `labelFor` are examples. Destroying the referencing control does not imply destruction of the referenced label. This distinction matters for shared accessible descriptions and cleanup.

Events announce interactions or state changes. A `press` handler receives a UI5 event whose source is the control and whose parameters depend on that event's contract. Do not assume a browser `MouseEvent` shape. Use documented parameters and bound contexts rather than deriving business identifiers from rendered text.

## 5. Module names are resource contracts

An application namespace such as `org.example.orders` conventionally maps to a resource path such as `org/example/orders`. A module at `webapp/util/amount.js` can be addressed as `org/example/orders/util/amount` once that namespace is mapped to the application's resource root. Names are case-sensitive on typical deployment servers even when a local workstation hides mistakes.

Use `sap.ui.define` for a module declaration and explicit dependencies. The dependency array and factory parameters must align. Modules should return their exported value and avoid side effects at import time. Loading a formatter should not send requests or create a page.

Place this original utility at `webapp/util/quantity.js`:

```javascript
sap.ui.define([], function () {
    "use strict";

    return {
        canApprove: function (iQuantity) {
            return Number.isInteger(iQuantity) && iQuantity > 0;
        }
    };
});
```

A controller imports it through its namespace path. `sap.ui.require` is useful for runtime asynchronous access, but should not replace clear static dependencies everywhere. Avoid synchronous loading and legacy globals such as constructing classes merely because another module happened to load them first. Production preload order can expose those hidden dependencies.

## 6. Types exist at several different layers

JavaScript primitives are only one kind of type. UI5 metadata also declares property types, enumerations, and allowable aggregation classes. Binding types parse, format, and validate values. OData metadata supplies service types, precision, scale, and nullability. TypeScript can add development-time static checking, but it does not replace any of these runtime contracts.

A text input emits text. A numeric-looking string is not automatically an integer suitable for a business calculation. A binding type such as `sap.ui.model.type.Integer` can validate and convert input according to configured constraints. Locale-aware types avoid the incorrect assumption that every user writes decimal separators the same way.

Use the OData model's documented types for protocol values. Decimal money values can exceed safe binary floating-point assumptions, and identifiers with leading zeroes are usually strings. Preserve business semantics rather than converting everything with `Number`. Backend validation remains essential even when a UI5 type rejects obviously invalid input.

TypeScript projects need compatible UI5 type definitions and an established transformation pipeline. Standard UI5 resource loading is not a promise that arbitrary TypeScript or native ECMAScript module source will execute directly. Consult the [official TypeScript guidance](https://github.com/SAP/openui5-docs/blob/main/docs/02_Read-Me-First/typescript-support-a7ee961.md) for the selected release and tooling combination.

## 7. Follow startup from HTML to component

A standalone page first loads `sap-ui-core.js`, establishes resource roots, and starts framework initialization. The component support mechanism can discover a declarative component container. UI5 then loads the component module and descriptor, creates declared models and the root view, and initializes application behavior. Asynchronous startup means network requests and module evaluation do not follow naive script ordering assumptions.

In a launchpad, the shell usually owns bootstrap and application hosting. The application's development `index.html` is not the production entry point. Put reusable application configuration in the descriptor and component, not only in that HTML page.

Use asynchronous views and component creation supported by the chosen baseline. Avoid code that asks for a control before its view exists, assumes translations are already loaded, or treats a component creation promise as the component itself. A loading indicator should describe genuine pending work, not compensate for synchronous blocking.

The coherent file set in [project setup](./project-setup.md) shows the standalone bootstrap. It deliberately uses relative runtime resources rather than an unversioned public CDN. Which server provides those resources is an environment decision, not a controller concern.

## 8. The component is the application boundary

Extend `sap.ui.core.UIComponent` for a conventional visual application. Declare `metadata: { manifest: "json" }` so the descriptor supplies application metadata. The descriptor contains the root view, libraries, models, and optionally routing. Call the base `init` implementation before relying on initialization it performs.

For an application whose descriptor actually declares routes and targets, a component implementation commonly includes:

```javascript
sap.ui.define([
    "sap/ui/core/UIComponent"
], function (UIComponent) {
    "use strict";

    return UIComponent.extend("org.example.orders.Component", {
        metadata: {
            manifest: "json"
        },

        init: function () {
            UIComponent.prototype.init.apply(this, arguments);
            this.getRouter().initialize();
        }
    });
});
```

This routing call is not needed in a minimal application without routing. A copied snippet must match its descriptor. A component's ownership boundary also supports namespaced IDs and lookup through the owning component instead of a global singleton application object.

Models created by custom code need an explicit destruction strategy. Do not destroy an externally supplied shared model just because one view disappears. Conversely, replacing references without releasing objects your code owns can leak listeners and data. Record who creates, shares, and destroys each long-lived object.

## 9. XML views express structure

XML views describe a control tree, not literal HTML. Namespace declarations resolve UI5 libraries; unprefixed tags in the `sap.m` namespace create mobile-library controls. Binding syntax connects properties and aggregations to model paths. Event attributes connect to controller methods.

For `webapp/view/Order.view.xml`, assume a named `order` JSON model exists on the component:

```xml
<mvc:View
    controllerName="org.example.orders.controller.Order"
    xmlns:mvc="sap.ui.core.mvc"
    xmlns="sap.m">
    <Page title="{i18n>orderTitle}">
        <content>
            <VBox class="sapUiSmallMargin">
                <Label text="{i18n>quantity}" labelFor="quantityInput"/>
                <Input
                    id="quantityInput"
                    value="{
                        path: 'order>/quantity',
                        type: 'sap.ui.model.type.Integer',
                        constraints: {minimum: 1}
                    }"/>
                <Button
                    text="{i18n>approve}"
                    press=".onApprove"/>
            </VBox>
        </content>
    </Page>
</mvc:View>
```

Provide all referenced resource keys and the matching controller before using this view. The integer type supports client-side validation, but the action still needs a deliberate validation and persistence flow. A validation error must prevent submission through application logic; a red border alone is not a transaction policy.

## 10. Controllers have a lifecycle, not just click handlers

`onInit` runs once per controller/view instance after the view is initialized. It is appropriate for attaching route handlers, establishing view state, and recording stable dependencies. It is not called every time a cached route becomes visible.

`onBeforeRendering` can run repeatedly before rendering. `onAfterRendering` can run repeatedly afterward. Use them sparingly for integrations requiring the rendered DOM. Sending a service request after every render can create feedback loops because the response changes state and triggers another render.

`onExit` is the cleanup point. Detach listeners from long-lived objects, clear timers, and destroy manually owned resources that the view does not own. Store listener functions or use the same function and context when detaching; a new `.bind(this)` expression creates a different function reference.

Do not assume framework hooks await an arbitrary returned promise. Start asynchronous work explicitly, handle failure, and guard against late completion after destruction. For route-driven requests, compare the requested object key with the currently active key or cancel transport work when supported. Otherwise a slow response for order A can overwrite order B's screen.

## 11. Rendering is managed and repeatable

Controls render managed state into DOM. Property changes can invalidate controls; UI5 schedules rendering rather than promising an immediate DOM update after each setter. Prefer changing properties or model data over manipulating generated HTML.

Do not cache a DOM node indefinitely. Rerendering can replace it, leaving stale event listeners attached to detached elements. Do not call rendering hooks manually to force updates. Explicit rendering synchronization belongs in narrowly controlled infrastructure or tests, not routine application interaction.

Third-party widgets require a documented bridge: create or update the widget after its host DOM exists, release it before replacement or on destruction, handle resizing, and provide accessibility. Embedding an inaccessible canvas does not become accessible because its parent is a UI5 control.

Custom renderers must escape untrusted text. Use `RenderManager.text` for text and supported attribute methods for attributes instead of raw HTML concatenation. Sanitization of rich content is a separate problem; it must follow a documented allowlist and cannot be solved by trusting a business system's data source.

## 12. Model inheritance and binding contexts

Models assigned to a component or view can propagate to descendants. A named model such as `view` isolates UI state from the default business model. A binding context establishes the starting point for relative paths. Absolute paths start at a model root; relative paths need a context.

If a list binds `items` to `/Orders`, each item receives an order context. Within that item, `{Description}` refers to the current order. `{view>/busy}` instead reads a named model's root. Confusing these scopes often produces empty controls without obvious exceptions.

Use `setProperty` on a JSON model to communicate changes. Mutating the original plain object does not reliably notify bindings. Avoid large unconditional refreshes as a substitute for understanding binding updates. They conceal state ownership problems and may trigger expensive remote reads in service models.

Binding modes govern direction: one-way reflects model changes, two-way can propagate user edits, and one-time evaluates once. Defaults vary by model and binding. A two-way binding is not a database save; the model's submission semantics determine when anything reaches the server.

## 13. Choose JSON and OData models intentionally

`JSONModel` is useful for small local datasets, flags, and prototypes. It does not provide server paging merely because a control supports growing. Loading a million rows into JSON and revealing twenty at a time remains a million-row download.

The OData V2 and V4 models are separate APIs, not versions of an interchangeable constructor. V2 request methods, batching behavior, and deferred groups should not be pasted into a V4 application. V4 is strongly binding-oriented and has its own context and group APIs. Confirm the service protocol and selected UI5 model documentation together.

For business editing, define when edits are staged, when they are submitted, what cancellation means, and how concurrent modifications are detected. An HTTP success does not always mean every change inside a batch succeeded. Inspect the model's documented completion and message mechanisms rather than assuming transport success equals business success.

Keep authorization on the backend. Hiding a button improves usability but does not prevent a user from constructing the corresponding request. Treat server messages as untrusted display content, and avoid logging full payloads containing personal information.

## 14. Navigation separates identity from screen instances

A route encodes navigation state; a target identifies what view is shown and where. The router maps hashes to views and restores deep links. A route parameter should identify a business object, not carry its entire serialized record or confidential attributes.

Resolve route parameters through service-aware key handling. Manual string concatenation into OData paths can mis-handle quotes, slashes, and composite keys. Validate expected parameter shapes, and let the service enforce access checks.

Distinguish navigating to a route from loading its data. The view may be cached while its object context changes. Attach object loading to the relevant route match, not only `onInit`. Design a not-found state for nonexistent or inaccessible records without leaking which protected records exist.

Browser Back and launchpad cross-application navigation are different concerns. Do not hardcode another application's URL into every controller. Shell integration APIs and supported services vary with hosting environment; isolate those dependencies and provide a meaningful standalone fallback where required.

## 15. Fragments and stable IDs

Fragments are reusable UI definitions without an independent controller lifecycle. A dialog fragment can use its containing controller's handlers. Use the release-supported fragment loading API and give instances suitable IDs, especially when opening multiple copies.

Controls created outside a view's normal content tree often belong in the view's `dependents` aggregation. This propagates models and ties destruction to the view. A dialog hidden with `close` still exists; choose whether to reuse or destroy it rather than creating a new instance every time.

Use view-local IDs and `this.byId("quantityInput")` in controllers. Do not build strings from generated component prefixes or use global DOM selectors. Stable application IDs support adaptation, automation, and support analysis, but uniqueness is scoped through the component and view machinery.

An asynchronous dialog creation promise should be cached if repeated clicks could create duplicates. Clear failed cached promises so retry is possible. During teardown, prevent a late-loaded dialog from attaching to an already destroyed view.

## 16. Performance follows architecture

Declare dependencies, use asynchronous loading, and deploy generated preload artifacts through the established build process. A preload bundles module definitions; it does not permit undeclared global dependencies or eliminate service latency.

Request only needed business data and use server-side filtering and paging where supported. Choose table controls based on interaction and data scale rather than appearance alone. Limit unnecessary model churn and deeply nested layouts before attempting browser micro-optimizations.

Measure startup separately from subsequent navigation. Warm caches can hide missing preload resources or excessive requests. Compare authenticated runs using representative permissions and data volumes. Network traces may contain confidential information; sanitize them before sharing.

Keep formatters deterministic and inexpensive because bindings can evaluate repeatedly. Do not make asynchronous service calls from a formatter. If calculated state needs remote information, load that state explicitly and expose it through a model.

## 17. Release-sensitive APIs and upgrade discipline

The framework version, library availability, browser support, and tooling version are distinct compatibility axes. A modern Node runtime does not upgrade the UI5 runtime installed in an ABAP system. A high `minUI5Version` declaration does not download missing framework features.

Consult the selected release's API reference, including `Since`, deprecation, and experimental markers. Examples using modern theming modules, fragment conveniences, OData V4 features, or asynchronous APIs may require newer baselines than an older enterprise landscape supplies.

Public contracts are more stable than internal markup. CSS targeting internal descendants, monkey patches, and private model fields commonly break during updates even when the public control API remains compatible. Prefer documented extension points, public events, and semantic styling facilities.

Plan an upgrade as a compatibility change: identify the currently deployed runtime, review intervening release notes, inspect deprecated usage, and rehearse critical tasks on a representative target. Do not silently switch production to a floating CDN version.

## 18. Common diagnostic pitfalls

An empty page can mean bootstrap failure, incorrect namespace mapping, a missing root view, or a controller import error. Start with the first console error and failed network request, not random framework configuration changes.

A binding that displays nothing often has the wrong model name, incorrect case, or missing relative context. Inspect those inputs before adding `refresh`. A duplicated event often means a route listener was attached repeatedly without detachment. A dialog showing old data often retained a binding context that was never updated.

A screen that works standalone but fails in a launchpad may depend on bootstrap settings found only in `index.html`, use absolute application paths, or assume a private framework version. Compare hosting contracts rather than patching the shell.

## Verification checklist

- [ ] Record the delivery, exact runtime version, required libraries, and supported browsers.
- [ ] Confirm namespace, descriptor ID, component name, and module paths agree.
- [ ] Check ownership and cleanup for models, dialogs, listeners, and timers.
- [ ] Verify relative binding contexts and explicit model names.
- [ ] Confirm submission, cancellation, validation, and concurrency behavior.
- [ ] Review keyboard access, accessible labels, translated text, and semantic states.
- [ ] Inspect asynchronous failures and late responses after navigation.
- [ ] Check public API availability against the actual deployment release.

These are recommended verification activities, not claims of completed execution.

## Official references and related foundations

- [OpenUI5 project overview and licensing](https://github.com/SAP/openui5)
- [OpenUI5 API reference](https://sdk.openui5.org/api)
- [SAPUI5 Demo Kit](https://ui5.sap.com/)
- [Official application initialization explanation](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/app-initialization-what-happens-when-an-app-is-started-d2f5869.md)
- [Official controller implementation and lifecycle documentation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/core/mvc/Controller.js)

Continue with [Fiori design](./fiori-design-system.md), [project setup](./project-setup.md), and [development environment](./development-environment.md). These sibling paths are the foundations listed in the [repository README](../README.md).
