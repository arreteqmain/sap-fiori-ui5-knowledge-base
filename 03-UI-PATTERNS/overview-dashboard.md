# Overview and dashboard patterns

## Scope and prerequisites

A dashboard helps a user notice a condition, understand its significance, and reach the next useful task. A procurement overview might show overdue confirmations, blocked invoices, and supplier delivery reliability. It should not become a gallery of unrelated charts. Start with the decisions a role must make, then select the smallest set of measures and navigation targets that supports those decisions.

This guide separates a **freestyle dashboard** from the **SAP Fiori elements Overview Page**. The working layout uses OpenUI5 `sap.f.Card`, card headers, `sap.ui.layout.Grid`, and `sap.m` controls. A separate optional microchart example requires the SAPUI5-only `sap.suite.ui.microchart` library. Declare the actual libraries in the manifest and use a distribution and deployment entitlement that provides them. `sap.ovp` Overview Page templates are also SAPUI5-specific; a freestyle collection of cards is not an Overview Page template.

Examples assume an MVC component, i18n resources, a named `dash` JSON model populated by an application-owned data adapter, and routes for drill-down. The adapter obtains already authorized aggregates and bounded preview records. JSON binding here is a presentation contract, not advice to download a full transactional database and calculate sensitive totals in the browser.

The [repository README](../README.md) lists the related [worklist](./worklist-pattern.md), [list report](./list-report-pattern.md), [object page](./object-page-pattern.md), and [transactional patterns](./transactional-patterns.md) guides.

## Choose the implementation model

Use a freestyle dashboard when composition, service contracts, or interactions differ materially from supported template behavior. `sap.f.Card` supplies a header and content container; the application owns queries, navigation, loading, errors, and synchronization. An integration card based on `sap.ui.integration.widgets.Card` is another card technology with its own manifest and host contract. It is not configured through the `sap.f.Card` content aggregation shown below.

Use the SAP Fiori elements Overview Page when supported annotation-driven cards, global filters, personalization, and established template behavior meet the requirements. Its card templates depend on the OData generation: official documentation distinguishes `sap.ovp.cards.v4.<cardType>` for V4 from `sap.ovp.cards.<cardType>` for V2 [2]. Do not copy a V2 template name into a V4 application and assume the model adapts.

Before choosing a template, verify the exact supported card types and capabilities for the deployed version. Current documentation notes V2-specific additional card types. A legacy screenshot is not proof that a card is available in every V4 runtime. Keep extensions within supported extension points instead of manipulating generated internal controls.

## Define a KPI contract

Every KPI needs a business definition: numerator, denominator, unit, time window, timezone, inclusion rules, and refresh timestamp. “Overdue orders: 27” is ambiguous unless overdue means something concrete, such as confirmed delivery date before the business date and remaining quantity greater than zero. Establish whether canceled, blocked, and partially delivered orders count.

The drill-down must reproduce the same definition. A KPI that counts item positions cannot navigate to an order-header list and promise the same record count. If the target intentionally aggregates differently, label that transition. Distinguish a count of affected suppliers from a count of late deliveries even when both numbers are useful.

Aggregate currency amounts only after agreeing on currency conversion and valuation date. Never sum mixed currencies and append the user's preferred currency symbol. For percentages, define the denominator and handle zero explicitly. Unknown is not zero; an unavailable KPI should say it is unavailable rather than display a reassuring green zero.

Thresholds encode business meaning. A high on-time delivery percentage may be good, while a high blocked-invoice percentage is bad. Store threshold policy centrally or receive semantic classification from the service. Do not decide color based solely on whether a number rose compared with yesterday.

## Compose responsive cards

The following OpenUI5 view uses a responsive grid and two cards. `defaultSpan` makes each card full width on phones, half width on medium screens, and one third on large screens. Card width follows its container. The first card displays a numerical KPI; its content provides an explicit accessible drill-down button rather than relying on a click anywhere in the card.

