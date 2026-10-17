# Plant Maintenance: Requests, Work Orders, and Asset Reliability

## Scope and implementation assumptions

This guide describes an original maintenance workspace for requests, maintenance notifications, work orders, technicians, assets, preventive maintenance, and operational handover. Assume SAP owns the authoritative maintenance documents and technical objects, while a UI5 application consumes a custom OData V4 façade. Scheduling, inventory, purchasing, safety processes, and financial settlement may involve separate services and organizational owners.

All `/maintenance/` routes, `Demo.Maintenance` actions, entity names, and state values below are **illustrative custom service contracts**. They are not official SAP API endpoints, standard status identifiers, or guaranteed application features. Use only supported business APIs and approved extensibility mechanisms for the actual installation. Never update maintenance, inventory, or accounting database tables directly.

SAP maintenance processes differ by product edition, release, configuration, and enabled scope. A phase-based process in one environment must not be assumed in another. Confirm what the installed system supports before mapping the proposed workflow to system statuses, user statuses, business functions, and integrations.

## Standard applications and research limits

First inspect the [SAP Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) for the exact product version and tasks: request intake, notification processing, order planning, execution, technical objects, and maintenance scheduling. Verify business catalogs, backend components, frontend prerequisites, device support, and extension options. Do not publish a fixed app ID as universally available.

This research retrieved official SAPUI5 documentation and OpenUI5 source from SAP's GitHub repositories. The SAP Help Portal and reference-library hosts could not be resolved from this environment. Thus release-specific maintenance application availability and API support remain explicit implementation gates. The designs below are recommendations requiring confirmation with the maintenance process owner, not certified descriptions of a particular release.

## Business notifications versus UI notifications

A **maintenance notification** is a persistent business record describing a malfunction, observation, request, or other maintenance-relevant issue according to the configured process. It can contain technical object references, dates, symptoms, damage/cause coding, activities, tasks, and links to follow-on work.

A **UI notification** is presentation feedback: a message strip, message popover, toast, shell alert, or inbox entry. Showing “request saved” does not create a maintenance notification unless the backend explicitly created one. Conversely, closing a message popover does not complete the business notification.

Use distinct names such as `MaintenanceNotifications` for business entities and `uiMessages` for client feedback. Explain whether an intake request is already a SAP notification or merely a custom precursor awaiting triage. Never label a temporary browser form as a durable notification.

## Roles and responsibility boundaries

An operator reports a symptom and immediate observations. A maintenance coordinator validates the request, identifies duplicates, and assesses priority. A planner defines scope, operations, resources, materials, and services. A supervisor authorizes release and coordinates access. Technicians execute assigned work and record actuals. Reliability engineers analyze failure patterns and preventive strategies.

Warehouse and purchasing teams own stock and external procurement actions. Safety personnel or authorized operating staff own permits and isolation processes as locally defined. Financial controllers own settlement configuration and financial closure. These responsibilities overlap operationally but must not collapse into unrestricted application permissions.

Scope access by maintenance plant, planning plant, work center, technical object, and action. A contractor may view assigned operations without seeing all asset histories or cost rates. A technician's completion statement is not automatically supervisory acceptance or financial closure.

## End-to-end corrective maintenance

1. An operator records the affected asset or location, symptom, discovery time, and urgency.
2. Triage validates the technical object, checks duplicates, and assesses operational impact.
3. The planner creates or links a work order and defines executable operations.
4. Materials, services, labor, permits, and access requirements are coordinated.
5. An authorized person releases the order when defined prerequisites are satisfied.
6. A scheduler assigns an approved time window and qualified technicians.
7. Technicians execute under applicable site procedures and record findings and actuals.
8. Supervisory review resolves incomplete work and accepts the technical result.
9. The source process performs technical completion and later financial closure as appropriate.
10. Reliability analysis updates failure knowledge and preventive strategy.

The application coordinates evidence and status; it is not a substitute for physical safety procedures, legal obligations, or competent authorization.

## Request triage and duplicate handling

