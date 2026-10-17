# MVC in UI5: boundaries, ownership, and lifecycle

## Scope and prerequisites

This chapter assumes SAPUI5 or OpenUI5 **1.120.x**, asynchronous module loading, XML views, and controls from `sap.m`. The application namespace is `demo.orders`, mapped to the application’s `webapp` directory. Examples use a client-side `JSONModel`, deliberately avoiding assumptions about an available OData service. The same separation applies to remote models, but persistence and request lifecycles require model-specific APIs. See [data-binding.md](data-binding.md) before substituting an OData model.

You should understand JavaScript objects, promises, XML namespaces, and the distinction between a control instance and its rendered HTML. UI5 manages a control tree; the browser displays a rendering of that tree. A controller normally changes control properties or model state, not arbitrary DOM nodes. The component supplies application ownership and shared infrastructure, as explained in [component-architecture.md](component-architecture.md).

MVC is useful because it gives changes a predictable home. A revised label belongs in the view or translation resources. A different discount calculation belongs in domain logic. Deciding which application action follows a button press belongs in the controller. These boundaries are not an automatic consequence of naming three directories. They emerge from dependency direction, state ownership, and disciplined lifecycle management.

## Assign responsibilities before writing handlers

The **model** exposes application data through bindings. It may hold a local draft, remote entities, localized text, or transient presentation state. It need not contain every business rule. A small pure domain module can calculate totals without inheriting from a UI5 model. The model stores observable results; the domain module defines how those results are derived.

The **view** declares controls, layout, bindings, and event connections. It describes a representation rather than orchestrating an entire workflow. XML is particularly effective for making relative paths and named model dependencies visible during review. Simple presentation expressions are appropriate; deeply nested business decisions inside XML obscure the behavior that stakeholders need to verify.

The **controller** translates interaction into application operations. It reads event parameters, requests validation, coordinates navigation or persistence, and changes explicitly owned state. A controller should not become a database abstraction, a renderer, and a financial calculation engine simultaneously. Extracting a helper is valuable when it creates a meaningful boundary, not merely when a method exceeds an arbitrary number of lines.

A useful review question is whether a calculation can run without creating a view. If a price rule requires a rendered table to obtain its input, data and presentation are probably entangled. Conversely, a helper that accepts a controller solely to call `byId` is usually not a domain service. It has simply hidden the same coupling behind another file.

## Wire a small application explicitly

The following descriptor belongs in `webapp/manifest.json`. It is a complete minimal descriptor for this chapter’s single-view example. Routing is intentionally absent; the component creates one root view. The manifest’s application ID, component class, controller name, and resource mapping must agree.

```json
{
  "_version": "1.58.0",
  "sap.app": {
    "id": "demo.orders",
    "type": "application",
    "title": "Order draft"
  },
  "sap.ui": {
    "technology": "UI5",
    "deviceTypes": { "desktop": true, "tablet": true, "phone": true }
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {} }
    },
    "rootView": {
      "viewName": "demo.orders.view.Order",
      "type": "XML",
      "id": "order",
      "async": true
    }
  }
}
```

Assume a same-origin UI5 runtime is mounted at `/resources/`. This bootstrap fragment in `webapp/index.html` establishes the namespace and uses declarative component support. A launchpad can provide equivalent startup infrastructure; do not add a second bootstrap inside an embedded application.

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

The component owns the shared order draft. Put the following definition in `webapp/Component.js`. The superclass initialization must run before application initialization that depends on component infrastructure. This example has no router to initialize.

```javascript
sap.ui.define([
  "sap/ui/core/UIComponent",
  "sap/ui/model/json/JSONModel"
], function (UIComponent, JSONModel) {
  "use strict";

  return UIComponent.extend("demo.orders.Component", {
    metadata: { manifest: "json" },

    init: function () {
      UIComponent.prototype.init.apply(this, arguments);
      this.setModel(new JSONModel({
        customer: "Ada",
        quantity: 2,
        unitPrice: 12.5,
        total: 25
      }), "order");
    }
  });
});
```

