# Modules, namespaces, and dependency direction

## Baseline and prerequisites

This chapter assumes **SAPUI5/OpenUI5 1.120.x**, asynchronous bootstrap, `sap.ui.define`, and `sap.ui.require`. The application namespace is `demo.orders`; its resource root maps to `webapp`. Examples use browser JavaScript modules in the UI5 AMD-style loader, not native ECMAScript `import`/`export` syntax and not CommonJS `require`.

You should understand JavaScript functions, object references, and asynchronous callbacks. The architecture examples in [mvc-pattern.md](mvc-pattern.md), [component-architecture.md](component-architecture.md), and [data-binding.md](data-binding.md) all rely on the module rules described here. A correct folder tree is only part of the design: dependency direction and exported state determine whether the application remains composable.

UI5’s module loader knows logical module identifiers and resolves them to resources. Application code should depend on those identifiers, not on deployment-specific file URLs. This lets the same application run beneath a local development server, an application router, or a shell without rewriting imports throughout the codebase.

## Map names to resources consistently

The class name `demo.orders.controller.List` uses dots. Its controller module identifier is `demo/orders/controller/List.controller`, using slashes and the controller suffix. Its file is `webapp/controller/List.controller.js`. The XML controller declaration uses the dotted controller name without the `.controller` suffix because the view/controller infrastructure applies that convention.

Likewise, the view name `demo.orders.view.List` resolves to `demo/orders/view/List.view.xml`. The logical component name `demo.orders` leads to its component resources, including `Component.js` and `manifest.json`. These related conventions do not mean every filename can be derived by blindly replacing dots with slashes; view and controller resource suffixes also matter.

In a standalone `webapp/index.html`, this bootstrap fragment supplies the application mapping. The runtime is assumed to be served from a trusted same-origin `/resources/` location:

```html
<script id="sap-ui-bootstrap"
  src="/resources/sap-ui-core.js"
  data-sap-ui-theme="sap_horizon"
  data-sap-ui-async="true"
  data-sap-ui-compat-version="edge"
  data-sap-ui-resourceroots='{"demo.orders":"./"}'
  data-sap-ui-on-init="module:sap/ui/core/ComponentSupport">
</script>
<div data-sap-ui-component
  data-name="demo.orders"
  data-id="orders"
  data-settings='{"id":"orders"}'></div>
```

This is a fragment of an HTML document, not an XML view. A shell can provide resource mappings and component creation instead. Do not bootstrap UI5 again inside an application already hosted by a UI5 shell.

Case sensitivity matters. A path that works on a case-insensitive workstation may fail on a Linux deployment if `formatter.js` is imported as `Formatter`. Keep module IDs, namespaces, manifest entries, and filenames aligned exactly. Avoid mapping an application namespace over an existing framework namespace.

## A practical folder layout

The following layout supports one component with several views and explicit helper modules:

```text
webapp/
  Component.js
  manifest.json
  index.html
  controller/
    List.controller.js
    Detail.controller.js
  view/
    App.view.xml
    List.view.xml
    Detail.view.xml
    fragments/
      Confirm.fragment.xml
  model/
    formatter.js
    orderRules.js
  service/
    OrderRepository.js
  i18n/
    i18n.properties
```

Directories express intent but do not enforce access restrictions. A controller can technically import another controller, yet doing so often creates undesirable coupling. Establish a direction: controllers depend on domain rules and service abstractions; domain rules do not depend on controllers or views. A reusable component should not import a host’s internal modules merely because its files are reachable.

Use a service boundary when it has a real responsibility, such as adapting V2 operations into an application-specific repository contract. Do not introduce a generic “utils” directory into which every unrelated behavior disappears. Names such as `orderRules` and `OrderRepository` communicate the domain and make ownership easier to review.

## Define one module per resource

Create `webapp/model/orderRules.js`:

```javascript
sap.ui.define([], function () {
  "use strict";

  return {
    canApprove: function (order) {
      return !!order &&
        order.status === "Open" &&
        Number.isInteger(order.quantity) &&
        order.quantity > 0;
    }
  };
});
```

The anonymous definition lets the loader assign the module’s identity from the resource being loaded. Avoid explicitly naming ordinary application modules. A named definition can disagree with its requested path, become fragile under bundling, or make one physical file masquerade as another module.

The factory returns the module export. That export can be a constructor, a plain object, or a function. The module is not required to register a global variable. Consumers receive the exported value through their dependency parameters, which makes imports visible and reduces reliance on loading order.

