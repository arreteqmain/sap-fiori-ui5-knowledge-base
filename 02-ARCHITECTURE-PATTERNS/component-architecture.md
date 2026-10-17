# Component architecture and feature composition

## Baseline and prerequisites

Examples target **SAPUI5/OpenUI5 1.120.x**, asynchronous loading, XML views, and `sap.m`. A component named `demo.orders` owns a host application; a reusable component named `demo.address` owns an address editor. Both namespaces resolve through the loader to separately named directories. Components may be deployed together without becoming the same lifecycle object.

You should already understand models, controllers, and view-local IDs. [mvc-pattern.md](mvc-pattern.md) describes those foundations. This chapter focuses on a larger question: which feature should own which objects, and through what contract should another feature interact with it? [module-structure.md](module-structure.md) explains the resource namespace and dependency mechanics beneath component loading.

A UI component is not merely a directory with a `Component.js` file. It is an instantiated ownership boundary with descriptor metadata, models, optional routing, and a root control. Its type is reusable; its instances have independent lifecycles. Two instances of the same component must not accidentally share mutable editor state through a module-level variable.

## Choose the smallest useful boundary

Use a fragment for reusable control structure that participates in the surrounding controller. Use a custom control for a cohesive visual abstraction with properties, aggregations, and events. Use a UI component for a feature that benefits from an independently configured root view, model scope, initialization, or navigation boundary.

For example, an address input layout may be a fragment. An address control with validation events and well-defined properties may be a custom control. An address-management feature that loads reference data, supports selection and editing, and appears in multiple host applications can justify a component. These are design alternatives, not a mandatory progression through increasingly prestigious abstractions.

Deployment independence is a separate decision. A nested component may live inside the host repository and share its release. Conversely, code in a separate library may provide a small custom control rather than a full component. Decide based on ownership and contract stability first; use packaging to support that design rather than dictate it.

Avoid turning every screen into a component just to reduce controller size. Component boundaries introduce asynchronous creation, model propagation choices, ID composition, event subscriptions, and destruction policies. A small helper module or a better MVC separation may solve the actual problem with fewer moving parts.

## The descriptor is part of the public architecture

The manifest describes application identity, libraries, data sources, models, root views, and routing. Put stable declarative dependencies there so the framework knows them before executing application initialization. A manifest-driven component can load dependencies asynchronously rather than discovering them through a chain of imperative calls.

Keep `sap.app/id` aligned with the component namespace. In a library or reusable feature, document supported configuration and expected models. A component that secretly requires a host model called `data` with undocumented paths has a fragile implicit interface even if its JavaScript classes are neatly separated.

This host manifest, `webapp/manifest.json`, declares one reusable component usage. It is intentionally independent of the routed application example in [routing-navigation.md](routing-navigation.md); merge relevant descriptor sections rather than placing multiple `sap.ui5` objects in one JSON document.

```json
{
  "_version": "1.58.0",
  "sap.app": {
    "id": "demo.orders",
    "type": "application",
    "title": "Order composition"
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {} },
      "components": { "demo.address": {} }
    },
    "rootView": {
      "viewName": "demo.orders.view.App",
      "type": "XML",
      "id": "app",
      "async": true
    },
    "componentUsages": {
      "addressEditor": {
        "name": "demo.address",
        "lazy": true
      }
    }
  }
}
```

The host bootstrap must resolve `demo.orders` to `./` and `demo.address` to `./reuse/address/`, assuming both paths are relative to the host’s `webapp/index.html`. If the address component is deployed elsewhere, change the resource mapping, not its logical module IDs. Configure trusted deployment locations; never derive a component resource URL from arbitrary user input.

The host `Component.js` is deliberately small:

```javascript
sap.ui.define(["sap/ui/core/UIComponent"], function (UIComponent) {
  "use strict";
  return UIComponent.extend("demo.orders.Component", {
    metadata: { manifest: "json" },
    init: function () {
      UIComponent.prototype.init.apply(this, arguments);
    }
  });
});
```

This host has no routes, so initialization does not call `getRouter().initialize()`. Routed hosts should initialize their configured router after superclass setup and essential model setup. A component with no routing section should not assume a router exists simply because it extends `UIComponent`.

## Embed a child through a declared usage

Create `webapp/view/App.view.xml`:

