# Data binding: state, transformation, and remote persistence

## Baseline and prerequisites

This chapter assumes **SAPUI5/OpenUI5 1.120.x**, asynchronous modules, XML views, and `sap.m`. Local examples use `JSONModel`. Remote examples are explicitly divided into **OData V2** and **OData V4** alternatives. Their service URLs, metadata, and entity sets are assumptions, not public demonstration endpoints. Do not combine the two model APIs into one implementation.

You should understand the ownership rules in [mvc-pattern.md](mvc-pattern.md) and [component-architecture.md](component-architecture.md). Binding connects a model location to a control property, aggregation, or context. It does not replace backend authorization, transaction design, or domain validation. Understanding those distinctions prevents a deceptively responsive screen from reporting changes as saved before the service has accepted them.

## Paths, model names, and contexts

A binding such as `{draft>/customer}` addresses an absolute path in the named model `draft`. A binding such as `{draft>customer}` is relative and needs a context on that model. The name before `>` selects the model; the slash after it selects absolute versus relative resolution. An unnamed binding addresses the default model, not whichever model happens to contain a matching property.

An element binding establishes a context for a control subtree. An aggregation binding creates repeated children with row-specific contexts. A property binding transfers one value, possibly through a type or formatter. These are complementary operations: a list binds its items, each item receives an entity context, and its text properties bind relatively beneath that context.

Do not treat a context path as a permanent business identifier. JSON list positions can change after sorting or replacement, and remote contexts have lifecycles managed by their bindings. Keep durable keys separately when designing routes. [routing-navigation.md](routing-navigation.md) explains how a route selects a context without exposing a model’s internal structure.

## Local wiring and binding modes

For the following local example, a controller creates a named JSON model on its view. Place this module at `webapp/controller/Editor.controller.js` under a `demo.orders` namespace mapping. The view is `webapp/view/Editor.view.xml`, created by the component’s root-view declaration or a routing target.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/ui/model/BindingMode",
  "../model/formatter"
], function (Controller, JSONModel, BindingMode, formatter) {
  "use strict";
  return Controller.extend("demo.orders.controller.Editor", {
    formatter: formatter,

    onInit: function () {
      this._draft = new JSONModel({
        customer: "Ada",
        quantity: 2,
        unitPrice: 12.5,
        editable: true
      });
      this._draft.setDefaultBindingMode(BindingMode.TwoWay);
      this.getView().setModel(this._draft, "draft");
    },

    onExit: function () {
      this._draft.destroy();
    }
  });
});
```

`OneWay` transfers model changes to the control but does not write user edits back. It is suitable for display values and for a screen whose controller deliberately collects edits elsewhere. A control can remain editable under a one-way binding, so visual editability alone does not prove the model is changing.

`TwoWay` transfers supported changes in both directions. It is appropriate for draft inputs and editable service properties when the model supports it. It does not mean every keystroke is necessarily committed to the model: the control’s change behavior matters. For example, input update timing depends on properties such as `valueLiveUpdate` as well as the binding mode.

`OneTime` initializes a value without continuing ordinary change observation after resolution. Use it for genuinely immutable presentation data, not for a label expected to react to locale changes or a status that will change after an operation. `Default` means the model’s default binding mode, so inspect that default rather than assuming all model classes behave identically.

Set a model-wide default early. A property binding can override it when necessary. Supported modes vary by model and binding class; aggregation binding does not become a two-way collection editor merely because the model default is `TwoWay`. Adding or removing entities still requires the appropriate model or list-binding operation.

## Types parse and validate; formatters represent

Create `webapp/model/formatter.js`:

```javascript
sap.ui.define([], function () {
  "use strict";
  return {
    estimate: function (quantity, unitPrice) {
      if (quantity == null || unitPrice == null) {
        return "";
      }
      var q = Number(quantity);
      var p = Number(unitPrice);
      if (!Number.isFinite(q) || !Number.isFinite(p)) {
        return "";
      }
      return (q * p).toFixed(2);
    }
  };
});
```

This formatter produces a simple display estimate. `toFixed` is not locale-aware currency formatting and binary floating-point multiplication is not an accounting policy. Use appropriate UI5 number/currency types for localized display and authoritative decimal rules for financial business operations.

The view wires the named model, typed input, composite formatter, and expression binding together:

```xml
<mvc:View
  controllerName="demo.orders.controller.Editor"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m">
  <VBox class="sapUiSmallMargin">
    <Label text="Customer" labelFor="customer"/>
    <Input id="customer" value="{draft>/customer}"
      editable="{draft>/editable}"/>
    <Label text="Quantity" labelFor="quantity"/>
    <Input id="quantity"
      value="{
        path: 'draft>/quantity',
        type: 'sap.ui.model.type.Integer',
        constraints: { minimum: 1 }
      }"/>
    <Text text="{
      parts: [
        { path: 'draft>/quantity' },
        { path: 'draft>/unitPrice' }
      ],
      formatter: '.formatter.estimate'
    }"/>
    <Button text="Continue"
      enabled="{= %{draft>/editable} &amp;&amp; %{draft>/quantity} > 0 }"/>
  </VBox>