The component attaches the model under the name `order`; bindings must therefore use `order>`. The view can inherit this model through its component-owned control tree. Do not create a second model named `order` on the view unless deliberately replacing the inherited scope. Such accidental shadowing can make edits appear to disappear when navigating between screens.

## Keep the view declarative

Create `webapp/view/Order.view.xml`:

```xml
<mvc:View
  controllerName="demo.orders.controller.Order"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m">
  <Page title="Order draft">
    <content>
      <VBox class="sapUiSmallMargin">
        <Label text="Customer" labelFor="customer"/>
        <Input id="customer" value="{order>/customer}"/>
        <Label text="Quantity" labelFor="quantity"/>
        <Input id="quantity"
          value="{
            path: 'order>/quantity',
            type: 'sap.ui.model.type.Integer',
            constraints: { minimum: 1 }
          }"
          change=".onRecalculate"/>
        <ObjectNumber number="{order>/total}"/>
        <Text text="{screen>/status}"/>
        <Button text="Recalculate" press=".onRecalculate"/>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

The integer type performs parsing and validation between the input’s string representation and the numeric model property. It is not a complete business validation framework. Invalid input can remain visible in a control while the model retains the previous valid value. A production save flow must inspect validation messages and applicable business constraints rather than assuming that reading the model proves every visible field is valid.

The view uses a second model, `screen`, for presentation status. That status is not part of the order payload. Keeping it separate prevents accidental submission of UI flags and makes ownership clearer. The page title and labels are literal only to keep the wiring visible; a localized application would move them into a resource bundle.

## Controllers coordinate, helpers calculate

Create `webapp/model/orderMath.js`:

```javascript
sap.ui.define([], function () {
  "use strict";

  return {
    total: function (quantity, unitPrice) {
      if (!Number.isInteger(quantity) || quantity < 1 ||
          !Number.isFinite(unitPrice) || unitPrice < 0) {
        throw new Error("Invalid order calculation input");
      }
      return quantity * unitPrice;
    }
  };
});
```

This demonstration uses JavaScript numbers and does not claim to implement accounting-grade decimal arithmetic. A real monetary calculation must follow the service’s precision, rounding, and currency rules. Those rules belong in a domain boundary and ultimately require authoritative server-side verification.

Now create `webapp/controller/Order.controller.js`:

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "../model/orderMath"
], function (Controller, JSONModel, orderMath) {
  "use strict";

  return Controller.extend("demo.orders.controller.Order", {
    onInit: function () {
      this._screen = new JSONModel({ status: "Ready" });
      this.getView().setModel(this._screen, "screen");
    },

    onRecalculate: function () {
      var model = this.getOwnerComponent().getModel("order");
      try {
        var total = orderMath.total(
          model.getProperty("/quantity"),
          model.getProperty("/unitPrice")
        );
        model.setProperty("/total", total);
        this._screen.setProperty("/status", "Calculated");
      } catch (error) {
        this._screen.setProperty("/status", error.message);
      }
    },

    onExit: function () {
      this._screen.destroy();
      this._screen = null;
    }
  });
});
```

The handler obtains the component-owned model through the owner, not through a global registry. It changes observable data using `setProperty`, allowing bindings to react. Directly mutating an object returned by `getData` can bypass the notification path on which the screen depends. A later forced refresh may hide that problem without fixing the underlying responsibility error.

The controller destroys the local model because it created and owns it. It does not destroy the shared order model: another view could still use that model. Establish the same ownership rule for timers, subscriptions, dialogs, and independently created bindings. Attachment to a view and ownership of a model are related concepts but are not interchangeable guarantees of destruction.

## Understand the four main lifecycle hooks

`onInit` executes when a controller’s view is initialized. Use it for one-time instance setup: attach a route handler, create a view-local model, or install a stable event subscription. It does not mean that remote metadata or business data has arrived. Nor does it mean every later navigation will create a new controller.

`onBeforeRendering` executes before rendering and can run repeatedly. Use it only when work genuinely depends on the upcoming rendering cycle. Reattaching the same event handler here without detaching it produces duplicate behavior. Recreating a JSON model here can discard user edits whenever an unrelated invalidation causes the view to render.

