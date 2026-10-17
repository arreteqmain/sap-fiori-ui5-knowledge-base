# Routing, navigation, and reconstructible deep links

## Baseline and prerequisites

This chapter targets **SAPUI5/OpenUI5 1.120.x**, `sap.m.routing.Router`, asynchronous XML views, and a component-based application. The namespace is `demo.orders`, mapped to `webapp`. The runnable wiring example uses a `JSONModel` with numeric order identifiers. It does not assume an OData service, and its JSON paths must not be copied into OData requests.

Prerequisites are the MVC lifecycle described in [mvc-pattern.md](mvc-pattern.md) and component ownership from [component-architecture.md](component-architecture.md). Understand that the browser fragment, commonly called the hash, is separate from the HTTP pathname. In a launchpad, shell navigation and application-internal routing are also separate responsibilities.

A route is a rule for interpreting a URL location. A target is a declaration of what UI content to create and where to place it. A navigation action selects a named route and supplies arguments. Keeping these concepts separate makes direct startup, browser Back, and reuse substantially easier than manually replacing page controls after every click.

## Start with a URL contract

Choose a stable route vocabulary before coding handlers. In this example, the empty application hash shows the order list and `orders/101` shows one detail. The identifier is a durable business key, not a table index or a serialized binding context path. This distinction matters as soon as rows are filtered or the backend changes paging order.

A good deep link contains enough information to reconstruct meaningful state after a full reload. It should not require a user to have visited the list first. Optional query arguments can represent shareable filters or tabs; transient loading flags, dialog instances, and unsaved field contents usually should not be in the URL.

Do not put credentials, access tokens, or sensitive draft content in a hash. Fragments are not sent as ordinary HTTP request targets, but remain visible to browser history, page scripts, screenshots, and application telemetry. Treat route arguments as untrusted input and apply both syntactic validation and backend authorization.

## Manifest routes and targets

The following `webapp/manifest.json` defines three targets. The root view contains an `sap.m.App` with the local ID `app`; routed pages are inserted into its `pages` aggregation. A `bypassed` target handles hashes that match no route.

```json
{
  "_version": "1.58.0",
  "sap.app": {
    "id": "demo.orders",
    "type": "application",
    "title": "Order navigation"
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {} }
    },
    "rootView": {
      "viewName": "demo.orders.view.App",
      "type": "XML",
      "id": "root",
      "async": true
    },
    "routing": {
      "config": {
        "routerClass": "sap.m.routing.Router",
        "viewType": "XML",
        "viewPath": "demo.orders.view",
        "controlId": "app",
        "controlAggregation": "pages",
        "async": true,
        "bypassed": { "target": "notFound" }
      },
      "routes": [
        { "name": "list", "pattern": "", "target": "list" },
        {
          "name": "detail",
          "pattern": "orders/{orderId}",
          "target": "detail"
        }
      ],
      "targets": {
        "list": { "viewName": "List", "viewLevel": 1 },
        "detail": { "viewName": "Detail", "viewLevel": 2 },
        "notFound": { "viewName": "NotFound", "viewLevel": 1 }
      }
    }
  }
}
```

The routing configuration uses the supported 1.120 `viewPath`/`viewName` view-target vocabulary consistently. Do not mix examples using newer target declaration styles without checking that their fields exist on your deployed version. `viewLevel` assists mobile navigation direction; it is not an authorization rank or a browser history index.

Create `webapp/view/App.view.xml`:

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
  <App id="app"/>
</mvc:View>
```

The root view is not itself a routed page. It supplies the container into which targets place the pages. If `controlId` does not resolve in the expected root-view scope, routes can match while no page appears. Debug placement and matching separately rather than repeatedly changing route patterns to solve a container lookup problem.

## Initialize models before processing the current hash

Create `webapp/Component.js`:

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
        orderList: [
          { id: "101", customer: "Ada" },
          { id: "102", customer: "Linus" }
        ],
        ordersById: {
          "101": { id: "101", customer: "Ada", status: "Open" },
          "102": { id: "102", customer: "Linus", status: "Approved" }
        }
      }), "orders");
      this.getRouter().initialize();
    }
  });
});
```

The local lookup structure makes direct detail startup independent of list selection. Both structures are static sample data; a production application should avoid maintaining inconsistent duplicate copies of mutable entities. A remote detail view should load or bind the requested entity even when the list cache is empty.