</mvc:View>
```

A type supplies formatting, parsing, and validation as a coherent conversion contract. It can reject a non-integer quantity before writing it into the model. A formatter normally supplies only the display direction. A formatted output is not automatically invertible; UI5 cannot infer how an edited string should be split back into multiple source values.

Invalid input can remain in a control while the model retains its previous valid value. Consequently a positive quantity in the model is not proof that every visible input is currently valid. Register the application’s validation/message handling and prevent save according to active messages and business constraints. Do not solve validation by silently replacing every invalid input with zero.

OData types also reflect service metadata. Decimal values may use string representations to preserve precision, and dates are not interchangeable with arbitrary JavaScript timestamps. Inspect the relevant model’s type contract before coercing values with `Number` or `Date`. The numeric conversions in this local JSON demonstration are not a generic remote decimal strategy.

## Expression bindings and XML escaping

Use `{= ... }` for a small reactive expression. The one-time variant `{:= ... }` does not maintain the same ongoing dependency observation. An expression is a restricted expression language, not an unrestricted controller method body. Keep branching shallow and move substantial rules into a tested domain module or a formatter with a clear purpose.

The `%{...}` form requests embedded binding values with target type `any`, avoiding an unwanted conversion to the target property’s type. This is useful when the output property is boolean but an embedded value is numeric or textual. The `${...}` form can apply target-type conversion; selecting between them should be intentional.

XML syntax applies before UI5 evaluates the binding expression. A literal `&&` must therefore appear as `&amp;&amp;` in an XML attribute, and `<` as `&lt;`. Escaping an XML operator is different from escaping a runtime string for HTML or a URL. These layers solve different problems.

Do not construct expressions by inserting user-controlled strings into XML or binding syntax. Treat view definitions as trusted application resources. Ordinary model values remain data. Converting those values into dynamically evaluated binding declarations removes that separation and creates unnecessary parsing and security risks.

## Computed values need an explicit owner

Choose a formatter when the value exists only to present source data. A formatted full name or status label need not be written back into the model. The dependency list should include every source property used by the calculation. Reading hidden unrelated model paths from inside a formatter produces updates that the binding cannot reliably observe.

Choose a controller or domain operation when the computed value is part of an editable draft or must be submitted. Update it at a defined transition, such as recalculation or save preparation. Avoid maintaining two independent authoritative values for the same fact. If both quantity and total are editable, define which change recalculates which field and how rounding is resolved.

Choose the backend for authoritative business results, especially prices, taxes, availability, and authorization-dependent decisions. A client-side preview can improve responsiveness, but it should be labeled and reconciled with the server response. Service-side computed properties may require refresh or side-effect handling after an update; a successful PATCH does not guarantee every dependent display value has already changed.

Never mutate a model from inside a formatter. The formatter may execute repeatedly, with temporarily unavailable values, and in an order unrelated to the user’s intended workflow. Side effects can create loops, redundant requests, and inconsistent totals. A pure calculation gives the framework freedom to reevaluate without changing application behavior.

## OData V2: model-level operations and deferred groups

This section assumes an **OData V2** service at `/odata/v2/catalog/`, an entity set `Products`, a string key `ID`, and a writable property `Name`. The following component initialization fragment imports `sap/ui/model/odata/v2/ODataModel` as `ODataModel` and creates the named model `v2`. It is an alternative to, not an extension of, the V4 configuration below.

```javascript
var model = new ODataModel({
  serviceUrl: "/odata/v2/catalog/",
  useBatch: true
});
this.setModel(model, "v2");
model.setDeferredGroups(
  model.getDeferredGroups().concat(["catalogWrite"])
);
```

Preserve existing deferred groups when adding an application group. Replacing the list without understanding the defaults can cause unrelated pending changes to be sent earlier than intended. Deferred submission, batch transport, and backend transaction boundaries are distinct settings, even though they are often configured together.

A controller can perform a filtered read without constructing an OData query string. Import `sap/ui/model/Filter` and `sap/ui/model/FilterOperator` as `Filter` and `FilterOperator` in the controller’s `sap.ui.define` dependency list:

```javascript
var model = this.getOwnerComponent().getModel("v2");
var name = this.byId("productSearch").getValue();
model.read("/Products", {
  filters: [new Filter("Name", FilterOperator.Contains, name)],
  success: function (data) {
    // Consume data.results or update an explicitly owned local projection.
  },
  error: function () {
    // Present a classified request error through application message handling.
  }
});
```

The example assumes a view input with ID `productSearch`; add it to the view before wiring a search button to this handler. The property path is application-controlled and the value is passed through a filter object. Do not concatenate the search value into `$filter`, even if it has first been URL-encoded. OData literal escaping and URL encoding are not the same operation.

For a known entity key, V2 provides metadata-aware `createKey`. Use it after metadata is available:

```javascript
model.metadataLoaded().then(function () {
  var keyPath = model.createKey("/Products", { ID: selectedId });
  model.read(keyPath, {
    success: function (product) {
      // Use the returned product without synthesizing an entity URL.
    },
    error: function () {
      // Handle missing, unauthorized, and transport outcomes deliberately.
    }
  });
});
```

Here `selectedId` is an application-validated string identifier, not an OData fragment. Metadata loading failure needs its own application-level handling; do not assume every version’s `metadataLoaded` promise rejects as a conventional fetch promise would. The key helper handles protocol key formatting, but it does not authorize the selected entity.

V2 creation uses the model’s `create` API and can be queued in the explicitly deferred group:

```javascript
model.create("/Products", { ID: "P100", Name: "Notebook" }, {
  groupId: "catalogWrite",
  success: function (createdProduct) {
    // This operation has succeeded; reconcile the UI with server values.
  },
  error: function () {
    // Retain useful draft state and expose the operation error.
  }
});
model.submitChanges({
  groupId: "catalogWrite",
  success: function (batchData) {
    // Inspect batch/change responses; transport success alone is insufficient.
  },
  error: function () {
    // The batch transport itself failed.
  }
});
```

The constant ID assumes a service that permits client-assigned keys and an unused key. A service with server-generated keys requires a different payload. This example is about V2 operation wiring, not a universal product schema.

Two-way property edits in V2 require their own change-group configuration and save/cancel policy. Do not infer that the explicit group on this `create` call automatically assigns every bound property edit to that group. V2 APIs such as `setChangeGroups`, `setDeferredGroups`, `resetChanges`, and `submitChanges` must be designed together for the application’s entity types.

## OData V4: bindings and contexts, not V2 CRUD methods

This alternative assumes an **OData V4** service at `/odata/v4/catalog/` with entity set `Products` and writable `Name`. Add the following sections to the appropriate manifest. The component uses manifest-driven model creation and does not instantiate a V2 model under the same name.

```json
{
  "sap.app": {
    "dataSources": {
      "catalog": {
        "uri": "/odata/v4/catalog/",
        "type": "OData",
        "settings": { "odataVersion": "4.0" }
      }
    }
  },
  "sap.ui5": {
    "models": {
      "v4": {
        "dataSource": "catalog",
        "type": "sap.ui.model.odata.v4.ODataModel",
        "settings": {
          "synchronizationMode": "None",
          "operationMode": "Server",
          "autoExpandSelect": true,
          "groupId": "$auto",
          "updateGroupId": "catalogEdit"
        }
      }
    }
  }
}
```

This is a descriptor **fragment**: retain the existing application ID, dependencies, root view, and routing when merging it. The service URL ends with a slash. The named application update group `catalogEdit` uses API submission semantics by default in this baseline, while `$auto` allows automatic read batching.

Bind a list in the view and allow contexts to represent its entities:

```xml
<List id="products"
  items="{
    path: 'v4>/Products',
    parameters: {
      $select: 'ID,Name',
      $$updateGroupId: 'catalogEdit'
    }
  }">
  <CustomListItem>
    <Input value="{v4>Name}"/>
  </CustomListItem>