`onAfterRendering` runs after rendering. It is appropriate for tightly controlled integration with a browser API or third-party widget that requires DOM nodes. It is usually the wrong place for ordinary data loading or text updates. Changing a property here can invalidate the view again, potentially creating a render loop. Prefer a control API or binding whenever one exists.

`onExit` runs during view destruction. Release resources created by the controller: detach external listeners using the same function and listener identity, clear timers, and dispose of owned helper objects. Navigation away is not necessarily destruction, because routed views are commonly cached. Therefore `onExit` is not a reliable “user left this screen” notification.

Lifecycle hooks are not general promise orchestration points. Returning a promise from `onInit` does not make ordinary MVC initialization wait for arbitrary application work. Start asynchronous work deliberately, expose loading and error state, and arrange that completion cannot update a destroyed view. If asynchronous view creation is required, use the documented asynchronous creation APIs rather than depending on an undocumented lifecycle return convention.

## Route activation and rendering are different events

Suppose a detail view is created once and reused for orders 101 and 102. Its controller’s `onInit` runs once, while its route-specific handler should execute for each matching order route. Bind the entity in that route handler. Do not read the current hash only during initialization and assume the detail view will forever represent that first order.

Attach the handler during initialization and detach it during exit. Keep the route object or retrieve the same named route for removal. Use route-specific `patternMatched` when only that route’s arguments should drive the screen. Broader router events may also report parent or nested matching behavior and require additional filtering.

The complete route and target configuration is in [routing-navigation.md](routing-navigation.md). That chapter also covers cold-start deep links. An MVC controller should receive a validated route identifier and select a context; it should not manufacture backend request URLs by joining untrusted hash fragments.

## Relative binding establishes a screen context

A detail view often binds its root to one entity and uses relative child properties. With the local JSON model, `bindElement({ path: "/selectedOrder", model: "orders" })` establishes a named context. A child binding such as `{orders>customer}` then resolves beneath that context. The view declaration stays reusable because the controller changes the context instead of rewriting each field.

Absolute paths begin at the model root and ignore the current context. They are useful for application-wide state, but a detail view filled with absolute paths is harder to reuse and easier to connect to the wrong record. Always include the model name in diagnostics: the same path on two differently named models can represent unrelated data.

In a list template, each row receives its own binding context. Event handlers should obtain that context from the event source or documented list-item parameter. A displayed row position is not a durable business identifier, particularly after sorting, filtering, or server-side paging. Binding context is the connection between the selected control and its current entity.

## Fragments, IDs, and reusable presentation

A fragment is a reusable declaration of controls, not an independent MVC application. It does not introduce a controller lifecycle of its own. In the 1.120 baseline, `Controller#loadFragment` is a convenient way to load controller-associated fragments asynchronously. Use view-local IDs and retrieve resulting controls with the corresponding view/controller lookup rules rather than hard-coding generated global IDs.

For a dialog loaded as a view dependent, inherited models and destruction are aligned with the view. Cache the loading promise so two rapid clicks do not create duplicate dialogs. Handle rejection so a failed load can be retried deliberately. If a dialog is created manually outside that convenience API, explicitly establish its dependent relationship or destroy it yourself.

Choose a fragment when repeated structure needs the parent controller’s orchestration. Choose a custom control when behavior and a public property/event contract should be encapsulated. Choose a component when the feature owns a larger application boundary, independent lifecycle, or routing concerns. Making every repeated field group a component adds deployment and communication complexity without necessarily improving reuse.

## Async work, race conditions, and user intent

A response can be technically successful and still be obsolete. If a user changes selection while an earlier request is pending, the earlier completion must not replace the newer selection. Use a request generation counter, a binding/context identity check, or an API-supported cancellation mechanism. Destruction checks alone do not solve this race because the view may still be alive.

Keep busy state scoped to the operation that owns it. If two operations overlap, clearing a single boolean when the first finishes can expose an editable screen while the second is still pending. Either serialize the operation, track outstanding work, or use binding events with a clearly defined ownership policy. Always clear busy state on both success and failure.

