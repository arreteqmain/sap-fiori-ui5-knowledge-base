# Services: promises, requests, cancellation, and safe errors

## Scope and prerequisites

The UI5 modules in this chapter target public APIs available in **UI5 1.120**, using a maintained patch or a verified newer runtime. They work in **OpenUI5 and SAPUI5** and require no SAPUI5-only service abstraction. Native `fetch`, `AbortController`, `URL`, and promises require a modern supported browser. A freestyle UI5 service module is simply a module with a clear contract; it is not a launchpad service, CAP service implementation, or dependency-injection framework.

Use namespace `com.acme.orders`, mapped to `webapp`. The OData examples assume an existing V2 model with accessible metadata and an authenticated same-origin endpoint exposing `OrderSet`. The fetch example deliberately targets a separate read-only REST endpoint `/api/orders`, not an OData endpoint. Its response contract is `{ "orders": [...] }`; do not use it to parse an OData V2 `d.results` envelope or a V4 `value` envelope without changing and documenting the adapter.

Each order has string `OrderID`, text `CustomerName`, decimal-string `Amount`, currency code, status code, and ISO calendar-date `OrderDate`. Leading zeros are significant. A service layer should never “clean” an order key by converting it to a number. Accounting calculations, currency conversion, and financial rounding are outside these transport adapters.