</List>
```

This snippet belongs inside a view with `sap.m` as the default namespace and a controller containing the following handlers. Ensure `ID` is actually a property in the assumed service. Editing `Name` through a supported two-way binding queues an update in the specified update group.

```javascript
onCreateProduct: function () {
  var binding = this.byId("products").getBinding("items");
  var context = binding.create({ Name: "New product" });
  context.created().then(function () {
    // Creation was persisted; use returned defaults and generated keys.
  }, function (error) {
    if (!error.canceled) {
      // Handle terminal/cancellation distinctions through application messaging.
    }
  });
},

onSaveProducts: function () {
  var model = this.getOwnerComponent().getModel("v4");
  return model.submitBatch("catalogEdit").then(function () {
    // Inspect messages and pending changes before declaring every edit saved.
  }, function () {
    // Batch transport failure; preserve the draft and offer an informed retry.
  });
},

onCancelProducts: function () {
  this.getOwnerComponent().getModel("v4").resetChanges("catalogEdit");
}
```

The creation payload assumes server-generated keys are allowed. `ODataListBinding#create` creates a transient context immediately. `Context#created()` represents persistence completion, not every failed attempt. A failed POST may be retried on a later submission, leaving the creation promise pending. Use the list binding’s `createCompleted` event and message handling to report individual failed attempts.

