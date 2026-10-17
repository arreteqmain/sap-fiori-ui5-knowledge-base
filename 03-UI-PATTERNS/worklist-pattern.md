# Worklist pattern with `sap.m.List`

## Scope and prerequisites

A worklist helps a person find the next business object to process. Its organizing question is “What needs my attention?” rather than “How can I analyze every record?” Typical examples include invoices awaiting clarification, maintenance notifications assigned to a technician, and purchase requisitions awaiting approval. A useful worklist combines a compact summary, understandable filtering, predictable ordering, selection when an action requires it, and direct navigation into an object.

This guide describes a **freestyle OpenUI5 application** using `sap.m.List`. It is not a Fiori elements List Report implementation. The controls and binding utilities in the executable examples belong to OpenUI5: `sap.m`, `sap.ui.core`, and the model packages. Declare application libraries in `manifest.json`, load the view through an MVC component, and provide an i18n resource model. The example strings are literal English to make the interaction visible; production applications should externalize them.

The first implementation uses a named JSON model, `work`, containing a bounded, already authorized collection. Its schema is `/WorkItems`, with `ID`, `Title`, `Status`, `Priority`, and `Owner` properties. A separate JSON model, `ui`, owns transient screen state. Later sections explain the different requirements of an OData V4 binding. No example assumes that client filtering enforces authorization, that growing provides offline storage, or that selecting an item locks the underlying business object.

Related guides listed in the [repository README](../README.md) are the [list report](./list-report-pattern.md), [object page](./object-page-pattern.md), [dashboard](./overview-dashboard.md), and [transactional patterns](./transactional-patterns.md).

## Decide whether a list is the right control

Choose `sap.m.List` when each row has a compact narrative and a single obvious destination. Titles, status text, attributes, and an optional number work well. Choose `sap.m.Table` when users must compare a small set of columns on different devices. Choose `sap.ui.table.Table` for dense desktop analysis, column menus, and many repeated business fields; it is not a phone-oriented substitute for a responsive list.

A worklist can contain a table, and a list report can contain a responsive table. These pattern names describe tasks, not mandatory control classes. Avoid presenting an approval queue as a generic reporting tool merely because both require filters. Conversely, do not hide advanced reporting requirements inside a list with increasingly elaborate custom rows.

Keep a primary row destination separate from selection. Navigation opens details; selection marks objects for an operation. If both are required, use the documented list modes and row item types rather than attaching DOM click handlers. Custom handlers on inner text controls easily create inconsistent keyboard behavior and accidental navigation while a user is selecting.

## Establish a binding contract

Build the screen against stable business identifiers, not array positions. `ID` is a durable application key; `/WorkItems/3` is only a JSON-model location. The route accepts the durable identifier, while the view binding resolves the current location. Real services should expose deterministic keys and an explicit sort tie-breaker.

The following XML uses a growing list with a navigation row. `includeItemInSelection="false"` keeps row activation distinct from selection. The search event intentionally applies a query on explicit submission, avoiding expensive requests on every keystroke. The filter selector uses business status keys rather than translated display labels.