`initialize` connects routing to the current location. Call it once during normal component startup, not after every button press. The superclass must establish manifest infrastructure first. The asynchronous target configuration accommodates asynchronous view creation; it does not make arbitrary model loading finish before a route handler executes.

The standalone bootstrap can use the same `ComponentSupport` arrangement described in [mvc-pattern.md](mvc-pattern.md). A host shell may create the component instead. In either case, the route handler must tolerate the initial location already being a detail route when the component starts.

## Navigate using a named route

Create `webapp/view/List.view.xml`:

```xml
<mvc:View
  controllerName="demo.orders.controller.List"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m">
  <Page title="Orders">
    <content>
      <List items="{orders>/orderList}">
        <StandardListItem
          title="{orders>id}"
          description="{orders>customer}"
          type="Navigation"
          press=".onOpen"/>
      </List>
    </content>
  </Page>
</mvc:View>
```

Create `webapp/controller/List.controller.js`:

```javascript
sap.ui.define(["sap/ui/core/mvc/Controller"], function (Controller) {
  "use strict";
  return Controller.extend("demo.orders.controller.List", {
    onOpen: function (event) {
      var context = event.getSource().getBindingContext("orders");
      this.getOwnerComponent().getRouter().navTo("detail", {
        orderId: context.getProperty("id")
      });
    }
  });
});
```

`navTo` centralizes pattern construction. Renaming a route pattern should not require changing a dozen string-concatenated hash assignments. The event source is the pressed list item, so its named binding context identifies the selected row. Do not take the item’s rendered position as its identity.

These example keys contain only digits. For general keys, define a deliberate route encoding contract. The router’s parameter encoding preserves some reserved characters according to its API; applications with keys containing reserved separators may need explicit encoding and corresponding decoding. Do not blindly add `encodeURIComponent` or `decodeURIComponent` at multiple layers. Test literal percent signs, slashes, Unicode, question marks, and already encoded strings against the chosen contract.

## Activate detail state on every route match

Create `webapp/view/Detail.view.xml`:

```xml
<mvc:View
  controllerName="demo.orders.controller.Detail"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m">
  <Page title="Order"
    showNavButton="true"
    navButtonPress=".onBack">
    <content>
      <VBox class="sapUiSmallMargin">
        <Title text="{orders>id}"/>
        <Text text="{orders>customer}"/>
        <ObjectStatus text="{orders>status}"/>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

The detail controller binds the whole view to one JSON entity. It validates the identifier before using it as a local JSON path segment:

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/core/routing/History"
], function (Controller, History) {
  "use strict";
  return Controller.extend("demo.orders.controller.Detail", {
    onInit: function () {
      this._route = this.getOwnerComponent().getRouter().getRoute("detail");
      this._route.attachPatternMatched(this._onMatched, this);
    },

    _onMatched: function (event) {
      var id = event.getParameter("arguments").orderId;
      var view = this.getView();
      view.unbindElement("orders");
      if (!/^[0-9]{1,12}$/.test(id)) {
        this._showMissing();
        return;
      }
      var path = "/ordersById/" + id;
      if (!view.getModel("orders").getProperty(path)) {
        this._showMissing();
        return;
      }
      view.bindElement({ path: path, model: "orders" });
    },

    _showMissing: function () {
      this.getOwnerComponent().getRouter()
        .getTargets().display("notFound");
    },

    onBack: function () {
      if (History.getInstance().getPreviousHash() !== undefined) {
        window.history.go(-1);
      } else {
        this.getOwnerComponent().getRouter().navTo("list", {}, true);
      }
    },

    onExit: function () {
      this._route.detachPatternMatched(this._onMatched, this);
    }
  });
});
```

The only concatenated data path here belongs to a local JSON model and follows a strict digit-only check. It is not an OData URL construction technique. With OData V2 use metadata-aware key construction such as `createKey`; with OData V4 prefer existing binding contexts or documented metadata-aware key utilities. Never concatenate raw route input into an OData entity path or `$filter` expression.

The previous binding is removed before validation so a missing or malformed identifier cannot leave the previous customer’s details visible. This is especially important with cached views. `patternMatched` runs for each matching detail navigation; `onInit` is only the place where the handler is installed.

Create `webapp/view/NotFound.view.xml`:

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
  <MessagePage title="Not found"
    text="This order or location is unavailable."
    showNavButton="false"/>
