# Transactional create, change, and display patterns

## Scope and prerequisites

A transactional application changes business state. Creating a requisition, changing a delivery date, approving an invoice, and canceling an order all require more than editable controls. The application must define when work becomes persistent, how failures are corrected, what Cancel means, and which operations are irreversible.

This guide covers freestyle OpenUI5 screens using `sap.m`, `sap.ui.layout`, and OData V4 binding APIs. Examples assume an MVC component, an i18n resource model, an authorized V4 service, and explicit transaction ownership. SAP Fiori elements can coordinate supported lifecycle behavior for compatible services, but the examples are not template extension code. A plain `ObjectPageLayout`, `Wizard`, or two-way binding does not automatically provide draft management.

Use a tested UI5 version supporting the APIs selected here. For the `sap/ui/core/Messaging` singleton integration shown below, target a modern 1.x release such as 1.120 or later and check the deployed API reference. The service supplies metadata, validation messages where supported, and optimistic concurrency semantics. Literal English messages illustrate behavior; externalize them for production.

See the [repository README](../README.md) and neighboring [object page](./object-page-pattern.md), [worklist](./worklist-pattern.md), [list report](./list-report-pattern.md), and [dashboard](./overview-dashboard.md) guides for composition and navigation.

## Model the transaction boundary

Decide what must succeed together. A form containing a header and items may represent one business transaction, while a mass change across independent orders may intentionally allow partial success. A shared screen does not imply shared atomicity. Confirm the server's changeset and action behavior rather than assuming a `$batch` request is a single database transaction.

Separate local input state, queued model changes, persisted draft state, and active business state. They answer different questions. A field can contain invalid text that never reached the model. A queued PATCH can remain unsent in memory. A server draft can be persisted without being activated. A released business object may trigger downstream processes that cannot be undone by resetting local bindings.

Define a lifecycle such as display, editing, validating, submitting, success, correction, conflict, and unavailable. Keep the busy state and operation identity explicit. Prevent a second Save while the first is pending, but do not freeze the entire application unnecessarily. State transitions should follow confirmed outcomes rather than optimistic assumptions.

## Choose create, change, or display

Display mode supports reading and navigation. Prefer text, links, formatted amounts, and statuses over disabled inputs. The user should immediately distinguish an immutable business fact from a field that is merely not currently editable.

Change mode edits an existing object. Determine which fields can change under the object's current business state, and recheck that policy at submission. Field enablement can reflect backend capabilities but does not enforce them. Server authorization and business validation remain mandatory even when the UI hides an action.

Create mode needs initial values, mandatory fields, key assignment, duplicate detection, and a post-create destination. Avoid fabricating persistent keys in the browser unless the service contract explicitly supports client-generated identifiers. After successful creation, navigate using the server-confirmed identity, not the temporary binding path of a transient context.

A copy operation is usually a specialized create flow. Define which fields copy, which reset, and whether attachments or child items are included. Copying a released order should not accidentally copy its released state, approval history, or external reference identifiers.

## Select a working-data strategy

Direct two-way binding to an OData V4 context is appropriate when service types and model messages support the form naturally. Use a dedicated API update group to queue edits until Save. This preserves metadata-aware typing and lets the model manage pending changes. It also requires careful ownership because unrelated controls must not accidentally share the same transaction group.

A JSON working copy can isolate a multi-step form, but it creates a mapping responsibility. Preserve decimal precision, null semantics, date types, and identifiers. Do not round-trip business data through `JSON.stringify()` merely to obtain an “easy clone” when values or application objects are not JSON-safe. A working copy is not a server draft, and closing the browser loses it unless a separately designed persistence mechanism exists.

Do not use `$auto` updates for a flow that promises “Nothing changes until Save.” Automatic submission can persist field changes before the explicit button is pressed. Pick the persistence policy first and configure the model and bindings accordingly. Autosave is valid when clearly communicated, but it needs its own failure and recovery behavior.

## Implement an explicit V4 edit group

The model configuration below is a `manifest.json` fragment for a V4 data source. It keeps normal reads automatic and assigns a default API update group. Application group IDs use API submission by default; declaring the submit mode makes the intent visible [1].

