# Procurement: From Request to Verified Supplier Liability

## Scope and assumptions

This guide follows an indirect-material purchase from employee request through purchase requisition, purchase order, approval, receipt, and invoice matching. The working example is replacement laboratory equipment purchased for a maintenance cost center. It emphasizes operational decisions and UI5 interaction design rather than reproducing a particular SAP screen. Service procurement, subcontracting, consignment, and stock transport orders require additional document semantics; they must not inherit this example's receipt rules unchanged.

The business discussion uses SAP purchasing concepts. Every entity, property, action, status, and endpoint introduced in code is an **illustrative custom OData V2 contract**, not an SAP-delivered API or a promise about a standard application. An implementation must map these concepts to supported interfaces for its actual product, deployment, and release. Flexible workflow examples are conditional on the installed and configured scenario. Classic release procedures and flexible workflow are not interchangeable configuration instructions.

Use this chapter alongside [Finance](finance.md) for liability and payment consequences, [Warehouse Management](warehouse-management.md) for physical execution, and the [repository document index](../README.md#document-index) for navigation. A purchase order authorizes a commercial commitment; it does not prove delivery, invoice accuracy, or payment.

## Discover the standard application before designing an extension

Start in the [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/). Record the target product and release, deployment model, application type, business role, supported devices, implementation information, and relevant help links. Search by business outcome as well as title: “Create Purchase Order,” “Manage Purchase Orders,” “Create Purchase Order - Advanced,” and “My Inbox” are useful discovery terms, not assertions that every system contains identically named tiles.

For PO approvals, inspect both the purchasing application's approval capabilities and the relevant My Inbox scenario. Establish whether the organization uses flexible workflow, another supported release mechanism, or an integrated process. Verify available decisions, agent determination, substitution, attachments, and task visualization. Do not infer that a workflow exists because someone can open My Inbox. Conversely, an absent launchpad tile might indicate catalog or space assignment, not missing backend functionality.

Compare the standard application against an explicit gap list. A bespoke receiving kiosk may be justified; cloning order maintenance merely to rearrange fields usually creates unnecessary upgrade and authorization work. Where supported, prefer configuration, adaptation, or documented extension points. Navigate to standard applications through configured intent-based navigation rather than concatenating guessed transaction URLs.

The official purchasing workflow and My Inbox references at the end describe supported concepts in specific documentation contexts. Their presence does not establish universal prerequisites or availability. Keep an implementation worksheet containing the library selection date and the actual system's checks. Avoid hard-coding app identifiers from an unrelated release into architectural guidance.

## Business objects and ownership

A purchase requisition expresses an internal demand. A purchase order records the purchasing organization's order to a supplier. Requisition approval may authorize spending intent without selecting final commercial conditions; order approval may evaluate the negotiated price, supplier, and terms. One requisition can feed multiple orders, and one order can consolidate demands. Preserve item-level references rather than assuming a one-to-one document relationship.

A supplier identifier is not simply an editable vendor name. In S/4HANA scenarios, supplier-related master data is maintained through the business partner approach, with relevant roles and organizational data. ECC landscapes have different master-data arrangements. Resolve the actual system's model instead of imposing a universal migration story. A business partner can exist without being eligible for the requested purchasing organization or company code.

For the example order, persist supplier key, purchasing organization, purchasing group, company code, plant, delivery address, account assignment, currency, payment terms, and item details. Item details include material or description, quantity, order unit, delivery date, price unit, and reference requisition item. Account-assigned consumption and stock procurement have different accounting outcomes. Displaying a cost center field is not evidence that the selected account-assignment category permits it.

| Responsibility | Typical operator | Required boundary |
|---|---|---|
| Explain demand and delivery need | Requester | Own organizational scope |
| Select source and negotiate terms | Buyer | Assigned purchasing organizations |
| Approve expenditure | Approver | Valid task and delegated authority |
| Confirm physical receipt | Receiver | Relevant plant and receipt authority |
| Verify supplier invoice | Accounts payable | Company code and invoice permissions |
| Resolve master-data defects | Master-data steward | Controlled change workflow |

These roles describe separation of duties, not delivered SAP role names. Local policy should decide whether requesters can receive their own purchases and whether buyers may approve orders they created.

## Workflow walkthrough: the maintenance purchase

### 1. Capture a demand that purchasing can act upon

The requester needs twelve replacement instruments next month. Ask for a usable description, quantity and unit, required delivery date, delivery location, and charge object. Store attachments with controlled access and malware inspection. A technical specification may contain sensitive drawings; a URL to an ungoverned public file share is not an acceptable substitute for attachment governance.

Validate organizational combinations early through value help filtered by authorization. Still repeat all checks when saving. Cost centers expire, budgets change, and a saved draft may survive a reorganization. Distinguish missing mandatory data from a policy exception: a missing plant is a correction; a request outside a preferred supplier agreement may require explanation and approval.

Present a review page showing the estimated commitment, not just unit price. Explain whether freight and tax estimates are included. A requester entering “12” must see whether the unit is each, box, or case. Capture when the price estimate was obtained. Treat a budget availability preview as advisory unless the backend transaction actually reserves or commits funds according to the configured process.

### 2. Submit the requisition and route approval

Submission should persist the request and transition it into the configured approval process atomically where possible. If workflow creation is asynchronous, show “Submitted; routing pending,” not “Awaiting manager approval” before an agent exists. Provide a support-visible correlation identifier and a reconciliation queue for requests whose workflow could not start.

The approval snapshot should include the requester, account assignment, estimated amount and currency, relevant attachments, and policy exceptions. Route by backend rules, not browser-calculated thresholds. A threshold comparison requires a defined valuation currency, conversion date, rate source, and rounding rule. A browser comparing two floating-point totals can neither establish authority nor provide reliable financial control.

An approver needs consequences, not just green and red buttons. Show what will be authorized, unresolved exceptions, and whether rejection returns the request for correction or ends the process. Require a rejection reason where policy demands it. Store the task version and document version considered by the decision so later changes cannot silently inherit an earlier approval.

### 3. Source and construct the purchase order

The buyer reviews released demand, checks available contracts or source arrangements, and confirms supplier eligibility. An order split may be necessary for different suppliers, currencies, plants, or organizational constraints. Explain splits to the requester using linked item histories; otherwise users mistake multiple orders for duplicate purchasing.

Price semantics deserve a dedicated review. A price of 240 EUR per 100 units is not 240 EUR per unit. Order unit and pricing unit can differ, and conversion factors must come from governed master data or explicitly approved transaction data. Show net value, conditions, and expected delivery schedule. Validate that rounding and minimum order quantities agree with the supplier arrangement.

Before submission, display differences from the approved requisition: quantity increases, changed supplier, new account assignment, and revised price. Decide which differences trigger renewed approval. Do not assume requisition release automatically releases the resulting order. Store references to the demand being fulfilled so downstream analysis can explain both outstanding requisition quantities and outstanding order quantities.

### 4. Approve, communicate, and amend the order

Order approval and supplier communication are distinct outcomes. A released order may still await output generation, transmission, or acknowledgement. Record these separately, including the order revision actually transmitted. A failed email or integration message should not reset a valid approval or cause another purchase order to be created.

Material changes after approval require a documented rule: restart approval, require a change approval, or allow only designated nonmaterial amendments. The rule must be evaluated server-side against the previously approved snapshot. If changing quantity restarts workflow, explain the resulting approval status before the buyer confirms the change.

Track supplier acknowledgement separately from internal release. Acknowledged dates can differ from requested dates and influence expediting. A supplier cancellation is a commercial event, not a deletion instruction. Preserve the original order, changes, outputs, and affected receipt or invoice references. Never remove posted history to make the order appear unused.

### 5. Receive goods without hiding exceptions

The receiver selects an eligible order item and records what physically arrived. In the example, eight instruments arrive first and four later. The first receipt should leave an explicit open quantity. Whether the order is considered delivery-complete is a separate governed decision, especially when short delivery is accepted.

Capture delivery reference, quantity and unit, receiving location, relevant batch or serial identifiers, and posting date. Distinguish arrival, stock receipt, inspection, and putaway according to the landscape. A UI that calls all four “Received” will mislead buyers and accounts payable. For warehouse-managed locations, coordinate with the warehouse process rather than creating a second independent confirmation channel.

Damage, wrong material, and overdelivery need explicit exception paths. A receiver should not inflate receipt quantity to clear an invoice mismatch. Returns and reversals must reference their original documents and respect subsequent activity. If a receipt was posted against the wrong order, show a supported reversal-and-repost path, not an unrestricted edit of posted quantities.

### 6. Match the invoice and resolve differences

For a three-way matching scenario, compare the order's commercial conditions, eligible receipt history, and supplier invoice. Matching depends on configuration: not every procurement category requires the same receipt reference, tolerance, or service acceptance process. Services may need an appropriate acceptance or service-entry process instead of a material receipt. Label the specific matching basis in the UI.

An invoice for twelve instruments after only eight have arrived is an exception to investigate, not proof of fraud or automatic evidence that the receipt is missing. The supplier may have billed early, delivery may be unrecorded, or a receipt may be posted elsewhere. Route the discrepancy to the responsible buyer or receiver while preserving invoice due-date visibility.

An approved order does not authorize payment of every subsequent invoice. Duplicate checks, tax validation, payment blocks, and accounting period controls remain relevant. Handover to [Finance](finance.md) should include supplier reference, invoice currency, posting outcome, matched documents, exception decisions, and any payment block. Avoid a procurement dashboard showing “Complete” while an invoice remains unresolved.

## State machines: related, not collapsed

The following are custom business states for this guide. They are not SAP status codes.

```text
Requisition:
Draft -> Submitted -> Routing -> InApproval -> Released
                           \-> RoutingError
InApproval -> Rejected -> Draft(new revision)
Released -> Converted(partial or complete, derived from item links)

Order approval:
Draft -> Submitted -> InApproval -> Released
InApproval -> Rejected
Released -> ChangePending -> InApproval

Order execution dimensions:
Output: Pending -> Sent -> Acknowledged | TransmissionError
Receipt: None -> Partial -> Fulfilled | Exception
Invoice: None -> Partial -> Matched | Blocked
```

Receipt and invoice states must be derived using quantities and document relationships, not only timestamps. Reversed receipts reduce net receipt quantity. Credit memos affect liability without necessarily reversing physical receipt. An order can be fully received and partially invoiced, or partially received and invoice-blocked. A single linear progress bar may summarize the journey, but drill-down must reveal each independent dimension.

| Event | Preconditions | Durable outcome | User recovery |
|---|---|---|---|
| Submit requisition | Complete valid draft, current revision | Submission recorded | Repair validation errors |
| Approve order | Active task, authorized agent, matching revision | Decision and next workflow step | Refresh stale task |
| Post receipt | Eligible item, valid quantity and location | Receipt reference | Investigate ambiguous timeout |
| Invoice exceeds tolerance | Valid invoice, mismatch found | Exception or block as configured | Assign owner and evidence |
| Reverse receipt | Referenced document, permitted follow-on state | Reversal reference | Resolve downstream dependencies |

Closing an order should be a deliberate business action or derived conclusion with documented rules. A manually closed indicator is not a substitute for explaining residual quantities, outstanding invoices, or commitments.

## Illustrative custom OData contract

Assume a gateway named `/example/procurement/` exposes the following V2 model. The name is deliberately illustrative; do not configure it as a standard SAP endpoint. Its metadata must be implemented and published by the custom service owner.

| Entity set | Selected fields | Contract semantics |
|---|---|---|
| `PurchaseOrders` | `OrderId`, `Revision`, `Currency`, `NetAmount`, `ApprovalState` | ETag protects each editable revision |
| `OrderItems` | `OrderId`, `ItemId`, `Quantity`, `Unit`, `Price`, `PriceUnit` | Composite keys; decimals encoded as strings |
| `ApprovalCommands` | `RequestId`, `OrderId`, `ExpectedRevision`, `Decision`, `Comment` | Create-only command with deduplication |
| `CommandResults` | `RequestId`, `Outcome`, `DocumentReference`, `ErrorCode` | Authorized lookup after uncertain response |

`OrderId`, `ItemId`, `RequestId`, `Revision`, and `ExpectedRevision` are strings. `Quantity`, `NetAmount`, and `Price` are `Edm.Decimal`; `PriceUnit` also has an explicit numeric definition in metadata. A UUID request identifier is represented as `Edm.String` here to keep the example unambiguous. `Decision` accepts only the contract's enumeration, such as `APPROVE` and `REJECT`.

Creating an approval command is an application-level command pattern, not a claim that any OData entity creation automatically performs workflow approval. The backend must authenticate the caller, authorize the decision, lock or compare the target revision, validate the active task, and persist the decision together with the deduplication record. Its response includes an authoritative outcome. A successfully accepted command can still be processing if the contract explicitly supports asynchronous execution.

Deduplication scope includes tenant and business operation; retain records for a defined retry horizon. Reusing a request identifier with different payload content must fail. Looking up a command result must enforce the same organizational authorization as the original operation. Do not expose other users' document existence through an unrestricted request-identifier search.

## UI5 example: approval review, not a replacement workflow engine

The view uses a read-only OData context named `po` and a local JSON model named `ui`. Initialize `ui` with `busy: false`, `canApprove: false`, and `comment: ""`. The backend supplies a capability hint for presentation; it is not an authorization token. The controller binds the selected order only after model metadata is ready.

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  controllerName="example.procurement.Approval">
  <VBox class="sapUiSmallMargin" busy="{ui>/busy}">
    <Title text="{po>OrderId}" />
    <ObjectNumber number="{
      path: 'po>NetAmount',
      type: 'sap.ui.model.odata.type.Decimal',
      constraints: {precision: 19, scale: 4}
    }" unit="{po>Currency}" />
    <ObjectStatus text="{po>ApprovalState}" />
    <Label text="Decision comment" labelFor="decisionComment" />
    <TextArea id="decisionComment" value="{ui>/comment}"
      maxLength="500" />
    <Button text="Approve" press=".onApprove"
      enabled="{= ${ui>/canApprove} &amp;&amp; !${ui>/busy} }" />
  </VBox>
