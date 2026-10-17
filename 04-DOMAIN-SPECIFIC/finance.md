# Finance: Reliable Invoice, Payment, and Accounting Workflows

## Scope, assumptions, and boundaries

This guide follows a supplier invoice into accounts payable, through payment preparation and settlement, and into general-ledger reporting. It also examines manual journals, employee expenses, and operational dashboards. The practical example is a laboratory-equipment invoice connected to the order in [Procurement](procurement.md). Warehouse receipt evidence comes from [Warehouse Management](warehouse-management.md). The [repository document index](../README.md#document-index) provides navigation.

The accounting principles here are deliberately separate from product configuration. Tax rules, account determination, payment formats, workflow availability, and closing procedures vary by jurisdiction and landscape. Examples do not establish legal or accounting advice. Confirm the organization's accounting policy and supported SAP interfaces with the responsible finance team.

All endpoints, entities, fields, statuses, and actions shown in snippets are **illustrative custom OData V2 contracts**. They are not standard SAP APIs. A production service must explicitly implement these contracts and map them to supported backend operations. No custom UI should directly update financial database tables or infer that successful HTTP transport proves successful accounting.

## Discover standard applications and supported scope

Use the [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) with the actual product, release, and deployment selection. Useful searches include “Manage Supplier Invoices,” “Manage Automatic Payments,” “Post General Journal Entries,” “Manage Journal Entries,” and supplier line-item or financial reporting applications. These are search terms and documentation starting points, not a universal list of installed tiles.

Record whether an application is transactional or analytical, its supported business process, implementation dependencies, organizational authorization, and navigation targets. Verify payment capabilities separately from bank connectivity and payment approval. A payment proposal screen does not prove that payment-media generation, bank transmission, and external acknowledgement are configured.

Expenses deserve their own discovery exercise. An organization might use an integrated travel-and-expense solution, an external expense platform, or a custom reimbursement process. Do not present one employee-expense screen as native functionality in every SAP finance deployment. Check how approved reports create payable or payroll consequences and who owns rejected integrations.

Prefer standard posting and approval experiences when they fit. A custom exception workbench can add value by bringing procurement evidence and financial status together without replacing the accounting engine. Keep links to standard applications configurable through launchpad navigation rather than embedding undocumented URLs or fixed application identifiers.

## Accounting objects and data semantics

A supplier invoice, an accounting journal entry, an open supplier item, and a payment are related but distinct objects. Posting an invoice can create accounting documents and open items; preparing a payment does not itself establish that cash moved at the bank. Clearing links eligible items according to configured accounting processes. Preserve these relationships so users can investigate discrepancies without guessing from matching amounts.

A document identifier often needs company code and fiscal year to be unique in its accounting context. An invoice reference supplied by a vendor is not a reliable primary key. Preserve leading zeros in identifiers. Carry legal entity, ledger where relevant, document type, posting date, document date, currency, and line assignments explicitly.

Amounts require exact decimal semantics. Keep document currency separate from company-code and other configured currencies. Do not sum 500 EUR and 500 USD into “1,000 payable.” A reporting conversion requires a stated target currency, rate type, valuation date, and rounding policy. Do not assume two fractional digits: currency and interface conventions vary.

Define debit and credit representation once. A contract may use positive decimal amount plus a debit/credit indicator, or signed decimal values with explicit account semantics. Mixing conventions causes plausible-looking but incorrect totals. The backend must verify balancing under the configured currency and ledger requirements; a browser total is only a preview.

Posting date determines the accounting period under the configured fiscal calendar. Document date records the external document's date; invoice receipt date and payment baseline date can drive other rules. These are not interchangeable with the UTC timestamp of a user clicking Post. A midnight time-zone conversion must not move a business date into another period.

## Roles and segregation of duties

Separate invoice entry, invoice exception resolution, payment preparation, payment release, bank-data maintenance, journal approval, and reconciliation where policy requires. A small organization may combine operational roles, but compensating controls should be documented rather than hidden in a shared administrator account.

| Role | Business responsibility | Important restriction |
|---|---|---|
| AP processor | Capture and verify invoices | Cannot bypass posting or payment blocks |
| Buyer or receiver | Explain price or receipt differences | Cannot silently change supplier liability |
| Treasury operator | Prepare payments and liquidity decisions | Cannot unilaterally alter supplier bank data |
| Payment approver | Review authorized payment scope | Must see relevant changes since review |
| Accountant | Post journals and reconcile accounts | Controlled company-code and ledger scope |
| Expense approver | Review employee business purpose | Avoid approving own reimbursement |
| Auditor | Inspect evidence and changes | Read-only, scoped, traceable access |

These are responsibilities, not delivered SAP role names. Authorization must constrain reads, value helps, exports, background jobs, and commands. A company-code selector is a convenience, not an access-control boundary. Restrict sensitive bank information and employee receipts even when a user can see summarized liabilities.

## Supplier invoice workflow

### Capture and identify the invoice

Start with supplier, company code, supplier invoice reference, document date, posting date, currency, gross amount, and appropriate line references. Preserve the original document or governed image reference. Optical extraction can suggest values, but low-confidence fields need review. Never treat an OCR result as an authorized posting instruction.

Duplicate detection should consider the configured business criteria, not only exact file hashes. A supplier might resend the same invoice with a different PDF encoding. Conversely, recurring invoices can legitimately share similar amounts and descriptions. Show candidate duplicates with identifying evidence and route exceptions rather than silently deleting a submission.

The supplier must be eligible for the organizational context. Payment blocks, alternative payee arrangements, withholding-tax requirements, and bank-data concerns may affect processing. Do not copy sensitive bank details from an invoice image into master data without the governed verification process. An invoice containing new banking instructions is a fraud-control event, not a routine field correction.

### Validate references and match evidence

For the laboratory order, the supplier bills twelve instruments while only eight are recorded as received. Display order quantity, net eligible receipt quantity, invoice quantity, unit conversion, price basis, and tolerance outcome. The AP processor should see whether the discrepancy is quantity, price, tax, or missing reference data.

An invoice may be parked, held, posted with a payment block, or rejected according to the actual supported process. These outcomes are not equivalent. A posted but blocked invoice may already affect accounts payable while being ineligible for payment. A parked item generally must not be presented as a fully posted liability without explaining its accounting status.

For a non-PO invoice, require an appropriate expense or asset account assignment and review path. For service invoices, apply the configured service acceptance basis instead of pretending a material goods receipt exists. Record the resolution owner and deadline, because unresolved matching exceptions affect cash forecasts and supplier relationships.

### Simulate, approve, and post

A simulation can reveal account determination, tax, balancing, and period errors before commitment. Label it as a preview with a calculation timestamp and relevant revision. Configuration or master data can change between simulation and posting; posting must repeat authoritative checks.

Approval and posting are separate transitions unless the implemented workflow explicitly combines them. Approval should bind to a specific document revision and material fields. Changes to supplier, amount, account assignment, or payment information may invalidate approval. The server determines that rule; the browser cannot declare a change “cosmetic” to avoid another review.

On posting, return the financial document references and final status. If processing is asynchronous, return a durable command identifier and expose progress. Do not mark the invoice Posted solely because a queue accepted it. After a timeout, reconcile the original intent before another submission.

### Correct posted errors through controlled follow-on actions

Posted financial documents generally require supported correction, reversal, or follow-on documents rather than arbitrary edits to accounting amounts. The precise options depend on document type and configuration. Retain references between original and corrective documents, including reversal reason, dates, and actor.

A credit memo and a reversal are not interchangeable. A supplier credit may represent a commercial adjustment; reversing an erroneous posting undoes a particular accounting event. A closed period can require a permitted correction date and accounting review. Never auto-open a period or shift dates simply to make a UI retry succeed.

## Payment preparation, execution, and reconciliation

Payment work begins with eligible open items, payment terms, baseline dates, blocks, payment methods, and banking arrangements. Build a proposal using the configured selection criteria and show exceptions. A proposal is a reviewable candidate set; it does not mean the bank has been instructed or the supplier has received funds.

Review company codes, due dates, discount opportunities, bank availability, currencies, and blocked items. A proposed payment can become stale if an invoice is blocked, cleared elsewhere, or changed through an allowed process. Revalidate eligibility when executing. Store the proposal or review revision so the approver knows what the final run contains.

Distinguish payment posting, media creation, release, transmission, bank acceptance, settlement evidence, and reconciliation. Their ordering and integration vary. Some accounting clearing may occur before external settlement is confirmed; therefore “cleared” and “bank settled” must remain separate dimensions. Bank rejection can require further accounting and operational steps.

```text
Payment preparation:
DraftParameters -> ProposalRequested -> ProposalReady -> Reviewed
ProposalRequested -> ProposalError
Reviewed -> ExecutionRequested -> Executed | ExecutionError

External processing:
MediaPending -> MediaCreated -> Released -> Transmitted
Transmitted -> Accepted | Rejected | AcknowledgementUnknown

Reconciliation:
AwaitingStatement -> Matched -> Reconciled
AwaitingStatement -> Investigate
```

These are custom explanatory states, not SAP payment-status values. A payment file with an uncertain delivery outcome must not be regenerated with a fresh external identifier just to clear an error icon. Preserve run, message, and bank references. Coordinate with treasury and the bank-connectivity owner to determine whether the original instruction was accepted.

| Observation | Do next | Do not do |
|---|---|---|
| Proposal has invalid payment data | Repair governed data and regenerate/review | Edit bank account ad hoc in the UI |
| Payment execution times out | Query original run and command outcome | Launch an unrelated replacement run |
| Bank rejects an instruction | Investigate rejection and accounting effect | Assume deleting a file reverses accounting |
| Statement amount differs | Reconcile fees, FX, dates, and partial settlement | Force a match solely by supplier name |
| Invoice paid outside the process | Record and reconcile through supported accounting | Hide the still-open item from reports |

## General-ledger journals and closing

A manual journal form should make the posting purpose explicit. Require company code, posting date, document type, currency, line accounts, debit/credit semantics, amounts, and required assignments. Explain missing cost centers or profit centers at the line where they matter. A generic “Document invalid” toast makes finance users reconstruct backend rules unnecessarily.

An illustrative accrual might debit an expense and credit an accrual account. This is a conceptual pattern, not a universal account-number recommendation. Account determination, tax implications, reversals, and permitted document types belong to the organization's chart and policy. Never embed sample account numbers in production defaults.

Separate journal preparation, validation, review, approval, posting, and scheduled reversal where required. A reversible accrual needs a clear reversal rule and follow-up evidence. If scheduled reversal fails because a period is closed, raise an exception; do not silently label the accrual fully processed.

At close, reconcile subledger balances with general-ledger views using the relevant company code, ledger, currency, and key date. Explain timing differences rather than comparing two totals from different refresh times. Restrict period controls to authorized finance administrators. Capture the reason and scope of temporary period access.

Concurrent period closure is a useful failure scenario: a journal preview succeeds, but the posting period closes before submission. The correct outcome is an actionable posting failure without an unauthorized date substitution. The preparer must obtain approval for any date or period change, and material changes may require another journal review.

## Employee expenses and reimbursement

Expense capture includes employee, business purpose, dates, expense category, amount and currency, receipt evidence, and allocation. Tax and reimbursement treatment vary by country and policy. A receipt image can include personal information; restrict visibility and retention appropriately. Do not log receipt contents in browser telemetry.

Differentiate employee-paid expenses, corporate-card transactions, cash advances, and company-paid bookings. Treating all four as a payable to the employee can duplicate reimbursement. Reconcile imported card items against expense reports and preserve source identifiers to prevent repeat imports from creating duplicate claims.

Approval should show policy exceptions, missing receipts, personal portions, attendees where required, and allocation changes. A manager approval is not necessarily tax validation or finance posting approval. Map each decision to its actual responsibility and make rejection or return-for-correction effects visible.

An approved report may still await integration into accounting or payroll. Show export, acceptance, posting, and reimbursement status separately. A retry must reuse the source report and integration identifier. If only some lines are rejected, preserve accepted line references and clearly specify whether correction requires reversing the complete document or resubmitting only rejected content.

## Illustrative custom service and UI5 implementation

Assume an OData V2 service at `/example/finance/` with `InvoiceDrafts`, `PostingCommands`, and `CommandResults`. This is a custom contract, not a standard endpoint. `InvoiceDrafts` has string keys `DraftId` and string `Revision`, plus `CompanyCode`, `Currency`, `GrossAmount`, `PostingDateText`, and `__metadata.etag` in V2 responses. `GrossAmount` is `Edm.Decimal(19,4)` in this example. `PostingDateText` is an explicitly constrained ISO business-date string, not an OData timestamp.

`PostingCommands` is create-only. Its properties are `RequestId`, `DraftId`, `ExpectedRevision`, and `Operation`, all strings. `Operation` accepts `POST_INVOICE`. The backend atomically enforces authorization, revision, duplicate controls, balancing, open period, and posting. `CommandResults` provides authorized recovery lookup and returns the final accounting references. Reject a reused request identifier with a different payload.

The following XML assumes an OData binding context named `fin` and a local JSON model `ui` initialized with `busy: false` and `mayPost: false`. Its decimal type validates display and parsing; it does not replace server-side accounting validation.

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  controllerName="example.finance.Invoice">
  <VBox class="sapUiSmallMargin" busy="{ui>/busy}">
    <Title text="{fin>DraftId}" />
    <Label text="Gross amount" labelFor="grossAmount" />
    <Input id="grossAmount" editable="false" value="{
      path: 'fin>GrossAmount',
      type: 'sap.ui.model.odata.type.Decimal',
      constraints: {precision: 19, scale: 4}
    }" description="{fin>Currency}" />
    <Label text="Posting date" />
    <Text text="{fin>PostingDateText}" />
    <Button text="Post verified invoice" press=".onPost"
      enabled="{= ${ui>/mayPost} &amp;&amp; !${ui>/busy} }" />
  </VBox>