Do not await `context.created()` before submitting an API group: persistence cannot finish while the POST is still waiting in that group. Likewise, a fulfilled `submitBatch` promise is not proof that every operation inside the batch succeeded. Inspect operation messages, transient state, and pending changes as appropriate to the workflow.

V4 reads are driven by bindings and contexts, using APIs such as `requestContexts` and `requestObject` where imperative access is necessary. V4 creation uses list bindings, updates use property bindings or context APIs, and deletion uses contexts. **Do not call V2 `read`, `create`, or `submitChanges` on the V4 model.** Similar business intent does not imply an interchangeable JavaScript interface.

## Batching, cancellation, and refresh

Batching reduces HTTP overhead; it does not automatically make all application changes one business transaction. Understand the service’s change-set behavior, concurrency control, and operation responses. A successful outer HTTP response can contain failed inner operations. Preserve enough state to explain which changes remain pending rather than replacing all errors with a generic success toast.

Choose submission boundaries around user intent. An explicit Save button usually needs a deliberate update group and cancellation policy. Immediate-edit applications may prefer automatic groups but still need clear validation, failure recovery, and concurrent-update handling. Avoid one global group if unrelated features must save independently.

Refresh can overwrite or conflict with pending edits. Do not issue a global refresh after every field change to force computed values to appear. Use the model-specific refresh or side-effect APIs at the correct scope, after understanding pending-change restrictions. Resetting changes is also not an undo for operations already committed on the server.

## Filtering, sorting, and collection completeness

A bound list is not necessarily a complete in-memory copy of an entity set. Remote bindings request ranges as controls need them. Summing the currently rendered rows can therefore produce a partial total, even when the list appears long enough to represent the entire business result. Use an appropriate backend aggregate or an explicitly complete dataset when correctness requires all records.

Likewise, a context count available immediately may not be the final server count. Design empty, loading, and partial-result states deliberately. A temporary absence of contexts while a request runs is not the same as a successful query returning no matches. Bindings expose lifecycle information to help the UI distinguish these situations.

Filtering and sorting should operate at the intended layer. For a JSON model, the data is local. For an OData model in server operation mode, the service evaluates supported operations. A custom client comparator cannot be assumed to execute on the server. Verify backend support for requested operators and avoid relying on client-only behavior that disappears when the model changes.

Use application-controlled property names in filters and sorters. The user can choose among supported sort fields, but that choice should map to a known metadata property rather than becoming a raw expression. Search values remain values. This design is both easier to validate and more portable between services.

## Projection and request ownership

Request only the data the screen needs, while ensuring keys and dependencies remain available. In V4, `autoExpandSelect` can help derive selections from bindings, but it is not a reason to stop understanding the entity graph. A formatter that secretly reads an unbound navigation property can defeat the assumptions behind an efficient projection.