```json
{
  "sap.ui5": {
    "models": {
      "": {
        "dataSource": "mainService",
        "settings": {
          "synchronizationMode": "None",
          "operationMode": "Server",
          "groupId": "$auto",
          "updateGroupId": "edit",
          "groupProperties": {
            "edit": { "submit": "API" }
          }
        }
      }
    }
  }
}
```

The surrounding manifest must define `mainService` as an OData V4 service with an appropriate service URL. A large application should override `$$updateGroupId` on page bindings or use separate model instances when independent transactions coexist. Do not let Save on one tab submit changes made in another tab accidentally.

A small typed field can use service metadata or an explicit type. This XML shows a named JSON working model so that its local validation contract is unambiguous:

```xml
<form:SimpleForm
  xmlns:form="sap.ui.layout.form"
  xmlns="sap.m"
  editable="true" layout="ResponsiveGridLayout">
  <Label text="Description" required="true" labelFor="description"/>
  <Input id="description"
    value="{
      path: 'edit>/Description',
      type: 'sap.ui.model.type.String',
      constraints: {minLength: 1, maxLength: 80}
    }"/>
  <Label text="Quantity" required="true" labelFor="quantity"/>
  <Input id="quantity"
    value="{
      path: 'edit>/Quantity',
      type: 'sap.ui.model.type.Integer',
      constraints: {minimum: 1, maximum: 9999}
    }"/>
</form:SimpleForm>
```

`required="true"` marks the label but does not itself enforce a business rule. Register validation with the message infrastructure, and validate required values and cross-field rules before submission. For a monetary amount, do not reuse the integer example: use the actual decimal type, precision, scale, and currency contract.

## Validate in layers

Validation has at least four layers. Parsing determines whether text can become a typed value. Type constraints check local ranges and lengths. Cross-field rules check relationships such as end date after start date. Server validation checks authoritative rules, authorization, current state, and data changed by other users.

Do not infer validity only from the model object. If a user types “many” into an integer field, parsing can fail while the model retains the previous valid number. Save must detect the control's validation error and must not quietly submit that old number. Conversely, an empty required field may never have been touched and may not yet have a message.

Register the view for validation handling through `Messaging.registerObject(view, true)` and unregister it when destroyed. Provide a message popover or equivalent accessible overview with field navigation. Scope business messages to the current transaction; indiscriminately checking every message in a shared application can block one form because another page has an unrelated error.

On Save, run required and cross-field checks, then inspect relevant error messages and control value states. Keep warnings distinct from blocking errors. If warnings can be overridden, make the override explicit and send any required acknowledgment to the service rather than simply deleting the message from the frontend.

## Submit changes without false success

The following controller methods are for an existing **V4-bound object**, not the JSON working-copy form above. Assume the view element binding owns the `edit` update group. `ui` is a named JSON model containing `busy` and `editing`. `_validateCurrentTransaction()` is an application integration function that returns a boolean after the required, cross-field, and scoped message checks just described.

```javascript
onSave: async function () {
  const view = this.getView();
  const ui = view.getModel("ui");
  if (ui.getProperty("/busy") ||
      !this._validateCurrentTransaction()) {
    return;
  }
  ui.setProperty("/busy", true);
  try {
    await view.getModel().submitBatch("edit");
    if (view.getElementBinding().hasPendingChanges()) {
      this._showTransactionMessages();
      return;
    }
    ui.setProperty("/editing", false);
    this._announceSaved();
  } catch (error) {
    this._showSaveFailure(error);
  } finally {
    ui.setProperty("/busy", false);
  }
},

discardLocalChanges: function () {
  this.getView().getElementBinding().resetChanges();
  this.getView().getModel("ui").setProperty("/editing", false);
}
```

The message-presentation methods are application integration points, not UI5 APIs. They should show localized business text and preserve a support correlation identifier without exposing raw response payloads. This example deliberately does not declare success solely because `submitBatch()` resolved: the official V4 guide notes that contained operation errors can coexist with a successful batch envelope [1].

