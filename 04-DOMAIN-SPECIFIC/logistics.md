# Logistics: Transportation Planning and Delivery Visibility

## Scope and assumptions

This guide describes an original application design for shipment tracking, freight orders, transport planning, delivery monitoring, and route optimization. The illustrative landscape has SAP S/4HANA, a transportation capability, warehouse execution, external carriers, and a custom UI5 application. It is not a promise that one standard application supports this entire process. Organizational responsibilities, deployment edition, licensed capabilities, and integration availability must be established before selecting an implementation.

**SAP Transportation Management (TM) and Logistics Execution transportation are not interchangeable.** A freight order in a TM process is not simply another name for an LE shipment document. Freight units, planning, tendering, execution, and settlement depend on the installed transportation solution. Older LE transportation scenarios may have different document flows and restrictions. Confirm the applicable product documentation, simplification information, supported APIs, and target release before mapping objects.

All `/logistics/` URLs, `Demo.Logistics` actions, entity names, and states below belong to a **hypothetical custom OData V4 façade**. They are not official SAP endpoints or prescribed SAP status codes. The backend translates domain commands into supported application APIs; it never writes directly to SAP database tables.

## Standard application discovery

Search the [SAP Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) for the actual business tasks and select the exact product version. Record required business catalogs, software components, frontend/backend prerequisites, supported devices, and available application configuration. Check whether the capability belongs to embedded TM, another TM deployment, or delivery processing. Do not infer availability from a screenshot, a similarly named application, or an app identifier copied from another release.

This research environment could retrieve official SAPUI5 documentation through SAP's GitHub repositories, but could not resolve the Help Portal or reference-library hosts. Consequently, no release-specific standard application availability is asserted here. The library verification is an implementation gate, not a completed certification.

## Process ownership and operating roles

A transport planner groups demand and chooses feasible resources. A dispatcher communicates the executable plan and handles deviations. Shipping personnel own loading readiness and delivery execution within their authorization. A carrier supplies acceptance, milestones, and proof of delivery. Customer service sees promises and exceptions without changing transportation commitments. A freight cost specialist reviews charges; an operations supervisor approves exceptional cancellation or reassignment.

Avoid a universal “logistics user” permission. Assign permissions by legal entity, shipping location, planning organization, document responsibility, and action. Carrier users should see only their assigned movements and the minimal customer information necessary for execution. A planner's ability to view another region does not imply permission to release its freight orders.

## End-to-end workflow

1. Receive transportation demand from the authoritative delivery or order process.
2. Validate addresses, time windows, dangerous-goods requirements, quantities, and readiness.
3. Build planning units using the installed transportation model.
4. Produce a feasible route proposal and compare cost, service, and operational risk.
5. Save a versioned plan, obtain required approval, and commit resource assignments.
6. Tender or otherwise communicate the movement according to the actual carrier process.
7. Confirm readiness, load, depart, collect milestones, and manage exceptions.
8. Confirm delivery evidence and resolve discrepancies.
9. Reconcile transportation completion with delivery, billing, and freight settlement.

These steps are a cross-system process, not one distributed database transaction. Each handoff needs a durable identifier, an explicit owner, and a reconciliation path when the next system is unavailable.

## State model: separate dimensions

Do not compress everything into `ShipmentStatus`. Use independently meaningful dimensions:

| Dimension | Illustrative progression | Important exception |
|---|---|---|
| Planning | Unplanned → Proposed → Committed | Proposal becomes stale |
| Carrier commitment | NotRequested → Pending → Accepted | Rejected or expired |
| Execution | NotStarted → Loading → Departed → Delivered | Delayed or interrupted |
| Evidence | Missing → Received → Verified | Disputed |
| Settlement | NotReady → Ready → InReview → Settled | Charge disputed |

A delivered movement can still have disputed evidence and unsettled charges. A carrier rejection does not cancel the customer's delivery automatically. Expose the actionable state and its owner rather than presenting one misleading green status.

Record transitions with server time, previous version, command identifier, actor, reason, and resulting version. Preserve the original event time separately because an event received today may describe yesterday's departure.

## Atomic block: releasing a freight order