```xml
<mvc:View
  controllerName="demo.orders.controller.App"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns:core="sap.ui.core"
  xmlns="sap.m">
  <Page title="Shipping address">
    <content>
      <core:ComponentContainer
        id="addressHost"
        usage="addressEditor"
        manifest="true"
        async="true"
        propagateModel="false"
        lifecycle="Container"
        componentCreated=".onAddressCreated"/>
      <Text text="{host>/summary}"/>
    </content>
  </Page>
</mvc:View>
```

The container is a control in the host’s view tree. The child component is a separate component instance hosted by that control. `usage` resolves the manifest declaration; asynchronous manifest-first loading gives UI5 the information needed to load child dependencies. The explicit lifecycle setting tells the container to manage the embedded component when the container is destroyed.

`propagateModel="false"` makes the intended isolation visible. A child should not accidentally inherit the host’s default business model. If propagation is enabled for a deliberately shared model design, document the expected model names and what happens when the child has a model with the same name. Model propagation does not transfer ownership of the underlying model object.

The child manifest belongs at `webapp/reuse/address/manifest.json`:

```json
{
  "_version": "1.58.0",
  "sap.app": {
    "id": "demo.address",
    "type": "component",
    "title": "Address editor"
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {} }
    },
    "rootView": {
      "viewName": "demo.address.view.Editor",
      "type": "XML",
      "id": "editor",
      "async": true
    }
  }
}
```

The child is a component, not an independently bootstrapped page. Do not put another `sap-ui-core.js` script into its view or root control. The host runtime creates it. Separate component identity does not imply a separate browser document, JavaScript realm, or security sandbox.

## Design an explicit data and event contract

The child’s public contract has two operations: accept a draft address and emit a confirmed address. It owns a private `draft` JSON model. Create `webapp/reuse/address/Component.js`:

```javascript
sap.ui.define([
  "sap/ui/core/UIComponent",
  "sap/ui/model/json/JSONModel"
], function (UIComponent, JSONModel) {
  "use strict";

  return UIComponent.extend("demo.address.Component", {
    metadata: {
      manifest: "json",
      events: {
        confirmed: {
          parameters: { address: { type: "object" } }
        }
      }
    },

    init: function () {
      UIComponent.prototype.init.apply(this, arguments);
      this._draft = new JSONModel({ street: "", city: "" });
      this.setModel(this._draft, "draft");
    },

    setAddress: function (address) {
      this._draft.setData({
        street: typeof address.street === "string" ? address.street : "",
        city: typeof address.city === "string" ? address.city : ""
      });
      return this;
    },

    confirmAddress: function () {
      var data = this._draft.getData();
      this.fireConfirmed({
        address: { street: data.street, city: data.city }
      });
    },

    destroy: function () {
      if (this._draft) {
        this._draft.destroy();
        this._draft = null;
      }
      UIComponent.prototype.destroy.apply(this, arguments);
    }
  });
});
```

`setAddress` expects a non-null address object supplied by the host; production APIs may additionally reject malformed input with a documented error. Copying supported scalar fields prevents accidental aliasing of the host object. The event also returns a fresh object. Editing inside the child therefore does not silently mutate the host before confirmation.

This contract is intentionally narrow. It does not expose the draft model, controller, or internal control IDs. Changing the child’s layout later should not require changes to the host. The event payload is not an authorization decision and is not automatically valid server data; persistence must still validate the address according to business rules.

Create `webapp/reuse/address/view/Editor.view.xml`:

```xml
<mvc:View
  controllerName="demo.address.controller.Editor"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m">
  <VBox class="sapUiSmallMargin">
    <Label text="Street" labelFor="street"/>
    <Input id="street" value="{draft>/street}"/>
    <Label text="City" labelFor="city"/>
    <Input id="city" value="{draft>/city}"/>
    <Button text="Use address" press=".onConfirm"/>
  </VBox>
</mvc:View>
```

The child controller delegates to its owner’s public operation:

```javascript
sap.ui.define(["sap/ui/core/mvc/Controller"], function (Controller) {
  "use strict";
  return Controller.extend("demo.address.controller.Editor", {
    onConfirm: function () {
      this.getOwnerComponent().confirmAddress();
    }
  });
});
```

Place that module at `webapp/reuse/address/controller/Editor.controller.js`. Its owner is the address component, not the outer orders component. Ownership is established during creation and is different from asking which visual container happens to surround a control.

## Connect the host without private lookups