Cancel should reset only the intended binding or group, after any necessary confirmation. Reset is not a database rollback for previously submitted changes. The V4 guide also warns that explicit operation invocations are not removed like ordinary property changes. Avoid queuing an irreversible action in a group where the UI promises Cancel will discard everything.

## Create entities with V4 contexts

V4 creates entities through a list binding. A bounded example establishes the main APIs:

```javascript
const binding = model.bindList("/Requests", null, null, null, {
  $$updateGroupId: "createRequest"
});
const context = binding.create({
  Description: workingData.Description,
  Quantity: workingData.Quantity
});

context.created().then(function () {
  // Use this persisted context to resolve the post-create destination.
  onPersisted(context);
}).catch(function (error) {
  if (!error.canceled) {
    onUnexpectedCreateFailure(error);
  }
});

await model.submitBatch("createRequest");
```

This excerpt assumes validated working data, a dedicated API group, and application callbacks. Retain the binding until the transient context is either persisted or canceled; destroy it when the flow is finished. Do not destroy the binding immediately after submission if the UI still needs the context.

The `created()` promise represents eventual persistence, not necessarily the completion of the first attempt. A failed create can remain transient and be retried. Use the list binding's supported create lifecycle events, including `createCompleted`, to clear per-attempt busy state and show service errors. Do not await `created()` to decide whether to remove a spinner after an individual failed request, because that can leave the form permanently busy.

Cancellation of a transient create must remove the transient context through the supported V4 context/binding lifecycle, with rejected cancellation promises handled. Deleting a persisted entity is a different server operation. The official create guide documents these distinctions and retry behavior [2]. Do not replace this flow with V2 `createEntry()` or callbacks from `model.create()`.

## Use a wizard for meaningful stages

A wizard suits a long, infrequent task with genuine dependency between stages: identify a request type, enter details, assign responsibility, then review. It is usually excessive for editing three familiar fields. Experienced users may prefer a compact form with clear sections.

The following OpenUI5 wizard uses a JSON working model. Moving through steps does not persist the request. The final review action is a separate integration point that performs full validation and then calls the create flow.

```xml
<mvc:View
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  controllerName="demo.controller.CreateRequest">
  <Wizard id="wizard" complete=".onReview">
    <WizardStep id="identity" title="Request details" validated="false">
      <VBox>
        <Label text="Description" labelFor="requestDescription"/>
        <Input id="requestDescription"
          value="{edit>/Description}" change=".onDetailsChanged"/>
      </VBox>
    </WizardStep>
    <WizardStep id="responsibility" title="Responsibility"
      validated="false">
      <VBox>
        <Label text="Owner" labelFor="owner"/>
        <Input id="owner" value="{edit>/Owner}"
          change=".onOwnerChanged"/>
      </VBox>
    </WizardStep>
  </Wizard>
</mvc:View>
```

```javascript
onDetailsChanged: function () {
  const value = this.getView().getModel("edit")
    .getProperty("/Description") || "";
  const wizard = this.byId("wizard");
  const step = this.byId("identity");
  if (value.trim().length > 0 && value.length <= 80) {
    wizard.validateStep(step);
  } else {
    wizard.invalidateStep(step);
  }
},

onOwnerChanged: function () {
  const owner = this.getView().getModel("edit").getProperty("/Owner");
  const wizard = this.byId("wizard");
  const step = this.byId("responsibility");
  if (owner && owner.trim()) {
    wizard.validateStep(step);
  } else {
    wizard.invalidateStep(step);
  }
}
```

An owner should normally come from a value help or validated identifier lookup; a nonempty string is only local completeness, not proof of a valid or authorized assignee. The official Wizard sample uses `validateStep()` and `invalidateStep()` to control progression and separates the review page from the wizard [3].

If an earlier answer changes the meaning of later answers, invalidate affected steps and use the supported progress-reset API, such as `discardProgress()` for the relevant step. Preserve safe values where appropriate, but do not let stale later-step validation authorize submission. The review page must summarize the actual working data and provide a path back to correct each section.