</mvc:View>
```

The fixed decimal scale belongs to this example metadata, not a universal currency rule. Production monetary display should use currency-aware formatting and the service's actual decimal semantics. Keep raw decimal strings intact for transmission; never round a monetary value by calling `Number` merely to make a label attractive.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/m/MessageBox"
], function (Controller, MessageBox) {
  "use strict";
  return Controller.extend("example.procurement.Approval", {
    onApprove: function () {
      const view = this.getView();
      const ui = view.getModel("ui");
      const model = view.getModel("po");
      const context = view.getBindingContext("po");
      if (ui.getProperty("/busy") || !context) {
        return;
      }
      const order = context.getObject();
      // Retain the same intent until its outcome is reconciled.
      if (!this._approvalIntent) {
        this._approvalIntent = {
          RequestId: globalThis.crypto.randomUUID(),
          OrderId: order.OrderId,
          ExpectedRevision: order.Revision,
          Decision: "APPROVE",
          Comment: ui.getProperty("/comment") || ""
        };
      }
      ui.setProperty("/busy", true);
      model.create("/ApprovalCommands", this._approvalIntent, {
        success: function (result) {
          ui.setProperty("/busy", false);
          if (result.Outcome === "COMPLETED") {
            this._approvalIntent = null;
            ui.setProperty("/canApprove", false);
            model.refresh(true);
            MessageBox.success("Approval recorded.");
          } else {
            MessageBox.information(
              "Decision accepted. Reconcile its result before another action."
            );
          }
        }.bind(this),
        error: function () {
          ui.setProperty("/busy", false);
          MessageBox.warning(
            "Outcome not confirmed. Check the command result before retrying."
          );
        }
      });
    }
  });
});
```