**Command:** release one approved plan version for execution.

**Preconditions:** the actor can release for the planning organization; the order is not canceled; demand references remain valid; mandatory resources and permits are present; the expected version matches; capacity allocation has not been superseded.

**Atomic effects:** freeze the released plan snapshot, record committed allocations, append an audit transition, and create an outbound integration task in the same local transaction. External carrier communication occurs afterward through a durable worker.

**Failure paths:** reject a stale plan with a conflict response; return structured validation messages for missing permits; retain the order in its previous state if the local transaction fails. If outbound delivery fails after commit, show “released, communication pending,” not “release failed.” A dispatcher must not release again merely to resend the carrier message.

## Data model and authoritative fields

Use technical immutable keys alongside readable business identifiers. A useful custom read model includes:

- `FreightOrders`: ID, display number, source system, organization, planning status, execution status, carrier reference, version, and timestamps.
- `Stops`: sequence, location reference, local time zone, planned/estimated/actual arrival and departure, and handling requirements.
- `DemandAssignments`: source document/item reference, quantity, unit, weight, volume, and assignment version.
- `Milestones`: external event identifier, event type, event time, received time, source, confidence, and correction reference.
- `PlanProposals`: input snapshot hash, solver version, constraints, objective values, expiration, and feasibility findings.
- `CommandReceipts`: idempotency key, action, target, payload hash, processing state, and resulting document reference.

Keep calculated ETA distinct from contractual delivery promise. Label stale projections and indicate their update time. Store monetary amounts with currency and measurements with units; never sum kilograms and pounds without an explicit conversion rule.

## Business invariants

An assignment cannot consume more available demand than the source permits. Shared capacity must account for other committed assignments, not merely the rows visible in the planner's browser. A route cannot violate mandatory vehicle, location, driver, or cargo restrictions. A canceled movement cannot accept a normal departure confirmation. A proof-of-delivery attachment cannot silently rewrite an actual delivered quantity.

Validate quantities at the appropriate granularity: shipment-level totals can conceal item-level shortages. Preserve source reference and item identity through splitting and consolidation. If multiple freight orders share a delivery, prohibit a simplistic “any freight order delivered means delivery complete” rule. Define completion from all relevant assignments and the source delivery's own processing rules.

## Service boundaries and integration

The delivery service owns delivery quantities, blocks, and goods-movement outcomes. Transportation owns planning decisions and transportation execution. Warehouse execution owns tasks and physical loading confirmations where applicable. The tracking ingestion service owns external event validation and normalization. The optimization service proposes routes but must not post goods issues or accept carrier charges.

A façade can combine these facts for a worklist, but it must not become a second source of truth. Commands route to the owning service. Cross-system orchestration records progress explicitly: “transport released,” “warehouse request pending,” and “carrier acknowledgement outstanding” are different facts. Use supported APIs and business events available in the target landscape; do not assume a publicly documented API is enabled or licensed in every deployment.

## Planning workspace

Organize demand by planning horizon, origin, destination cluster, readiness, and exception. Distinguish demand that is absent from demand that is filtered out. Let planners inspect unassigned quantities before running optimization. A “plan all” button should describe which filtered population, planning snapshot, and time window it will use.

Display hard constraint violations separately from soft penalties. An expensive feasible route is different from an inexpensive impossible route. Provide an explanation panel showing the binding constraint: dock window, truck capacity, vehicle restriction, or required service time. Allow a supervisor to approve a permitted soft-constraint exception, but never silently convert a legal prohibition into a penalty.

## Route optimization contract

Build an immutable input containing demand, locations, travel-time assumptions, capacity, service times, time windows, driver restrictions, and the current resource-reservation version. Record objective weights and the solver build. Do not optimize using an unversioned live table while the planner changes demand underneath it.

Run expensive optimization asynchronously. Return a job identifier, progress, cancellation state, and a proposed plan. Cancellation requests may race with completion; the service must report the actual terminal outcome. Revalidate the proposal when committing because resources or delivery requirements may have changed. A proposal is not a reservation unless the contract explicitly creates a time-limited hold.