```xml
<mvc:View
  controllerName="demo.controller.Worklist"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns:core="sap.ui.core"
  xmlns="sap.m">
  <Page title="My work">
    <content>
      <VBox>
        <SearchField id="search" search=".onSearch"
          placeholder="Search title or ID"/>
        <Select id="status" selectedKey="ALL" change=".onStatusChange">
          <items>
            <core:Item key="ALL" text="All statuses"/>
            <core:Item key="OPEN" text="Open"/>
            <core:Item key="BLOCKED" text="Blocked"/>
          </items>
        </Select>
        <List id="worklist"
          items="{work>/WorkItems}"
          mode="MultiSelect"
          includeItemInSelection="false"
          growing="true"
          growingThreshold="30"
          itemPress=".onItemPress"
          selectionChange=".onSelectionChange"
          noDataText="No work matches these filters">
          <headerToolbar>
            <OverflowToolbar>
              <Title text="Work items" level="H2"/>
              <ToolbarSpacer/>
              <Button text="Group by status" press=".onGroup"/>
              <Button text="Assign"
                enabled="{ui>/hasSelection}" press=".onAssign"/>
            </OverflowToolbar>
          </headerToolbar>
          <items>
            <ObjectListItem title="{work>Title}"
              intro="{work>ID}" type="Navigation">
              <firstStatus>
                <ObjectStatus text="{work>Status}"/>
              </firstStatus>
              <attributes>
                <ObjectAttribute title="Owner" text="{work>Owner}"/>
              </attributes>
            </ObjectListItem>
          </items>
        </List>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

This is a complete view structure, not a complete deployable application. The controller below supplies the filtering and grouping handlers; routing, loading data, and the assignment transaction are application integration points. In production, give status values human-readable translated labels and an independently defined semantic state.

## Compose search and structured filters

Represent query state explicitly, then derive a single filter tree. Search across multiple fields is usually an OR expression; status and search are usually combined with AND. Applying each independently to the same application filter bucket can accidentally replace the previous filter. Central composition prevents that bug and makes the active query explainable.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "sap/ui/model/FilterType",
  "sap/ui/model/Sorter",
  "sap/m/GroupHeaderListItem"
], function (
  Controller, JSONModel, Filter, FilterOperator,
  FilterType, Sorter, GroupHeaderListItem
) {
  "use strict";

  return Controller.extend("demo.controller.Worklist", {
    onInit: function () {
      this.getView().setModel(new JSONModel({
        query: "", status: "ALL", hasSelection: false
      }), "ui");
      this.byId("worklist").getBinding("items").sort([
        new Sorter("Title"),
        new Sorter("ID")
      ]);
    },

    onSearch: function (event) {
      this.getView().getModel("ui")
        .setProperty("/query", event.getParameter("query").trim());
      this._applyFilters();
    },

    onStatusChange: function (event) {
      this.getView().getModel("ui").setProperty(
        "/status", event.getSource().getSelectedKey()
      );
      this._applyFilters();
    },

    _applyFilters: function () {
      const state = this.getView().getModel("ui").getData();
      const filters = [];
      if (state.query) {
        filters.push(new Filter({
          filters: [
            new Filter("Title", FilterOperator.Contains, state.query),
            new Filter("ID", FilterOperator.Contains, state.query)
          ],
          and: false
        }));
      }
      if (state.status !== "ALL") {
        filters.push(new Filter(
          "Status", FilterOperator.EQ, state.status
        ));
      }
      this.byId("worklist").removeSelections(true);
      this.getView().getModel("ui")
        .setProperty("/hasSelection", false);
      this.byId("worklist").getBinding("items")
        .filter(filters, FilterType.Application);
    },

    onGroup: function () {
      this.byId("worklist").bindItems({
        path: "work>/WorkItems",
        template: this.byId("worklist").getBindingInfo("items").template,
        templateShareable: true,
        sorter: [
          new Sorter("Status", false, true),
          new Sorter("Title"),
          new Sorter("ID")
        ],
        groupHeaderFactory: function (group) {
          return new GroupHeaderListItem({
            title: group.key, upperCase: false
          });
        }
      });
      this._applyFilters();
    },

    onSelectionChange: function () {
      const count = this.byId("worklist").getSelectedContexts(true).length;
      this.getView().getModel("ui")
        .setProperty("/hasSelection", count > 0);
    },

    onItemPress: function (event) {
      const context = event.getParameter("listItem")
        .getBindingContext("work");
      this.getOwnerComponent().getRouter().navTo("object", {
        id: encodeURIComponent(context.getProperty("ID"))
      });
    }
  });
});
```

The `object` route must declare an `id` parameter and decode it exactly once on receipt. The example assumes string IDs and a route definition compatible with encoded values. For arbitrary identifiers containing reserved delimiters, define and verify a route-safe encoding contract rather than concatenating raw model paths.

The grouping handler reuses the XML template deliberately and marks it shareable. Keep its ownership with the view and do not reuse this pattern across unrelated views. A simpler production alternative is to declare the sorter and `groupHeaderFactory` in binding information once, then toggle grouping through `binding.sort()`. When rebinding, preserve every relevant binding parameter, event, filter, and template; this bounded JSON example has no additional parameters to preserve.

## Make grouping semantics explicit

Grouping is an ordering operation with group headers, not a separate analytical aggregation. The first sorter determines the group key and must make equal keys contiguous. Secondary sorters order records inside each group. A status group does not calculate server totals, and its visible item count is not necessarily the count of all matching objects.

