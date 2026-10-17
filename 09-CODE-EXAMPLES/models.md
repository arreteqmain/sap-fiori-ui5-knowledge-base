# Models: JSON, resources, OData V2, and OData V4

## Scope and version assumptions

Examples use public APIs available in UI5 **1.120**. Deploy a maintained patch and consult the documentation for the exact runtime you ship. JSONModel, ResourceModel, and both OData model implementations are available in **OpenUI5 and SAPUI5**. The SAP backend URLs are illustrative deployment contracts, not a requirement to use SAPUI5-only libraries. Do not infer control compatibility from model availability: some SAPUI5 smart controls support V2 but not V4.

The namespace is `com.acme.orders`, mapped to `webapp`. All JavaScript examples are UI5 modules with declared dependencies. Backend-dependent examples require an authenticated same-origin route or development proxy. The service must provide working metadata, paging, error responses, and the OData version advertised by its endpoint. No code here creates the backend or grants authorization.

The contract is `OrderSet` with string key `OrderID`, `CustomerName` string, `Amount` decimal with precision 15 and scale 2, `Currency` string, `Status` string, and `OrderDate`. V2 represents a date-only field here as `Edm.DateTime` with `sap:display-format="Date"`; V4 uses `Edm.Date`. That distinction changes the client value representation. Do not send the same JavaScript date object to both services merely because the property names match.

