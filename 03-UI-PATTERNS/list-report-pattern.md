# List report pattern with `sap.ui.table.Table`

## Scope and prerequisites

A list report supports finding, comparing, and processing a substantial collection of business objects. Users typically start with structured criteria, inspect a consistent set of columns, refine the result, and open an object or invoke a scoped operation. Unlike a work queue, its primary question is often “Which records meet these conditions?” rather than “What should I do next?”

This guide implements a **freestyle desktop report** with OpenUI5 `sap.ui.table.Table`. It is not a Fiori elements List Report template or a `sap.ui.comp.smarttable.SmartTable` configuration. A plain grid table does not generate a filter bar from annotations, persist variants automatically, provide export automatically, or enforce backend authorization.

Examples assume `sap.ui.table`, `sap.m`, and `sap.ui.core` are declared dependencies and the component supplies a named `report` JSON model containing a bounded `/Orders` array. Its fields are string `ID`, `Title`, `Status`, `SupplierName`, plus numeric `Quantity`. A separate section describes V4 server binding and mass changes. The optional XLSX example requires the **SAPUI5-only `sap.ui.export` library**; it must not be assumed available in an OpenUI5 deployment.

Related pages listed in the [repository README](../README.md) are the [worklist](./worklist-pattern.md), [object page](./object-page-pattern.md), [dashboard](./overview-dashboard.md), and [transactional patterns](./transactional-patterns.md).

## Choose a desktop grid deliberately

`sap.ui.table.Table` is useful for dense, column-oriented desktop work. It reuses rendered rows as the user scrolls, which reduces DOM cost compared with rendering every record. That virtualization does not remove backend query cost, turn row indexes into stable identifiers, or make complex cell controls inexpensive.

Use a responsive `sap.m.Table` when the same task must work naturally on phones. A desktop grid inside a responsive container is still a desktop grid. Horizontal scrolling may be acceptable for an expert reporting task, but it should be a conscious design decision supported by the application's device policy.

Choose columns around comparisons and decisions. Put the identifying link first, preserve units beside amounts, and avoid forcing users to decode status codes. A table with forty initially visible columns is rarely a good default, even when personalization can hide them. Select a meaningful initial layout and make less frequent information discoverable.

## Define the query model

Keep search text, structured filters, column filters, sorting, and pagination conceptually separate. Search across ID and description is often an OR condition. Company, status, and date criteria are usually AND conditions. Document the resulting expression so that both developers and users can predict the result.

Decide whether searching is explicit through Go or triggered while typing. Explicit execution is often preferable for expensive enterprise queries. If using live search, debounce requests and prevent older responses from replacing newer state. Do not send a request on every intermediate keystroke without measuring server load and accessibility impact.

Defaults should bound the workload without hiding their meaning. A recent date range or business-unit restriction can improve performance, but users need to see and change it where permitted. A hidden default filter that explains a missing document is a support problem, not an optimization.

## Build the table

This view uses built-in column filter and sort properties. A custom column-filter handler centralizes filtering with search. The table has a deliberate desktop row count; for a resizable production layout, use the row-mode API supported by the deployed UI5 version and give its parent a resolvable height.