For priorities, lexical order is often wrong: “High,” “Low,” and “Medium” do not sort in urgency order. A bounded JSON list can use a custom comparator over numeric ranks. An OData service cannot execute an arbitrary JavaScript comparator. Expose a sortable `PriorityRank` property on the service, or agree on a server-supported ordering. Do not silently sort only the currently loaded server page.

Group labels should explain business meaning. A raw status code is acceptable in a technical prototype but not in a finished operations tool. Translate the group label without changing the underlying key. If “Unassigned” represents a null owner, distinguish that from a failed owner-name lookup. Avoid placing important actions only in group headers; those headers are not business records.

## Define selection policy before implementing actions

`getSelectedContexts(true)` asks for selected binding contexts, including remembered selections where supported by the list. It does not mean “all records matching the query.” A server collection may contain thousands of unloaded objects. Define whether the action applies to explicitly selected objects, loaded objects, or the entire filtered result.

The example clears selection whenever the query changes. This conservative policy avoids processing hidden items after a filter switch. An application that intentionally preserves selection must expose a count and an easy “clear all selections” action, including hidden selections. Never show a selected count derived only from currently rendered controls if the actual command includes remembered contexts.

Extract identifiers immediately before invoking a command, then confirm its scope. For assignment, show the target owner and object count. For approval, consider whether business policy allows heterogeneous states. Disable impossible commands for convenience, but send the same constraints to a backend action that checks authorization, current state, and concurrency. A button’s enabled state is not a security boundary.

Selection is not locking. Another user can change a work item after it is selected. The command response should identify successes, conflicts, and rejected items. Keep failed items available for correction and refresh the successful objects. A single green toast after a partially failed operation is misleading.

## Introduce OData V4 deliberately

Replace the bounded JSON model only after defining service behavior. For a V4 model, configure server operation mode, bind `/WorkItems`, and use supported `Filter` operators and `Sorter` properties. The official filtering guide distinguishes static `$filter` binding parameters from dynamic filters; the two are combined, and a static filter cannot be replaced by a later `filter()` call [2].

Do not carry a JSON-only custom filter function into a server binding. Check whether the service supports `contains` on each chosen field, whether matching is case-sensitive, and whether exact ID lookup would be clearer than substring search. Backend search capabilities are service-specific; a text search box does not automatically enable OData `$search`.

Use `$select` to request the fields needed for rendering, sorting, key resolution, and navigation. Avoid expanding every child collection merely to show a header row. A server should enforce row-level access before returning results and should apply a stable sort before paging. Client-side hiding after retrieval can leak both records and aggregate counts.

Growing improves the initial interaction by requesting additional contexts as needed. It does not make a huge collection cheap, and it does not render every row efficiently forever. Establish a meaningful initial filter such as “Assigned to me,” constrain default date ranges, and profile payloads on representative devices. The `sap.m.List` source explicitly recommends limiting mobile item counts and using growing for initial rendering [1].

Do not mix V2 calls such as `model.read()` or `submitChanges()` into a V4 controller. V4 uses bindings and contexts for reads and `submitBatch()` for API-group submission. A migration must replace the data-access contract, not merely change the model constructor. Transaction details belong in [transactional patterns](./transactional-patterns.md).

## Preserve navigation context

A drill-down should preserve enough worklist state for the user to continue after processing one object. Persist query values, grouping, sorting, and explicit business identifiers in a component-level state model or a supported application-state mechanism. Restore them before the list requests data. Restoring controls after loading creates duplicate requests and a visible jump.

Treat scroll position as a best effort rather than a durable identifier. Paging, new records, and changed sort values can move a row. Prefer returning near a known business key and explain when the previously opened object no longer matches the filter. Do not store selection objects or binding contexts in local storage; they are runtime references and may contain sensitive data.

On a deep link, resolve the object independently of the worklist. The object page must not require the row to have been loaded previously. Handle deleted, unauthorized, and malformed identifiers with a recoverable message and a safe route back. Never disclose whether an inaccessible identifier exists if the service intentionally masks that distinction.

## Represent loading, empty, and failure states

An empty list can mean that no objects exist, no objects match a filter, or the request failed. These are different states. Give a filtered-empty state a reset action. Give an unfiltered-empty queue a task-oriented explanation. Give a failure a retry action and retain the user’s query rather than replacing the screen with “No data.”