Do not suppress every error to preserve a clean console. Classify validation errors, authorization failures, missing entities, and transport problems. A controller coordinates the appropriate user response, while a reusable service can normalize technical error information. Avoid displaying raw response bodies that may contain implementation details or sensitive data.

## Decision guidance and common pitfalls

Keep a model on the view when only that screen owns its lifecycle and state. Move it to the component when several screens need the same authoritative client state. Do not move it to the global core simply to avoid passing a dependency. Global placement makes multiple application instances interfere and makes cleanup difficult.

Use a formatter for deterministic representation, a type for parsing and validation, and a domain function for business rules. A formatter should not issue requests or write other model properties. Such side effects can be triggered repeatedly by unrelated binding updates and are difficult to reason about during rendering.

Prefer stable local control IDs for controls that code, tests, or accessibility associations must reference. Avoid reaching into another controller’s view to modify its controls. Communicate through shared application state or a defined component contract instead. A screen that can only refresh by calling a sibling controller’s private method has crossed the intended boundary.

Finally, resist a universal base controller containing every feature’s utilities. A small base controller can provide stable application-wide conventions, but domain-specific methods create inheritance coupling between unrelated screens. Modules with explicit imports, described in [module-structure.md](module-structure.md), usually make these dependencies easier to see.

## Review a feature by following one user action

A practical architectural review starts with a concrete action rather than a directory diagram. Follow an edit from the input control through parsing, the model update, recalculation, save preparation, and server response. At each transition, identify the owner and the source of truth. This exposes duplicated state more effectively than checking whether files have the expected suffixes.

For example, suppose the controller stores a quantity in an instance field while the view edits the same quantity in a JSON model. Which one is submitted? A successful display update can conceal that the controller later saves its stale copy. Prefer one authoritative draft location and derive temporary local variables from it when an operation begins.

Next, follow failure through the same path. A parsing failure should remain associated with the field; a business rule failure should describe the relevant action; a persistence failure should preserve enough draft information for recovery. A general controller catch block that resets the whole screen can erase the evidence needed to correct the problem.

## Accessibility and localization fit the same separation

MVC boundaries also help accessibility. The view declares label associations, meaningful control types, headings, and appropriate interaction semantics. The controller coordinates focus only when the workflow requires it, such as after a dialog closes or an error summary is selected. Direct DOM focus hacks are fragile because rendering can replace the element.

Localization should not be scattered through string concatenation in handlers. Bind static labels to resource-model keys and use parameterized messages for sentences containing dynamic values. Format numbers and dates with locale-aware types rather than assuming that a decimal point or an ISO-looking string is suitable for every user.

These concerns are not decorative finishing work. A controller that constructs entire layouts as strings makes accessible labeling and translation review harder. A declarative view plus explicit state transitions allows the same functional behavior to remain understandable across themes, languages, and input methods.

## Verification checklist

- Confirm the namespace resolves to `webapp` and all names match their file paths exactly.
- Load the component directly and verify the root XML view is created asynchronously.
- Change the quantity to a valid integer and verify model-driven recalculation without manual DOM changes.
- Enter invalid input and confirm validation state is not mistaken for a valid save-ready screen.
- Revisit a routed screen and verify handlers do not multiply or retain stale entity contexts.
- Destroy the view and verify owned models, external listeners, and timers are released.
- Simulate overlapping requests and confirm obsolete results cannot overwrite current selection.
- Instantiate two components and confirm their draft state does not leak between instances.

## Official sources

The explanations were checked against the official OpenUI5 1.120.0 implementation and its embedded public API documentation. Source references pin the baseline rather than silently following the latest SDK.

- [Controller API and lifecycle implementation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/mvc/Controller.js): lifecycle integration, owner access, and fragment loading.
- [UIComponent API](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/UIComponent.js): root-view and router ownership.
- [BindingMode definitions](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/BindingMode.js): direction of model/control synchronization.
- [Versioned SDK Controller reference](https://ui5.sap.com/1.120.0/#/api/sap.ui.core.mvc.Controller): public method contracts and lifecycle hook descriptions.
