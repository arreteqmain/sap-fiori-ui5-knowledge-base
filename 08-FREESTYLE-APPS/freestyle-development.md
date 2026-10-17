# Freestyle SAPUI5 Development

## Scope and prerequisites

This chapter explains how to choose and structure a freestyle application, not how to reproduce every framework feature manually. The running example is a maintenance work queue: technicians find work orders, inspect one order, and navigate to a separate edit experience. The application owns its interaction design; an OData service owns business rules and persistence. Freestyle development is appropriate when that ownership is intentional and funded throughout the application's lifetime.

Examples assume SAPUI5 or OpenUI5 **1.120**, asynchronous modules, XML views, JavaScript controllers, and the `sap.ui.core`, `sap.m`, and `sap.f` libraries. The service example uses the UI5 **OData V4 model against an OData V4 service**. Identifiers, routes, entity sets, and service URLs are illustrative contracts, not standard SAP APIs. A production project must select a supported patch and verify its deployment platform's library availability. A minimum version in a descriptor does not download or pin that runtime.

Readers should understand binding contexts, component ownership, promises, and basic OData metadata. Read [project setup](../01-FOUNDATIONS/project-setup.md), [MVC](../02-ARCHITECTURE-PATTERNS/mvc-pattern.md), and [data binding](../02-ARCHITECTURE-PATTERNS/data-binding.md) first. SAP's official [app development guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/app-development-b1fbe1a.md) explains the framework/toolchain boundary. This chapter's organizational recommendations are application architecture choices rather than additional framework requirements.

## Decide whether freestyle is justified

Start with a representative user task and real service metadata. A requirement saying “custom UI” is insufficient: a standard list report with different labels might only require annotations, configuration, or documented extensions. Conversely, forcing an equipment-dispatch board into a generated table can make the application harder to operate and upgrade than a deliberately designed freestyle screen.

Fiori elements supplies floorplans and behaviors driven by metadata, annotations, and supported extension points. Freestyle supplies controls and framework services, while the application implements the page composition and operational behavior. Neither approach creates missing backend authorization, atomic transactions, or good service semantics. Evaluate the actual release's Fiori elements capabilities before selecting an approach, especially where flexible programming features or supported custom sections can cover the unusual part.

| Use case | First candidate | Evidence needed before choosing freestyle |
| --- | --- | --- |
| Search, filter, inspect, and edit ordinary business objects | Fiori elements list report and object page | A tested requirement that supported configuration or extensions cannot satisfy |
| Annotation-rich analytical overview | Fiori elements analytical floorplan | Interaction or visualization needs outside the supported floorplan |
| Scan, validate, and confirm warehouse steps | Freestyle task flow | Device input, latency, error recovery, and keyboard behavior are understood |
| Dispatcher assigns jobs across a time/resource board | Freestyle composition | Board interaction is genuinely central, with an accessible equivalent |
| Standard object page with one specialized calculation section | Fiori elements with documented extension | Extension lifecycle and service contract are sufficient |
| Multi-system exception cockpit | Freestyle, or a composed shell of applications | Identity, ownership, and cross-system consistency are explicitly designed |
| Simple form over a weak, undocumented service | Repair service first | Freestyle cannot compensate for missing business operations |

Record the decision as a short architecture decision: the task, rejected alternatives, target runtime, service assumptions, accessibility consequences, and maintenance cost. Include the cost of dirty-state protection, message navigation, variants, accessibility verification, and integration testing. These are often invisible benefits of generated floorplans that must be supplied consciously in freestyle applications.

Avoid an all-or-nothing decision across a product portfolio. One standard purchasing object can use Fiori elements while a scanning task uses freestyle and navigates to that object. Keep semantic navigation and business identifiers consistent. Do not embed a generated application and manipulate its private control tree to simulate an extension; supported public extension APIs are the boundary.

## Design the task before selecting controls

