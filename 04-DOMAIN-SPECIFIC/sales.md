# Sales: Quotations, Orders, Returns, and Performance

## Scope and deployment assumptions

This guide designs a practical sales workspace spanning quotations, sales orders, fulfillment tracking, customer returns, and performance dashboards. Assume a UI5 client, a custom OData V4 façade, an authoritative SAP sales process, and asynchronous integrations with credit, availability, logistics, billing, and analytics. The process descriptions are deliberately product-neutral where SAP editions or releases differ.

All `/sales/` routes, `Demo.Sales` operations, entities, and state labels below are **illustrative custom contracts**, not official SAP API URLs, standard application names, or SAP status codes. Implement backend adapters using supported APIs and released extensibility mechanisms for the actual target. Direct database writes to sales, pricing, delivery, or accounting tables are not a valid shortcut.

Before building, use the [SAP Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) with the exact SAP product version to assess standard functionality. Verify business catalogs, deployment prerequisites, supported extension points, and application configuration. Names and capabilities can change; an application available in one edition is not evidence that it exists in another. No fixed application IDs are assumed.

## Research boundary

Official SAPUI5 documentation was retrieved from SAP's GitHub repositories for the V4 model and operation bindings. The reference-library and Help Portal hosts could not be resolved in this research environment. Therefore this chapter does not claim that a particular standard sales application, API, or return scenario has been verified for a specific release. Treat product selection and release-specific API validation as mandatory discovery tasks.

The custom workflow is an architectural example. Its terms “submitted,” “approved,” and “fulfilled” must be mapped explicitly to the installed application's statuses, blocks, and document flow. Avoid storing one custom lifecycle flag that contradicts the source application's actual processing state.

## Roles and separation of duties

A sales representative prepares quotes and customer orders within an assigned sales area. A sales manager approves exceptional discounts or terms. Credit specialists own credit decisions; a salesperson cannot bypass a credit block by changing a frontend property. Fulfillment teams manage delivery execution. Billing owns invoices and credit documents. Returns specialists coordinate authorization, receipt, inspection, and disposition. Analysts consume governed measures.

Permissions depend on sales organization, distribution channel, division, customer responsibility, and business action. A manager may approve a discount but have no authority to release credit. Service identities must not gain unrestricted access simply because the frontend already filtered records. Define delegation, temporary coverage, and approval expiration rather than sharing accounts during absences.

## End-to-end sales workflow

1. Identify the customer and validate partner roles and sales-area eligibility.
2. Prepare a quotation with products, requested quantities, validity, and commercial terms.
3. Calculate pricing and obtain approvals for exceptions.
4. Issue the versioned quotation and record customer acceptance or rejection.
5. Convert an eligible accepted quotation into an order with preserved references.
6. Determine the order's availability, credit, and compliance outcomes.
7. Track confirmations, deliveries, goods movement, billing, and customer exceptions.
8. Initiate a controlled return when required.
9. Reconcile return receipt, inspection, credit, and financial effects.
10. Feed governed operational and commercial metrics.

Each step has a different authoritative owner. An order appearing in a worklist does not prove that availability confirmation, delivery creation, or billing has completed.

## Quotation lifecycle and versioning

Use an explicit quotation revision: Draft → SubmittedForApproval → Approved → Issued → Accepted, Rejected, or Expired. This is a proposed custom model, not a standard SAP status sequence. A revised quotation should invalidate or supersede its earlier approval according to commercial policy. An accepted revision must be immutable enough to explain exactly what the customer accepted.

Record validity dates as business dates in an agreed business time zone. Define whether acceptance must occur before the end of the final day or at a precise instant. Preserve issued document artifacts and pricing snapshots. Do not recalculate an old issued quotation in place and then claim its new total was always offered.

Partial acceptance needs an explicit rule. If ten quoted units are accepted now and five later, track conversion quantities and prevent over-conversion through a server-side quantity invariant.

## Atomic block: convert quotation to order

**Inputs:** quotation identifier, accepted revision, selected eligible items, expected version, and an idempotency key.

**Preconditions:** authorized actor, valid customer and sales area, eligible quotation state, unexpired terms where required, remaining conversion quantity, and required approvals still valid.