For binding-oriented operations see [models.md](./models.md), for callers and lifecycle see [controllers.md](./controllers.md), for screen integration see [views.md](./views.md), and for reusable value rules see [utilities.md](./utilities.md). The files occupy the requested paths in the repository [README](../README.md#repository-structure).

## Design the boundary around outcomes

A useful service contract tells its caller what can complete, fail, or be cancelled. It does not require the caller to inspect a vendor-specific response object or guess whether a promise represents queueing or persistence. A method named `queueUpdate` should not imply immediate submission. A method named `save` should document whether it includes validation, transport submission, individual operation results, and server reconciliation.

Keep services independent of controls. A service must not call `byId`, set a button busy, or open a MessageBox. The controller decides whether a failure is relevant to the current screen and how to present it. This separation is especially important when an old request finishes after navigation: the transport can report an outcome without opening a dialog on an unrelated page.

Do not make every model operation into an extra abstraction just for symmetry. A V4 context already exposes meaningful operations and promises. Wrap it when you need domain-specific orchestration, stable error mapping, or a shared request policy, not to recreate the whole model API with different names. Preserve the distinction between OData V2 and V4 instead of hiding incompatible semantics behind a misleading universal `crud` object.

## Safe error contract

This complete module belongs at `webapp/service/ServiceError.js`. It carries only a fixed code and optional HTTP status. It intentionally omits backend text, request bodies, arbitrary headers, and raw causes. Approved operational telemetry can record a separately vetted correlation identifier; that is not a reason to expose arbitrary backend details to the UI.

```javascript
sap.ui.define([], function () {
  "use strict";
  class ServiceError extends Error {
    constructor(code, status) {
      super(code);
      this.name = "ServiceError";
      this.code = code;
      this.status = Number(status) || 0;
    }
  }
  ServiceError.fromStatus = function (status) {
    const number = Number(status) || 0;
    const code = number === 401 ? "AUTH_REQUIRED" :
      number === 403 ? "FORBIDDEN" :
      number === 404 ? "NOT_FOUND" :
      number === 409 || number === 412 ? "CONFLICT" :
      number === 429 ? "RATE_LIMITED" : "REQUEST_FAILED";
    return new ServiceError(code, number);
  };
  return ServiceError;
});
```

The code is for application decisions, not for direct display. Map it to resource keys such as `authRequired` or `ordersLoadFailed`, then resolve them through the asynchronous i18n model. An unknown code must fall back to a generic translated message. Never display `error.message` merely because the error happens to inherit from Error; other code paths may supply a raw browser or backend exception.

Authentication expiry and authorization denial are distinct. A 401 may need a centrally managed sign-in flow. A 403 should not trigger endless reauthentication attempts or claims that the user can solve the problem by refreshing. Conflicts need reconciliation. Rate limits need a deliberate retry policy, not immediate repeated requests.

## Abortable V2 read adapter

Place this module at `webapp/service/readOrdersV2.js`. Its caller supplies an owned or shared `sap.ui.model.odata.v2.ODataModel` whose metadata has loaded. The adapter does not destroy that model. It returns both an abort function and a promise so cancellation cannot leave a promise permanently pending when a transport's abort behavior differs.

```javascript
sap.ui.define([
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "com/acme/orders/service/ServiceError"
], function (Filter, FilterOperator, ServiceError) {
  "use strict";

  return function readOrdersV2(model, query) {
    let request = null;
    let settled = false;
    let rejectPromise;
    const promise = new Promise(function (resolve, reject) {
      rejectPromise = reject;
      try {
        request = model.read("/OrderSet", {
          filters: query ? [
            new Filter("CustomerName", FilterOperator.Contains, query)
          ] : [],
          urlParameters: {
            "$select": "OrderID,CustomerName,Amount,Currency,Status,OrderDate",
            "$top": "50"
          },
          success: function (data) {
            if (settled) { return; }
            settled = true;
            resolve(data.results || []);
          },
          error: function (error) {
            if (settled) { return; }
            settled = true;
            reject(ServiceError.fromStatus(error.statusCode));
          }
        });
      } catch (error) {
        settled = true;
        reject(new ServiceError("REQUEST_FAILED"));
      }
    });
    return {
      promise: promise,
      abort: function () {
        if (settled) { return; }
        settled = true;
        rejectPromise(new ServiceError("CANCELLED"));
        if (request) { request.abort(); }
      }
    };
  };
});
```

The cancellation branch marks the operation settled before calling the transport's abort method. A synchronous abort callback therefore cannot replace the intended `CANCELLED` result with a generic failure. Calling abort after settlement is harmless. The caller must attach a rejection handler immediately, even if it expects to cancel the operation later.

The adapter uses a Filter object rather than concatenating user text into `$filter`. The OData model performs literal encoding and request construction. It still depends on the service supporting the chosen filter operation. Encoding is not authorization: the backend must enforce which entities and properties this user may read.

The fifty-row cap makes the returned array a limited search result, not a complete dataset. If the UI needs full pageable browsing, bind the list to the OData model instead of repeatedly copying capped arrays. If it needs export, define a separate export contract with bounded resource use and explicit paging.

This is a read adapter only. A V2 write wrapper must retain operation callbacks and batch submission outcomes; a V4 wrapper must use binding/context APIs and `submitBatch`. [models.md](./models.md#odata-v2-crud-and-explicit-submission) shows why an outer successful batch response is not a proof of successful business writes.

## Fetch adapter with timeout and validation

Place this independent REST adapter at `webapp/service/RestOrders.js`. It depends on the ServiceError module above. The server responds with JSON and caps its own results; the client additionally rejects an unexpectedly oversized array. A reverse proxy must configure response-size limits because parsing a huge JSON body already consumes memory before client-side validation runs.

```javascript
sap.ui.define([
  "com/acme/orders/service/ServiceError"
], function (ServiceError) {
  "use strict";

  function projectOrder(row) {
    if (!row || typeof row !== "object" ||
        typeof row.OrderID !== "string" ||
        !/^\d{6}$/.test(row.OrderID) ||
        typeof row.CustomerName !== "string" ||
        row.CustomerName.length > 80 ||
        typeof row.Amount !== "string" ||
        !/^-?\d{1,13}\.\d{2}$/.test(row.Amount) ||
        typeof row.Currency !== "string" ||
        !/^[A-Z]{3}$/.test(row.Currency) ||
        typeof row.Status !== "string" ||
        typeof row.OrderDate !== "string" ||
        !/^\d{4}-\d{2}-\d{2}$/.test(row.OrderDate)) {
      throw new ServiceError("INVALID_RESPONSE");
    }
    return {
      OrderID: row.OrderID,
      CustomerName: row.CustomerName,
      Amount: row.Amount,
      Currency: row.Currency,
      Status: row.Status,
      OrderDate: row.OrderDate
    };
  }

  async function search(query, options) {
    options = options || {};
    if (typeof query !== "string" || query.length > 80) {
      throw new ServiceError("INVALID_QUERY");
    }
    const controller = new AbortController();
    const signal = options.signal;
    let timedOut = false;
    const forwardAbort = function () { controller.abort(); };
    if (signal) {
      signal.addEventListener("abort", forwardAbort, { once: true });
      if (signal.aborted) { controller.abort(); }
    }
    const timer = setTimeout(function () {
      timedOut = true;
      controller.abort();
    }, 15000);
    try {
      const url = new URL("/api/orders", window.location.origin);
      url.searchParams.set("q", query);
      url.searchParams.set("limit", "50");
      const response = await fetch(url.toString(), {
        method: "GET",
        credentials: "same-origin",
        headers: { "Accept": "application/json" },
        signal: controller.signal
      });
      if (!response.ok) {
        throw ServiceError.fromStatus(response.status);
      }
      const contentType = response.headers.get("Content-Type") || "";
      if (!contentType.toLowerCase().includes("application/json")) {
        throw new ServiceError("INVALID_RESPONSE");
      }
      let body;
      try {
        body = await response.json();
      } catch (error) {
        if (controller.signal.aborted) { throw error; }
        throw new ServiceError("INVALID_RESPONSE");
      }
      if (!body || !Array.isArray(body.orders) ||
          body.orders.length > 50) {
        throw new ServiceError("INVALID_RESPONSE");
      }
      return body.orders.map(projectOrder);
    } catch (error) {
      if (controller.signal.aborted) {
        throw new ServiceError(timedOut ? "TIMEOUT" : "CANCELLED");
      }
      if (error instanceof ServiceError) { throw error; }
      throw new ServiceError("NETWORK_FAILED");
    } finally {
      clearTimeout(timer);
      if (signal) {
        signal.removeEventListener("abort", forwardAbort);
      }
    }
  }

  return { search: search };
});
```

The date check validates only the wire shape. Calendar validity and the business-allowed date range remain server responsibilities, and any editable client field needs the stricter date validation described in [utilities.md](./utilities.md). The projection prevents accidental propagation of unexpected fields into UI state. It is not an HTML sanitizer; render customer text through ordinary text properties, not unescaped HTML.

HTTP errors do not make fetch reject automatically. Check `response.ok` before parsing success data. A redirected login page may arrive as HTTP 200 with HTML, which is why content-type and shape checks matter. Production authentication should be designed so APIs provide a recognizable authentication outcome instead of silently returning a login document.

Timeout is a client waiting policy, not proof that the server did no work. That distinction becomes critical for POST or PATCH. This adapter performs GET only; adding writes requires CSRF protection, concurrency handling, idempotency decisions, and a backend-supported request contract. Do not manually fetch OData writes merely to avoid learning the model's grouping semantics.

## Integrating cancellation with controller state

The caller owns an AbortController for each active REST read. Before starting a new read, increment a request generation and abort the previous controller. Capture the generation in the new completion handlers. Update bound rows, busy flags, and error messages only when the controller remains alive and that generation is still current.

On controller exit, mark the controller inactive before aborting requests, clear any debounce timer, and detach non-view subscriptions. Cancellation can trigger rejection immediately. The rejection handler must recognize `CANCELLED` as an expected navigation or supersession outcome, not display a failure dialog.

Use the same guard in `finally` as in success and catch. An unconditional old `finally` that sets `/busy` to false can hide a newer in-flight request. Likewise, an unconditional cleanup assignment can erase the newer request handle, making exit cleanup ineffective. [controllers.md](./controllers.md#latest-request-wins-remote-controller) demonstrates this ordering with V2 callbacks.

Do not use cancellation as a write rollback mechanism. A user navigating away after Save may leave a request already accepted by the backend. Preserve an explicit “outcome unknown” state and reconcile when returning. For destructive actions, the service and user interaction need a policy more careful than “abort on exit.”

## Retry and observability policy

Retry only operations whose semantics permit it. An idempotent read can often be retried after a transient network failure with capped exponential backoff and jitter. A create needs a server-supported idempotency key or a reconciliation lookup. Merely retrying because the promise rejected can create duplicate orders after a timeout.

Respect server rate limits and Retry-After according to your deployed API policy. Do not retry authentication, validation, or authorization failures as if they were temporary network outages. Keep retry controls cancellable, including any scheduled timer, so a destroyed controller does not continue background traffic.

Log safe categories, operation names, elapsed durations, and approved opaque correlation identifiers. Do not log cookies, authorization headers, full URLs containing sensitive filters, complete order bodies, or raw backend response text. UI5 logging is not a secure storage system, and browser diagnostics may be visible to end users.

Error handling must preserve enough distinctions to support recovery without exposing details. “Conflict” can offer reload-and-review; “not found” can return to the list; “forbidden” can explain lack of access. A generic fallback is still necessary because networks, proxies, and unexpected server behavior cannot be exhaustively enumerated.

## Pitfalls and manual verification checklist

- Return valid JSON, malformed JSON, HTML, and an oversized array from the REST endpoint.
- Confirm identifiers retain leading zeros and amounts remain strings.
- Abort before a request starts, during transfer, and after completion.
- Verify timeout cleanup removes timers and external abort listeners.
- Reverse the completion order of two searches and check current-screen relevance.
- Return 401, 403, 404, 409, 412, 429, and 500; verify safe differentiated outcomes.
- Inspect all visible messages and diagnostic output for raw backend text or credentials.
- Fail an inner OData batch request while the outer request succeeds.
- Navigate away during reads and while a debounce timer is pending.
- Simulate an uncertain write outcome before defining a retry policy.

These are manual integration cases for a consuming application. They are not a claim of live endpoint execution.

## Official API references

Research access: the SDK host could not be resolved during research. API behavior was checked through SAP's official OpenUI5 GitHub sources instead. The [V2 request API reference](https://ui5.sap.com/#/api/sap.ui.model.odata.v2.ODataModel) is a navigation link, not a claim of successful SDK access; select your deployed version when consulting it.

- [V2 ODataModel read handles, errors, and CRUD semantics](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js).
- [V4 ODataModel application groups and submitBatch](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js).
- [V4 Context asynchronous access and write promises](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/Context.js).
- [Controller lifecycle and exit ownership](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/core/mvc/Controller.js).