Capture a concise symptom, technical object or location, reporter contact, discovery time, operational impact, and relevant evidence. Separate the reporter's requested urgency from the maintenance team's assessed priority. Require enough information to triage without forcing an operator to diagnose a failure mechanism they cannot know.

Potential duplicates can be suggested using asset, symptom, and time window, but do not merge automatically. Two similar reports may describe separate incidents or a recurring fault. Link duplicate reports to a canonical issue while preserving original reporters, timestamps, and evidence. If a request is rejected as not maintenance-related, record the reason and destination owner rather than silently deleting it.

For emergencies, display approved local escalation instructions. Do not imply that submitting a form guarantees an immediate response, or that application availability replaces emergency communication.

## Independent state machines

Keep request, notification, order, operation, and permit lifecycles separate. An illustrative request can progress from New → Triaged → Accepted → LinkedToOrder → Resolved, with Rejected and Duplicate as controlled alternatives.

A custom order projection might show Planned → Approved → Released → InExecution → AwaitingReview → TechnicallyComplete → FinanciallyClosed. These labels require mapping to the actual configured source statuses. Operations can be unstarted, partially confirmed, completed, or requiring follow-up independently of the header.

A notification may remain open while a linked order is completed because cause coding or requested tasks are unresolved. A technically completed order may still have financial work outstanding. A permit's validity can expire while the order remains released. Avoid a single green “done” label that conceals these distinctions.

## Atomic block: accept a maintenance request

**Inputs:** request identifier, triage decision, expected version, responsible team, reason, and idempotency key.

**Preconditions:** the actor has triage authority for the plant; required asset/location information is valid; the request is still eligible; duplicate disposition has been considered; the expected version matches.

**Effects:** record the accepted decision, assign responsibility, append the audit transition, and create a durable follow-on task if a source notification or order must be created asynchronously.

**Exception path:** if source creation fails, keep the accepted request visibly awaiting integration. Do not reset it to new and cause another coordinator to create duplicate work. A source response lost after successful creation requires correlation-based reconciliation before any controlled resubmission.

## Technical objects and asset identity

Distinguish a physical equipment item from a functional location or hierarchical position. An equipment item can be installed, removed, or transferred; the location's history and the equipment's history answer different reliability questions. Preserve installation intervals and the technical-object reference applicable when the incident occurred.

Store immutable technical identifiers alongside readable labels and serial/manufacturer references where appropriate. Avoid using an editable equipment description as a key. A retired or deactivated asset should remain available for authorized historical analysis without accepting ordinary new work contrary to policy.

Handle missing identification through a controlled “location-only” or “unknown object” intake path where permitted. Do not invent an equipment master record on the fly simply to satisfy a required frontend field.

## Illustrative data model

Expose a read projection with:

- `MaintenanceRequests`: ID, reporter reference, discovery timestamp, symptom, assessed priority, plant, state, and version.
- `MaintenanceNotifications`: source reference, technical object, catalog-coded findings, malfunction interval, tasks, and linked orders.
- `WorkOrders`: order number, planning plant, maintenance plant, work center, lifecycle dimensions, cost currency, source references, and version.
- `Operations`: sequence, description, required qualifications, planned effort, work center, execution status, and version.
- `Assignments`: technician, operation, planned interval, assignment state, and scheduling version.
- `Assets`: equipment/location identifiers, hierarchy, installation validity, criticality, and responsible organization.
- `Confirmations`: immutable confirmation identifier, actual effort, unit, time interval, completion indicator, and reversal reference.
- `MaintenancePlans`: scheduling basis, cycle/counter definition, last call, next due window, and source plan reference.
- `CommandReceipts`: target, command, request ID, payload hash, result reference, and processing state.

Represent measurements with units, money with currency, and versions as opaque strings. Large counters and decimal quantities must not be rounded through JavaScript's binary floating-point arithmetic.

## Business invariants

Only eligible operations can receive ordinary execution confirmations. A technician must be authorized and, where required, qualified for the assignment. Confirmed labor must have a valid unit and plausible interval. Material consumption must reference the appropriate stock-owning process; a local decrement in a browser model is not a goods movement.