This is a focused handler, not a complete application. It assumes a modern secure browser providing `crypto.randomUUID`, a default nondeferred create group, and framework-managed CSRF handling. In another browser baseline use an approved UUID provider, never a timestamp-only identifier. A V2 model configured with deferred groups needs explicit submission and response inspection. Production code must also handle route changes, request cancellation, message binding, and completion after the view is destroyed.

Disable navigation away from an unresolved intent or persist that intent in an approved recovery mechanism. In-memory retention does not survive reload. The recovery screen queries `CommandResults` using the retained identifier, keeps the original payload immutable, and permits a new intent only after a definitive failure or completion. Changing the comment while retrying must not mutate the existing command.

For editing an order rather than approving it, use metadata-generated keys and concurrency information:

```javascript
const path = "/" + model.createKey("PurchaseOrders", {
  OrderId: order.OrderId
});
model.update(path, { BuyerNote: note }, {
  eTag: order.__metadata.etag,
  success: onSaved,
  error: onSaveFailed
});
```

Here `BuyerNote` is an additional illustrative editable string property. Require an actual ETag in the contract; do not silently substitute a wildcard. A stale precondition should trigger comparison and reload, not an automatic overwrite. Approval commands use `ExpectedRevision` for their own atomic business precondition because an ETag on the newly created command would not protect the target order.