</mvc:View>
```

This review view deliberately does not let users modify amount and immediately post unsaved values. If an editable draft is introduced, save and validate it first, resolve pending changes, and then post the resulting revision. The capability flag improves usability but does not confer posting permission.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/m/MessageBox"
], function (Controller, MessageBox) {
  "use strict";
  return Controller.extend("example.finance.Invoice", {
    onPost: function () {
      const view = this.getView();
      const ui = view.getModel("ui");
      const model = view.getModel("fin");
      const context = view.getBindingContext("fin");
      if (!context || ui.getProperty("/busy")) {
        return;
      }
      if (model.hasPendingChanges()) {
        MessageBox.warning("Save and validate draft changes before posting.");
        return;
      }
      const draft = context.getObject();
      if (!this._postingIntent) {
        this._postingIntent = {
          RequestId: globalThis.crypto.randomUUID(),
          DraftId: draft.DraftId,
          ExpectedRevision: draft.Revision,
          Operation: "POST_INVOICE"
        };
      }
      ui.setProperty("/busy", true);
      model.create("/PostingCommands", this._postingIntent, {
        success: function (result) {
          ui.setProperty("/busy", false);
          if (result.Outcome === "COMPLETED") {
            this._postingIntent = null;
            ui.setProperty("/mayPost", false);
            model.refresh(true);
            MessageBox.success("Invoice posting confirmed.");
          } else {
            MessageBox.information(
              "Posting accepted; check its result before another submission."
            );
          }
        }.bind(this),
        error: function () {
          ui.setProperty("/busy", false);
          MessageBox.warning(
            "Posting outcome unconfirmed. Reconcile the existing request."
          );
        }
      });
    }
  });
});
```