Technical completion must respect the configured process, unresolved required operations, and mandatory evidence. Do not impose a universal rule that every linked notification must be closed first; the actual source rules and local policy decide the permitted relationship. Financial closure is distinct and may have additional prerequisites.

Asset criticality does not grant authorization. An “urgent” flag cannot bypass permits, isolation verification, or procurement controls. Server-side invariants must remain in force even if a mobile client submits a crafted request.

## Work order planning

Translate the accepted problem into operations with clear scope, expected outcome, work center, estimated effort, qualifications, and dependencies. Distinguish troubleshooting from known repair work. Link drawings or procedures using controlled document references and current revisions.

Plan stock materials and external services through their owning processes. Show availability and procurement status with timestamps. A displayed material quantity is not a reservation guarantee unless the source system confirms it. Record access constraints, production windows, scaffolding, lifting resources, and required inspections as structured prerequisites where practical.

If scope grows during execution, use an authorized change process. Preserve the released baseline and explain additional cost or schedule impacts. Do not silently replace the original plan with actuals, because that erases planning accuracy evidence.

## Release and safety-sensitive prerequisites

Release is a business transition, not proof that the equipment is physically safe to work on. Display the source process's release outcome and the independently verified state of relevant permits or access prerequisites. Where the application references isolation or permit systems, preserve source identifiers, validity, and verification timestamps.

The backend must reject prohibited release attempts even when the frontend button appears enabled. If prerequisite information is unavailable or stale, apply the organization's approved fail-safe policy and escalation path. Never suggest bypassing safety controls to restore application flow.

An expired permit during execution requires the responsible operating process to determine the next action. The application should prominently communicate the block and owner; it should not automatically extend validity.

## Technician scheduling

Schedule against operation duration, qualifications, availability, shift calendars, travel, and dependencies. Assignments are tentative until committed against a scheduling version. Prevent overlapping assignments according to the site's rules, including travel and handover time where relevant.

Resource capacity is not merely headcount. Two technicians with different qualifications may not substitute for each other. A contractor's site access may expire before the planned date. Revalidate qualification and assignment eligibility at the appropriate workflow transitions.

For reassignment, preserve who was originally assigned, who accepted the work, and why the change occurred. A dispatcher should see the effect on both the receiving technician and other commitments. Do not erase a technician's existing progress when changing the remaining assignment.

## Technician execution experience

Provide an operation-focused view with asset identity, symptom, scope, prerequisites, approved documents, required measurements, and confirmation controls. Let the technician record partial progress, additional findings, and work not completed. Avoid a single “complete” button that bundles time booking, material movement, notification closure, and order completion without explanation.

Record actual start/end or actual effort according to the supported process; these are not interchangeable. Capture the relevant unit and business date. Explain whether “final confirmation” means the operation is fully complete under the configured source semantics.

Support follow-up requests without rewriting the original symptom. Photograph and document uploads need size limits, malware inspection, authorization, and retention. Equipment tags and photos can reveal sensitive facility layouts; treat them as protected business data.

## Contextual UI5 XML: operation confirmation

The route has bound the view to an `Operations` entity from the illustrative V4 service. The named `entry` JSON model holds unsent form values; it is not a backend draft. This example records labor hours and completion intent, not goods movements or permit authorization.