```xml
<mvc:View
  controllerName="demo.controller.Dashboard"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns:f="sap.f"
  xmlns:cards="sap.f.cards"
  xmlns:layout="sap.ui.layout"
  xmlns="sap.m">
  <Page title="Procurement overview">
    <content>
      <VBox class="sapUiResponsiveContentPadding">
        <Text text="{dash>/contextLabel}"/>
        <Text text="{dash>/asOfLabel}"/>
        <layout:Grid defaultSpan="XL4 L4 M6 S12" width="100%"
          hSpacing="1" vSpacing="1">
          <layout:content>
            <f:Card width="100%" busy="{dash>/overdue/busy}">
              <f:header>
                <cards:NumericHeader
                  title="Overdue orders"
                  subtitle="Outstanding delivery, current scope"
                  number="{dash>/overdue/countText}"
                  unitOfMeasurement="orders"
                  details="{dash>/overdue/detailText}"/>
              </f:header>
              <f:content>
                <VBox class="sapUiSmallMargin">
                  <ObjectStatus text="{dash>/overdue/statusText}"
                    state="{dash>/overdue/state}"/>
                  <Button text="Review overdue orders"
                    enabled="{dash>/overdue/available}"
                    press=".onOverduePress"/>
                </VBox>
              </f:content>
            </f:Card>
            <f:Card width="100%">
              <f:header>
                <cards:Header title="Suppliers requiring attention"
                  subtitle="Highest late-delivery counts"/>
              </f:header>
              <f:content>
                <List items="{dash>/suppliers}"
                  noDataText="No suppliers require attention"
                  itemPress=".onSupplierPress">
                  <ObjectListItem title="{dash>Name}"
                    type="Navigation" number="{dash>LateCount}"
                    numberUnit="late deliveries"/>
                </List>
              </f:content>
            </f:Card>
          </layout:content>
        </layout:Grid>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

The official `sap.f.Card` sample demonstrates header and content aggregations with ordinary controls inside them [1]. A card is not automatically actionable: this example deliberately supplies actions. Preformat `countText` and timestamps through locale-aware application formatters; keep raw numerical values available for calculations and accessibility descriptions. Use i18n texts in a deployed application.

Avoid fixed card heights chosen to align one language's content. Long translations and larger text settings change wrapping. If consistent rows are essential, choose a card layout strategy designed for that requirement and verify minimum heights against actual content. Do not clip error messages to preserve decorative alignment.

## Add microcharts selectively

A microchart answers a narrow visual question inside a compact space. A radial microchart can express completion percentage; a bullet chart can compare actual performance against target; a small trend can show direction. It is not a substitute for a detailed analytical view with axes, filtering, and inspection.

The following optional fragment is **SAPUI5-only** because it uses `sap.suite.ui.microchart`. Add that library to dependencies only when the deployed distribution supplies it. The `dash` model must contain a finite percentage between zero and one hundred and a semantic color supported by the runtime.

```xml
<core:FragmentDefinition
  xmlns:core="sap.ui.core"
  xmlns="sap.m"
  xmlns:micro="sap.suite.ui.microchart">
  <VBox class="sapUiSmallMargin">
    <Text text="On-time delivery"/>
    <micro:RadialMicroChart
      percentage="{dash>/delivery/percentage}"
      valueColor="Good"
      size="M"/>
    <Text text="{dash>/delivery/accessibleSummary}"/>
    <Button text="Analyze delivery performance"
      press=".onDeliveryPress"/>
  </VBox>
</core:FragmentDefinition>
```

`Good` is illustrative; bind the approved policy classification in a real dashboard. The official radial-chart documentation explains percentage input and the alternative fraction/total calculation, including the invalid zero-total case [3]. It also documents changes in accepted color types for SAPUI5 2.0, so do not assume a 1.x enum contract applies unchanged across major versions.

For an OpenUI5-only deployment, use `sap.m.ProgressIndicator` with explicit percentage and descriptive text, or choose another available representation. Do not silently depend on a commercial library merely because its XML namespace resolves in a developer's SAPUI5 sandbox. The textual summary remains important even when the chart is available: color and shape alone cannot convey business meaning accessibly.

## Preserve filters during drill-down

A card's destination should receive business filter values, not the displayed count. For an internal route, agree on a small route contract and validate it at the target. The following controller methods assume the target route declares optional query parameters and the component owns a named `dash` model:

```javascript
onOverduePress: function () {
  const model = this.getView().getModel("dash");
  const scope = model.getProperty("/scope");
  this.getOwnerComponent().getRouter().navTo("orders", {
    "?query": {
      plant: scope.plant,
      businessDate: scope.businessDate,
      condition: "overdueOutstanding"
    }
  });
},