```xml
<mvc:View
  controllerName="demo.controller.Report"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns:table="sap.ui.table"
  xmlns="sap.m"
  height="100%">
  <Page title="Order report" enableScrolling="false">
    <content>
      <table:Table id="orders"
        rows="{report>/Orders}"
        visibleRowCount="12"
        selectionMode="MultiToggle"
        selectionBehavior="RowSelector"
        filter=".onColumnFilter"
        ariaLabelledBy="reportTitle">
        <table:extension>
          <OverflowToolbar>
            <Title id="reportTitle" text="Orders" level="H2"/>
            <ToolbarSpacer/>
            <SearchField width="18rem"
              placeholder="Search ID or title" search=".onSearch"/>
            <Button text="Clear column filters" press=".onClearFilters"/>
          </OverflowToolbar>
        </table:extension>
        <table:columns>
          <table:Column width="9rem"
            sortProperty="ID" filterProperty="ID">
            <Label text="Order"/>
            <table:template>
              <Link text="{report>ID}" press=".onOpenOrder"/>
            </table:template>
          </table:Column>
          <table:Column width="20rem"
            sortProperty="Title" filterProperty="Title">
            <Label text="Title"/>
            <table:template>
              <Text text="{report>Title}" wrapping="false"/>
            </table:template>
          </table:Column>
          <table:Column width="10rem"
            sortProperty="Status" filterProperty="Status">
            <Label text="Status"/>
            <table:template>
              <ObjectStatus text="{report>Status}"/>
            </table:template>
          </table:Column>
          <table:Column width="12rem" sortProperty="SupplierName">
            <Label text="Supplier"/>
            <table:template>
              <Text text="{report>SupplierName}" wrapping="false"/>
            </table:template>
          </table:Column>
        </table:columns>
      </table:Table>
    </content>
  </Page>
</mvc:View>
```

`visibleRowCount` is appropriate for an explicitly targeted 1.x compatibility example; newer applications should check its deprecation and prefer the documented row-mode aggregation when available. Do not combine competing sizing APIs. The official basic table sample confirms the `rows`, `columns`, cell-template, and extension-toolbar structure [1].

## Combine filtering and search correctly