## Decimal, unit, date, and organizational semantics

Do arithmetic on an exact-decimal backend or another explicitly governed decimal implementation. Quantities need scale rules by unit, and some materials disallow fractional pieces. Store entered quantity and unit alongside normalized quantity and conversion provenance where required. A conversion from boxes to each must be reproducible after master data changes.

Distinguish net price, net item value, tax amount, freight, and gross invoice value. Never aggregate amounts across currencies without showing a conversion basis. Approval valuation may differ from the order currency. Display both when meaningful, identifying the rate date rather than implying today's conversion was approved.

Required delivery dates are business dates, not browser-local instants. Audit decisions and system events are instants, normally stored with UTC semantics and displayed with an identified user time zone. Posting dates follow company and accounting rules. A user receiving at midnight in another country must not accidentally shift a delivery date through JavaScript UTC conversion.

Organizational scope belongs on every query and command. A purchasing group filter improves usability but does not authorize access. Server-side filtering must prevent unauthorized company codes, plants, suppliers, attachments, and related documents from leaking through value helps, counts, exports, or error messages.

## Error recovery and operational support

Classify errors before choosing a retry. Validation failures require correction. Authorization failures require an appropriate role or assignment change. Stale revisions require refresh and comparison. A transient transport failure after a commit produces an unknown outcome. Retrying that last case with a new request identifier can create duplicate receipts or decisions.