```xml
<mvc:View controllerName="demo.maintenance.controller.Operation"
  xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
  <Page title="{OperationDescription}" busy="{ui>/busy}">
    <content>
      <ObjectIdentifier title="{EquipmentLabel}" text="{OperationNumber}"/>
      <ObjectStatus title="{i18n>execution}" text="{ExecutionStatus}"/>
      <MessageStrip text="{i18n>followApprovedSiteProcedures}"
        type="Warning" showIcon="true"/>
      <VBox class="sapUiSmallMargin">
        <Label text="{i18n>actualHours}" labelFor="hours"/>
        <Input id="hours" value="{entry>/hours}" type="Text"/>
        <Label text="{i18n>findings}" labelFor="findings"/>
        <TextArea id="findings" value="{entry>/findings}" rows="4"/>
        <CheckBox text="{i18n>operationFullyCompleted}"
          selected="{entry>/final}"/>
        <Button text="{i18n>recordConfirmation}" type="Emphasized"
          enabled="{= !${ui>/busy} }" press=".onConfirm"/>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

The text input deliberately retains the decimal as a string. This small example specifies dot-decimal input through localized help text; a production locale-aware field should parse and validate with a suitable UI5 decimal type before producing the service's canonical decimal string. Do not use `parseFloat` on arbitrary localized input.

## Contextual UI5 JavaScript: confirmation command

The custom `ConfirmWork` bound action accepts `RequestId`, `ExpectedVersion`, and `Findings` as `Edm.String`, `ActualHours` as `Edm.Decimal` with precision 7 and scale 2, and `Final` as `Edm.Boolean`. It returns a command receipt. The backend determines the eligible technician from authenticated identity and assignment; the browser does not submit a trusted technician ID.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox"
], function (Controller, JSONModel, MessageBox) {
  "use strict";
  return Controller.extend("demo.maintenance.controller.Operation", {
    onInit: function () {
      this.getView().setModel(new JSONModel({ busy: false }), "ui");
      this.getView().setModel(new JSONModel({
        hours: "", findings: "", final: false
      }), "entry");
    },
    onConfirm: async function () {
      const oView = this.getView();
      const oUi = oView.getModel("ui");
      const oEntry = oView.getModel("entry");
      const oContext = oView.getBindingContext();
      const oBundle = oView.getModel("i18n").getResourceBundle();
      if (oUi.getProperty("/busy") || !oContext) {
        return;
      }
      const sHours = String(oEntry.getProperty("/hours")).trim();
      if (!/^(?:0|[1-9]\d{0,4})(?:\.\d{1,2})?$/.test(sHours)
          || /^0(?:\.0{1,2})?$/.test(sHours)) {
        MessageBox.warning(oBundle.getText("enterPositiveDotDecimalHours"));
        return;
      }
      const sRequestId = globalThis.crypto.randomUUID();
      const oAction = oView.getModel().bindContext(
        "Demo.Maintenance.ConfirmWork(...)", oContext
      );
      oAction.setParameter("RequestId", sRequestId);
      oAction.setParameter("ExpectedVersion", oContext.getProperty("Version"));
      oAction.setParameter("ActualHours", sHours);
      oAction.setParameter("Findings", oEntry.getProperty("/findings"));
      oAction.setParameter("Final", oEntry.getProperty("/final"));
      oUi.setProperty("/busy", true);
      try {
        await oAction.invoke("$direct");
        MessageBox.information(
          oBundle.getText("confirmationAccepted", [sRequestId])
        );
      } catch (oError) {
        MessageBox.error(
          oBundle.getText("checkConfirmationReceipt", [sRequestId])
        );
      } finally {
        oAction.destroy();
        oUi.setProperty("/busy", false);
      }
    }
  });
});
```

The default model is `sap.ui.model.odata.v4.ODataModel`, configured with `serviceUrl: "/maintenance/"` and `synchronizationMode: "None"`. The service path requires same-origin routing to the custom backend. Verify `invoke` support in the deployed UI5 version and secure-context support for `crypto.randomUUID`.

The regex limits decimal shape, not legitimate business effort. The server applies shift, assignment, date, plausibility, and policy checks. The example preserves entered values on both outcomes. A production confirmation journal marks a form snapshot as submitted and prevents a second new command until the receipt is resolved. It must not silently retry with a new identifier after an unknown outcome.

## Atomic block: record labor confirmation

Validate operation version, actor assignment, allowed confirmation state, actual-hour range, final-confirmation semantics, and required evidence. Reserve the command identifier atomically. Record the confirmation, resulting operation transition, audit event, and outbound work as one local transaction where the façade owns that data.

If SAP owns the confirmation, the façade records a durable command and invokes the supported API with correlation where available. A lost response must be reconciled before resending. Do not claim that a local receipt alone makes a non-idempotent remote API idempotent.

Correct a booked confirmation through the source process's reversal or correction mechanism. Preserve the original record and link the correction. Editing historical hours in a custom projection must not conceal a financial or labor posting that remains unchanged in SAP.