Describe the user's starting information, desired outcome, and likely interruptions. For a technician, the initial question may be “What can I finish at this location?” rather than “Show all work orders.” That changes default filters, grouping, primary actions, and empty-state text. Document whether work can be interrupted safely and whether a saved object is different from a posted or released object.

Sketch the narrowest supported viewport before polishing the desktop layout. A three-column desktop display becomes a sequence of screens on a phone; the task must remain understandable without simultaneous visibility. Keep the object identity visible while editing. Reserve explicit space for business messages and unavailable-data states rather than designing only the successful loaded page.

Use actual SAP Fiori design guidance alongside control APIs. The official [FlexibleColumnLayout usage documentation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.f/src/sap/f/FlexibleColumnLayout.js) describes the list-detail-detail paradigm and its design intent. The [DynamicPage documentation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.f/src/sap/f/DynamicPage.js) describes title, collapsible header, content, and footer responsibilities. These are useful SDK-backed design sources when reviewing a composition; they are not permission to put every application into three columns.

## Select controls by interaction and data volume

Choose a control because its behavior matches the task, not because a screenshot resembles the mockup. Changing a table class late affects selection, navigation, keyboard behavior, and binding strategy. Prototype the largest realistic dataset, translated text, and at least one assistive-technology path before committing to the layout.

| Need | Candidate | Decision constraint |
| --- | --- | --- |
| Short queue with varied item summaries | `sap.m.List` with `ObjectListItem` | Keep the primary identity and status easy to scan |
| Responsive tabular comparison | `sap.m.Table` | Prioritize columns and define pop-in behavior |
| Dense desktop row/column work | `sap.ui.table.Table` | Add `sap.ui.table`; verify model and device support |
| Persistent title and collapsible filters | `sap.f.DynamicPage` | One clear scrolling strategy and meaningful snapped title |
| Related list and detail levels | `sap.f.FlexibleColumnLayout` | Model layout in routing and verify phone behavior |
| Simple labeled edit fields | `sap.ui.layout.form.SimpleForm` | Declare `sap.ui.layout`; maintain labels and responsive layout |
| Small set of fixed choices | `sap.m.Select` or `RadioButtonGroup` | Choice count and comparison needs determine the control |
| Large business value domain | Input plus value help | Search remotely and return stable keys, not display labels |
| Critical multi-field business operation | Dedicated page or dialog | Preserve context, validation, and recoverable input |

The official [`sap.m.Table` documentation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.m/src/sap/m/Table.js) requires column order to match cell order and warns about mobile row volume. Growing helps incremental rendering; it does not make thousands of complex controls inexpensive. Desktop row virtualization and responsive table growing solve different problems. Neither is a substitute for selective server queries.

Prefer an existing control's public properties and events over inheritance. Use a fragment for a repeated application-specific composition. Create a reusable control only when a stable public contract and independent behavior justify it; see [custom controls](custom-controls.md). A formatter that returns HTML is not a lightweight alternative to a control.

## Declare a coherent application contract

The following is a **merge excerpt** for `webapp/manifest.json`, not a complete descriptor. It assumes component namespace `demo.workorders`, root view `demo.workorders.view.App`, a root `sap.m.App` with local ID `app`, and views named `Worklist` and `Detail`. Resource bundle keys belong in `webapp/i18n/i18n.properties`. The component must initialize its router once after the base `UIComponent.init`.