</mvc:View>
```

Displaying a target directly does not rewrite the hash. For an unavailable order, this example preserves the requested URL so refresh remains diagnostic and the location can be corrected. An application can instead navigate to a dedicated error route when that is its intended URL contract.

## Separate unmatched locations from missing data

An unmatched hash never selects the detail route; the router’s `bypassed` configuration displays `notFound`. A syntactically valid detail hash can still identify a missing order. That is a data resolution problem handled after matching. These cases may share a presentation but should remain distinct in diagnostics and telemetry.

A remote service adds more outcomes: unauthorized access, temporary transport failure, malformed response, or a deleted entity. Do not label every request failure “not found.” An authorization-aware application may intentionally use a neutral message to avoid disclosing entity existence, while still recording the correct internal classification.

Metadata and entity data arrive asynchronously. A route can select a binding before the entity has loaded. Observe documented binding events and contexts rather than testing an immediate cached object and treating `undefined` as proof of absence. [data-binding.md](data-binding.md) explains why V2 and V4 require different request and completion APIs.

## Browser history and replacement navigation

The third argument in `navTo("list", {}, true)` replaces the current history entry. Use replacement when correcting a location, implementing a fallback after direct startup, or avoiding a dead-end intermediate state. Use ordinary navigation when the user should be able to return with Back.

The history check above is suitable for this small standalone application. A shell-hosted application may require shell-provided back navigation rather than assuming application history represents the entire user journey. Even within a standalone app, decide whether returning to the previous hash is semantically appropriate after destructive actions or discarded drafts.

Do not use target display and hash routing interchangeably. Calling `display` changes visible content without adding a shareable route state. Calling `navTo` expresses an intended location and lets routing select targets. Direct display is useful for auxiliary or error presentation, but primary user journeys should normally remain reconstructible from the location.

Unsaved edits require a policy before navigation occurs. A warning shown after the hash and context have already changed can be too late. Coordinate confirmation, pending updates, and the chosen navigation action. Browser unload protection, application-internal navigation, and shell navigation are related but separate mechanisms and must be verified in the actual hosting environment.

## Query arguments, optional state, and precedence

UI5 patterns support mandatory parameters, optional parameters, query parameters, and rest parameters. Introduce only the forms your application needs. A route with a query declaration can represent a selected tab or filter without creating a distinct path for each combination. Supply and read query arguments according to the documented router contract.

Normalize optional state at the route boundary. Unknown tabs should select a documented default or an error state, not reach arbitrary controller method names. A string query parameter is not automatically a boolean merely because it looks like `false`. Parse and validate values explicitly and preserve only supported state when generating links.

Order overlapping routes carefully. A broad pattern can capture locations intended for a more specific route. Do not enable greedy matching merely to compensate for unclear patterns: multiple matches can activate more handlers than expected. Use route-specific events for detail work and understand parent matching when adding nested routes.

Avoid a rest parameter that carries an opaque serialized model path. Such links expose implementation details, couple URLs to a particular model version, and invite unsafe path construction. Prefer stable domain identifiers and a small, documented vocabulary of optional state.

## Targets, caching, and nested layouts

Targets encapsulate view creation and placement. A target name is not necessarily a route name, and several routes may intentionally reuse one target. The target/view machinery commonly reuses views, which is why controller initialization and route activation must remain separate.

For a split or flexible-column layout, multiple targets can represent different areas. Define their container IDs and aggregations explicitly and decide which route states populate each area. A detail deep link must reconstruct required parent content as well as the final pane; it cannot assume a previous list route created those controls.

Nested components introduce another ownership layer. Use component targets and documented route prefix arrangements when independent child routing is needed. Do not start multiple unrelated routers against the same hash and hope registration order will arbitrate. [component-architecture.md](component-architecture.md) discusses when a child actually needs a router.

## Route transitions are state transitions

Treat navigation as more than a hash mutation. A transition can invalidate a previous selection, begin a data request, update a title, and change available actions. Define the sequence so the user never sees controls enabled for the old entity while the new entity is loading. Clearing the old binding in the example is one small part of that policy.

With remote data, keep the requested identifier separate from the resolved entity state. The URL can already name order 102 while the last completed request represented order 101. A loading state should make that distinction visible, and an obsolete response must not restore order 101 after the user has moved on.

A generation counter can guard imperative requests: increment it for each route activation, capture the current value when starting work, and ignore completion if the value changed. Binding-driven solutions should similarly verify the current context or use the binding’s documented lifecycle. The important property is that completion belongs to the current navigation, not merely to a still-alive controller.

## Canonical links and shareable presentation state

Define one canonical representation for equivalent locations. If tabs are case-sensitive internally, normalize supported external values deliberately. If a default tab is omitted from the URL, generate links consistently rather than alternating between two equivalent forms. Canonicalization improves support conversations, analytics, and the predictability of browser history.

Only normalize information that the application understands. Blindly rewriting an unfamiliar identifier can accidentally refer to a different entity. Avoid decoding arbitrary input repeatedly in an attempt to make malformed links work. A malformed escape sequence should produce a controlled error, not an exception that leaves the navigation container half updated.

Shareable filters need a bounded schema. A route query can identify a supported status and search term, but should not contain executable filter syntax or unrestricted backend property names. Restore values into ordinary filter controls, validate them, and construct typed model filters. The route is a public state description, not a tunnel for arbitrary query execution.

When generating a link for another application, use the host environment’s supported cross-application navigation service where available. Internal `navTo` knows this component’s route names; it does not know shell semantic objects, external application intents, or the deployment host’s URL policy. Keeping these layers separate avoids hard-coded launchpad URLs in reusable controllers.

## Titles, focus, and navigation feedback

A matched route should lead to understandable navigation feedback. Update the visible page title and any application-managed document title according to the resolved state. Do not show a sensitive entity name in a title before authorization and data resolution have established that it may be displayed.

Focus behavior also belongs in the navigation design. Mobile navigation containers provide their own transition behavior, while complex desktop layouts may require a deliberate focus destination after content becomes available. Avoid stealing focus whenever a background binding update occurs. A route change and a minor data refresh are not the same user interaction.

For an unavailable detail link, provide a usable recovery path in the actual application. This chapter’s minimal message page intentionally has no navigation handler, but a production design can offer a list action, shell Back, or a retry depending on the cause. Preserve the original requested identifier when retrying a transport failure rather than silently navigating to a different entity.

## Distinguish routing errors from deployment errors

An asynchronous target can fail because its XML or controller resource is unavailable even though the route pattern is correct. Inspect module requests and resource-root mappings before changing the route contract. A case mismatch in `Detail.view.xml` may work on one workstation and fail only after deployment to a case-sensitive server.

Cold-start testing should include the actual deployed path. Hash routing normally keeps the fragment client-side, but the server must still serve the application document and resources at that pathname. A reverse proxy or application router can break startup before UI5 ever processes the hash. Such a failure is not solved by adding another `bypassed` target.

Keep diagnostic categories separate: no matching route, target creation failure, invalid route argument, missing entity, and transport failure. They can share a user-facing recovery design while retaining distinct technical causes. That separation gives support teams useful evidence without requiring raw backend response bodies in visible messages.

## Verification checklist

- Open the empty hash and verify the list appears in the root `App`.
- Open `#/orders/101` in a fresh tab without visiting the list first.
- Navigate repeatedly between orders and verify cached detail content changes correctly.
- Open an invalid identifier, an unknown valid identifier, and a completely unmatched hash.
- Verify a missing entity never leaves stale details from a previous route visible.
- Exercise browser Back and the detail back button after both direct startup and list navigation.
- Verify replacement navigation does not create a Back loop.
- For broader identifier contracts, test reserved characters and avoid double encoding.
- Simulate slow or failed data resolution and distinguish unavailable data from unmatched routes.
- In a shell, verify shell/application hash coordination rather than assuming standalone behavior.

## Official sources

- [Router public API and route configuration, pinned to 1.120.0](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/routing/Router.js), including `navTo` and target integration.
- [UIComponent routing metadata and initialization infrastructure](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/UIComponent.js).
- [Versioned SDK Route reference](https://ui5.sap.com/1.120.0/#/api/sap.ui.core.routing.Route), for pattern syntax and matching events.
- [Versioned SDK mobile Router reference](https://ui5.sap.com/1.120.0/#/api/sap.m.routing.Router), for navigation-container integration.
- [Versioned SDK History reference](https://ui5.sap.com/1.120.0/#/api/sap.ui.core.routing.History), for previous-hash semantics.
