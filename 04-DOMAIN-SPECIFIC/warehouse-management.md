# Warehouse Management: Traceable Physical Execution with UI5

## Scope and landscape assumptions

This guide follows inbound receipt, putaway, internal movement, outbound picking, packing, goods issue, and physical inventory. Its working example is a warehouse receiving laboratory instruments and later supplying a maintenance team. [Procurement](procurement.md) covers the commercial order and receipt relationship; [Finance](finance.md) covers accounting consequences. The [repository document index](../README.md#document-index) provides the wider navigation.

First identify the warehouse solution. Classic ECC Warehouse Management uses concepts such as transfer requirements, transfer orders, and quants in its applicable processes. SAP Extended Warehouse Management uses warehouse requests, warehouse tasks, warehouse orders, resources, and other EWM objects. SAP S/4HANA Stock Room Management is another scope requiring its own product documentation. Do not relabel a classic WM transfer order as an EWM warehouse order or assume a common interface.

Embedded and decentralized EWM landscapes have different integration boundaries and operational responsibilities. Inventory Management, warehouse execution, delivery processing, and financial accounting may not update in one database transaction. Establish which system owns each document and what eventual consistency means operationally. An app saying “confirmed” must identify which confirmation occurred.

All code below uses **explicitly illustrative custom OData V2 contracts**. No entity, field, status, or endpoint is presented as an SAP standard API. The adapter behind a custom service must use interfaces supported by the actual landscape. The example does not turn a UI5 app into an RF framework or provide automatic offline posting.

## Discover the supported execution experience

Use the [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) to search the selected product and release for inbound deliveries, outbound deliveries, warehouse tasks, stock movements, and physical inventory. Review application type, supported devices, implementation information, roles, and help links. Some warehouse execution experiences are RF or other supported clients rather than interchangeable UI5 applications.

Compare a proposed mobile app with the supported RF and standard application processes. A custom narrow scanning station can solve a specific need, but reproducing task assignment, exception handling, packing hierarchies, and resource management is a substantial responsibility. Avoid designing a touchscreen facade around guessed movement APIs.

Test actual hardware before choosing controls. Desktop keyboard input, Bluetooth scanner keyboard emulation, integrated industrial scanners, and camera-based capture have different timing, focus, permission, and reliability characteristics. A responsive layout is not proof that a device can scan the required symbology or survive an entire warehouse shift.

Document the supported connectivity model. “Mobile” does not mean “offline.” SAPUI5 and an OData model do not automatically provide durable offline storage, stock reservations, conflict resolution, or secure synchronization. Those capabilities require an explicit architecture and process agreement.

## Object semantics and operational roles

Separate a warehouse instruction from its execution confirmation and from the inventory or delivery posting it may trigger. In EWM, a warehouse task represents work such as a product or handling-unit movement; warehouse orders group tasks for execution according to configured rules. The details and supported transitions belong to the relevant EWM documentation.

A storage bin identifies a physical location within the warehouse structure. A handling unit represents packaging and its contents, not simply an alternate product identifier. Product, batch, serial number, stock type, owner, and party entitled to dispose can matter to stock identity in the applicable landscape. A quantity alone cannot identify which stock may be moved.

| Role | Responsibility | Essential boundary |
|---|---|---|
| Receiving operator | Identify arrivals and record exceptions | Authorized receiving warehouse and activity |
| Putaway operator | Execute destination movement | Assigned resource or permitted tasks |
| Picker | Confirm selected stock and quantity | Task assignment and stock eligibility |
| Packer | Build and validate shipping units | Delivery and packing restrictions |
| Inventory counter | Record observed stock | Blind-count and separation rules |
| Supervisor | Resolve exceptions and reassign work | Controlled override with reason |
| Integration support | Reconcile messages and posting failures | No untracked physical corrections |

These are operational responsibilities, not universal SAP role names. An operator authorized for one warehouse must not retrieve another warehouse's stock through a crafted query. A supervisor's ability to reassign work does not imply authority to post financial inventory differences.

## Inbound: arrival, receipt, inspection, and putaway

### Identify the arrival

Start with the expected inbound reference where available: delivery, purchase order, shipment, or another supported warehouse request. Confirm supplier or source, expected products, packaging, quantities, and unloading location. Preserve external references without assuming they are globally unique.

For twelve expected instruments, eight may arrive on the first vehicle. Record eight physically observed items rather than automatically accepting the delivery's planned quantity. Capture damage, missing labels, packaging discrepancies, and unexpected products with explicit exception codes. Photographs and attachments need governed storage and access, especially where labels expose customer information.

Scan handling units and products according to the actual process. A pallet label can identify a handling unit whose contents must still be verified. A product barcode may identify a trade item with a packaging level, not one base unit. Explain ambiguous scans instead of selecting the first matching product.

### Post the appropriate receipt event

Goods receipt changes a particular business state under the configured process. It is not automatically equivalent to unloading, quality acceptance, stock becoming unrestricted, or final putaway. The UI should label each event and show document references or downstream pending states.

Coordinate with [Procurement](procurement.md) so the same physical arrival is not independently posted from both a purchasing receipt screen and a warehouse confirmation screen. Define the supported integration route and ownership. If warehouse execution triggers a receipt update, the purchasing view should reflect that result rather than offering an ungoverned duplicate action.

Posting dates follow business rules; arrival timestamps describe physical events. Keep these distinct. A delayed inbound message may post after the truck arrived. A user in another time zone should not inadvertently shift the business posting date when selecting a date-only value.

### Inspect and put away

Quality inspection may restrict availability or require a decision before certain movements. Putaway chooses eligible destination locations based on configured rules, capacity, compatibility, and stock properties. Do not let a custom app override a proposed bin simply because any free-text value is syntactically valid.

Guide the operator through source confirmation, product or handling-unit identification, quantity, and destination confirmation. Record the actual destination. If a proposed bin is physically blocked, capture an exception and request a supported alternate destination; do not confirm the proposed bin while placing goods elsewhere.

Partial putaway leaves explicit residual work. If six of eight instruments fit in a bin, the remaining two need traceable stock and a supported split or follow-on instruction. A client must not invent a second task locally without backend validation and assignment.

## Internal movements and replenishment

An internal move changes location, availability, or stock attributes according to its supported process. Distinguish a bin-to-bin move from a stock-type change and from an Inventory Management transfer. The same visible action label can hide different accounting and execution consequences.

Replenishment should preserve the relationship between demand, source stock, destination, and task priority. Two operators seeing a low pick-face quantity must not both create overlapping replenishment blindly. The backend coordinates eligibility, capacity, reservations where applicable, and task creation.

Require reason codes for unplanned movements and supervisor review where policy demands. A “move anything” screen can bypass stock segregation, hazardous-material rules, or quality restrictions if authorization only checks the user's warehouse. Validate stock type, batch, ownership, unit, and destination at execution time.

If physical movement occurred but confirmation failed, establish a supervisor recovery process. The operator should retain the original task and observed locations, stop conflicting follow-on activity, and report the uncertain result. Automatically reversing physical instructions in the browser cannot return a pallet to its original bin.

## Outbound: allocate, pick, pack, and issue

### Establish eligible work

Outbound work starts from the applicable delivery or warehouse request and configured stock-selection rules. Allocation and task creation are not the same as physical picking. Availability may depend on stock type, batch, shelf life, customer restrictions, and ownership.

Show the operator enough information to distinguish stock safely: source bin, product, expected unit, batch or serial requirements, and handling-unit context. Do not expose unnecessary customer details on a shared handheld screen. Respect resource and assignment controls so operators do not confirm someone else's task by guessing an identifier.

### Confirm picking with controlled exceptions

Require scans in a deliberate sequence where appropriate: source bin, product or handling unit, quantity, then destination or staging location. A source-bin scan prevents a correct product being picked from the wrong stock segment. A serial-controlled process may require each individual identifier, not merely a total quantity.

Short picks should capture observed quantity and reason. The remainder might require another task, replenishment, delivery adjustment, or supervisor resolution. Do not automatically substitute a different batch or location because the originally proposed stock is absent. The substitution must be permitted and traceable.

For a task of four units, a confirmation of three must not silently become four after a scanner suffix or default-field behavior. Show the final quantity and unit clearly before commitment, using confirmations appropriate to risk and throughput. Excessive confirmation dialogs can reduce attention; use them selectively, not as a substitute for validation.

### Pack and validate shipping units

Packing associates actual contents with packaging and handling-unit identifiers. Maintain containment rules: a unit cannot be packed into itself, duplicated under two parents, or assigned contradictory delivery contents. Weight, dimensions, packaging compatibility, and maximum loads may require checks under the specific process.

Label creation has its own lifecycle. A successful print request is not proof the label physically printed or was attached to the right carton. Track print job and label identity; support controlled reprints with visible reprint history. Reprinting a label should not create another business handling unit unless that is the intended operation.

Verify packed contents against delivery requirements and scanned identifiers. A carton containing three items cannot be declared complete merely because the delivery expected four. For regulated or serialized goods, retain the relationship between item, serial, package, and dispatch reference.

### Stage, load, and post goods issue

Staging and loading are physical execution steps distinct from final goods issue under the configured process. Gate the appropriate actions on required picking, packing, and checks. A carrier reference should correlate to the correct shipment or delivery, not be copied across unrelated loads by a default.

Goods issue can have inventory, delivery, and accounting consequences. Its reversal is a governed business operation, not a local status reset. If issue posting fails after loading, notify the operational owner and prevent users from assuming dispatch is fully recorded.

In a distributed landscape, show “warehouse confirmed; inventory update pending” when that is the truthful state. Preserve integration references and monitor aging. A queue error should not prompt another physical pick or another goods issue with a fresh identifier.

## Physical inventory and difference handling

Physical inventory requires a defined population and counting method. Establish whether the count is by bin, product, handling unit, or another supported basis, and how movements during the count are controlled. A count document's snapshot and current stock are not necessarily the same observation.

Blind counting can reduce anchoring. If policy requires it, do not reveal book quantity through a hidden field, an export, or an accessible API response. Capture counted quantity and unit, batch, serials where applicable, counter, observation time, and explicit zero-count indication. Empty input must not mean zero.

A difference may trigger recount, investigation, or authorized adjustment. Separate the person counting from the person approving significant differences where required. A recount is a new observation, not an overwrite that erases the first count.

| Condition | Action | Evidence |
|---|---|---|
| No stock physically found | Explicit zero count | Counter and count reference |
| Unexpected product in bin | Record supported exception | Observed identifier and location |
| Difference beyond review threshold | Recount or supervisor review | Original and subsequent observations |
| Movement occurs during count | Apply configured count control | Movement and snapshot references |
| Adjustment approved | Supported difference posting | Approver and posting reference |

Never make a difference disappear by editing a stock total directly. Inventory adjustments can affect valuation and accounting; coordinate with [Finance](finance.md). Report counted, approved, and posted states separately so a completed handheld form is not mistaken for a posted adjustment.

## State machines and decision boundaries

The following custom states explain a mobile execution design. They do not reproduce EWM or WM status codes.

```text
Task execution:
Available -> Assigned -> InProgress -> ConfirmationPending -> Confirmed
InProgress -> Exception -> Replanned | Cancelled
ConfirmationPending -> OutcomeUnknown -> Reconciled

Mobile scan session:
AwaitSource -> AwaitProduct -> AwaitQuantity -> AwaitDestination -> Review
Review -> Sending -> Confirmed | NeedsCorrection | OutcomeUnknown

Inventory count:
Created -> Counting -> Submitted -> RecountRequired | Reviewed
Reviewed -> AdjustmentPending -> Posted
```

Do not automatically return `OutcomeUnknown` to `InProgress`. The original confirmation may have committed. Resolve the command result first, then reload the authoritative task. Likewise, cancellation must check whether stock already moved and whether follow-on work exists.

A task state, integration state, and physical observation are different dimensions. A supervisor investigating an incident needs all three. Store “operator reports pallet at staging lane” as evidence, not as a replacement for the confirmed stock location.

## Illustrative custom contract and UI5 scan interaction

Assume an OData V2 service `/example/warehouse/` with these explicitly custom entities:

| Entity set | Selected properties | Semantics |
|---|---|---|
| `ExecutionTasks` | `TaskId`, `Revision`, `SourceBin`, `DestinationBin`, `ProductId`, `ExpectedQuantity`, `Unit` | Read model; current revision required |
| `ConfirmationCommands` | `RequestId`, `TaskId`, `ExpectedRevision`, `ScannedSource`, `ScannedProduct`, `ActualQuantity`, `Unit` | Create-only confirmation intent |
| `CommandResults` | `RequestId`, `Outcome`, `ConfirmationReference`, `IntegrationState` | Authorized recovery and synchronization status |

All identifiers are strings. `ActualQuantity` and `ExpectedQuantity` are `Edm.Decimal(15,3)` in this example. The service permits only full confirmation here; partial quantity, batch, serial, handling-unit, and destination exceptions require a richer implemented contract and workflow. Do not deploy the simplified contract for tasks requiring those controls.

The server validates source, product, unit, quantity, current revision, task assignment, stock eligibility, and authorization atomically. A request identifier deduplicates the business intent; a different payload with the same identifier fails. Return the original outcome for a duplicate retry. A separate integration status explains downstream synchronization without inviting another confirmation.

The XML uses a local `scan` JSON model with `source`, `product`, `quantity`, `busy`, and `ready`. Quantity is a canonical decimal string validated before it reaches this review screen. Localized entry parsing belongs in a dedicated entry step. The `task` model supplies the authorized task context.

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  controllerName="example.warehouse.Confirm">
  <VBox class="sapUiSmallMargin" busy="{scan>/busy}">
    <Title text="{task>TaskId}" />
    <Label text="Source bin scan" labelFor="sourceScan" />
    <Input id="sourceScan" value="{scan>/source}"
      submit=".onSourceSubmit" />
    <Label text="Product scan" labelFor="productScan" />
    <Input id="productScan" value="{scan>/product}" />
    <Text text="{scan>/quantity}" />
    <Text text="{task>Unit}" />
    <Button text="Confirm reviewed movement" press=".onConfirm"
      enabled="{= ${scan>/ready} &amp;&amp; !${scan>/busy} }" />
  </VBox>
</mvc:View>
```

Scanner integration is intentionally limited to keyboard-wedge input. A `submit` event depends on device configuration and terminator; verify it on supported hardware. Camera or native scanner integration needs its own documented adapter and permission handling. The example does not assume a universal browser scanning API.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/m/MessageBox"
], function (Controller, MessageBox) {
  "use strict";
  return Controller.extend("example.warehouse.Confirm", {
    onSourceSubmit: function () {
      this.byId("productScan").focus();
    },
    onConfirm: function () {
      const view = this.getView();
      const scan = view.getModel("scan");
      const model = view.getModel("task");
      const context = view.getBindingContext("task");
      if (!context || scan.getProperty("/busy") ||
          !scan.getProperty("/ready")) {
        return;
      }
      const task = context.getObject();
      if (!this._confirmationIntent) {
        this._confirmationIntent = {
          RequestId: globalThis.crypto.randomUUID(),
          TaskId: task.TaskId,
          ExpectedRevision: task.Revision,
          ScannedSource: scan.getProperty("/source"),
          ScannedProduct: scan.getProperty("/product"),
          ActualQuantity: scan.getProperty("/quantity"),
          Unit: task.Unit
        };
      }
      scan.setProperty("/busy", true);
      model.create("/ConfirmationCommands", this._confirmationIntent, {
        success: function (result) {
          scan.setProperty("/busy", false);
          if (result.Outcome === "COMPLETED") {
            this._confirmationIntent = null;
            scan.setProperty("/ready", false);
            model.refresh(true);
            MessageBox.success("Movement confirmation recorded.");
          } else {
            MessageBox.information(
              "Request accepted. Reconcile before repeating the movement."
            );
          }
        }.bind(this),
        error: function () {
          scan.setProperty("/busy", false);
          MessageBox.warning(
            "Outcome unknown. Keep the request and check its result."
          );
        }
      });
    }
  });
});
```

This handler assumes a nondeferred create group, normal V2 CSRF handling, and a secure browser supporting `crypto.randomUUID`. A production app needs safe lifecycle handling and a durable recovery design; controller memory does not survive reload. Block switching tasks while a confirmation is unresolved or persist the immutable intent under approved device policy.

`ready` is a presentation flag, not an authorization or validation result. Client-side checks reduce errors, but the service must repeat them. A supervisor resolving a definite validation failure may permit a corrected new intent. For an uncertain response, retain the original payload and identifier until authorized result lookup establishes the outcome.

## Scanning correctness, units, and time

Define supported symbologies and payload formats. Preserve leading zeros and meaningful separators; scanner preprocessing must not blindly trim identifiers or remove control characters used by a structured barcode format. Parse known formats in a governed adapter and reject malformed or ambiguous data.

Account for duplicate scanner events, rapid repeated scans, a terminator arriving late, and focus moving into the wrong field. A debounce can prevent UI noise but cannot provide transactional idempotency. Audible and visual feedback should distinguish capture, validation, and successful backend posting.

Keep quantity as an exact decimal with unit. One scanned case may represent multiple each, and variable-weight items need additional semantics. Use governed conversion factors and capture actual weight where required. Never infer quantity from the number of scan events unless the process explicitly defines one scan per physical unit.

Record physical observation time separately from server receipt and posting times. Use UTC instants for audit events and explicit zones for operational display. Business dates remain dates. Device clocks can drift, so synchronization and conflict rules must not depend solely on a handheld timestamp.

## Connectivity and synchronization without magical offline support

The simplest safe model is online-only posting with clear connectivity feedback. If connectivity drops before an action, stop submission and preserve entered data according to policy. If it drops after submission, treat the result as unknown rather than declaring failure.

An offline-enabled design needs secure local storage, user and device identity, queue durability, immutable request identifiers, retry policy, expiry, logout behavior, and conflict resolution. It also needs a business rule for what physical work may occur without server authorization. Offline tasks do not reserve stock automatically.

On reconnection, authenticate, recover authoritative command results, and revalidate remaining intents against current tasks and stock. Do not use last-write-wins for confirmations. A task may have been reassigned or confirmed elsewhere. Expired or conflicting work should enter a supervisor queue with physical observations preserved.

Define downstream synchronization separately from device upload. A confirmation may reach the warehouse service while an inventory integration remains pending. Monitor both links with correlation identifiers, aging, retry ownership, and reconciliation procedures. Repeatedly refreshing a UI does not repair a blocked integration queue.

## Security, concurrency, audit, and recovery

Enforce warehouse, activity, stock, and assignment authorization on every command. Restrict exports, labels, attachments, and value helps consistently. Shared devices need accountable sessions, controlled handover, inactivity rules appropriate to operations, and protection against stale data from the previous user.

Concurrent confirmations require backend locking or equivalent transactional controls. A revision check protects task changes, but stock availability and handling-unit location also need validation. Two workers must not both confirm the same pallet because each saw an Available task earlier.

Record actor, resource or device context where appropriate, task revision, scanned identifiers, quantity and unit, exception reason, request identifier, event times, and resulting document references. Log privileged overrides and label reprints. Avoid storing unnecessary customer or employee data in scanner telemetry.

Recover according to failure class. Invalid scans require correction. Stale tasks require reload and comparison. Authorization failures require proper assignment. Unknown posting outcomes require reconciliation. Physical mismatch requires operational investigation, not an automatic technical retry.

## Shift handover and unresolved physical work

At shift change, distinguish available work, assigned but untouched work, goods physically moved with unknown confirmation, and confirmed work awaiting integration. Reassigning a task must not erase an unresolved observation from the previous operator. Require supervisor acknowledgement where another worker could otherwise repeat a movement.

Handover should include affected bins or staging lanes, task and handling-unit references, command identifiers, and the immediate safety or stock-control action. A device battery failure is not justification to confirm work under another person's session. Restore accountable access, reconcile the original intent, and only then resume execution.

For a scanner replacement during a shift, distinguish cached display data from unresolved commands. The replacement device should load current authorized work rather than importing a previous device's local task list as authoritative. If an unresolved intent is transferred through an approved recovery mechanism, preserve its identifier and payload exactly. Record the handover so support can determine which device captured the observation and which device performed reconciliation. Never let both devices independently create new confirmations for the same physical movement.

## Tests and operational readiness checklist

Use real supported devices and representative labels. Include damaged barcodes, long identifiers, leading zeros, duplicate scans, lost focus, and gloves or accessibility constraints. Test screen-reader labels and keyboard-only alternatives where applicable.

| Scenario | Required evidence |
|---|---|
| Two workers confirm the same task | One valid confirmation; no duplicate movement |
| Connection drops after commit | Original confirmation recoverable |
| Same identifier, changed quantity | Payload conflict rejected |
| Wrong bin with correct product | Server blocks confirmation |
| Zero count versus blank count | Distinct, correctly validated outcomes |
| Partial inbound arrival | Residual quantity remains traceable |
| Reprint shipping label | No duplicate handling unit created |
| Integration queue blocked | Pending state and support ownership visible |
| Task reassigned during offline interval | Conflict investigated, not overwritten |
| Unauthorized warehouse query | No stock leakage through lists or counts |

Before release:

- Confirm the landscape is ECC WM, EWM, Stock Room Management, or another explicitly documented scope.
- Verify standard application or RF support against the selected product and release.
- Document task ownership, movement semantics, unit conversion, and exception workflows.
- Prove backend authorization, idempotency, stock concurrency, and recovery after uncertain responses.
- Validate scanners, labels, focus behavior, battery expectations, and connectivity at actual work locations.
- Separate physical execution, warehouse confirmation, integration, and accounting status.
- Define count controls, recount approval, and inventory-difference posting ownership.
- Establish supervisor procedures for goods moved physically but not confirmed electronically.

## Official references and research notes

Research used search-indexed SAP Help documentation and SAP-maintained UI5 sources. Direct Help Portal and app-library fetches were unavailable, so no release-independent application prerequisites or identifiers are asserted. Historical documentation links below explain concepts; verify behavior in the actual product version.

1. [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) — application discovery and implementation review.
2. [SAP Help: Extended Warehouse Management](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/9832125c23154a179bfa1784cdc9577a/4ecb88b8b2422afee10000000a42189e.html) — EWM process context.
3. [SAP documentation: Warehouse Task](https://help.sap.com/saphelp_SCM700_ehp02/helpdata/en/8f/cccb53ad377114e10000000a174cb4/content.htm) — product and handling-unit task concepts in its documented release.
4. [SAP Help: Warehouse Order Creation](https://help.sap.com/docs/SAP_EXTENDED_WAREHOUSE_MANAGEMENT/3d97bec9bf1649099384bb8167df3cf2/48890776d2b21b5ae10000000a42189c.html) — grouping warehouse tasks into warehouse orders.
5. [SAP Help: Physical Inventory](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/9832125c23154a179bfa1784cdc9577a/4cb4e7d54c1f1921e10000000a15822b.html) — inventory process context.
6. [SAP-maintained SAPUI5 documentation: OData V2 Model](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md) — server-backed bindings, CRUD, batching, concurrency, and token handling; not a promise of offline execution.
7. [SAP Help: Transfer Requirement](https://help.sap.com/docs/SAP_ERP_SPV/95c61932dbb841a9a32b10ccd6d72d3f/8226bd53e3acb64ce10000000a174cb4.html) — classic ERP warehouse planning terminology, distinct from EWM task terminology.
8. [SAP Help: Stock Room Management](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/b2dee5e83e2446149294f9860a7c08f0/ee8abd534f22b44ce10000000a174cb4.html) — separately documented warehouse-management scope.