Assumptions include a secure browser with `crypto.randomUUID`, a nondeferred create group, and the V2 model's normal token handling. Configure deferred batching deliberately if used; submission success and individual business success are different. A full application also needs message-model integration, route lifecycle handling, and a recovery page.

The recovery page must query `CommandResults` with the retained request identifier, preserving it across reload through an approved mechanism. Do not claim in-memory controller state is durable. On definite validation failure, let the user correct the draft and start a new intent. On unknown outcome, retain the old intent until reconciled. Never regenerate identifiers on every retry.

Ordinary draft edits use ETag-protected updates. Metadata-generated keys avoid broken paths when identifiers contain special characters:

```javascript
const draftPath = "/" + model.createKey("InvoiceDrafts", {
  DraftId: draft.DraftId
});
model.update(draftPath, { PostingDateText: "2026-10-31" }, {
  eTag: draft.__metadata.etag,
  success: onDraftSaved,
  error: onDraftSaveFailed
});
```

The literal date is illustrative. The service must validate its format, calendar validity, authorization, and accounting-period eligibility. Require an ETag rather than replacing missing concurrency data with `*`. The command uses its own atomic revision comparison because the ETag of a newly created command does not guard the invoice draft.

## Dashboards that reconcile to accounting

Define each measure before drawing charts. “Overdue payable” needs a key date, due-date rule, treatment of blocked items, organizational scope, and currency basis. “Invoices awaiting approval” should not include posted invoices merely because an unrelated task remains visible. Show counts and amounts separately.