```json
{
  "sap.app": {
    "id": "demo.workorders",
    "type": "application",
    "dataSources": {
      "mainService": {
        "uri": "/odata/v4/maintenance/",
        "type": "OData",
        "settings": { "odataVersion": "4.0" }
      }
    }
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {}, "sap.f": {} }
    },
    "rootView": {
      "viewName": "demo.workorders.view.App",
      "type": "XML",
      "id": "root",
      "async": true
    },
    "models": {
      "": {
        "dataSource": "mainService",
        "type": "sap.ui.model.odata.v4.ODataModel",
        "settings": {
          "synchronizationMode": "None",
          "operationMode": "Server",
          "autoExpandSelect": true,
          "groupId": "$auto",
          "updateGroupId": "workOrderEdit",
          "groupProperties": {
            "workOrderEdit": { "submit": "API" }
          }
        }
      },
      "i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "settings": { "bundleName": "demo.workorders.i18n.i18n" }
      }
    },
    "routing": {
      "config": {
        "routerClass": "sap.m.routing.Router",
        "viewType": "XML",
        "viewPath": "demo.workorders.view",
        "controlId": "app",
        "controlAggregation": "pages",
        "async": true
      },
      "routes": [
        { "name": "worklist", "pattern": "", "target": "worklist" },
        { "name": "detail", "pattern": "orders/{id}", "target": "detail" }
      ],
      "targets": {
        "worklist": { "viewName": "Worklist", "viewId": "worklist" },
        "detail": { "viewName": "Detail", "viewId": "detail" }
      }
    }
  }
}
```

The service URL is same-origin and should be routed through the deployment platform's approved destination or reverse proxy. Never put credentials into the manifest. The update group is application-defined and submits explicitly; it is not a database transaction configuration. All edits using that group share its submission boundary, so do not reuse it indiscriminately for independent open editors.

The official [V4 model source and API comments](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js) document group configuration and supported parameters. The similarly named [V2 model](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js) has different APIs and behavior. Copying `submitChanges` or `read` into this V4 setup is incorrect; V4 primarily works through bindings and contexts.

## Compose a worklist with explicit presentation state

This `Worklist.view.xml` uses an OData collection for business data and a named `ui` JSON model for presentation state. It assumes `WorkOrders` exposes `ID`, `Description`, and `Status`, with `ID` an `Edm.Guid`. Translation keys must be supplied for every `i18n` binding. A string such as “Released” is displayed as service text here; production services should define localized texts or the application should map documented codes.

```xml
<mvc:View
  controllerName="demo.workorders.controller.Worklist"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  xmlns:f="sap.f">
  <f:DynamicPage id="queuePage">
    <f:title>
      <f:DynamicPageTitle>
        <f:heading>
          <Title text="{i18n>workQueueTitle}" />
        </f:heading>
      </f:DynamicPageTitle>
    </f:title>
    <f:content>
      <VBox>
        <SearchField
          id="queueSearch"
          placeholder="{i18n>searchDescription}"
          search=".onSearch" />
        <List
            id="orders"
            busy="{ui>/loading}"
            busyIndicatorDelay="0"
            noDataText="{i18n>noMatchingOrders}"
            growing="true"
            growingThreshold="30"
            items="{
              path: '/WorkOrders',
              parameters: { $select: 'ID,Description,Status' },
              events: {
                dataRequested: '.onDataRequested',
                dataReceived: '.onDataReceived'
              }
            }">
          <ObjectListItem
              title="{Description}"
              intro="{ID}"
              type="Navigation"
              press=".onOpenOrder">
            <firstStatus>
              <ObjectStatus text="{Status}" />
            </firstStatus>
          </ObjectListItem>
        </List>
      </VBox>
    </f:content>
  </f:DynamicPage>
</mvc:View>
```