Compare alternatives on consistent dimensions: cost in one declared valuation basis, expected lateness, empty distance, utilization, and emissions assumptions if calculated. Explain uncertain travel times. Avoid false precision such as a minute-level arrival promise from yesterday's traffic model.

## Delivery monitoring and customer promises

A monitoring page should connect order, delivery, freight movement, stop, and evidence references without pretending they are the same document. Present the contractual delivery date, currently estimated arrival, actual milestone, and projection timestamp together. Include the local destination time zone and avoid treating date-only commitments as UTC instants.

For a delayed stop, show downstream impacts and responsible teams. Customer service may request a revised promise, but changing a transport ETA does not revise the sales commitment automatically. Warehouse readiness may be late while the vehicle remains on time. Separate operational readiness, transportation performance, and customer service measures so supervisors can identify the actual bottleneck.

## Bulk dispatch and operational metrics

Treat bulk release as a set of explicitly identified commands unless the backend genuinely supports an atomic multi-order transaction. Show the selection count, relevant plan versions, and organization scope before submission. Return an outcome for each order: accepted, rejected, already processed, or unresolved. A transport-level batch success does not prove that every contained business operation succeeded.

For mixed results, retain successful receipts and allow users to review only the unresolved subset. Never repeat the entire selection with new identifiers. If some orders share a capacity reservation, define whether the reservation commits atomically or whether partial acceptance requires replanning. The browser cannot establish that guarantee by sending requests simultaneously.

Define on-time delivery using the relevant promised window and verified arrival event, not the time the carrier feed arrived. Define utilization using a declared limiting dimension: weight, volume, loading meters, or another supported measure. Averaging percentages across differently sized vehicles can misrepresent capacity; publish the aggregation rule. Freight cost per delivery requires a documented allocation method when one movement serves several customers.

Track exception age and resolution time alongside service performance. Otherwise teams may improve apparent delivery performance by leaving disputed events unprocessed. Include a data-quality indicator for movements missing evidence rather than excluding them silently from the denominator.

## Dispatcher shift handover

Provide a handover queue ordered by risk and next decision deadline. Each entry should contain the movement reference, last verified physical state, pending commitment, responsible external party, next contact time, and the person accepting ownership. Preserve the chronology of previous decisions so the incoming dispatcher does not repeat an already rejected carrier option.

If tracking is unavailable, show the last verified milestone and its age. A manually confirmed milestone requires an authorized actor, evidence, and a reason identifying its source. Restoring the feed must reconcile later electronic events with that manual record rather than overwrite it without explanation.

## Tracking ingestion and correction

Authenticate carrier feeds and bind each external identifier to an authorized carrier/document relationship. Validate event type, timestamp plausibility, location, and payload size. Deduplicate by source plus immutable event ID; if a carrier lacks IDs, agree on a documented composite key and collision policy. Keep raw evidence in controlled storage with retention limits.

Events arrive late, out of order, and sometimes incorrectly. A late “loading started” event must not regress a departed movement to loading. Store it in history and recompute permitted projections using domain rules. Corrections should reference the superseded event rather than erase it. Quarantine impossible events, such as delivery before departure beyond an allowed clock-skew tolerance, for operational review.

## Contextual UI5 XML: exception worklist

The view below uses the default V4 model and a named `ui` JSON model. `ShipmentNumber`, `ExecutionStatus`, `ExceptionSummary`, and `Version` exist in the illustrative service metadata. Buttons issue commands rather than editing a status property directly. Resource-bundle texts and the controller's initialization are required.

```xml
<mvc:View controllerName="demo.logistics.controller.Worklist"
  xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
  <Page title="{i18n>freightOrders}" busy="{ui>/busy}">
    <content>
      <MessageStrip text="{i18n>trackingMayBeDelayed}"
        type="Information" showIcon="true"/>
      <Table id="orders" items="{
        path: '/FreightOrders',
        parameters: {
          $select: 'ID,ShipmentNumber,ExecutionStatus,ExceptionSummary,Version',
          $orderby: 'ShipmentNumber'
        }
      }">
        <columns>
          <Column><Text text="{i18n>movement}"/></Column>
          <Column><Text text="{i18n>execution}"/></Column>
          <Column><Text text="{i18n>exception}"/></Column>
          <Column><Text text="{i18n>action}"/></Column>
        </columns>
        <items>
          <ColumnListItem>
            <cells>
              <ObjectIdentifier title="{ShipmentNumber}"/>
              <ObjectStatus text="{ExecutionStatus}"/>
              <Text text="{ExceptionSummary}"/>
              <Button text="{i18n>requestReplan}" press=".onRequestReplan"
                enabled="{= !${ui>/busy} }"/>
            </cells>
          </ColumnListItem>
        </items>
      </Table>
    </content>
  </Page>
</mvc:View>
```