## Concurrency, idempotency, and offline work

Use ETags for source-supported entity edits and expected business versions for custom operations. Two technicians may confirm different operations concurrently; avoid unnecessarily locking the whole order if the business boundary is smaller. Conversely, a final-order transition needs a consistent view of required operation outcomes.

Scope idempotency by tenant, action, and operation. Store payload hash, receipt, authoritative confirmation reference, and retention period. Concurrent identical commands converge through atomic uniqueness. A reused key with different hours is a conflict, not an update.

Offline execution requires a designed synchronization protocol, not local storage plus blind replay. Capture the observed source version, stable command identifier, local timestamp, and submitted snapshot. Encrypt and minimize cached data under the organization's device policy. On reconnect, revalidate assignment and eligibility; route stale or conflicting work to review. Never infer current permit validity from an old offline screen.

## Preventive maintenance scheduling

Distinguish time-based, counter-based, and combined strategies supported by the target system. A maintenance plan is not merely a recurring calendar reminder. The source scheduling logic may create calls or follow-on documents according to configured cycles, scheduling parameters, and completion behavior.

For time-based plans, define calendar basis, tolerance, lead time, and how early or late execution affects future dates. For counter-based plans, identify the authoritative measuring point, unit, reading timestamp, expected progression, and estimated usage basis. Handle counter replacement, rollover, reset, and corrected readings explicitly.

Display next due and overdue status from the authoritative schedule or a clearly labeled projection. A frontend timer must not create maintenance orders. If forecasting is custom, label it as a recommendation until the source scheduling process confirms a call.

## Atomic block: generate preventive calls

A scheduler identifies eligible plan/cycle occurrences using a stable occurrence key, plan version, and scheduling horizon. It records the decision and durable creation intent before dispatch. Re-running the same scheduling window must not create duplicate orders for the same occurrence.

Overlap between two scheduler instances requires locking or atomic uniqueness at the occurrence boundary. If a maintenance plan changes, determine whether already generated calls remain valid, require rescheduling, or need a controlled cancellation. Do not delete historical calls to make the schedule look clean.

Reconcile generated intents against source call and order references. A failed integration should appear as “call creation pending” or “requires review,” not as an empty schedule. Escalate overdue safety- or compliance-relevant work through approved organizational procedures.

## Measurements, findings, and reliability analysis

Keep observations, symptoms, causes, and corrective activities distinct. A symptom reported by an operator is not a validated root cause. Use configured catalogs and allow explanatory text where policy permits. Track who classified the failure and when the classification changed.

Measurements require point, value, unit, timestamp, quality indicator, and source. Detect implausible jumps and unit mismatches before they influence preventive forecasts. Corrections retain provenance. Sensor events can create candidate issues, but automatic order creation requires defined thresholds, deduplication, and business approval rules.

Reliability analysis should separate repeat failures from duplicate reports. Equipment replacement and installation changes matter when calculating exposure and comparing failure rates. Do not attribute a location's entire history to the equipment currently installed there.

## Exceptions and completion review

Common exceptions include missing materials, inaccessible equipment, unavailable qualifications, expired access authorization, additional damage, contractor no-show, and failed post-maintenance inspection. Record a structured reason, owner, next action, and next review date. Pausing execution should not erase already confirmed labor or consumed materials.

For incomplete work, create follow-up scope or retain open operations according to the source process. Supervisory review distinguishes “work performed,” “equipment restored,” “required documentation complete,” and “accepted for operation.” These facts may be recorded by different authorized people.

Reopening after technical completion must follow supported source controls and preserve audit history. Financial closure or settled costs can impose additional restrictions; a custom status toggle must not bypass them.

## Security, audit, and attachments

Authorize every read and command on the backend, including asset history, technician schedules, exports, attachments, and receipt lookup. Frontend assignment filters improve usability but do not protect records. Restrict sensitive personnel data and cost rates independently from operation descriptions.