The corresponding controller deliberately uses the `search` event rather than issuing a backend request for every keystroke. Its filter applies to the bound collection; it does not copy service data into a parallel JSON array. The request events own loading state. Transport failures should additionally feed the application's persistent message handling, as described in [operational applications](operational-apps.md).

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "sap/ui/model/FilterType"
], function (Controller, JSONModel, Filter, FilterOperator, FilterType) {
  "use strict";

  return Controller.extend("demo.workorders.controller.Worklist", {
    onInit: function () {
      this.getView().setModel(new JSONModel({ loading: false }), "ui");
    },

    onSearch: function (oEvent) {
      const sQuery = (oEvent.getParameter("query") || "").trim();
      const aFilters = sQuery
        ? [new Filter("Description", FilterOperator.Contains, sQuery)]
        : [];
      this.byId("orders").getBinding("items")
        .filter(aFilters, FilterType.Application);
    },

    onDataRequested: function () {
      this.getView().getModel("ui").setProperty("/loading", true);
    },

    onDataReceived: function () {
      this.getView().getModel("ui").setProperty("/loading", false);
    },

    onOpenOrder: function (oEvent) {
      const oContext = oEvent.getSource().getBindingContext();
      this.getOwnerComponent().getRouter().navTo("detail", {
        id: oContext.getProperty("ID")
      });
    }
  });
});
```

For simultaneous independent requests, use separate region flags or a request counter rather than allowing one completion to hide another request's busy state. An empty result is not a failed request: preserve different messages and recovery actions. Do not show “no orders” while the first request is still loading.

## Route by identity, bind by context

A route is a durable description of the selected business object. A list index is not: sorting, filtering, server paging, and concurrent updates all change indexes. Use a stable key and validate route input before constructing a binding path. In this example the service declares a single GUID key, so the V4 key predicate is unquoted. A string key would require OData string-literal escaping, not this construction.

```javascript
// Methods inside demo.workorders.controller.Detail.
onInit: function () {
  this._detailRoute = this.getOwnerComponent().getRouter().getRoute("detail");
  this._detailRoute.attachPatternMatched(this._onMatched, this);
},

_onMatched: function (oEvent) {
  const sId = oEvent.getParameter("arguments").id;
  const rGuid = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (!rGuid.test(sId)) {
    this.getView().unbindElement();
    this.getOwnerComponent().getRouter().navTo("worklist", {}, true);
    return;
  }
  this.getView().bindElement({
    path: "/WorkOrders(" + sId + ")",
    parameters: { $select: "ID,Description,Status" }
  });
},