**Effects:** reserve or consume conversion quantities, create the order or a durable creation request, record the quotation/order relationship, and persist a command receipt. If the SAP order is created asynchronously, return “creation pending” rather than inventing a final order number.

**Exception path:** if the external response is lost, reconcile using the supported external reference before attempting another creation. A local transaction cannot roll back a remote order that already exists. Provide a manual investigation queue when the target API cannot safely correlate ambiguous outcomes.

## Order status is multidimensional

An order can be commercially accepted but credit-blocked, partially confirmed, partly delivered, and partially billed. Model these dimensions independently. A dashboard's “open” category needs a definition: unfulfilled quantity, unbilled value, unresolved exceptions, or some combination.

| Dimension | Example facts | Responsible owner |
|---|---|---|
| Commercial | Draft, accepted, rejected item | Sales |
| Credit | Not checked, blocked, released | Credit team |
| Supply | Requested, confirmed, backordered | Availability process |
| Fulfillment | Not delivered, partial, complete | Logistics |
| Billing | Not billed, partial, complete | Billing |
| Returns | Authorized, received, resolved | Returns team |

Do not let a user directly PATCH `FulfillmentStatus` to “complete.” Commands must enforce real business prerequisites and update the status through the authoritative process.

## Business data model

A custom sales read model might expose `Quotations`, `QuotationItems`, `SalesOrders`, `SalesOrderItems`, `ScheduleLines`, `DocumentLinks`, `ReturnRequests`, `ReturnItems`, `ApprovalDecisions`, and `CommandReceipts`.

Each header carries immutable technical ID, source system, business number, sales area, customer references, document currency, version token, and timestamps. Items carry product reference, requested quantity/unit, relevant pricing components, rejection reason, and source item reference. Schedule lines distinguish requested and confirmed dates and quantities. Document links retain item-level relationships across splits, deliveries, invoices, and returns.

Do not store money as floating-point arithmetic in business logic. Use decimal types with currency metadata and server-calculated rounding. Distinguish document currency from reporting currency. Quantities require units and controlled conversion rules. A cancellation or credit is not simply a negative number inserted into the wrong entity.

## Commercial invariants

Only valid partner combinations can be used for the selected sales area. Approvals apply to the exact commercial revision and authorized discount scope. The total converted quantity cannot exceed the quotation's permitted quantity. Delivered quantities cannot be reduced by editing the order as if no delivery existed.

Return authorization must consider original eligible quantities, prior returns, product identity, policy, and legal requirements. A commercial credit may be approved without a physical return in some business policies, but that must be explicit; do not force fictitious warehouse receipts. Conversely, receiving returned goods does not automatically authorize every requested credit amount.

Protect these invariants in the backend transaction or source process. Browser calculations are useful previews, not authoritative price, tax, credit, or quantity decisions.

## Pricing, availability, and credit boundaries

Pricing owns conditions, validity, scales, rounding, and applicable approval controls. Availability owns confirmations based on the target planning process. Credit owns credit evaluation and release. Tax determination may involve additional services. The sales façade orchestrates these results without replacing them with browser heuristics.

Present each result's timestamp and version. A price calculated before an address change may no longer apply. A confirmed date can change after supply replanning. A credit release can be withdrawn. Before final submission, the backend revalidates the complete order snapshot and reports which inputs changed. Use a deliberate “review updated terms” step when commercially significant recalculation occurs; never silently accept a changed customer price.

## Approval escalation and customer communication

Approval requests need a defined decision scope, commercial snapshot, approver rule, deadline, and substitute rule. Escalation changes responsibility; it should not silently change the permitted discount threshold. Record rejection reasons and distinguish a request sent back for correction from a terminal rejection. A sales representative who changes payment terms after approval must trigger the applicable revalidation rather than inherit approval for unrelated terms.

Customer communications require their own delivery record. An issued quotation can exist even if email delivery failed. Retrying email transmission must not create a second quotation revision or repeat order conversion. Store the approved document reference, recipient decision, channel, delivery status, and communication correlation identifier, while observing privacy and retention policy.

For changed delivery promises, distinguish a proposed date from one accepted by the customer. Capture the acceptance evidence where policy requires it. A customer service note saying “called customer” is not equivalent to contractual acceptance. These distinctions make later service disputes and performance analysis explainable without exposing unrestricted internal notes to customers.