## Integrate draft as a backend lifecycle

Draft support requires an explicit service protocol and orchestration. A server draft can survive navigation or sessions under defined rules; a JSON buffer or queued in-memory PATCH cannot make that promise. Fiori elements can manage supported draft services, while freestyle applications must implement the compatible lifecycle themselves.

Document how a draft is created, edited, saved, activated, discarded, resumed, and cleaned up. Explain ownership and locking messages. Distinguish “Saved as draft” from “Created” or “Released.” Activation can fail business validation even after the draft itself has been persisted successfully.

Draft is not offline support. Connectivity may be necessary for each draft save or activation. An offline design additionally requires durable protected storage, synchronization, identity and authorization handling, conflict rules, and explicit user communication. Never assure users that unsaved work survives a refresh unless that behavior has actually been implemented and verified.

## Confirm consequential operations

Use confirmations where the consequence warrants interruption: deleting a persisted object, discarding substantial unsaved work, or releasing a document into an irreversible process. Do not ask “Are you sure?” after every ordinary Save; that trains users to dismiss dialogs without reading.

Name the operation, identify the object or selection scope, and explain important consequences. Prefer “Delete 4 selected requests?” over a generic warning. If an operation is reversible, consider an appropriate undo flow instead of a blocking confirmation, but only when the backend truly supports reversal.

```javascript
MessageBox.confirm("Discard the unsaved changes to this request?", {
  actions: [MessageBox.Action.OK, MessageBox.Action.CANCEL],
  emphasizedAction: MessageBox.Action.CANCEL,
  onClose: function (action) {
    if (action === MessageBox.Action.OK) {
      this.discardLocalChanges();
    }
  }.bind(this)
});
```

Import `sap/m/MessageBox` through `sap.ui.define`. For a final product, use action labels that communicate the domain operation when supported and keep the non-destructive choice easy to identify. A confirmation only records user intent; it neither grants authorization nor guarantees the object has remained unchanged.

## Handle concurrency and uncertain outcomes

Use the service's ETag and optimistic concurrency mechanism where available. A conflict must offer a recoverable path: inspect current data, preserve the user's intended changes where safe, and decide whether to reapply. Do not automatically overwrite newer server values or suppress a precondition failure.

A timeout after submission is an uncertain outcome, not necessarily failure. The server may have committed before the response was lost. For creates and irreversible actions, define idempotency keys or a reconciliation lookup with the service team. Blindly retrying can create duplicate business documents or repeat an external side effect.

Separate transport errors from business errors and per-item failures. In a mass action, report what succeeded and what needs correction. Keep unsuccessful selections or provide a result list with identifiers. Request logs should avoid sensitive payloads while preserving correlation information.

## Verification checklist

- [ ] Create, change, display, and copy have documented field and lifecycle rules.
- [ ] Save boundaries match the backend transaction and atomicity contract.
- [ ] Invalid text and untouched required fields both block submission.
- [ ] Field messages are accessible and scoped to the current transaction.
- [ ] A resolved batch envelope cannot produce false success.
- [ ] Failed creates clear attempt-level busy state while remaining correctable.
- [ ] Cancel affects only the intended local transaction.
- [ ] Wizard changes invalidate dependent steps and review data.
- [ ] Draft persistence and active business persistence have distinct messages.
- [ ] Destructive confirmations state object identity and consequence.
- [ ] Conflicts and uncertain outcomes have recovery and reconciliation paths.
- [ ] Backend authorization is enforced for every write and action.

## Official sources

1. [SAPUI5 OData V4 batch control](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/batch-control-74142a3.md): API groups, reset behavior, changesets, retry, and batch versus operation failures.
2. [SAPUI5 creating an entity in a collection](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/creating-an-entity-in-a-collection-c9723f8.md): transient contexts, persistence promises, lifecycle events, and create retries.
3. [OpenUI5 Wizard sample](https://github.com/SAP/openui5/blob/master/src/sap.m/test/sap/m/demokit/sample/Wizard/view/Wizard.view.xml): validated steps, progression, review separation, and final submission.