Expose stable application error codes and field targets, while keeping stack traces and sensitive supplier data out of end-user messages. Include a support correlation identifier. For multi-item posting, specify whether the command is all-or-nothing or returns independent item outcomes. “Nine of ten saved” is acceptable only if the user can identify the ninth success and retry only the failed business intent.

Workflow routing failures need an operational owner and aging alert. A buyer should not cancel and recreate an order solely to regenerate a missing task. Reconciliation should inspect the submission, workflow reference, and revision, then resume or repair through supported backend procedures. Audit any administrative intervention and retain the original failed event.

For integration output, use a durable outbound mechanism with deduplicated message identifiers. Delivering an order twice to a supplier can be as harmful as creating two internal orders. Include revision and communication purpose so a resend is distinguishable from a new order. Supplier acknowledgements should correlate to the correct revision.

## Audit, security, and concurrency controls

Record who acted, on whose behalf, under which delegation, against which document revision, with which decision and reason. Store event time, correlation identifier, and linked evidence. Business audit events should survive normal application log rotation and follow retention policy. Do not put bank details, personal addresses, or attachment contents into diagnostic logs.

Separate master-data stewardship from transactional use where policy requires it. A buyer changing a supplier bank account and immediately approving an invoice introduces risk beyond the purchasing screen. Cross-domain controls must connect the partner change, purchasing activity, and finance payment process.

Concurrent buyers can revise the same order, and concurrent receivers can consume the same open quantity. ETags protect ordinary edits; posting commands also need transactional quantity checks and locks or equivalent concurrency control. A preview saying “four remaining” is not a reservation. Two receipts for four each must not both succeed merely because both clients loaded the same preview.

Administrative overrides require distinct authorization, reason capture, and traceability. Hiding an override button is not protection. Apply the same constraints to background interfaces, imports, and mobile clients. An approval API that trusts a caller-supplied approver identifier bypasses the control even if its UI looks correct.

## Test scenarios and acceptance evidence

Test the workflow as a set of business outcomes rather than only controller callbacks. Prepare authorized and unauthorized users, two organizational scopes, partial receipts, rejected tasks, changed documents, and suppliers with invalid purchasing data. Verify that errors leave no partial commitments unless explicitly designed.

| Scenario | Expected evidence |
|---|---|
| Twelve ordered, eight received | Four remain open; receipt history shows eight |
| Same receipt intent submitted twice | One posting; same authoritative result |
| Same identifier, changed quantity | Rejected payload conflict |
| Order changes during approval | Old task cannot approve new revision |
| Two receivers post final four | Only allowed net quantity posts |
| Supplier lacks required organizational data | Action blocked with actionable explanation |
| Network lost after commit | Result lookup finds original posting |
| Unauthorized plant in crafted request | Backend denial without data leakage |
| Reversal after invoice activity | Configured dependency checks enforced |

Include decimal boundary tests, price-per-hundred calculations, zero-decimal and three-decimal currency display, fractional unit restrictions, and delivery dates around daylight-saving changes. Check keyboard navigation, screen-reader labels, focus after validation, long supplier names, and narrow mobile layouts.

Integration acceptance should follow a single requisition item through order creation, approval, supplier output, partial receipt, final receipt, invoice block, resolution, and finance handover. Capture document references and compare them with backend history. A screenshot of a green status is insufficient evidence that the accounting or workflow transaction actually completed.