onExit: function () {
  this._detailRoute.detachPatternMatched(this._onMatched, this);
}
```

This is a read-only route skeleton. A complete detail screen must distinguish malformed keys, not-found responses, authorization failures, and temporary network failures. It must also prevent a previous object's messages from being displayed as if they belong to the new object. Do not redirect every failure silently to the list; a user who followed a bookmark needs an explanation and an intentional recovery path.

Only put safe navigation state into the URL. Access tokens, confidential notes, and full filter payloads do not belong there. A route key is an identifier, not authorization evidence. The service must check access even when the key came from a previously authorized list.

## Separate business data from UI state

Business data includes quantities, statuses, organizational assignments, and values that the service persists or computes. UI state includes which panel is open, whether a region is loading, and which draft interaction step is active. Store these in separate named models or controller-owned state with explicit ownership. Do not add `/isEditing` to a business entity simply because binding makes it convenient.

There are three further categories worth distinguishing. Route state is bookmarkable identity and layout. Personalization is a user's durable preference such as table columns. Edit state is pending input that may or may not already exist in the backend. A single global JSON model that mixes all four categories makes navigation and reset behavior unpredictable.

The application should document which component owns each state value and when it is cleared. A component-level model is appropriate for a shared selection summary; a view-level model is usually better for a dialog's busy flag. Avoid module-level mutable variables because multiple component instances can coexist in the same launchpad session.

## Treat editing as a contract, not a button toggle

Before implementing Save, choose between service-bound edits and a detached edit buffer. Service-bound edits use typed OData bindings and pending changes; a detached buffer makes cancellation easy but requires explicit serialization, validation, concurrency handling, and message mapping. Mixing both strategies on the same fields creates two competing truths.

The backend owns atomicity and state transitions. Submitting a batch does not establish a business transaction spanning arbitrary services, nor does an enabled Save button prove the operation is authorized. Draft activation, release, goods issue, and financial posting are different operations with different consequences. See [transactional patterns](../03-UI-PATTERNS/transactional-patterns.md) and the detailed V2/V4 distinctions in [operational applications](operational-apps.md).

For navigation away from pending changes, use an explicit application policy: save, discard, or stay. Do not reset all model changes merely because one page closes when other pages share the model. Test a user opening a deep link while an edit is pending, not only clicking the application Back button.

## Organize modules for change

Keep views declarative and controllers focused on interaction coordination. Put pure formatting in formatter modules, reusable business-operation adapters in service modules, and genuinely reusable widgets in a control namespace. Do not create an abstract base controller containing every application's service and layout decision; such inheritance hides dependencies and makes isolated testing difficult.

An effective feature folder contains its view, controller, fragments, and focused helpers. Shared models and service adapters should have clear contracts rather than becoming a miscellaneous directory. Stable view IDs improve test selectors and adaptation support. When creating a fragment asynchronously, cache its promise and let the owning view manage its destruction.

The official [performance guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/performance-speed-up-your-app-408b40e.md) recommends asynchronous loading and manifest-declared dependencies. A controller should not load libraries synchronously during a button event. Nor should it fetch data in a formatter, which can execute repeatedly during binding updates. Measure startup, first useful data, and task completion separately.

## Common implementation traps

Avoid copying the entire service cache into a JSON model to simplify display: doing so loses OData types, server paging, and automatic message targets. Avoid binding one object to several independent editors without defining their update groups and cancellation ownership. Avoid style rules that target internal UI5 IDs or undocumented classes; a patch can change those details without breaking the public API.

Do not use a success toast as the only feedback for a failed or partial operation. Do not make every action a confirmation dialog; reserve confirmation for consequential or ambiguous actions. Do not hide all errors behind a generic busy overlay, and do not let a long-running export block unrelated navigation. Accessibility, responsiveness, and recovery are part of the interaction contract, not a final cosmetic pass.

## Review release readiness with the service owner

Agree on service error formats, maximum supported result sizes, expected latency, and backward compatibility before deployment. A frontend can be technically compatible with UI5 while depending on a service field that was renamed during a backend transport. Keep a small contract inventory for fields used in routes, filters, object identity, and business actions.

Review deployment assumptions separately from local development. Relative service URLs, launchpad navigation, authentication redirects, and content security policy can behave differently behind the production proxy. The initial application route should work when opened directly, not only after following a development server's landing page. These checks belong to application integration, even though no custom control or controller method implements them alone.

## Manual verification checklist

- Confirm the chosen floorplan against the decision matrix and document why supported Fiori elements options were insufficient.
- Open the worklist, inspect filtering requests, and verify server paging with realistic volume.
- Open a detail URL directly; test valid, malformed, missing, and unauthorized objects.
- Verify request failures differ from empty results and that busy state always clears.
- Navigate entirely by keyboard and check meaningful labels at narrow width and browser zoom.
- Check that route handlers, timers, dialogs, and subscriptions are released by their owner.
- Verify library declarations and the deployed runtime version rather than assuming the descriptor pins them.
- Review Save, Cancel, and navigation against the backend transaction contract.

## Related chapters and source boundaries

Continue with [operational applications](operational-apps.md), [custom controls](custom-controls.md), and [freestyle patterns](freestyle-patterns.md). Repository-wide companions are [routing and navigation](../02-ARCHITECTURE-PATTERNS/routing-navigation.md), [accessibility](../06-BEST-PRACTICES/accessibility.md), and [performance optimization](../06-BEST-PRACTICES/performance-optimization.md). These paths match the repository README's requested structure; they identify companion chapters even when a partial checkout has not populated them yet.

The linked SAP documentation on the moving `main` branch supplies conceptual guidance. Version-tagged OpenUI5 sources supply the concrete API baseline used here. Where a newer conceptual example mentions APIs introduced after 1.120, verify availability before adopting it; the examples in this chapter intentionally stay within the stated baseline.

Research used the official SAP GitHub documentation and OpenUI5 source files cited above. Direct SDK and Fiori design hosts were unavailable during preparation; this chapter does not claim to have inspected their live pages. The [UI5 SDK](https://ui5.sap.com/) remains the deployment-version reference, not evidence of executed application checks.