## Sales order entry experience

Use progressive disclosure: customer and sales area first, then products and quantities, then requested dates and terms. Provide value helps filtered by server-side eligibility. A product description is not a stable identifier. Preserve entered values when validation fails and focus the first actionable message.

For long orders, support item-level validation and a clear header summary. Avoid saving each field automatically if the process requires a coherent approval snapshot. Where drafts are supported, distinguish draft ownership and lifecycle from active order state. Do not advertise backend draft semantics merely because a JSONModel temporarily holds unsaved values.

Prevent navigation loss through an explicit unsaved-change policy. An accepted command in progress is different from an unsaved draft; abandoning the page must not cancel work already committed on the server.

## Contextual UI5 XML: order submission

The page has already been bound by routing to a valid `SalesOrders` context from the custom V4 service. The `CanSubmit` property is a server-computed usability hint. The backend still authorizes and validates the action. Resource bundle keys and the `ui` JSON model are initialized by the application.

```xml
<mvc:View controllerName="demo.sales.controller.Order"
  xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
  <Page title="{OrderNumber}" busy="{ui>/busy}">
    <content>
      <ObjectHeader title="{CustomerName}" number="{NetAmount}"
        numberUnit="{Currency}"/>
      <ObjectStatus title="{i18n>credit}" text="{CreditStatus}"/>
      <ObjectStatus title="{i18n>fulfillment}" text="{FulfillmentStatus}"/>
      <Table items="{
        path: 'Items',
        parameters: {
          $select: 'ID,ProductDescription,RequestedQuantity,Unit,ConfirmedDate'
        }
      }">
        <columns>
          <Column><Text text="{i18n>product}"/></Column>
          <Column><Text text="{i18n>quantity}"/></Column>
          <Column><Text text="{i18n>confirmedDate}"/></Column>
        </columns>
        <items>
          <ColumnListItem>
            <cells>
              <Text text="{ProductDescription}"/>
              <ObjectNumber number="{RequestedQuantity}" unit="{Unit}"/>
              <Text text="{ConfirmedDate}"/>
            </cells>
          </ColumnListItem>
        </items>
      </Table>
    </content>
    <footer>
      <OverflowToolbar>
        <ToolbarSpacer/>
        <Button text="{i18n>submitOrder}" type="Emphasized"
          enabled="{= ${CanSubmit} &amp;&amp; !${ui>/busy} }"
          press=".onSubmit"/>
      </OverflowToolbar>
    </footer>
  </Page>
</mvc:View>
```

Format dates and decimals using metadata-aware types in the production view; include explicit display-currency formatting where the service metadata does not supply it. Never communicate credit blocks using color alone. For message navigation, associate backend validation targets with the corresponding editable fields.

## Contextual UI5 JavaScript: guarded submission