onSupplierPress: function (event) {
  const supplier = event.getParameter("listItem")
    .getBindingContext("dash").getObject();
  this.getOwnerComponent().getRouter().navTo("supplier", {
    id: encodeURIComponent(supplier.ID)
  });
}
```

The target maps `condition` to a known filter definition; it must not execute arbitrary filter expressions received from the URL. Validate plant and date formats, then let the server authorize the resulting query. Dates should preserve the agreed business timezone. A browser in another timezone must not change the meaning of “overdue today.”

For cross-application navigation in a launchpad, use its supported navigation service and semantic object/action contract. Resolve service availability asynchronously through the shell integration provided by the host. Do not assume a global shell container exists in standalone execution. Navigation-target availability can improve the UI, but it cannot replace data-service authorization.

## Synchronize global filters

Global filters are a promise that cards share the selected scope. Not every card can honor every filter, especially when they use different services. Make unsupported scope visible rather than silently ignoring a selected company code. If a card is intentionally global, label it accordingly.

Apply a filter change atomically to the dashboard's state, then issue card requests for that state. Tag requests with a generation number so that late responses from the previous scope are ignored. Otherwise a slow KPI can display values for plant A beneath a header already showing plant B.

Keep the previous result only when it is clearly marked as stale during refresh. A screen that mixes new and old values without timestamps can imply relationships that never existed. For decisions requiring a consistent snapshot, obtain a server-side snapshot token or coordinated analytical result; parallel requests alone do not guarantee a shared transactional point in time.

## Isolate card failures

Each card needs loading, available, empty, stale, and failed states. A failed supplier-preview query should not blank a successfully loaded overdue count. Use independent request state and a retry action scoped to the failing card. Retain global filter values when retrying.

Do not convert a failed aggregate response to zero. Show “Unavailable” with a concise explanation and keep detailed technical diagnostics out of normal business text. Use a correlation identifier for support. Differentiate authorization failure from transient network failure without leaking restricted information.

Refresh frequency is a business and capacity decision. A queue with human-scale changes may not need polling every few seconds. The official Overview Page guide describes its own refresh interval setting [2]; that setting does not automatically configure a freestyle dashboard. In freestyle, stop timers on destruction, avoid overlapping refreshes, and consider whether hidden pages should continue requesting data.

## Design for responsive priority

Responsive design is not just reducing columns. Put the highest-priority card first in reading and keyboard order so that the phone layout remains useful. Avoid changing visual order with CSS while leaving a contradictory focus order. Make secondary cards available further down rather than squeezing every metric above the fold.

Small cards should not contain horizontally scrolling analytical tables. Show a bounded preview with a clear “View all” destination. State whether the preview is top ranked, most recent, or otherwise sampled. The user must not mistake three visible rows for the complete set of exceptions.

Check chart labels, units, and drill-down controls at phone width and browser zoom. Tooltip-only explanations are unreliable on touch devices. A card's title should explain its purpose without needing to inspect a legend elsewhere on the page.

## Protect data and control cost

Aggregates can reveal sensitive information even when detail rows are hidden. Authorize KPI queries at the backend using the same business scope as detail queries. Frontend visibility and launchpad intent checks are presentation aids; they do not enforce access to the data. The official card authorization guide describes navigation-intent-based visibility, not a replacement for backend checks [4].

Request aggregates directly from an analytical or purpose-built endpoint. Avoid issuing one request per supplier merely to calculate a count. Bound preview result sizes, select only needed properties, and agree on indexes and server paging. Measure dashboard startup with the real number of cards and realistic filter selectivity.

Cache only under an explicit freshness and authorization policy. A cache key must include relevant user or tenant scope, filters, and metric definition version. Browser storage is not automatically an appropriate place for financial or personnel summaries. In-memory data disappears on reload and is not offline support.

## Verification checklist

- [ ] Each KPI has a documented formula, unit, period, and timezone.
- [ ] Drill-down reproduces the KPI's scope or explains any aggregation change.
- [ ] Mixed currencies and zero denominators are handled explicitly.
- [ ] Unknown and failed values never appear as successful zero results.
- [ ] Each card can fail and retry without resetting the whole page.
- [ ] Late responses cannot mix different global-filter generations.
- [ ] Phone order reflects business priority and keyboard order.
- [ ] Charts have textual meaning independent of color.
- [ ] Optional SAPUI5 libraries are declared and actually available.
- [ ] V2 and V4 Overview Page templates are not mixed.
- [ ] Navigation checks are not treated as data authorization.
- [ ] Request counts, refresh intervals, and payload sizes meet capacity goals.

## Official sources

1. [OpenUI5 `sap.f.Card` XML sample](https://github.com/SAP/openui5/blob/master/src/sap.f/test/sap/f/demokit/sample/Card/V.view.xml): card header/content composition with standard controls.
2. [SAPUI5 Overview Page card types](https://github.com/SAP-docs/sapui5/blob/main/docs/06_SAP_Fiori_Elements/types-of-cards-8ed3f76.md): available template families, OData-version distinction, and Overview Page refresh configuration.
3. [SAPUI5 radial microchart guide](https://github.com/SAP-docs/sapui5/blob/main/docs/10_More_About_Controls/radial-micro-chart-aa93bdb.md): percentage, denominator, semantic colors, and responsive sizing.
4. [SAPUI5 Overview Page card authorization settings](https://github.com/SAP-docs/sapui5/blob/main/docs/06_SAP_Fiori_Elements/setting-authorizations-for-cards-00683f9.md): navigation-intent-based card visibility and its scope.