The local interface is in [views.md](./views.md), lifecycle and ownership examples in [controllers.md](./controllers.md), request adapters in [services.md](./services.md), and representation helpers in [utilities.md](./utilities.md). These exact filenames are listed in the repository [README](../README.md#repository-structure).

## Model roles and ownership

A JSON model is a client-side object graph. It is excellent for filters, busy flags, local drafts, and small already-loaded datasets. It does not provide server paging, ETags, changesets, or persistence. An OData model understands service metadata, binding caches, entity identity, and transport behavior. A ResourceModel resolves translatable text; it is not a general-purpose configuration database.

Choose an owner for every model. A component commonly owns shared OData and resource models. A controller may own a small view-state model. Attaching a shared model to a child view does not mean that child should destroy it on exit. Conversely, independently creating a JSONModel and forgetting to destroy it can leave event relationships and bindings alive longer than intended.

Use explicit names when multiple models coexist. `orders>/OrderSet` refers to the business model; `view>/busy` refers to presentation state; `i18n>save` refers to a resource key. The default model is a valid design choice, but mixing unnamed and named bindings accidentally produces empty controls that can look like transport failures.

Avoid copying an entire OData response into JSON simply to make binding easier. That discards metadata-based typing, cache identity, server paging, and automatic concurrency behavior. A deliberate projection for a read-only dashboard is different: it has an explicit refresh boundary and no implied persistence. Document which kind of copy you are making.

## JSON state with explicit notification

Place this factory at `webapp/model/createViewState.js`. Import it from a controller, call it once during initialization, attach the result as `view`, and destroy it during that controller's exit. Its defaults are intentionally plain serializable values, not controls, request handles, or cyclic references.

```javascript
sap.ui.define(["sap/ui/model/json/JSONModel"], function (JSONModel) {
  "use strict";
  return function createViewState() {
    return new JSONModel({
      busy: false,
      query: "",
      selectedId: "",
      draft: {
        CustomerName: "",
        Amount: "0.00",
        Currency: "EUR"
      },
      validation: {
        customer: "",
        amount: ""
      }
    });
  };
});
```

Update it through `setProperty("/busy", true)` or replace a complete subtree through `setProperty("/draft", newDraft)`. For list edits, construct a replacement array and set the collection property. Mutating `getData().rows.push(...)` does not clearly notify bindings. Calling `refresh(true)` everywhere masks this mistake and makes it harder to reason about update scope.

An edit buffer must be an actual copy of the editable fields. Assigning the selected entity object directly to `/draft` creates an alias: typing can mutate what the user thinks is the persisted record. Copy only whitelisted writable properties. For nested structures, a shallow copy is not enough; define a domain-specific mapping rather than copying model internals and navigation links.

JSONModel has a size limit affecting the number of entries exposed to bindings. Raising it is sometimes appropriate for a bounded value-help list, but it is not server pagination. Loading hundreds of thousands of entities into JSON and increasing the limit shifts the problem to browser memory and rendering. Prefer a service-backed binding for large collections.

## Resource model and supported locales

This complete factory belongs at `webapp/model/createI18n.js`. It assumes English text in `webapp/i18n/i18n.properties` and German translations in `webapp/i18n/i18n_de.properties`. The empty locale string denotes the base bundle. Only these two bundles are claimed to exist, and unsupported locales fall back to the English base.

```javascript
sap.ui.define([
  "sap/ui/model/resource/ResourceModel"
], function (ResourceModel) {
  "use strict";
  return function createI18n() {
    return new ResourceModel({
      bundleName: "com.acme.orders.i18n.i18n",
      supportedLocales: ["", "de"],
      fallbackLocale: "",
      async: true
    });
  };
});
```

The complete base bundle for the keys used in this chapter is:

```properties
ordersTitle=Orders
save=Save
saveFailed=The order could not be saved. Try again.
savedOrder=Order {0} was saved.
invalidCustomer=Enter a customer name of 1 to 80 characters.
```

The German bundle contains the same keys:

```properties
ordersTitle=Aufträge
save=Speichern
saveFailed=Der Auftrag konnte nicht gespeichert werden. Versuchen Sie es erneut.
savedOrder=Auftrag {0} wurde gespeichert.
invalidCustomer=Geben Sie einen Kundennamen mit 1 bis 80 Zeichen ein.
```

Attach the model as `i18n` and bind XML text with `{i18n>save}`. Imperative code must account for asynchronous bundle loading. `Promise.resolve(resourceModel.getResourceBundle()).then(function (bundle) { return bundle.getText("savedOrder", ["000042"]); })` works whether the API returns an already available bundle or a promise. Handle the promise rejection where the message is consumed.

Do not list every possible locale “for completeness.” `supportedLocales` is an availability declaration that controls bundle lookup, not a translation service. Language-region variants may resolve through their language fallback; verify your desired fallback chain. If you introduce `en` as a separate file, update the configuration and deploy that file consistently. Missing base keys should be treated as a release defect, not shown to users as cryptic resource names.

For languages with different sentence order, translate the whole sentence and substitute parameters. Concatenating `"Order " + id + " saved"` prevents natural translation. Dates and amounts should be formatted with locale-aware types, not embedded as raw backend strings in a translated sentence.

## OData V2 CRUD and explicit submission

Place this module at `webapp/model/V2Orders.js`. It is an independent demonstration repository. Its model is not shared with any other editor, which makes the dedicated `orderWrites` deferred group meaningful. The service supports creating a supplied unused key and updating customer names. A real service may generate keys instead; adapt the create payload to its metadata and business contract.

```javascript
sap.ui.define([
  "sap/ui/model/odata/v2/ODataModel"
], function (ODataModel) {
  "use strict";

  function safeError() {
    return new Error("ORDER_REQUEST_FAILED");
  }

  return function createV2Orders() {
    const model = new ODataModel({
      serviceUrl: "/sap/opu/odata/sap/ZACME_ORDERS_SRV/",
      useBatch: true,
      defaultBindingMode: "TwoWay"
    });
    model.setDeferredGroups(
      model.getDeferredGroups().concat(["orderWrites"])
    );

    function keyPath(id) {
      if (typeof id !== "string" || !/^\d{6}$/.test(id)) {
        throw new Error("INVALID_ORDER_ID");
      }
      return "/" + model.createKey("OrderSet", { OrderID: id });
    }

    function queue(method, path, payload) {
      const operation = new Promise(function (resolve, reject) {
        const options = {
          groupId: "orderWrites",
          changeSetId: "orderTransaction",
          success: function (data) { resolve(data); },
          error: function () { reject(safeError()); }
        };
        if (method === "remove") {
          model.remove(path, options);
        } else {
          model[method](path, payload, options);
        }
      });
      // The caller may submit later; mark early rejection as handled.
      operation.catch(function () {});
      return operation;
    }

    return {
      model: model,
      ready: function () { return model.metadataLoaded(); },
      read: function (id) {
        return model.metadataLoaded().then(function () {
          return new Promise(function (resolve, reject) {
            model.read(keyPath(id), {
              success: resolve,
              error: function () { reject(safeError()); }
            });
          });
        });
      },
      queueCreate: function (order) {
        return queue("create", "/OrderSet", {
          OrderID: order.OrderID,
          CustomerName: order.CustomerName,
          Amount: order.Amount,
          Currency: order.Currency,
          Status: "NEW"
        });
      },
      queueUpdate: function (id, customerName) {
        return queue("update", keyPath(id), {
          CustomerName: customerName
        });
      },
      queueDelete: function (id) {
        return queue("remove", keyPath(id));
      },
      submit: function () {
        return new Promise(function (resolve, reject) {
          model.submitChanges({
            groupId: "orderWrites",
            success: resolve,
            error: function () { reject(safeError()); }
          });
        });
      },
      destroy: function () { model.destroy(); }
    };
  };
});
```

The V2 service permits omitted `OrderDate` during creation and assigns its default date. If your metadata requires the field, provide the correct V2 date representation. The six-digit validation is an application rule for this illustration, not a general limitation of OData keys. `createKey` still handles OData literal escaping; do not replace it with string interpolation.

Call `ready()` before queuing an update or delete because key construction depends on metadata. Do not await a deferred write before submission: it cannot complete until its group is sent. The order is “queue, submit, await both,” as in this integration function:

```javascript
sap.ui.define([
  "com/acme/orders/model/V2Orders"
], function (createV2Orders) {
  "use strict";
  return async function changeCustomer(id, name) {
    const repository = createV2Orders();
    try {
      await repository.ready();
      await repository.read(id);
      const write = repository.queueUpdate(id, name);
      const batch = repository.submit();
      await Promise.all([write, batch]);
    } finally {
      repository.destroy();
    }
  };
});
```

The preliminary read makes the entity and its model-managed ETag available before updating. The individual operation promise matters. An HTTP-successful `$batch` may contain a failed changeset. Resolving `submitChanges` alone does not prove that the order changed. For multiple queued writes, await every operation as well as the submission. The sample's single changeset ID groups those writes in one changeset when the service supports the corresponding transaction semantics; multiple changesets are not one atomic business transaction.

V2 `metadataLoaded()` is not a complete metadata-failure strategy in this baseline: metadata failure is reported through model events and the readiness promise can remain pending. A production owner must handle `metadataFailed`, provide a safe startup failure state, and define a retry or bounded waiting policy. The abortable read adapter in [services.md](./services.md) addresses individual data requests; it does not magically repair unavailable metadata.

## OData V4 binding and context operations

V4 is binding-oriented. There is no model-level V2 `read`, `create`, `update`, `remove`, or `submitChanges` API to copy across. The following factory at `webapp/model/V4Orders.js` creates one owned list binding and a dedicated application update group. The V4 endpoint is `/odata/v4/orders/`, including its trailing slash.

```javascript
sap.ui.define([
  "sap/ui/model/odata/v4/ODataModel"
], function (ODataModel) {
  "use strict";
  return function createV4Orders() {
    const model = new ODataModel({
      serviceUrl: "/odata/v4/orders/",
      synchronizationMode: "None",
      groupId: "$auto",
      updateGroupId: "orderWrites",
      autoExpandSelect: true
    });
    const list = model.bindList("/OrderSet", null, null, null, {
      "$select": "OrderID,CustomerName,Amount,Currency,Status,OrderDate",
      "$$updateGroupId": "orderWrites"
    });
    return {
      model: model,
      list: list,
      requestPage: function (start, length) {
        return list.requestContexts(start, length);
      },
      queueCreate: function (order) {
        const context = list.create({
          OrderID: order.OrderID,
          CustomerName: order.CustomerName,
          Amount: order.Amount,
          Currency: order.Currency,
          Status: "NEW",
          OrderDate: order.OrderDate
        });
        context.created().catch(function () {
          // Cancellation rejects this promise; business failure may retry.
        });
        return context;
      },
      queueCustomerChange: function (context, name) {
        const update = context.setProperty(
          "CustomerName", name, "orderWrites"
        );
        update.catch(function () {});
        return update;
      },
      queueDelete: function (context) {
        const deletion = context.delete("orderWrites");
        deletion.catch(function () {});
        return deletion;
      },
      submit: function () {
        return model.submitBatch("orderWrites");
      },
      hasPending: function () {
        return model.hasPendingChanges("orderWrites");
      },
      discard: function () {
        model.resetChanges("orderWrites");
      },
      destroy: function () {
        list.destroy();
        model.destroy();
      }
    };
  };
});
```

`requestPage(0, 20)` resolves to contexts, not plain entity objects. Use each context's `getProperty` for already available fields or `requestObject` for asynchronous object access. A relative binding below that context can bind to `CustomerName` without reconstructing an entity URL. If you create a temporary programmatic binding, destroy it when finished; a control owns its own binding and should not have it destroyed externally without a deliberate rebind.

For a customer update, first obtain a context from `requestPage`, queue `setProperty`, start submission, and then await both promises. Do not await the update promise before sending the application group. For deletion, await the context's deletion promise as well as submission. A successful outer batch promise alone is not an individual-operation success signal.

Creation requires special care. `context.created()` resolves when the transient entity has been persisted. A backend validation failure can leave that promise pending while UI5 retains the transient context for retry. It is therefore incorrect to implement every Save as `await created()` with no failure UI or cancellation path. Observe the list binding's `createCompleted` event and its `success` parameter to display a safe failure and preserve the draft. The event also supplies the affected context, allowing multiple create operations to be distinguished.

The following complete helper module sends a single queued create and reports that attempt's outcome. It expects the owned list and group from the factory above, no other create activity on that repository, and a validated payload. It leaves a failed transient entity available for the caller to correct or discard.

```javascript
sap.ui.define([], function () {
  "use strict";
  return function createAndSubmit(repository, order) {
    return new Promise(function (resolve, reject) {
      let context;
      let settled = false;
      function finish(error) {
        if (settled) { return; }
        settled = true;
        repository.list.detachCreateCompleted(completed);
        if (error) { reject(new Error("ORDER_CREATE_FAILED")); }
        else { resolve(context); }
      }
      function completed(event) {
        if (event.getParameter("context") === context) {
          finish(!event.getParameter("success"));
        }
      }
      repository.list.attachCreateCompleted(completed);
      try {
        context = repository.queueCreate(order);
        repository.submit().catch(function () { finish(true); });
      } catch (error) {
        finish(true);
      }
    });
  };
});
```

Do not discard or destroy the repository while that helper is pending; the owner must implement a corresponding cancellation policy if navigation can interrupt it. After a failed attempt, do not call `queueCreate` again for the same draft: correct the existing transient context and resubmit, or call `discard` and start over deliberately. The server must enforce uniqueness and authorize every write.

## Batches, pending changes, and concurrency

`$auto` is suitable for automatically grouped reads and, when chosen, updates. An application group such as `orderWrites` gives the application an explicit submission boundary. This grouping decision is separate from the business definition of atomicity. A batch can contain multiple requests and changesets; verify the actual service's transactional guarantees.

Before showing “saved,” inspect operation results and relevant model messages. `hasPendingChanges` is useful for navigation guards and retained failed changes, but it is not a universal substitute for operation outcomes: a read can fail without pending changes, and an operation may finish with no editable state remaining. Track the operations your user action actually initiated.

Let the OData model manage CSRF tokens and normal entity ETags. Avoid copying stale `__metadata` into a new payload or forcing wildcard concurrency to silence conflicts. A conflict means the user's draft and the server version need reconciliation. A refresh must not silently overwrite an unsaved draft.

Treat decimal quantities and money as strings at service boundaries when metadata requires it. Formatting does not perform authoritative accounting. Do not sum amounts from different currencies, round with `Math.round(value * 100)`, or assume all currencies have two minor-unit digits. The illustrative `Amount` field has an explicit two-digit scale; currency conversion and tax calculation remain server/domain responsibilities.

## Refresh without destroying the user's intent

A model refresh is a cache and request operation, not a universal repair button. Before refreshing an editor, decide how pending changes are handled. Refreshing a list after a successful write is different from replacing an unsaved draft with the latest server object. The first reconciles confirmed state; the second can discard user intent.

For a detail screen, bind through a stable entity key or an existing context and request only the properties the screen needs. Do not reconstruct identity from customer names or visible sort order. If a server-generated key replaces a transient creation identity, let the model maintain that transition and update application navigation only after persistence is confirmed.

Avoid competing copies with independent refresh schedules. If a dashboard uses a JSON projection while an editor uses OData, document when the dashboard becomes stale and how it is refreshed. Otherwise a successful Save can coexist with an apparently unchanged total, leading users to submit the same action again.

## Manual verification checklist

- Confirm the service metadata matches keys, nullability, decimal precision, and date types.
- Keep `"000042"` unchanged through reads, bindings, updates, and routes.
- Verify JSON draft edits do not alias persisted objects.
- Switch between German and an unsupported locale; inspect requests and fallback text.
- Confirm V2 requests use V2 methods and V4 requests use binding/context APIs.
- Queue a deferred write and verify it is not sent until explicit submission.
- Fail one inner batch request while the outer batch succeeds; the UI must not claim success.
- Reject a V4 create and verify a visible recoverable error rather than an endless success wait.
- Exercise cancel, retry, conflict, metadata failure, and navigation with pending changes.
- Verify owner destruction does not destroy models shared by other views.

These checks require a consuming application and live or controlled backend. No backend execution is implied by this reference.

## Official API references

Research access: the SDK host could not be resolved during research. API behavior was checked through SAP's official OpenUI5 GitHub sources instead. The [V2 model SDK reference](https://ui5.sap.com/#/api/sap.ui.model.odata.v2.ODataModel) and [V4 model SDK reference](https://ui5.sap.com/#/api/sap.ui.model.odata.v4.ODataModel) are navigation links, not claims of successful SDK access; select your deployed version when consulting them.

- [JSONModel public methods and observation caveats](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/json/JSONModel.js).
- [ResourceModel and bundle configuration](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/resource/ResourceModel.js).
- [V2 ODataModel CRUD, metadata, deferred groups, and submission](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js).
- [V4 ODataModel grouping and submitBatch](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js).
- [V4 list binding creation and createCompleted events](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataListBinding.js).
- [V4 Context property access, updates, deletion, and created promise](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/Context.js).