Aggregate on the server using authorized data. The currently loaded page of an OData list is not the full population. A browser summing twenty visible items and labeling the result “Total liability” creates a misleading dashboard even if the arithmetic is exact.

Display the last refresh timestamp, selected filters, reporting currency, and data-source scope. Analytical replication and operational transactions may have different latency. Drill-down should preserve filter semantics so users can reproduce totals. Do not silently exclude exceptions to make dashboard and accounting balances appear consistent.

Use critical states purposefully: overdue is not the same as blocked, and blocked is not necessarily erroneous. Show owners and next actions. Financial dashboards are operational control surfaces; color without definition encourages inappropriate bulk action.

## Recovery, audit, and concurrency

Distinguish field validation, policy failure, authorization denial, stale revision, business lock, transient infrastructure error, and unknown commit outcome. Each needs a different response. Retrying a closed-period failure is useless without a valid correction; retrying an unknown bank transmission can duplicate a payment.

Record the authenticated actor, delegated actor where applicable, operation, document revision, business reason, timestamp, request identifier, and resulting document references. Audit privileged overrides separately. Keep sensitive values masked in operational logs while retaining controlled financial evidence under the relevant retention policy.

Idempotency and duplicate-invoice detection solve different problems. Idempotency prevents the same submitted intent from executing twice. Duplicate-invoice checks identify separate submissions that may represent the same economic obligation. Both are needed. Financial uniqueness checks must execute transactionally so simultaneous submissions cannot both pass a preview check.