Explicit `$select` and `$expand` settings are service contracts. A property missing from a response may be intentionally unrequested rather than null in the business entity. Do not automatically turn every unavailable value into an empty string and persist it back. Distinguish “not loaded,” “not applicable,” and “explicitly empty” in editing workflows.

A programmatically created binding is a resource with an owner. Destroy it when the owning operation or feature no longer needs it, according to the relevant API contract. Do not destroy a V4 context independently: its binding manages its lifetime. Control-created bindings are normally tied to their controls, whereas standalone application bindings require deliberate cleanup.

## Validation, concurrency, and authoritative state

Validation has several stages. A type checks representation and local constraints; a domain rule checks relationships among fields; the server checks authoritative business constraints and permissions. Passing the first stage does not imply the others will pass. Keep rejected drafts recoverable and associate errors with the relevant fields or actions.

Concurrency failures require a user-facing decision. If another user changed an entity, blindly retrying with stale values can overwrite meaningful work or repeatedly fail. Respect the service’s concurrency mechanism, explain the conflict, and offer a refresh or reconciliation workflow. Do not remove concurrency headers merely to make an update appear successful.

A save operation also needs a snapshot policy. If users can continue editing while a request is pending, distinguish the values included in that submission from newer unsent changes. Clearing all dirty indicators when the earlier request completes can falsely report the newer changes as saved. Either lock the relevant draft scope or track submission and current state separately.

These policies depend on the model generation. V2 and V4 expose different pending-change, request, and context mechanisms. Write an application-specific adapter if a shared workflow must support both, but keep each adapter internally consistent. A wrapper with a method named `save` should not hide whether it waits for transport completion, entity persistence, or business operation completion.

## Diagnose bindings from the outside inward

When a field is empty, first inspect the intended model name and current context. Then check the resolved path, requested projection, and current request state. Only after those agree should you investigate the formatter or control. Replacing the binding with a hard-coded value can confirm rendering works, but it does not explain why the original path failed.

For an edit that appears to vanish, inspect parsing messages, update timing, binding mode, and whether a parent context was replaced. For an edit that remains visible but is not saved, inspect the update group and pending changes before blaming rendering. This ordered diagnosis avoids broad refresh calls that temporarily hide the problem while introducing extra requests.

Keep diagnostic logging proportional to the issue. Paths and lifecycle categories are often sufficient; complete entity payloads can contain personal or commercially sensitive information. Remove temporary instrumentation after the cause is understood and retain only the application’s deliberate logging policy.

## Security and verification

Standard text controls handle their own text rendering, but **binding encoding is not a universal sanitizer**. A bound value used as an external URL still requires a scheme and destination policy. Raw HTML, rich text, CSS, and backend expressions have different rules. Model values must remain data rather than becoming dynamically assembled XML, JavaScript, or OData expressions.

Keep service URLs and query-property names application-controlled. Pass user values through typed filters, validated payloads, and metadata-aware key construction. URL encoding alone neither makes an OData expression safe nor grants access to an entity. The server must enforce authorization and validate every submitted field.

### Verification checklist

- Confirm model names and absolute/relative paths resolve in the intended context.
- Check input update timing independently from the selected binding mode.
- Enter invalid values and inspect both control state and the retained model value.
- Verify formatters are pure and declare all their binding dependencies.
- Confirm computed financial results follow backend precision and rounding policy.
- Inspect the manifest and imports to prove each remote example uses only its stated model generation.
- Trace read groups, update groups, batch responses, messages, and pending changes.
- Simulate failed creates, retries, cancellation, and concurrent updates.
- Confirm save success is not inferred solely from outer batch completion.
- Review user-controlled values for OData, URL, HTML, and expression-context hazards.

## Official sources

- [BindingMode definitions, 1.120.0](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/BindingMode.js).
- [Expression parser and supported expression behavior](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/base/ExpressionParser.js).
- [OData V2 model public API documentation in source](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js).
- [OData V4 model and supported model parameters](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js).
- [V4 list binding creation, retry, and completion semantics](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataListBinding.js).
- [V4 Context lifecycle and operation contracts](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/Context.js).
- [Versioned SDK binding syntax guide](https://ui5.sap.com/1.120.0/#/topic/e2e6f4127fe4450ab3cf1339c42f8321), including expression and composite binding context.