The controller below handles string column filtering intentionally as Contains matching. It is not a generic parser for numeric ranges, dates, or user-entered filter expressions. Preventing the table's default filter behavior means the application must also update filter indicators. All criteria go into one application filter tree, so later search does not erase column filters.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "sap/ui/model/FilterType",
  "sap/ui/model/Sorter"
], function (Controller, Filter, FilterOperator, FilterType, Sorter) {
  "use strict";

  return Controller.extend("demo.controller.Report", {
    onInit: function () {
      this._columnFilters = Object.create(null);
      this._query = "";
      this.byId("orders").getBinding("rows").sort([
        new Sorter("Title"),
        new Sorter("ID")
      ]);
    },

    onSearch: function (event) {
      this._query = event.getParameter("query").trim();
      this._applyFilters();
    },

    onColumnFilter: function (event) {
      event.preventDefault();
      const column = event.getParameter("column");
      const value = event.getParameter("value").trim();
      const property = column.getFilterProperty();
      if (value) {
        this._columnFilters[property] = value;
      } else {
        delete this._columnFilters[property];
      }
      column.setFilterValue(value);
      column.setFiltered(Boolean(value));
      this._applyFilters();
    },

    _applyFilters: function () {
      const filters = Object.keys(this._columnFilters).map(function (key) {
        return new Filter(
          key, FilterOperator.Contains, this._columnFilters[key]
        );
      }, this);
      if (this._query) {
        filters.push(new Filter({
          filters: [
            new Filter("ID", FilterOperator.Contains, this._query),
            new Filter("Title", FilterOperator.Contains, this._query)
          ],
          and: false
        }));
      }
      const table = this.byId("orders");
      table.clearSelection();
      table.getBinding("rows").filter(filters, FilterType.Application);
      table.setFirstVisibleRow(0);
    },

    onClearFilters: function () {
      this._columnFilters = Object.create(null);
      this.byId("orders").getColumns().forEach(function (column) {
        column.setFilterValue("");
        column.setFiltered(false);
      });
      this._applyFilters();
    },

    onOpenOrder: function (event) {
      const context = event.getSource().getBindingContext("report");
      this.getOwnerComponent().getRouter().navTo("object", {
        id: encodeURIComponent(context.getProperty("ID"))
      });
    }
  });
});
```

“Clear column filters” here retains search, matching the implementation's separate query state. Explicitly clear the search field and `_query` if the intended action is “Reset all.” Avoid a mismatch between visible input values and the executed query.

For typed fields, use an appropriate value help, date control, or filter parser and the correct `FilterOperator`. Do not apply Contains to a decimal amount. Treat null separately from an empty string when the service does. UI strings should not be concatenated into raw `$filter` syntax.

## Preserve ordering across server pages

The initial sort uses `Title`, then `ID` as a deterministic tie-breaker. Built-in user sorting may replace that initial array. If deterministic secondary ordering is required for every sort, handle the table sort event, prevent default behavior, build the complete sorter array, and update column indicators consistently. Do not claim a one-time initial sorter guarantees every subsequent ordering.

Sorting by formatted strings can produce incorrect dates and amounts. Bind sorting to the underlying service property, not to a cell formatter's output. For a semantic priority, expose a server-sortable rank. A custom JavaScript comparator only affects client data and cannot sort an entire server-side result.

Document whether multi-column sorting is supported and how users reset it. When a variant restores sorting, apply it before requesting the result. Reapplying a variant after the table loads can cause unnecessary requests and confusing row movement.

## Move to OData V4 server binding

For production-sized datasets, bind the table's `rows` to a V4 entity set and configure server operation mode. Use the same `Filter` and `Sorter` concepts only where the service supports their operators and properties. The official V4 filtering guide explains the distinction between static binding `$filter` and dynamic filters, which are combined [2].

Select only the key and fields needed for cells, sorting, navigation, and commands. Add associations deliberately rather than expanding every child collection. Server paging requires deterministic ordering, appropriate indexes, and realistic limits. Row virtualization does not prevent large requests caused by an overly broad query.

OData `$search` is not synonymous with an OR tree of Contains filters. `$search` has service-defined searchable fields and linguistic semantics. If the service supports it, use the supported V4 binding parameter change API and its documented restrictions for the runtime. Otherwise label the search scope honestly, such as “Search ID or title.”

Never mix V2 `model.read()` or `submitChanges()` with V4 binding code. A table's visual API can stay the same while its model integration changes substantially. V4 reads arise from bindings and contexts; writes and explicit operations use their V4 APIs. Keep separate adapters if an application must consume both service generations.

## Treat selection as business scope

Grid tables expose selected row indexes, but indexes are not durable identifiers. Sorting, filtering, refresh, and virtual scrolling change their meaning. Resolve each selected index to its binding context immediately before a command and extract stable keys or retain the relevant contexts only for the operation's lifetime.

`getContextByIndex()` may encounter an unloaded context in server scenarios. Do not skip missing contexts silently and act on a smaller set than the user selected. Either request the necessary data through a supported bounded strategy or prevent the operation until the selected contexts are available. Communicate “selected rows” separately from “all matching records.”

The example clears selection on filter changes. If selection persists, expose the hidden-selection policy and count clearly. A select-all affordance on a virtualized table must not be described as selecting an unbounded server result unless a dedicated backend operation implements that scope safely.

## Implement batch changes deliberately

Batch transport reduces round trips; it does not by itself define business atomicity or partial-success policy. A mass status update may require all selected orders to succeed together, or it may permit independent results. Agree on that policy with the service and choose changeset and action behavior accordingly.

For a **V4-only** property-change operation, the following excerpt queues selected contexts in a dedicated API group and observes individual promises. It assumes all contexts have already been resolved, current values are loaded, no unrelated edits exist, and the service permits this property change. `bRetry=false` makes failures visible to each promise rather than relying on automatic retry behavior.

```javascript
const group = "massEdit";
const changes = selectedContexts.map(function (context) {
  return context.setProperty("Status", "REVIEW", group, false);
});
const outcomesPromise = Promise.allSettled(changes);
await model.submitBatch(group);
const outcomes = await outcomesPromise;
const failed = outcomes.filter(function (outcome) {
  return outcome.status === "rejected";
});
```

Surround this excerpt with busy-state cleanup and transport-error handling. Do not announce success merely because `submitBatch()` resolved; its envelope can succeed while contained requests fail [3]. Preserve failed row identities and show meaningful per-row messages. Where a business transition is not a simple property edit, expose and invoke a backend action instead.

Do not implement approval by changing a displayed status string unless the service explicitly defines that operation. Approval may require authorization, locking, audit logging, and downstream effects. A confirmation should state the object count and consequences, but only the backend can enforce those rules.

## Export a bounded snapshot

Export is a data operation, not a screenshot of rendered rows. Decide whether it includes selected records, the full filtered result, visible columns, or a defined reporting schema. Preserve filter and sort semantics and tell users which scope is being exported.

The following **SAPUI5-only** example uses `sap.ui.export.Spreadsheet` and `sap.ui.export.EdmType`. It exports the complete filtered and sorted result of the bounded JSON binding used earlier, not an OData result. Import both modules through `sap.ui.define`, declare `sap.ui.export`, and provide localized headers.

```javascript
async function exportBoundedJson(table, Spreadsheet, EdmType) {
  const binding = table.getBinding("rows");
  const rows = binding.getContexts(0, binding.getLength())
    .map(function (context) {
      return context.getObject();
    });
  const sheet = new Spreadsheet({
    workbook: {
      columns: [
        { label: "Order", property: "ID", type: EdmType.String },
        { label: "Title", property: "Title", type: EdmType.String },
        { label: "Status", property: "Status", type: EdmType.String }
      ]
    },
    dataSource: rows,
    fileName: "Orders.xlsx"
  });
  try {
    await sheet.build();
  } finally {
    sheet.destroy();
  }
}
```

Do not replace the JSON binding with an OData binding and assume `getContexts()` loads the whole result synchronously. Server export needs a supported data-source configuration or a backend export job with limits, cancellation, and authorization. Never loop over the entire service blindly in the browser to imitate a complete export.

The official export guide requires explicit column and data-source configuration for plain tables [4]. Its restrictions include unsupported custom formatter functions, charts, icons, and aggregated rows, plus row and cell limits [5]. Choose export types from underlying data, not cell text. Preserve leading-zero identifiers as strings and verify decimals, currencies, dates, and timezones with real spreadsheet consumers.

For OpenUI5-only deployments, omit this library and use a deliberately designed alternative such as a backend-generated download. CSV is not equivalent to XLSX: it requires delimiter and quoting policy, encoding decisions, formula-injection protection for untrusted values, and careful treatment of identifiers and dates.

## Persist variants without persisting runtime objects

A useful variant contains filter values, sort definitions, column order, visibility, widths, and a schema version. It should not contain binding contexts, control instances, or raw pending changes. Validate property names against an allowlist when restoring state, especially if variant state can arrive through a URL.

Persist user preferences through the supported application or shell mechanism chosen for the product. A plain `sap.ui.table.Table` does not automatically provide a complete variant-management solution. Avoid storing sensitive query values in browser storage without a retention and security policy. If metadata changes, migrate or reject incompatible variants rather than leaving the table partly configured.

## Distinguish loading, empty, and failed results

A report needs separate states for an initial query that has not run, an active request, an empty result, and a failed request. “No matching orders” is inaccurate when the service could not answer. Keep the user's criteria after failure and offer a retry without forcing them to rebuild the query.

Counts should describe their source. The number of rendered rows is unrelated to the total because the grid reuses row controls. A binding length can also be provisional until the service has provided enough information. If the design needs an exact total, request one under the same authorized filter scope and identify the cost with the backend team.

Use a request generation identifier when application-managed requests can overlap. Clearing the busy state on the first completed request is wrong when a newer request remains pending. Likewise, a late failure from an abandoned query must not replace a successful result for the current query. Binding-managed and application-managed request lifecycles should have one clearly defined owner.

## Plan navigation and return behavior

Opening an order should preserve the report's criteria, column arrangement, and sorting. On return, restore these before binding the result so that the first request already represents the intended state. A stored row number is only a position hint, not proof that the same business object remains there.

If an edited order no longer matches the report, explain why it disappeared rather than manufacturing a temporary row that violates the active filter. Refresh affected data while retaining useful orientation. Deep-linked object pages must still resolve their objects independently; they cannot depend on a virtual row context surviving in the source table.

For audit-sensitive exports or mass changes, record the query definition or selected business keys through the approved backend audit mechanism. Do not treat a browser console log or a downloaded filename as an audit trail. Data access, query retention, and action logging must follow the application's privacy policy.

## Keep cell templates safe under virtualization

A rendered row control can represent different business objects as the user scrolls. Do not attach business identity to a row's DOM position or capture a context once during initial rendering and reuse it forever. Event handlers should resolve the current binding context from the event source when the user acts.

Cell templates should also remain lightweight. Creating nested forms, large fragments, or independent service requests in every cell defeats the benefit of row reuse. Use concise display controls and move complex editing into a focused interaction when the task permits. Profile both the initial viewport and rapid scrolling with realistic data, because they exercise different costs.

Validation and status indicators must follow binding context changes correctly. A local error flag stored on a reused control can appear on the wrong record after scrolling. Put business-specific state in the model or a stable-keyed state map, and clear purely transient control state through a deliberate lifecycle.

## Define totals and aggregation separately

A plain grid table does not make every column analytically additive. Quantities with different units, amounts in different currencies, and percentages usually cannot be summed meaningfully without additional rules. If the report needs totals, define aggregation at the appropriate service or analytical layer and display the relevant unit and scope.

Distinguish a total for the full filtered result from a subtotal for selected rows or a loaded page. A footer that changes as more contexts load must not be labeled as a complete business total. Export requirements should state whether totals are independently calculated, included as additional records, or omitted; spreadsheet presentation limitations do not justify silently changing the meaning.

Grouping also has different implementations across table controls and data models. Do not carry `sap.m.List` group-header assumptions into a flat `sap.ui.table.Table` binding. For genuinely hierarchical or analytical tasks, assess the appropriate table type and supported binding contract rather than simulating hierarchy with indentation and fabricated data rows.

## Bound expensive queries before execution

Use mandatory or recommended criteria when an unfiltered query would be operationally unsafe. Validate the criteria before issuing the binding request, not after downloading the first large result. Explain why a plant, period, or organizational scope is required.

Backend limits remain necessary even with frontend checks. Another client can call the same endpoint directly, and a saved variant can contain broader values than expected. Agree on maximum result windows, query timeouts, and an export-job alternative for legitimate large workloads. A useful error should identify the restriction and suggest a narrower query instead of presenting an unexplained empty table.

## Verification checklist

- [ ] Desktop-only assumptions are explicit and an appropriate mobile path exists.
- [ ] Search and column filters compose without overwriting each other.
- [ ] Typed criteria use correct operators and null semantics.
- [ ] Every server sort has the required deterministic tie-breaker.
- [ ] Query changes reset or clearly preserve selection under a documented policy.
- [ ] Commands resolve stable keys, including unloaded-context handling.
- [ ] Batch outcomes are checked per operation and partial success is visible.
- [ ] Backend actions enforce authorization, state, and concurrency.
- [ ] Export scope matches the stated filters, columns, and selection.
- [ ] OpenUI5 builds do not depend accidentally on `sap.ui.export`.
- [ ] Large exports use a bounded supported strategy rather than rendered rows.
- [ ] Variants restore before the first query and reject unsupported properties.

## Official sources

1. [OpenUI5 basic grid-table sample](https://github.com/SAP/openui5/blob/master/src/sap.ui.table/test/sap/ui/table/demokit/sample/Basic/View.view.xml): rows, columns, templates, selection, and extension toolbar.
2. [OpenUI5 OData V4 filtering](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/filtering-5338bd1.md): server mode and static/dynamic filter composition.
3. [SAPUI5 OData V4 batch control](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/batch-control-74142a3.md): submission groups, changesets, retry, and individual failure semantics.
4. [SAPUI5 spreadsheet export](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/spreadsheet-export-2691788.md): module loading and explicit configuration for non-smart controls.
5. [SAPUI5 spreadsheet export restrictions](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/spreadsheet-export-restrictions-2c64148.md): supported output limits and unsupported presentation features.

SDK reference: [`sap.ui.table.Table`](https://ui5.sap.com/#/api/sap.ui.table.Table). Research used successfully retrieved official GitHub files; direct SDK access failed DNS resolution.