Use binding request events or a request counter appropriate to the chosen model to drive busy state. Multiple overlapping requests need reference counting or generation tracking; a single boolean toggled by whichever request finishes first can clear the spinner too early. A failed older request must not overwrite the result of a newer query.

Counts also need a contract. `getItems().length` counts controls, potentially including group headers; it is not a reliable server total. Binding length can be provisional. Label loaded counts honestly and request an authorized total only when it contributes to the task. A dashboard KPI must use the same business filter definition if it links into this worklist.

## Address accessibility and localization

Use a visible heading and meaningful search labels. Status must be conveyed through text as well as color. Ensure the navigation affordance has a clear accessible name derived from the row’s business title. Keyboard users must be able to move through rows, change selection, and invoke toolbar commands without triggering drill-down.

Long localized titles, right-to-left layout, and text enlargement can change row height substantially. Test those conditions with real content rather than fixed-width filler strings. Avoid custom CSS that forces single-line status text when the status contains legally or operationally significant information. Keep touch targets consistent with the control’s density settings.

## Operational pitfalls

### Design assignment as a recoverable command

An assignment dialog should resolve a valid owner identifier, not send the display name typed into a free-text input. Show the chosen owner and the number of selected objects before submission. Preserve the original selection while the dialog is open, but revalidate the selected keys when the command starts. A refresh or background update must not silently redirect an assignment to different rows.

For a large selection, prefer a purpose-built backend action with a documented result structure. Include a client request identifier when the service supports idempotent processing, and reconcile an uncertain timeout before inviting a retry. The server may have completed an assignment even if the response never reached the browser.

### Keep application state independent of result data

Keep query and preference state in `ui`, not as extra properties on individual work items. Adding `Selected` or `Busy` to service entities can accidentally create pending OData changes or conflict with real metadata. A per-object activity map keyed by stable business ID is safer when rows need individual progress feedback.

If worklist refresh removes an item that was just processed, announce the successful result and preserve keyboard focus at a useful location. Do not leave focus pointing at a destroyed row control. When an operation fails, retain an obvious path to retry or inspect the object, rather than automatically removing it because its status was optimistically changed in the browser.

Do not fetch every record into JSON merely to gain simple filtering. Do not infer business priority from color or translated strings. Do not silently discard a filter when grouping rebinds the aggregation. Do not depend on rendered row order for command identifiers. Do not refresh the entire application after every successful action if refreshing the affected binding is sufficient.

For observability, log correlation identifiers and command outcomes, not unrestricted search text or full business objects. Consider that free-text searches may contain personal data. Monitor server latency separately from rendering time, since both can make a list feel slow but require different fixes. Agree on timeout and retry behavior with the service team.

## Verification checklist

- [ ] Default filters match the work queue’s documented business definition.
- [ ] Search OR conditions remain ANDed with structured filters.
- [ ] Clearing search retains other filters and produces a predictable request.
- [ ] Sorting has a deterministic key tie-breaker across server pages.
- [ ] Group headers are translated and never counted as business objects.
- [ ] Selection scope is explicit, including behavior after filtering.
- [ ] Navigation uses durable keys and supports direct deep links.
- [ ] Back navigation restores filters without duplicate initial requests.
- [ ] Empty, failed, loading, and partially successful states are distinct.
- [ ] Keyboard selection and navigation do not interfere with each other.
- [ ] Backend authorization and concurrency checks reject stale commands.
- [ ] A phone, long translations, and a slow connection remain usable.

## Official sources

1. [OpenUI5 `sap.m.List` source and control documentation](https://github.com/SAP/openui5/blob/master/src/sap.m/src/sap/m/List.js): list purpose, inheritance from `ListBase`, and mobile performance guidance.
2. [OpenUI5 official filtering guide](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/filtering-5338bd1.md): OData V4 server operation mode and static versus dynamic filtering.
3. [SAPUI5 official batch-control guide](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/batch-control-74142a3.md): V4 binding update groups, submission, pending changes, and error semantics.

These official sources were consulted through GitHub. Resolve API details against the exact UI5 version deployed by the application; repository default branches can describe newer behavior than a maintained enterprise runtime.