The illustrative bound action `Demo.Sales.Submit` accepts `RequestId` and `ExpectedVersion` as `Edm.String`. It returns a durable receipt. The deployed UI5 version must support the V4 operation-binding `invoke` method. The application supplies its default V4 model with `/sales/` as its same-origin service URL.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox"
], function (Controller, JSONModel, MessageBox) {
  "use strict";
  return Controller.extend("demo.sales.controller.Order", {
    onInit: function () {
      this.getView().setModel(new JSONModel({ busy: false }), "ui");
    },
    onSubmit: async function () {
      const oView = this.getView();
      const oUi = oView.getModel("ui");
      const oModel = oView.getModel();
      const oContext = oView.getBindingContext();
      const oBundle = oView.getModel("i18n").getResourceBundle();
      if (oUi.getProperty("/busy") || !oContext) {
        return;
      }
      if (oModel.hasPendingChanges()) {
        MessageBox.warning(oBundle.getText("saveChangesBeforeSubmit"));
        return;
      }
      const sRequestId = globalThis.crypto.randomUUID();
      const oOperation = oModel.bindContext(
        "Demo.Sales.Submit(...)", oContext
      );
      oOperation.setParameter("RequestId", sRequestId);
      oOperation.setParameter("ExpectedVersion",
        oContext.getProperty("Version"));
      oUi.setProperty("/busy", true);
      try {
        await oOperation.invoke("$direct");
        MessageBox.information(
          oBundle.getText("submissionAccepted", [sRequestId])
        );
      } catch (oError) {
        MessageBox.error(
          oBundle.getText("verifySubmissionOutcome", [sRequestId])
        );
      } finally {
        oOperation.destroy();
        oUi.setProperty("/busy", false);
      }
    }
  });
});
```

Configure the default model as `sap.ui.model.odata.v4.ODataModel` with `serviceUrl: "/sales/"` and `synchronizationMode: "None"`. `crypto.randomUUID` requires a supported secure browser context. Saving edits is a separate explicit workflow: inspect individual failed changes, not merely the batch transport result. `hasPendingChanges` prevents submission of a visibly edited order whose action would otherwise use older server data.

No automatic retry occurs after a timeout. In a complete application, retain the request identifier in a protected command journal and provide receipt lookup. The message is deliberately “accepted,” not “delivered” or “billed.” Use the operation binding's returned context if receipt details must be displayed; `invoke` does not resolve directly to the returned business entity.

## Concurrency and command recovery

Two representatives can edit the same order, and pricing or credit can change independently. Use source-supported ETags for entity changes and an explicit expected version for the custom action. Reject stale changes with a conflict that identifies the changed scope without disclosing unauthorized fields. Offer reload and review; do not overwrite another person's changes through a blind force-save.

Persist idempotency receipts keyed by tenant, target, operation, and request identifier. Compare payload hashes and reject key reuse for a different command. A unique database constraint or equivalent atomic mechanism prevents simultaneous duplicates. Propagate an appropriate correlation reference to the source system if supported.

After an ambiguous timeout, query the receipt and the authoritative document relationship. Do not retry order creation using a new identifier merely because the browser did not receive a response. Where an external system lacks idempotency support, serialize ambiguous creation attempts and require reconciliation before any controlled resubmission.

## Tracking the fulfillment document flow

Show the order at item and schedule-line granularity. One item can have multiple confirmations, deliveries, and invoices. Aggregate quantities only after normalizing units and understanding rejected quantities. Link to source documents through authorized navigation targets; a document number alone does not grant access.

Make delays explainable: credit block, incomplete data, supply shortage, delivery block, warehouse delay, transport disruption, or billing issue. Provide owner and next action. A frontend timeline assembled from event arrival order can mislead users; order events by business occurrence time while retaining receipt time and correction history.

Customer-facing tracking should expose approved milestones, not internal pricing exceptions or employee notes. Apply a separate response projection and access policy rather than reusing the internal service with hidden columns.

## Returns workflow

Start with a return request referencing the original order, delivery, or invoice as appropriate to the supported process. Capture item, quantity, unit, reason, condition, serial or batch evidence where required, and requested remedy. Validate eligibility before promising a refund. Authorize the return and provide controlled instructions for shipment or drop-off.

Track physical receipt separately from commercial approval. Inspection determines condition and disposition: restock, repair, scrap, return to supplier, or quarantine under the actual warehouse and quality processes. Billing creates the appropriate credit or follow-on document. Resolve deductions, restocking fees, and tax effects under policy rather than calculating an arbitrary frontend refund.

Exception paths include unidentifiable goods, excess quantity, damaged transport packaging, duplicate requests, missing accessories, disputed serial numbers, and credit already issued. Keep these exceptions visible until an accountable decision is recorded.

## Atomic block: authorize a return

Check the customer's entitlement, original eligible quantity, prior authorized and completed returns, reason-specific policy, and approver authority. Reserve the permitted return quantity atomically so two agents cannot authorize the same remaining units. Record authorization expiration and whether unreceived quantities release the reservation.

Authorization is not receipt, inspection, or credit. If goods arrive after expiration, route to a controlled exception instead of deleting the historical authorization. Reversing a credit or physical receipt belongs to its authoritative process and may require additional financial or inventory controls. A cancel button should state exactly which layer is being canceled.

## Metrics and dashboard definitions

Define every measure with grain, timestamp, denominator, exclusions, currency basis, and refresh latency. Quote conversion can mean quotations converted by count or quoted value converted; these are different metrics. Conversion cohorts should use a stable issue period rather than compare today's conversions with today's newly created quotes.

Order intake, backlog, delivered value, invoiced revenue, and recognized revenue are not synonyms. A sales operational dashboard must not label order value as accounting revenue. Return rate can use units, value, or order count and should align returns with the original sales cohort when answering product-quality questions.

On-time fulfillment needs a defined baseline promise, measurement event, and tolerance. Freeze or version the original commitment to prevent performance from improving simply by moving the target date after a delay.

## Dashboard architecture and drilldown

Compute governed aggregates server-side or in an analytical service. Do not download every order to the browser and sum mixed currencies. Return valuation date, conversion-rate basis, data-as-of timestamp, and row-level authorization scope. Ensure total tiles and drilldown lists apply the same filters and definitions.

Separate operational queues from historical analytics. A near-real-time exception count can coexist with a nightly financial aggregate, but label both clocks. Avoid summing order value after joining orders to deliveries and invoices, which multiplies rows. Aggregate at the declared grain before combining measures. Reconcile sample documents to source totals before making the dashboard a management target.

## Security, audit, and privacy

Enforce authorization for reads, actions, exports, attachments, analytics, and receipt queries. Use backend row restrictions and action checks; frontend `CanSubmit` is not a security boundary. Customer personal information, negotiated prices, credit assessments, and payment details need distinct access and retention policies.

Audit pricing exceptions, approval decisions, partner changes, quotation issuance, order submission, return authorization, and credit-related actions. Capture actor, acting role, reason, before/after version, correlation ID, and authoritative document references. Do not copy unrestricted payloads or credentials into logs. Scan and restrict attachments, prevent unsafe HTML rendering, and protect spreadsheet exports against formula injection where applicable.

## Testing and operational handover

Test quote expiration boundaries, partial conversion, concurrent conversion, approval invalidation after repricing, credit changes between load and submit, duplicate submission, lost responses, and stale version conflicts. Test split deliveries, canceled items, mixed units, multi-currency reporting, partial returns, over-returns, and credits without physical returns under explicitly permitted policy.

UI tests should cover keyboard navigation, meaningful validation targets, empty and unauthorized states, and recovery after failed submissions. Contract tests should reject forbidden direct state changes. Dashboard checks must catch join multiplication and inconsistent drilldown filters.

Publish runbooks for stuck order creation, missing document links, approval backlog, return quantity discrepancies, and delayed analytical refresh. Assign owners across sales, integration, credit, logistics, and billing. Reconcile command receipts with created documents and source status. Restrict repair actions, preserve evidence, and never advise support to replay all failed POST requests indiscriminately.

## Delivery checklist and cross-links

- [ ] Verify standard applications and APIs for the exact release and edition.
- [ ] Define quotation revisions, order status dimensions, and approval ownership.
- [ ] Enforce quantity, price, and return invariants on the backend.
- [ ] Demonstrate stale-edit and ambiguous-submission recovery.
- [ ] Reconcile dashboard measures to authorized source documents.
- [ ] Hand over integration and business exception queues to named teams.

Related repository paths: [logistics](./logistics.md), [finance](./finance.md), [warehouse management](./warehouse-management.md), [transactional patterns](../03-UI-PATTERNS/transactional-patterns.md), [overview dashboards](../03-UI-PATTERNS/overview-dashboard.md), [data protection](../05-SECURITY/data-protection.md), [controller examples](../09-CODE-EXAMPLES/controllers.md), and [testing strategy](../06-BEST-PRACTICES/testing-strategy.md). They follow the [root README](../README.md) structure.

## Official references

Research date: 2026-10-17.

1. SAP, [OData V4 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v4-model-5de13cf.md), retrieved: model capabilities and official Sales Orders sample reference. That sample is a framework demonstration, not a production sales API contract.
2. SAP, [OData Operations](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-operations-b54f789.md), retrieved: deferred bound actions, primitive parameters, invocation promises, and result contexts.
3. SAP OpenUI5, [ODataModel source and API documentation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js), retrieved: model parameters and request grouping.
4. SAP, [Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) and [SAP S/4HANA documentation](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE), required release-specific verification sources; hosts unavailable during research.

The workflow, entity schema, metrics, and exception controls are original design guidance, not quotations from SAP process documentation. Validate them with business owners and the deployed product's supported interfaces. Pin UI5 documentation to the chosen runtime rather than assuming the evolving GitHub branch matches an older system.