Use a single ordinary module definition in each source file. Preload bundles generated by UI5 tooling can contain multiple module resources through the framework’s supported packaging format; they are not a reason to hand-write several unrelated anonymous definitions in one application file.

Do not execute application startup side effects at definition time unless the module is explicitly designed as an entry point. Creating dialogs, starting requests, or changing global configuration while a helper loads makes importing that helper an unpredictable operation. Prefer exported functions or constructors with explicit initialization.

## Dependencies are positional and explicit

Create `webapp/controller/List.controller.js`:

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "../model/orderRules"
], function (Controller, JSONModel, orderRules) {
  "use strict";

  return Controller.extend("demo.orders.controller.List", {
    onInit: function () {
      this._screen = new JSONModel({
        approvalAllowed: orderRules.canApprove({
          status: "Open",
          quantity: 2
        })
      });
      this.getView().setModel(this._screen, "screen");
    },

    onExit: function () {
      this._screen.destroy();
    }
  });
});
```

The first dependency corresponds to the first factory parameter, and so on. Reordering one list without the other can produce failures far from the import statement. Keep the correspondence visually obvious. Do not infer dependency order from the names of local variables.

`../model/orderRules` resolves relative to the importing module’s logical location, not the current page URL. Relative imports are convenient inside a cohesive feature. Absolute application module IDs can be clearer across larger boundaries. Both should omit the JavaScript extension in ordinary `sap.ui.define` dependencies.

A dependency declaration means the factory can use the resolved export. It does not mean arbitrary asynchronous work started by that dependency has completed. If a repository starts loading metadata, export an explicit readiness promise or expose operations that handle readiness internally. Do not mistake “module loaded” for “application data available.”

To see the controller’s result, declare `controllerName="demo.orders.controller.List"` on an XML view and bind a button’s `enabled` property to `{screen>/approvalAllowed}`. Point the component’s root-view declaration or a target at that view. This completes the connection from module identifier to controller instance to observable view state.

## Require modules at runtime deliberately

`sap.ui.require` loads a dependency set for an imperative operation. Its asynchronous array form invokes a callback after the requested modules are available. This is useful for an optional feature that should not increase initial loading work.

For example, a controller can load a message box when the user requests a confirmation:

```javascript
onExplainApproval: function () {
  sap.ui.require([
    "sap/m/MessageBox"
  ], function (MessageBox) {
    MessageBox.information("Approval checks the current order state.");
  }, function (error) {
    // Route the module-load failure to application error handling.
  });
}
```

Place this method inside the controller definition rather than in an unrelated global script. Because the callback above uses no controller state, it cannot accidentally update a destroyed view. A callback that does reference the view must handle the possibility that the controller has been destroyed while the module was loading.

The single-string lookup form of `sap.ui.require` is different from the asynchronous array form. It looks for an already available module export; it must not be treated as a request that will synchronously fetch a missing resource. Likewise, loading a module in a callback does not make its export available immediately on the next line outside that callback.

Prefer static dependencies for features required whenever a module is used. Runtime loading should be an intentional boundary, not a way to hide normal dependencies from reviewers. For recurring optional workflows, wrap loading in an application method with a documented return promise and error behavior.

## Avoid globals and synchronous compatibility paths

Legacy code often references constructors through globals such as `sap.m.Button`. That can accidentally work because another module loaded the library first. It can then fail under asynchronous startup, optimized bundles, or a different navigation sequence. Import `sap/m/Button` explicitly when constructing that class programmatically.

Do not use `jQuery.sap.require` or `sap.ui.requireSync` as a default escape hatch. Synchronous compatibility loading undermines asynchronous startup and can conceal an incomplete dependency graph. Modernizing such code means identifying the actual dependency and placing it in the correct definition or asynchronous require call.

Setting a module’s export as a global is generally unnecessary. A globally reachable object does not become a well-defined application service merely because it is convenient to call. Globals also complicate two simultaneous component instances, make replacement in tests harder, and blur destruction responsibility.

## Mutable exports are shared state

A module factory normally runs once for a resolved module identity. If it returns one mutable object, every consumer can observe that same object. A module-level `JSONModel` therefore behaves like a singleton, even when several components import it. This is frequently the cause of cross-instance draft leakage.

Export a factory or constructor when each owner needs an independent instance. For example, `createDraftModel` can return a new JSON model for each component. The component then destroys its own model. A stateless rules object like the earlier `orderRules` export is naturally safe to share because its methods do not mutate retained state.

Shared state can be intentional, such as a carefully scoped cache. Document its lifetime, invalidation policy, keying, and sensitivity. A cache keyed only by product ID may be wrong when values vary by user, tenant, language, or service root. Module caching does not provide an application-aware cache eviction policy.

## Cycles expose unclear responsibility

A circular dependency arises when module A depends on B and B eventually depends on A. Depending on loading and evaluation order, an export in the cycle can be unavailable or incomplete when consumed. Do not depend on a particular loader scheduling accident to make a cycle work.

A common cycle is controller → repository → controller. The repository should not know about the controller. Return data or a promise and let the controller decide how to present it. Another cycle is formatter → model factory → formatter; separate the pure transformation from model construction.

Break cycles by extracting a lower-level contract, introducing dependency injection, or moving orchestration into a higher-level owner. For example, both repository and formatter may depend on a small domain conversion module without depending on each other. This creates a directed dependency graph that mirrors architectural responsibility.

Dynamic `sap.ui.require` can postpone a reference but should not be the automatic cure for a design cycle. It may hide the cycle until a user activates a particular workflow. Use lazy loading for a legitimate optional boundary, not merely to move an initialization error into production interaction.

## Libraries, preloads, and deployment

Declare required UI5 libraries in `sap.ui5/dependencies/libs` in the manifest. Library dependencies describe framework resources; JavaScript module dependencies describe the concrete exports a factory uses. Declaring `sap.m` in the manifest does not remove the need to import a control constructor used in JavaScript.

UI5 tooling can build preload bundles from the application’s module graph. Keep source modules independently valid and let the established build configuration package them. Do not edit generated preload files as the primary source of a fix. Verify that development and production deployments resolve the same logical modules even if the network request shapes differ.

Use trusted resource roots. A dynamic module ID or resource location derived from arbitrary user input can turn a navigation parameter into a code-loading capability. Prefer an application-controlled allowlist mapping feature names to fixed module identifiers. Encoding the input does not make arbitrary executable resources trustworthy.

## Keep module loading separate from business readiness

An optional module can load successfully while the operation it implements fails. For example, an export dialog module may be available but lack permission to retrieve the selected document. Classify loader failures separately from validation, authorization, and transport errors so retry behavior addresses the actual problem.

Avoid top-level promises whose rejection is never observed because no consumer happens to use the module immediately. Prefer an explicit operation that returns a promise to its caller. The caller can then own busy state, report failure, and determine whether the result is still relevant to the current screen.

This distinction also improves reuse. A domain module that merely exports functions can be loaded by multiple features without starting their workflows. A module that begins work as soon as it is imported couples resource discovery to application behavior and makes preload timing observable to users.

## Decision guidance and verification checklist

Choose a static `sap.ui.define` dependency for a normal implementation requirement. Choose asynchronous `sap.ui.require` for an optional runtime boundary. Choose a constructor or factory export for owned mutable state. Choose a shared plain export for stateless behavior or explicitly designed shared infrastructure.

Before adding another abstraction, ask whether it clarifies dependency direction. A small explicit import is often better than a generic registry, global service locator, or deeply inherited base controller. Predictable loading and obvious ownership are the practical goals, not maximizing the number of folders.

- Check module IDs, file suffixes, resource roots, and letter case together.
- Confirm factory parameters match dependency-array positions.
- Load the application with asynchronous bootstrap and without relying on a prior screen.
- Review every programmatically used UI5 constructor for an explicit import.
- Ensure optional-load callbacks handle failures and owner destruction.
- Inspect exported objects for unintended shared mutable state.
- Instantiate two components to expose singleton draft leakage.
- Trace suspected cycles and remove the architectural back-reference.
- Compare source and packaged deployment behavior without editing generated bundles.
- Keep module identifiers and executable resource locations application-controlled.

## Official sources

- [Official UI5 loader source at 1.120.0](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/ui5loader.js), which implements module definition, dependency resolution, and require behavior. Use its public APIs rather than internal loader functions.
- [Official Controller implementation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/mvc/Controller.js), illustrating controller module naming and asynchronous resolution.
- [Official UIComponent implementation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/UIComponent.js), illustrating explicit dependencies and component infrastructure.
- [Versioned SDK `sap.ui.define` reference](https://ui5.sap.com/1.120.0/#/api/sap.ui/methods/sap.ui.define).
- [Versioned SDK `sap.ui.require` reference](https://ui5.sap.com/1.120.0/#/api/sap.ui/methods/sap.ui.require).

The conventions here intentionally use the 1.120 public loader model. A toolchain that transpiles native ES modules into UI5 modules can be valid, but its transformation and supported runtime must be documented separately rather than assumed.
