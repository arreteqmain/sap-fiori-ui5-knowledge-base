# Controllers: lifecycle, events, binding, and CRUD

## Scope and prerequisites

These examples target the public APIs available in UI5 **1.120**. Use a maintained patch of that release or a newer compatible release, and check the API documentation shipped with your actual runtime. They use OpenUI5 modules also included in SAPUI5: `sap.ui.core`, `sap.m`, and the standard model packages. No SAPUI5-only smart controls, Fiori elements extension hooks, or launchpad services are assumed. A freestyle controller is not a substitute for a Fiori elements extension controller.

The illustrative application namespace is `com.acme.orders`, mapped to its `webapp` directory by the UI5 loader. JavaScript modules below belong beneath that directory. A running UI5 application must bootstrap asynchronously, declare its libraries, and create an XML view. The complete local view is in [views.md](./views.md#complete-local-orders-view). Pair it with the first controller below; it needs no server. The remote controller is a separate alternative, not another object to merge wholesale into that controller.

The shared order contract has `OrderID` as an opaque string, `CustomerName` as text, `Amount` as a canonical decimal string, `Currency` as an ISO currency code, `Status` as a business code, and `OrderDate` as an ISO calendar date. Remote services expose `OrderSet`. `"000042"` is different from `"42"`; never normalize keys through `Number`, `parseInt`, or spreadsheet-style inference. The local demo deliberately implements only customer-name editing, so it does not pretend that a JSON array supplies server-side transactions or authorization.

See [models.md](./models.md) for model ownership and OData versions, [services.md](./services.md) for transport boundaries, and [utilities.md](./utilities.md) for validation and formatting. These files occupy the paths listed in the repository [README](../README.md#repository-structure).

## Choose responsibilities before writing handlers

A controller coordinates user intent with view state. It should decide whether a Save button is enabled, collect validated input, call an appropriate service, and present a safe outcome. It should not construct authentication tokens, implement accounting policy, or become the sole repository for an application's domain rules. A helper that accepts values and returns a value can usually live outside the controller. A helper that needs `this.byId` probably still belongs near the view.

Separate persistent business data from temporary interaction data. In the local example, `orders` holds the list while `view` holds the draft and selection. In a remote application, the corresponding business model should normally remain the OData model rather than a second, silently diverging JSON copy. An explicit edit buffer is reasonable, but its copy-in, save, conflict, and cancel behavior must be deliberate.

Use binding to connect state to controls. Calling `setText` in five different handlers creates five opportunities for the screen and model to disagree. Updating `/draft/CustomerName` or `/canSave` makes the state observable and inspectable. This does not mean every transient variable needs a model path: request handles, timer identifiers, and generation counters are controller internals, not presentation data.

## Complete local CRUD controller

Place this module at `webapp/controller/Orders.controller.js` and use the complete view linked above. The two named models are owned by this controller and destroyed during exit. New IDs are local demonstration identifiers, not a safe production numbering strategy.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox"
], function (Controller, JSONModel, MessageBox) {
  "use strict";

  return Controller.extend("com.acme.orders.controller.Orders", {
    onInit: function () {
      this._nextId = 43;
      this._orders = new JSONModel({
        OrderSet: [{
          OrderID: "000042",
          CustomerName: "Acme North",
          Amount: "1250.00",
          Currency: "EUR",
          Status: "NEW",
          OrderDate: "2026-10-17"
        }]
      });
      this._viewState = new JSONModel({
        selectedId: "",
        draft: { CustomerName: "" },
        canSave: false
      });
      this.getView().setModel(this._orders, "orders");
      this.getView().setModel(this._viewState, "view");
    },

    onSelect: function (event) {
      const item = event.getParameter("listItem");
      const context = item && item.getBindingContext("orders");
      if (!context) {
        return;
      }
      const order = context.getObject();
      this._viewState.setProperty("/selectedId", order.OrderID);
      this._viewState.setProperty("/draft", {
        CustomerName: order.CustomerName
      });
      this._viewState.setProperty("/canSave", false);
    },

    onDraftChange: function (event) {
      const value = event.getParameter("value");
      this._viewState.setProperty("/draft/CustomerName", value);
      this._viewState.setProperty("/canSave",
        value.trim().length > 0 && value.trim().length <= 80);
    },

    onNew: function () {
      this.byId("ordersTable").removeSelections(true);
      this._viewState.setProperty("/selectedId", "");
      this._viewState.setProperty("/draft", { CustomerName: "" });
      this._viewState.setProperty("/canSave", false);
    },

    onSave: function () {
      const name = this._viewState
        .getProperty("/draft/CustomerName").trim();
      if (!name || name.length > 80) {
        MessageBox.error("Enter a customer name of 1 to 80 characters.");
        return;
      }
      const id = this._viewState.getProperty("/selectedId");
      const rows = this._orders.getProperty("/OrderSet").slice();
      const index = rows.findIndex(function (row) {
        return row.OrderID === id;
      });
      if (id && index < 0) {
        MessageBox.error("This order is no longer available.");
        this.onNew();
        return;
      }
      if (index >= 0) {
        rows[index] = Object.assign({}, rows[index], {
          CustomerName: name
        });
      } else {
        const newId = String(this._nextId++).padStart(6, "0");
        rows.push({
          OrderID: newId,
          CustomerName: name,
          Amount: "0.00",
          Currency: "EUR",
          Status: "NEW",
          OrderDate: "2026-10-17"
        });
      }
      this._orders.setProperty("/OrderSet", rows);
      this.onNew();
    },

    onDelete: function () {
      const id = this._viewState.getProperty("/selectedId");
      if (!id) {
        return;
      }
      const rows = this._orders.getProperty("/OrderSet");
      this._orders.setProperty("/OrderSet",
        rows.filter(function (row) { return row.OrderID !== id; }));
      this.onNew();
    },

    onExit: function () {
      this._orders.destroy();
      this._viewState.destroy();
    }
  });
});
```

Saving creates a new array and, for updates, a new row object. That makes the update boundary obvious and avoids accidentally editing the original row while typing into a draft. `setProperty` notifies bindings; directly pushing into an array obtained through `getData` is not a reliable notification strategy. There is no need to call `refresh(true)` after every explicit JSON property update.

The selection handler uses the binding context of the selected item, not a DOM row number. Sorting, filtering, and growing lists can all change visible positions without changing entity identity. Store the key, then locate the current entity when applying a local update. In a remote application retain or resolve the appropriate OData context instead of assuming that a list index is a stable resource address.

The example uses English literal messages to remain independent of a resource bundle. Before shipping, replace those literals with the resource-model pattern in [models.md](./models.md#resource-model-and-supported-locales), including translations for validation and destructive actions. A production Delete interaction should normally request confirmation or offer a genuinely reversible undo. Do not claim undo if the backend deletion has already become irreversible.

## Lifecycle hooks are not interchangeable

`onInit` runs once for an instance. Create view-owned state, attach route handlers, and establish persistent subscriptions there. Controls from the view are available through `byId`, but rendered DOM is not guaranteed. Do not start a second router or recreate a component model just because a route matched.

`onBeforeRendering` can run repeatedly. It is appropriate for work that truly depends on an impending render, not for ordinary initial data loading. Starting network requests there can create request storms when unrelated bound properties invalidate controls. Avoid changing state repeatedly from rendering hooks without understanding whether that change triggers another rendering cycle.

`onAfterRendering` also runs repeatedly. Prefer control properties and public APIs to DOM manipulation. If integration with an external widget genuinely requires DOM access, ensure its previous instance and listeners are removed before installing another one. Never append an unmanaged listener on every render and hope view destruction will clean it up.

`onExit` is the controller's cleanup boundary. Abort owned requests, clear timers, detach subscriptions, and destroy objects that are not owned by the view or component. Do not destroy a shared component model from a child controller. Conversely, attaching a model to a view does not transfer all lifecycle responsibilities for an independently created model; retain a clear owner and destroy it explicitly.

An asynchronous `onInit` return value is not a universal loading barrier. Start asynchronous work and handle its rejection explicitly, or use a component initialization design that the framework supports. A rejected promise from a fire-and-forget event handler must not become an unhandled rejection merely because UI5 invoked the handler.

## Latest-request-wins remote controller

This alternative module belongs at `webapp/controller/RemoteOrders.controller.js`. It uses OData **V2**, not V4. The service must expose the contract above at the same-origin URL `/sap/opu/odata/sap/ZACME_ORDERS_SRV/`. Configure a development proxy or deployment destination; do not paste credentials into the source. Its view can contain a SearchField with `search=".onSearch"` and a table bound to `orders>/OrderSet`. Bind the table's `busy` property to `view>/busy`. The controller starts an initial read, so no additional startup method is required.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/odata/v2/ODataModel",
  "sap/ui/model/json/JSONModel",
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "sap/m/MessageBox"
], function (Controller, ODataModel, JSONModel,
  Filter, FilterOperator, MessageBox) {
  "use strict";

  return Controller.extend("com.acme.orders.controller.RemoteOrders", {
    onInit: function () {
      this._alive = true;
      this._generation = 0;
      this._request = null;
      this._remote = new ODataModel({
        serviceUrl: "/sap/opu/odata/sap/ZACME_ORDERS_SRV/",
        useBatch: true
      });
      this._rows = new JSONModel({ OrderSet: [] });
      this._state = new JSONModel({ busy: false });
      this.getView().setModel(this._rows, "orders");
      this.getView().setModel(this._state, "view");
      this._load("");
    },

    onSearch: function (event) {
      this._load((event.getParameter("query") || "").trim());
    },

    _load: function (query) {
      const generation = ++this._generation;
      const previous = this._request;
      this._request = null;
      if (previous) {
        previous.abort();
      }
      this._state.setProperty("/busy", true);
      const current = function () {
        return this._alive && generation === this._generation;
      }.bind(this);
      const finish = function () {
        this._request = null;
        this._state.setProperty("/busy", false);
      }.bind(this);

      this._request = this._remote.read("/OrderSet", {
        filters: query ? [
          new Filter("CustomerName", FilterOperator.Contains, query)
        ] : [],
        urlParameters: {
          "$select": "OrderID,CustomerName,Amount,Currency,Status,OrderDate",
          "$top": "50"
        },
        success: function (data) {
          if (!current()) { return; }
          this._rows.setProperty("/OrderSet", data.results || []);
          finish();
        }.bind(this),
        error: function () {
          if (!current()) { return; }
          finish();
          MessageBox.error("Orders could not be loaded. Try again.");
        }
      });
    },

    onExit: function () {
      this._alive = false;
      ++this._generation;
      if (this._request) {
        this._request.abort();
        this._request = null;
      }
      this._remote.destroy();
      this._rows.destroy();
      this._state.destroy();
    }
  });
});
```