The host controller at `webapp/controller/App.controller.js` listens for the container’s asynchronous completion. It then attaches a public component event and supplies initial data:

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel"
], function (Controller, JSONModel) {
  "use strict";
  return Controller.extend("demo.orders.controller.App", {
    onInit: function () {
      this._host = new JSONModel({ summary: "No address confirmed" });
      this.getView().setModel(this._host, "host");
    },

    onAddressCreated: function (event) {
      this._address = event.getParameter("component");
      this._address.attachConfirmed(this._onConfirmed, this);
      this._address.setAddress({
        street: "42 Example Street",
        city: "Berlin"
      });
    },

    _onConfirmed: function (event) {
      var address = event.getParameter("address");
      this._host.setProperty("/summary",
        address.street + ", " + address.city);
    },

    onExit: function () {
      if (this._address) {
        this._address.detachConfirmed(this._onConfirmed, this);
        this._address = null;
      }
      this._host.destroy();
    }
  });
});
```

The host does not destroy the child manually because the container has the `Container` lifecycle policy. Explicit destruction in both places can conceal an unclear ownership design. Conversely, if a component is created programmatically and merely associated with a container, verify the configured lifecycle rather than assuming aggregation-style destruction.

The example handles successful creation to keep the contract visible. A production host should also handle the container’s failure notification, provide an accessible error state, and offer a retry policy that does not create duplicate instances. Component creation failure is different from the child successfully starting but later failing to load business data.

## Ownership, IDs, and instance isolation

An object can belong to an owner component without being a child control aggregation of that component. Ownership is used by APIs such as controller owner lookup and by component-oriented infrastructure. The visual hierarchy, model propagation hierarchy, and owner relationship are related but distinct structures.

Use component-local and view-local IDs. Let the framework compose prefixes instead of constructing global IDs by string concatenation. From a host, use the container’s documented component access or the component’s public API. Do not rely on the full generated ID of an input inside the child’s root view.

For programmatic creation of controls or views, use the owning component’s documented ownership context where required, such as `runAsOwner`. Its scope applies while the callback executes; it is not an ambient ownership guarantee for arbitrary work scheduled later. Arrange asynchronous creation through component-aware framework APIs or re-enter the appropriate owner context when needed.

Repeated instances reveal architectural leaks quickly. Render two address editors with different initial values. If typing in one changes the other, investigate a shared module-level model or object reference. UI5 modules are evaluated once and their exported values can be shared; component instances are not automatically isolated from a singleton object imported by both.

## Communication alternatives and their trade-offs

A public event is suitable for a discrete fact: an address was confirmed, a document was selected, or a task finished. Define payload shape, whether the payload is copied, and whether the host is expected to respond. Do not encode an entire mutable application state into an event stream when consumers need a coherent current snapshot.

A shared named model can be appropriate when several features intentionally collaborate on the same draft. Define who creates and destroys it, which paths each participant may change, and how validation and cancellation work. Sharing a model is a stronger coupling than sharing an immutable identifier. It should be an explicit integration choice rather than the accidental effect of propagation.

A component-scoped event bus can decouple participants that cannot easily hold direct references. Specify channel and event names as a contract and unsubscribe on destruction. Avoid a global bus as a default: unrelated app instances can receive the same event, and the flow becomes difficult to trace. Prefer direct component events when there is already a clear parent-child relationship.

A service module is useful for domain operations that do not belong to a visual child. It can accept explicit model or transport dependencies. Avoid making the child component a service locator for all host functionality. That reverses the intended dependency direction and makes the child unusable without the exact original host.

## Nesting and navigation

A child with its own routes requires deliberate hash coordination. Two independent routers reacting to the same unpartitioned hash can compete. UI5 supports component targets and nested routing arrangements; configure their route prefixes and ownership together. Do not solve collisions by having each child replace `window.location.hash` manually.

Decide whether the host URL represents only the selected feature or also the child’s internal location. If a user bookmarks a specific child detail screen, cold startup must recreate the host, child, and selected entity without relying on an earlier event. [routing-navigation.md](routing-navigation.md) explains this reconstruction principle and the difference between routes and targets.

For a simple editor like this chapter’s example, no child router is necessary. The host chooses where the editor appears and supplies its draft. Adding routing would create an unnecessary second navigation vocabulary. Choose composition and routing independently: a reusable component can have no routes, and a multi-page application can use one component.

## Operational and security boundaries

Component boundaries are organizational, not security isolation. A nested component runs in the same application environment and generally has access to the same origin’s capabilities. Use only trusted component code and trusted resource locations. Server authorization must still apply to every business operation, irrespective of which component initiated it.

Model data is not automatically safe for every output context. The summary above is displayed by `sap.m.Text`, not inserted into raw HTML. Binding encoding is not a universal sanitizer for URL schemes, HTML fragments, CSS, or backend expressions. When a child accepts rich text or external links, define and enforce a context-specific policy.

Plan shared infrastructure explicitly. Theme and locale normally belong to the runtime or host environment. A child should not casually change global configuration. Decide how technical messages, business validation messages, and busy states are surfaced so nesting does not produce stacked spinners or duplicate error dialogs for one failure.

## Configuration versus mutable interaction

Separate startup configuration from ongoing interaction. Startup configuration identifies capabilities, service assumptions, or initial mode. A setter such as `setAddress` changes the current business draft after the instance exists. Treating every later state change as a new component creation is expensive and can discard unsaved work or duplicate subscriptions.

Component data can carry creation-time information, but its shape should be documented and validated by the receiving component. Do not assume the manifest’s arbitrary custom fields automatically become constructor settings or public properties. Distinguish descriptor configuration, component data, declared metadata properties, and explicitly exported methods.

When a host supplies configuration asynchronously, decide whether the child can render a safe empty state or whether creation should wait. An editor that briefly shows another customer’s cached values before configuration arrives has violated the host’s expectations even if it eventually displays the correct address. Make initialization readiness visible rather than timing host calls with arbitrary delays.

## Evolve a component contract deliberately

A component can change internally without breaking its host when the contract remains stable. Adding a new private input control is usually safe. Renaming an event, changing the meaning of `confirmed`, or replacing an address payload with a live model reference is an integration change. Review these changes as carefully as a backend API change.

Document whether confirmation means local acceptance, successful validation, or completed persistence. In this chapter it means the user accepted the local draft; no service has saved it. A host that interprets the event as durable persistence would report a misleading success. Event names and accompanying documentation should make that distinction explicit.

Prefer additive evolution when feasible. An optional payload field can be easier to integrate than changing the type of an existing field. However, merely adding data is not always harmless: event consumers may serialize entire objects, and new sensitive fields can unintentionally cross feature boundaries. Keep payloads minimal and purpose-specific.

## Integration diagnostics without breaking encapsulation

Encapsulation does not mean making failures opaque. A reusable feature should provide useful technical diagnostics and predictable failure notifications without exposing private control instances. Include the component type, operation category, and safe correlation information where the application’s logging policy permits it. Avoid logging complete addresses or other personal data merely to identify the failing instance.

Hosts should distinguish resource loading, initialization, validation, and persistence failures. Retrying resource loading may help a transient deployment error, but recreating the component is not a suitable default response to an invalid address. Conversely, a component whose JavaScript failed to load cannot emit its own business-level validation event.

During integration review, inspect both sides of every ownership handoff. Who destroys a model supplied by the host? Who detaches a subscription if the child is replaced? Who preserves a draft if the host changes the selected order? Explicit answers prevent a component architecture from becoming a collection of individually correct features with unreliable interactions.

## Shared service models require a separate agreement

If host and child deliberately share an OData model, define the model generation, update-group ownership, and save boundary as part of their integration contract. A child must not submit or reset unrelated host edits merely because they use the same model instance. [data-binding.md](data-binding.md) explains why V2 deferred groups and V4 update groups require different implementations.

Sharing a connection is not the same as sharing a draft. A child may need read access to reference data while keeping its editable address in a private JSON model. That arrangement can preserve cancellation semantics without duplicating every remote lookup.

## Verification checklist

- Resolve both component namespaces and verify manifest-first asynchronous creation.
- Confirm child initialization finishes before the host invokes its public methods.
- Edit a draft and verify the host changes only after the confirmed event.
- Create two instances and verify their draft models and IDs remain independent.
- Destroy the host view and verify the container destroys the embedded child.
- Confirm listener cleanup and avoid duplicate subscriptions after recreation.
- Simulate a missing child resource and inspect the host’s failure presentation.
- If routes are added, open a deep link directly and verify hash ownership.
- Review propagated models, event payloads, and resource URLs as explicit contracts.

## Official sources

- [ComponentContainer, including async loading and lifecycle settings](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/ComponentContainer.js).
- [UIComponent, root views and routing integration](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/UIComponent.js).
- [Versioned SDK Component API](https://ui5.sap.com/1.120.0/#/api/sap.ui.core.Component), including ownership, component usages, and creation contracts.
- [Versioned SDK ComponentContainer API](https://ui5.sap.com/1.120.0/#/api/sap.ui.core.ComponentContainer), including `componentCreated`, `propagateModel`, and `lifecycle`.

The source inspection uses the official 1.120.0 tag. Recheck public contracts against the patch level deployed by your organization before introducing newer composition features.