Audit request disposition, priority changes, order release, reassignment, confirmation, reversal, technical completion, and preventive plan changes. Capture authenticated actor, delegated identity, plant scope, expected/resulting version, reason, and source correlation. Keep business audit evidence separate from debug logging.

Attachments need content validation, malware scanning, access checks, safe content disposition, and retention. Never log credentials, unrestricted findings, or facility photos. Support access should be read-only by default, with time-bounded exceptional repair privileges and review.

## Testing strategy

Test complete transition matrices, including forbidden actions for each role. Race two triagers accepting the same request, two confirmations using the same key, and a release against a concurrently changed operation. Simulate source success followed by transport failure and verify reconciliation without duplicate confirmations.

Test retired equipment, changed installation history, missing technical objects, partial confirmations, reversal, reassignment after partial work, and qualification expiration. Exercise decimal hours, unit conversion, midnight and daylight-saving intervals, counter rollover, corrected readings, overlapping preventive jobs, and a changed plan during call generation.

UI tests cover screen-reader labels, focus after validation, preserved input on failure, busy-state recovery, and differentiation between business notification state and UI messages. Integration tests must use the target system's actual completion and confirmation rules. A mock that accepts every transition cannot validate maintenance controls.

## Operational handover

Publish ownership for request backlog, source-notification creation failures, stuck confirmations, preventive call backlog, stale asset projections, and attachment processing errors. Monitor oldest pending command, duplicate suppression rate, unresolved ambiguous outcomes, failed source correlations, and age of untriaged critical requests.

Provide reconciliation reports connecting requests, business notifications, orders, operations, confirmations, and preventive occurrences. Dead-letter handling preserves original keys and evidence; replay requires diagnosis and verified deduplication behavior. Manual fixes should use supported correction actions, not database updates.

Define support escalation across maintenance, production operations, integration, inventory, purchasing, and finance. Include degraded-mode instructions that do not weaken site safety procedures. Document receipt retention, offline cache cleanup, attachment retention, backup/restore effects, and how restored workers avoid replaying already completed commands.

## Readiness checklist and cross-links

- [ ] Verify release-specific standard applications and supported maintenance APIs.
- [ ] Separate requests, business notifications, orders, operations, permits, and UI messages.
- [ ] Map source statuses and completion rules with the maintenance process owner.
- [ ] Prove concurrent confirmation and ambiguous-outcome recovery.
- [ ] Validate preventive scheduling deduplication and counter correction.
- [ ] Approve technician privacy, attachment controls, and operational escalation.

Related paths from the [root README](../README.md): [procurement](./procurement.md), [warehouse management](./warehouse-management.md), [finance](./finance.md), [object pages](../03-UI-PATTERNS/object-page-pattern.md), [operational apps](../08-FREESTYLE-APPS/operational-apps.md), [authorization](../05-SECURITY/authorization.md), [security patterns](../05-SECURITY/security-patterns.md), [model examples](../09-CODE-EXAMPLES/models.md), and [testing strategy](../06-BEST-PRACTICES/testing-strategy.md).

## Official sources and interpretation

Research date: 2026-10-17.

1. SAP, [OData V4 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v4-model-5de13cf.md), retrieved: model capabilities, entity changes, and operation invocation.
2. SAP, [OData Operations](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-operations-b54f789.md), retrieved: deferred bound actions, parameter handling, and invocation result semantics.
3. SAP OpenUI5, [MessagePopover source and API documentation](https://github.com/SAP/openui5/blob/master/src/sap.m/src/sap/m/MessagePopover.js), retrieved: UI feedback control behavior. This is not a maintenance-notification persistence mechanism.
4. SAP OpenUI5, [ODataContextBinding source and API documentation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataContextBinding.js), retrieved: action binding paths and request lifecycle.
5. SAP, [Fiori apps reference library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) and [SAP S/4HANA documentation](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE): required product/release verification, unavailable during this research.

Framework sources substantiate UI5 patterns, not the hypothetical service schema or maintenance policies. Confirm domain behavior with current target-release documentation and responsible business owners. Pin documentation to the deployed UI5 version because GitHub's evolving branches can describe APIs newer than the installed runtime.