Protect invoice and payment eligibility under concurrency. Another process may clear an open item or apply a block after selection. The backend must recheck current state, not trust the table row originally selected. A long-running payment process also needs explicit ownership, locking strategy, restart semantics, and reconciliation for partial outcomes.

## Remittance evidence and supplier inquiries

A supplier asking “Which invoice did you pay?” needs a traceable allocation, not merely a payment-run status. Preserve the relationship between the payment, eligible invoice items, discounts, credit memos, and remittance advice. Delivery of remittance advice is another communication outcome; it does not independently establish bank settlement.

For partial payments and residual-item processing, explain the selected accounting treatment using the actual supported configuration. Do not describe every smaller payment as clearing the complete original invoice. Users investigating an open balance should see the original obligation, amounts already processed, remaining items, and the references that explain the current balance.

Customer-service or supplier-support access should be narrower than payment-execution access. A support user may need masked payment references and status without permission to view full bank details or release another payment. Record who disclosed remittance information and use an approved communication channel rather than exporting unrestricted account data.

For disputed balances, capture a case reference and distinguish the supplier's claim from the accounting balance. A supplier statement might omit a credit memo, include an invoice not yet received, or use a different cutoff date. Reconcile those differences explicitly before changing payment eligibility. Preserve correspondence and approved decisions under the document retention policy. The support workbench should link to the responsible AP or treasury owner, not offer an unrestricted adjustment button merely because an inquiry is urgent.