The generation increments **before** aborting the previous request. Some transports invoke an error callback during cancellation. That callback already belongs to an obsolete generation and cannot clear the new request's busy state or display an error. Aborting alone is insufficient: a response may already be delivered, a transport may complete despite cancellation, or an intermediary may have accepted a write before the client disconnected.

The `alive` check protects against late completion after navigation destroys the view. Neither success nor failure should touch destroyed models, show an unrelated page's error dialog, or retain a controller indefinitely. The counter also ensures that an older request cannot overwrite a newer result if responses arrive in reverse order. This is a read-only snapshot example; do not edit this JSON copy and expect the OData model to persist it.

The `$top` cap is intentional. It is not full pagination, and the interface must communicate that the results are limited. A production worklist generally binds directly to the OData collection and lets the list binding manage paging. If an export or background process needs all matching data, follow the service's paging protocol rather than deleting `$top` and assuming unlimited results are safe.

## Events, contexts, and route subscriptions

UI5 event parameters differ by event. `selectionChange` on a list supplies `listItem`; a button `press` supplies its source; `liveChange` supplies the new input value. Read the event's public API instead of depending on the shape of a browser DOM event. When a button is inside a repeated row, `event.getSource().getBindingContext("orders")` identifies that row without walking parent controls.

Use `.bind(this)` when a callback needs the controller instance, or use an arrow function with a deliberate lexical scope. Save the exact bound function when you need to detach it later. Repeating `handler.bind(this)` during detach produces a different function and leaves the original subscription attached.