The table deliberately uses textual status rather than an unvalidated color mapping. Add a localized formatter for business labels and semantic states. Offer keyboard-accessible detail navigation and avoid exposing sensitive customer addresses in list columns.

## Contextual UI5 JavaScript: one explicit command

This example targets a modern UI5 release supporting V4 operation binding `invoke`; verify the deployed API version. The custom bound action `RequestReplan` accepts two `Edm.String` parameters, `RequestId` and `ExpectedVersion`, and returns a command receipt. `Version` is a string concurrency token, not a JavaScript integer.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox"
], function (Controller, JSONModel, MessageBox) {
  "use strict";
  return Controller.extend("demo.logistics.controller.Worklist", {
    onInit: function () {
      this.getView().setModel(new JSONModel({ busy: false }), "ui");
    },
    onRequestReplan: async function (oEvent) {
      const oView = this.getView();
      const oUi = oView.getModel("ui");
      if (oUi.getProperty("/busy")) {
        return;
      }
      const oContext = oEvent.getSource().getBindingContext();
      if (!oContext) {
        return;
      }
      const oAction = oView.getModel().bindContext(
        "Demo.Logistics.RequestReplan(...)", oContext
      );
      const sRequestId = globalThis.crypto.randomUUID();
      oAction.setParameter("RequestId", sRequestId);
      oAction.setParameter("ExpectedVersion", oContext.getProperty("Version"));
      oUi.setProperty("/busy", true);
      try {
        await oAction.invoke("$direct");
        MessageBox.information(
          oView.getModel("i18n").getResourceBundle()
            .getText("replanAccepted", [sRequestId])
        );
      } catch (oError) {
        MessageBox.error(
          oView.getModel("i18n").getResourceBundle()
            .getText("checkCommandBeforeResubmitting", [sRequestId])
        );
      } finally {
        oAction.destroy();
        oUi.setProperty("/busy", false);
      }
    }
  });
});
```

Configure the default `sap.ui.model.odata.v4.ODataModel` with service URL `/logistics/`, `synchronizationMode: "None"`, and server-supported metadata. This same-origin path must be routed to the custom backend. The secure-context browser must support `crypto.randomUUID`; choose a supported UUID utility for older browsers instead of generating weak identifiers.

The success message means the replan request was accepted, not that optimization finished. An invocation promise does not contain the returned entity directly; use the operation binding's bound context for action results when needed. The example intentionally performs no automatic mutation retry and no post-command refresh that could make a successful command look unsuccessful if a later read fails.

## Concurrency and idempotency

The backend compares `ExpectedVersion` atomically with the freight order's current version. It should also use its native concurrency mechanism when calling the authoritative system. For ordinary entity changes, honor ETags and conditional requests; do not assume a bound action automatically enforces your business version rule.

Store command receipts before dispatching external work. Scope the idempotency key by tenant, action, and target; retain a payload hash. A repeat with the same key and payload returns the original outcome. A reused key with a different payload is a conflict. Concurrent duplicate requests must converge through a unique constraint or equivalent atomic reservation, not a check-then-insert race.

For a network timeout, retain the request identifier and query its receipt. A production command journal should survive navigation and browser recovery using an appropriately protected design. Resending with a fresh identifier is a new business command. Only replay the original command under the service's explicit idempotency contract after checking its outcome.

## Exceptions and compensation

If a carrier rejects a committed order, dispatch decides whether to retender, choose another carrier, or escalate. Release allocated resources through the owning service rather than merely hiding their rows. If loading has begun, replanning may require stopping warehouse activity and confirming the physical state before a replacement plan is valid.

If a delivery is canceled after departure, do not roll the movement back to “not started.” Initiate a return, redirection, or exception workflow with authorization and evidence. Record cost impacts separately. If goods issue reversal is required, the inventory-owning process decides whether it is allowed. Transportation cancellation alone must not fabricate stock.

## Audit, security, and privacy

Persist command identity, actor identity, delegated technical identity, authorization scope, old/new versions, reason, and external correlation IDs. Keep business audit trails separate from diagnostic logs. Do not place delivery addresses, signatures, access tokens, or unrestricted event payloads in browser consoles or monitoring labels.

Enforce row-level authorization and action-level checks on every backend request, including receipt lookup, optimization jobs, attachments, and exports. Hiding a release button is usability, not authorization. Use transport encryption, approved destinations, CSRF protections appropriate to the service, and short-lived credentials. Validate file content and access rights before serving proof-of-delivery attachments. Carrier links should not reveal predictable unrestricted document URLs.

## Verification scenarios

Test state transitions as a matrix, including prohibited transitions and each role. Race two planners assigning the final available capacity. Submit the same release key concurrently. Deliver a timeout after the backend commits and verify receipt recovery without duplicate carrier messages. Send a late milestone, a correction, and a duplicate carrier event.

Test partial deliveries, split demand, unit conversion, daylight-saving boundaries, midnight crossings, missing coordinates, disconnected optimization workers, expired proposals, and a carrier whose clock is wrong. UI tests should assert focus recovery, busy-state cleanup, error accessibility, and separation between accepted jobs and completed plans. Integration tests must validate the actual target system's supported APIs and document flows, not just a permissive mock.

## Operational handover

Provide a runbook mapping symptoms to owners: ingestion backlog to integration operations, infeasible plans to transport planning, missing source deliveries to delivery support, and authorization failures to security administration. Publish queue age, oldest unprocessed milestone, receipt resolution time, stale ETA count, and outbound acknowledgement backlog.

Reconciliation compares released orders with outbound tasks and carrier acknowledgements. Alert on stuck intermediate states rather than only HTTP error rates. Dead-letter replay requires a reviewed cause, the original business key, and documented deduplication behavior. Define retention for tracking events, attachments, and command receipts; do not expire receipts earlier than the permitted retry window. Give support read-only evidence before granting exceptional repair permissions.

## Implementation checklist and related guides

- [ ] Identify TM versus LE transportation and verify release-specific application support.
- [ ] Document state dimensions, authoritative owners, and cross-system reconciliation.
- [ ] Validate plan feasibility and resource versions again at commitment.
- [ ] Prove duplicate-event and ambiguous-command recovery.
- [ ] Approve carrier authorization, privacy, and evidence retention.
- [ ] Assign operational owners for every pending integration state.

Continue with [warehouse execution](./warehouse-management.md), [sales promises](./sales.md), [worklist design](../03-UI-PATTERNS/worklist-pattern.md), [data binding](../02-ARCHITECTURE-PATTERNS/data-binding.md), [service examples](../09-CODE-EXAMPLES/services.md), [authorization](../05-SECURITY/authorization.md), and [testing strategy](../06-BEST-PRACTICES/testing-strategy.md). These paths match the repository structure specified in the [root README](../README.md).

## Official sources and research limitations

Retrieved on 2026-10-17:

1. SAP, [OData V4 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v4-model-5de13cf.md): supported model capabilities, server filtering, and operation invocation.
2. SAP, [OData Operations](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-operations-b54f789.md): deferred bindings, `invoke`, parameters, and result binding behavior.
3. SAP OpenUI5, [ODataContextBinding implementation and API documentation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataContextBinding.js): relative bound actions and request lifecycle.
4. SAP, [Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) and [SAP S/4HANA documentation entry point](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE): required deployment checks; unavailable to this research environment.

Sources support framework behavior, not the illustrative logistics service contract. Domain states, service boundaries, optimization controls, and operating procedures are design recommendations requiring validation with the implementation's business owners. GitHub `main` and `master` describe evolving versions; pin documentation to the deployed UI5 version before delivery.