## Tests, pitfalls, and release checklist

Use business acceptance cases with observable accounting references. Unit tests for formatting cannot establish that the correct company code or period was posted.

| Scenario | Expected outcome |
|---|---|
| Duplicate click with same request identifier | Exactly one posting and repeatable result |
| Different identifiers for same supplier invoice | Configured duplicate detection still runs |
| Amount contains more scale than allowed | Explicit validation, not silent truncation |
| Period closes after simulation | Posting fails without changing the date |
| Unauthorized company code in crafted request | Server denial on reads and writes |
| Concurrent payment selections | No duplicate economic payment |
| Network fails after journal commit | Original document recoverable by request |
| Bank acknowledges late | Existing instruction reconciled, not recreated |
| Expense import repeats | Same source items are not reimbursed twice |
| Report refresh differs from operational view | Timing and scope differences explained |

Test zero-decimal and three-decimal currencies, tax rounding, negative credit documents, partial payments, residual items where supported, exchange differences, daylight-saving boundaries, and fiscal-year transitions. Verify the meaning of each debit and credit display. Exercise rejection, return for correction, reversal, and reconciliation paths, not just successful posting.

Common pitfalls include mixing payment and settlement status, using invoice references as unique technical keys, summing mixed currencies, changing dates to bypass period controls, and interpreting approval as posting. Another is showing technical batch success while ignoring an error in the individual posting response.

Before release:

- Verify application scope and implementation information in the selected library release.
- Sign off posting, approval, reversal, tax, expense, and payment rules with finance owners.
- Confirm decimal, debit/credit, currency, fiscal-period, and business-date contracts.
- Prove backend authorization and separation of duties across alternate interfaces.
- Reconcile ambiguous invoice posting and payment transmission without duplicate execution.
- Show audit evidence and actual accounting references for acceptance cases.
- Establish owners for invoice exceptions, rejected payment media, integration failures, and closing errors.
- Reconcile dashboard definitions and drill-down totals to the agreed accounting population.

## Official references and research notes

Research used SAP-maintained UI5 documentation and search-indexed official SAP Help pages. Direct Help Portal and app-library fetches were unavailable; implementation details must therefore be checked in the actual target product documentation. No app identifiers, universal releases, or unverified prerequisites are prescribed.

1. [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) — standard application discovery and release-specific implementation review.
2. [SAP Help: Manage Supplier Invoices](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/af9ef57f504840d2b81be8667206d485/4e246b54f94c8f4ce10000000a4450e5.html) — supplier-invoice application context.
3. [SAP Help: Manage Automatic Payments](https://help.sap.com/docs/r/3eb1567cf97543c08087efb0936964e6/2020.000/en-US/45698054f87c033de10000000a441470.html) — payment proposal and payment-run concepts in the documented release.
4. [SAP Help: Journal Entries](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/651d8af3ea974ad1a4d74449122c620e/b050d7531a4d424de10000000a174cb4.html) — accounting-document context.
5. [SAP Help: Accounts Payable and Accounts Receivable Full Clearing](https://help.sap.com/docs/SAP_S4HANA_CLOUD/b978f98fc5884ff2aeb10c8fdeb8a43b/fffe9bc0c80f4609970e25a95df127f2.html) — clearing concepts, distinct from external bank settlement.
6. [SAP Help: Resetting Unsuccessful Payments or Proposals](https://help.sap.com/docs/SAP_S4HANA_CLOUD/031c345485b84c8c94265be9ef61d3a8/26996f2bdf2f4dc0bbc318db34eeb203.html) — controlled recovery context; follow the target system's supported procedure.
7. [SAP-maintained SAPUI5 documentation: OData V2 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md) — CRUD, server-backed data, batching, ETags, and CSRF handling.