For routing, the prerequisite is a component router with a route named `order` and a pattern such as `orders/{OrderID}`. Attach with `route.attachPatternMatched(this._onOrderMatched, this)` during initialization and detach with the same method and listener object during exit. Route parameters are transport strings, not authorization claims. Validate the key's expected shape and build the entity path through model APIs; never splice an untrusted parameter into an OData key predicate.

Navigation does not necessarily destroy a cached view. A route match can happen several times while `onInit` happens once. Refresh or rebind the selected entity when its route changes, and invalidate stale asynchronous work even if the controller remains alive. Otherwise a slow detail request for the previous key can populate the newly selected page.

## Remote writes and transaction feedback

For V2, use `create`, `update`, `remove`, and `submitChanges` as described in [models.md](./models.md#odata-v2-crud-and-explicit-submission). For V4, use list bindings, contexts, `setProperty`, `delete`, and `submitBatch`; copying a V2 controller and changing only the import is not a migration. An OData V4 model does not implement the V2 `read` method.

A Save handler should distinguish validation failure, request submission, accepted business changes, and final UI reconciliation. Disable repeated submission while a write is pending. Retain the draft after a recoverable failure. Do not clear the editor merely because the outer HTTP batch returned success: an individual changeset or request can have failed.

For optimistic concurrency, preserve model-managed ETags and handle conflicts as a separate user decision. A generic automatic retry could overwrite someone else's update. For create operations, a timeout can mean “outcome unknown,” not “nothing happened.” Reconcile using a server-supported idempotency mechanism or a lookup before issuing another create.

Never display `responseText`, exception stacks, SQL fragments, or arbitrary backend messages directly in a MessageBox. Map known failure classes to translated application messages. Diagnostics should carry an approved correlation identifier, not customer payloads or authentication headers. The service wrapper in [services.md](./services.md#safe-error-contract) defines a small error vocabulary suitable for this boundary.

## Unsaved changes and screen transitions

Define what “dirty” means for each editor. In the local example it is a draft differing from the selected row, not merely the fact that an input received focus. For a remote OData editor it can include model pending changes, transient created contexts, and invalid control text that never reached the model. Looking only at a JSON draft or only at `hasPendingChanges` can miss one of those states.

Before switching selection, decide whether to save, discard, or cancel the transition. A confirmation dialog is asynchronous: by the time it closes, the user may have navigated elsewhere or selected another entity. Capture the intended transition and recheck relevance before applying it. Do not let a delayed confirmation switch the current page back to an obsolete order.

A navigation guard should not silently submit every pending group on the component model. Another view may own changes in a different group, and some pending state may not yet be valid. Submit the transaction scope that belongs to this editor. Likewise, avoid a global reset that discards unrelated work; target the documented group or entity scope.

Busy indicators communicate waiting but do not provide concurrency control by themselves. Disable or guard actions that could queue duplicate writes, and preserve a clear distinction between a user cancelling a read and a user trying to cancel a submitted write. If a save response is lost, display a recovery path that acknowledges uncertainty rather than confidently stating that nothing changed.

For recoverable validation errors, preserve field contents and focus the relevant field or message summary. For a deleted entity, clear the obsolete context and return to a meaningful list state. For authorization denial, remove actions that no longer apply but still rely on the backend for enforcement. These outcomes are controller decisions informed by service error codes; they are not reasons to expose arbitrary backend strings.

## Manual verification checklist

- Load the local view and select `"000042"`; its zeros must survive editing and deletion.
- Create, edit, and delete a row; confirm draft changes do not mutate an unsubmitted row.
- Enter blank and overlong names; Save must not change the list.
- Sort or filter a list before editing; verify identity comes from context, never visible position.
- With a real V2 endpoint, throttle two searches so the older response arrives last.
- Confirm only the current response changes rows or clears busy state.
- Navigate away during loading; no late dialogs or accesses to destroyed models should occur.
- Fail a request and inspect the displayed message for accidental backend text.
- Exercise a failed changeset and a concurrency conflict before accepting remote Save behavior.
- Revisit a cached route and check that subscriptions and requests are not duplicated.

These are manual integration checks, not a claim that a live backend was exercised while writing this reference. Rendering, network, and authorization behavior must be verified in the consuming application with its actual metadata and deployment configuration.

## Official API references

Research access: the SDK host could not be resolved during research. API behavior was checked through SAP's official OpenUI5 GitHub sources instead. The [Controller SDK reference](https://ui5.sap.com/#/api/sap.ui.core.mvc.Controller) is a navigation link, not a claim of successful SDK access; select your deployed version when consulting it.

The researched sources are SAP's public OpenUI5 implementation and API JSDoc. The pinned sources describe the baseline; the Controller source also documents current lifecycle behavior. Public APIs, not internal implementation details, are the contract.

- [Controller lifecycle and controller methods](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/core/mvc/Controller.js).
- [V2 ODataModel: request handles, CRUD, keys, deferred groups](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js).
- [JSONModel: data updates and binding behavior](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/json/JSONModel.js).
- [V4 Context: asynchronous access and context operations](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/Context.js).