## Daily exception work: expediting and commercial closure

Give buyers a worklist organized by decisions rather than a flat list of every order. Useful queues include overdue supplier acknowledgements, delivery dates at risk, receipts with unexplained differences, and invoices waiting for purchasing evidence. Each queue needs an owner, aging definition, and navigation to the relevant document revision. A supplier changing a promised date should update the operational expectation without rewriting the original requested date.

For expediting, retain the contact event, supplier response, revised promise, and affected item quantities. A buyer's telephone note does not constitute a warehouse receipt. If replacement goods are promised after a return, decide whether the existing order remains open or another supported commercial document is needed. Preserve links so replenishment is not mistaken for an additional unapproved purchase.

Commercial closure should reconcile ordered, cancelled, received, returned, invoiced, and credited quantities or values under the relevant item category. Do not require those measures to be numerically identical in every scenario: accepted short delivery, nonquantity charges, and price adjustments can be legitimate. Instead, require an explained disposition for residual commitments and unresolved liabilities.

Measure exception age from meaningful events. “Awaiting receipt evidence for five days” is more actionable than “Order is old.” Keep both supplier-facing and internal delays visible so a receiver is not blamed for a late truck and a supplier is not blamed for an unprocessed internal task. Escalations should carry evidence and the next requested action, not just repeat notification emails.

Keep substitutions visible in the history. If the supplier proposes a different instrument model, the requester must confirm technical suitability and the buyer must evaluate commercial changes before accepting delivery against the original specification. A text note saying “equivalent item” should not bypass material, safety, or inspection requirements. Where an alternative changes account assignment, valuation, or serial tracking, involve the relevant owners before updating the transaction. The receipt operator should see the approved substitution and its effective order revision, rather than resolving a commercial ambiguity while the truck waits.

## Practical pitfalls and release checklist

Common failures include treating saved drafts as approved requests, conflating order release with supplier transmission, hiding the price unit, and accepting duplicate receipt submissions after a timeout. Another is implementing a custom approval screen before checking whether standard My Inbox already meets the need.

Before release:

- Confirm target applications and workflow support using the actual library product and release selection.
- Document supplier eligibility, account assignment, tolerance, and workflow restart rules.
- Verify every write against server-side authorization and current business state.
- Demonstrate exact decimal, currency, unit, and business-date handling.
- Reconcile ambiguous writes using retained request identifiers.
- Test partial execution, reversals, routing failures, and concurrent posting.
- Provide buyer, receiver, approver, and accounts-payable support ownership.
- Link receipt and invoice exceptions across domain boundaries without exposing unauthorized data.
- Review audit retention, attachment security, substitution, and administrative overrides.

## Official references and research notes

Sources were researched through SAP's official documentation repository and search-indexed SAP Help pages. Direct Help Portal and app-library fetches were unavailable in the research environment; therefore no target-system implementation prerequisites or app identifiers are asserted. Recheck documentation in the relevant product version before implementation.

1. [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) — application discovery and release-specific implementation information.
2. [SAP Help: Flexible Workflow for Purchase Requisitions](https://help.sap.com/docs/SAP_S4HANA_CLOUD/0e602d466b99490187fcbb30d1dc897c/8536fac83ec045d796cb6f12a5f21bf1.html) — requisition approval concepts in its documented product context.
3. [SAP Help: Flexible Workflow for Purchase Orders](https://help.sap.com/docs/SAP_S4HANA_CLOUD/0e602d466b99490187fcbb30d1dc897c/fbc1a9b4600745c1939276a0d9ce62ec.html) — conditional order workflow and approval concepts.
4. [SAP Help: Create Purchase Order - Advanced](https://help.sap.com/docs/SAP_S4HANA_CLOUD/0e602d466b99490187fcbb30d1dc897c/28f492016a52485784d85b7a31f7542d.html) — standard application discovery starting point.
5. [SAP Help: My Inbox](https://help.sap.com/docs/SAP_FIORI/d2c296c4f32d4f2a9e3752f58d5ef222/70735854bc73e630e10000000a44538d.html) — task-processing application concepts.
6. [SAP Help: Manage Supplier Invoices](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/af9ef57f504840d2b81be8667206d485/4e246b54f94c8f4ce10000000a4450e5.html) — invoice-processing context.
7. [SAP-maintained SAPUI5 documentation: OData V2 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md) — server-backed bindings, CRUD, batching, ETags, and CSRF behavior. Check against the UI5 version actually deployed.
