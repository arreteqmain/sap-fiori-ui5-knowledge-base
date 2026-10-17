# Operational Freestyle Applications

## Scope and prerequisites

Operational applications help users change business reality: release an order, confirm a repair, reserve inventory, or resolve an exception. Their success is measured by a trustworthy outcome and recoverable failures, not simply by whether the screen renders. This chapter develops a work-order editing scenario with list/detail navigation, explicit save boundaries, validation, concurrency, and long-running operations.

Examples assume UI5 **1.120**, asynchronous JavaScript modules, XML views, `sap.ui.core`, `sap.m`, and optionally `sap.f`. OData **V4** examples use `sap.ui.model.odata.v4.ODataModel`; the separately labeled V2 section uses `sap.ui.model.odata.v2.ODataModel`. The two models are alternatives, not interchangeable adapters. Entity names and actions are illustrative and must exist in the actual service metadata. The primary example assumes a non-draft V4 service and one active editor per update group.

Prerequisites are [freestyle development](freestyle-development.md), [data binding](../02-ARCHITECTURE-PATTERNS/data-binding.md), and a documented service contract. Obtain that contract before implementing Save: key types, nullable fields, precision, ETags, business actions, authorization failures, message formats, and atomicity guarantees all affect the screen. The official [V4 model API documentation in source](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js) and [V2 counterpart](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js) are the baseline references.

## Distinguish a UI edit from a business transaction

A UI edit starts when the user changes a field. A pending model update is a client-side intention. A successful HTTP response confirms a particular service request, not necessarily completion of a wider process. A business transaction is the backend-defined consistency boundary that enforces invariants and commits or rejects related changes.

For example, editing an order description may be a PATCH, while releasing the order may validate material availability and change several backend records. Saving the description does not release the order. A multi-service cockpit cannot create atomicity by placing requests into one JavaScript promise array. Its backend must provide a coordinating operation or the interface must expose partial completion and compensation explicitly.

OData batches collect requests. Changesets or atomicity groups can provide protocol-level all-or-nothing behavior where supported, but their actual scope and backend implementation must be verified. UI5 update groups choose when requests are submitted; they are not a general distributed transaction manager. Document the business meaning of “Save,” “Submit,” “Post,” and “Activate” separately.

Draft services introduce another distinction: saving a draft can persist unfinished work without making the active business object effective. Draft ownership, lock expiry, resume, discard, and activation require the service's supported draft protocol. Do not add a JSON property named `IsDraft` and assume equivalent semantics. If a standard draft-driven object fits a supported Fiori elements floorplan, reevaluate the cost of implementing that lifecycle in freestyle.

## Model the interaction as a state machine

Define allowed actions for each state before attaching handlers. A collection of unrelated booleans such as `editing`, `saving`, `hasError`, and `done` can represent impossible combinations. One explicit interaction phase plus derived flags makes it easier to reason about keyboard shortcuts, route changes, and repeated clicks.

| Phase | Meaning | Allowed actions |
| --- | --- | --- |
| Loading | Identity is known; data is unresolved | Cancel navigation where safe |
| Display | Server state is shown | Edit, refresh, business action if authorized |
| Editing | Local changes may exist | Save, cancel, permitted field interactions |
| Saving | Submission is in progress | Wait; do not start a second submit |
| Validation failure | Input or business rules reject changes | Correct fields, retry, discard |
| Conflict | Another actor changed the object | Compare, reload intentionally, or preserve a local copy |
| Not available | Missing, forbidden, or unavailable data | Explain and provide a safe destination |

A transport interruption after submission may leave the result **unknown**, which is not equivalent to a rejected operation. For consequential actions, provide a reconciliation path: retrieve current status or query an operation identifier. Retrying a financial posting blindly can duplicate business effects. This requirement belongs in service design, not just error-message wording.

Store the phase in a view-local JSON model. Derive enabled states from it and from permissions. Do not derive backend authorization from the phase. A disabled button prevents accidental interaction but the service must enforce access for every operation.

## Preserve identity through master/detail navigation

The master list supplies contexts; the detail route carries a stable business key. Selection and layout are UI state, while the selected work order remains service data. On desktop, the master can remain visible next to detail; on a phone, the same relationship becomes sequential navigation. The official [FlexibleColumnLayout documentation](https://github.com/SAP/openui5/blob/1.120.0/src/sap.f/src/sap/f/FlexibleColumnLayout.js) explains the list-detail-detail structure, column aggregations, layout property, and `stateChange` event.

When using `sap.f.routing.Router`, route targets must be placed into the appropriate column aggregations and layout must be managed coherently. Do not copy a `sap.m.routing.Router` descriptor and merely replace the root control. Use the semantic helper where appropriate, and record explicit user layout changes in navigation state without creating history entries for every resize event.

Navigating to object B while A is dirty requires a decision before rebinding. “Stay” preserves A and its selection; “Discard” resets only A's pending scope; “Save” waits for verified completion before continuing. A controller that immediately binds B and then asks the question has already endangered A's context. Coordinate route-triggered and list-triggered navigation through the same policy.

Not every navigation comes from an application button. Test direct hashes, browser Back, launchpad navigation, and closing a tab. Use the host's supported dirty-state/navigation integration where available; do not overwrite global shell handlers or assume a browser unload prompt can perform asynchronous persistence.

## Establish the V4 update boundary

Use the named group `workOrderEdit` from the [development chapter](freestyle-development.md) with submit mode `API`. Bind the detail object with `$$updateGroupId` if its scope differs from the model default. The service below exposes `WorkOrders` with GUID `ID`, text `Description`, and integer `Priority`. The form is deliberately small; add field-specific validation and a message summary before expanding it.

```xml
<mvc:View
  controllerName="demo.workorders.controller.Edit"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  xmlns:core="sap.ui.core">
  <Page title="{i18n>editWorkOrder}">
    <content>
      <VBox class="sapUiSmallMargin">
        <Label text="{i18n>description}" labelFor="description" />
        <Input id="description"
          value="{
            path: 'Description',
            type: 'sap.ui.model.odata.type.String',
            constraints: { nullable: false, maxLength: 80 }
          }"
          editable="{= ${ui>/phase} === 'Editing' }" />
        <Label text="{i18n>priority}" labelFor="priority" />
        <Input id="priority"
          value="{
            path: 'Priority',
            type: 'sap.ui.model.odata.type.Int32',
            constraints: { nullable: false }
          }"
          editable="{= ${ui>/phase} === 'Editing' }" />
      </VBox>
    </content>
    <footer>
      <OverflowToolbar>
        <ToolbarSpacer />
        <Button text="{i18n>save}" type="Emphasized"
          enabled="{= ${ui>/phase} === 'Editing' }"
          press=".onSave" />
        <Button text="{i18n>cancel}"
          enabled="{= ${ui>/phase} === 'Editing' }"
          press=".onCancel" />
      </OverflowToolbar>
    </footer>
  </Page>
</mvc:View>
```

The integer example validates an Int32 value and disallows null. The business priority range, for example one through five, needs separate domain validation; the OData integer type does not implement arbitrary minimum/maximum constraints. Domain rules still belong on the server. A required indicator or `nullable: false` is not a substitute for rules about blank text after trimming. Decimal quantities, currencies, and dates need their appropriate OData types and units; avoid converting decimal strings through JavaScript floating-point arithmetic.

Register the view with `sap/ui/core/Messaging` and automatic validation handling. The following complete controller skeleton assumes the view is bound by its route coordinator, contains the two local field IDs above, and has the stated V4 model. Translation keys include `saveFailed`, `saveRejected`, and `saved`. The message count intentionally filters client validation messages only; server failures are also checked through pending changes after submission.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/ui/core/Messaging",
  "sap/ui/core/library",
  "sap/m/MessageBox",
  "sap/m/MessageToast"
], function (Controller, JSONModel, Messaging, coreLibrary,
    MessageBox, MessageToast) {
  "use strict";
  const MessageType = coreLibrary.MessageType;

  return Controller.extend("demo.workorders.controller.Edit", {
    onInit: function () {
      this._ui = new JSONModel({ phase: "Editing" });
      this.getView().setModel(this._ui, "ui");
      Messaging.registerObject(this.getView(), true);
    },

    _text: function (sKey) {
      return this.getView().getModel("i18n")
        .getResourceBundle().getText(sKey);
    },

    _hasInputErrors: function () {
      const aControlIds = ["description", "priority"]
        .map((sId) => this.byId(sId).getId());
      return Messaging.getMessageModel().getData().some(function (oMessage) {
        return oMessage.getValidation()
          && oMessage.getType() === MessageType.Error
          && oMessage.getControlIds().some((sId) => aControlIds.includes(sId));
      });
    },

    onSave: async function () {
      if (this._ui.getProperty("/phase") !== "Editing"
          || this._hasInputErrors()) {
        return;
      }
      const oModel = this.getView().getModel();
      this._ui.setProperty("/phase", "Saving");
      try {
        await oModel.submitBatch("workOrderEdit");
        if (this._leaving) {
          return;
        }
        if (oModel.hasPendingChanges("workOrderEdit")) {
          MessageBox.error(this._text("saveRejected"));
          return;
        }
        MessageToast.show(this._text("saved"));
      } catch (oError) {
        if (!this._leaving) {
          MessageBox.error(this._text("saveFailed"));
        }
      } finally {
        if (!this._leaving) {
          this._ui.setProperty("/phase", "Editing");
        }
      }
    },

    onCancel: function () {
      if (this._ui.getProperty("/phase") === "Editing") {
        this.getView().getModel().resetChanges("workOrderEdit");
      }
    },

    onExit: function () {
      this._leaving = true;
      Messaging.unregisterObject(this.getView());
    }
  });
});
```

This skeleton illustrates the submission boundary, not a complete navigation workflow. Cancel resets pending model changes in the exclusive group; it does not undo previously committed backend changes. Invalid input can remain in a control even when it never reached the model. A full Cancel implementation must clear scoped validation messages and rebind or recreate the form to restore displayed values, then navigate only after discard succeeds.

`submitBatch` can resolve even when an individual change request fails; its promise alone is not a Save-success test. For this isolated editor, retained pending changes prevent a false success message. A richer application should also inspect scoped service messages and operation-specific outcomes. Do not clear unrelated messages to make the Save button green. The official [V4 context implementation and public API comments](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/v4/Context.js) describe entity-level pending changes and asynchronous updates.

## Handle server messages as actionable information

Show field-level errors near their inputs and provide a message summary for multiple errors. Keep persistent business rejection messages available until corrected or deliberately dismissed. A toast is appropriate for a brief confirmation, not for an error containing instructions the user must remember.

Message targets are part of the service contract. A target for `Priority` can be associated with a property binding; a global business error may have no field target. Preserve the distinction instead of assigning every error to the first input. When a message summary entry is activated, reveal the relevant section and focus the field through public control APIs.

Client validation improves feedback speed, while backend validation protects consistency. Both may reject different aspects of the same field. Do not suppress backend errors because the UI already checked a regular expression. When errors contain technical details, show a safe business explanation first and expose support information separately without leaking confidential payloads.

Use object-scoped message ownership when navigating between orders. Messages for A should not appear as validation errors on B, yet unresolved global service warnings may still matter. Registering and unregistering the view manages validation integration; it does not authorize indiscriminate removal of every message in the shared message model.

## Define V2 behavior separately

For a V2 service, configure deferred groups and change groups using V2 APIs. The following is a **V2-only setup excerpt** executed once after the component has created its V2 model. `Maintenance.WorkOrder` must exactly match the entity type's qualified name in that service's metadata.

```javascript
const oModel = this.getModel(); // sap.ui.model.odata.v2.ODataModel
const sGroup = "workOrderEdit";
const aDeferred = oModel.getDeferredGroups();
if (!aDeferred.includes(sGroup)) {
  oModel.setDeferredGroups(aDeferred.concat(sGroup));
}
const mChangeGroups = Object.assign({}, oModel.getChangeGroups());
mChangeGroups["Maintenance.WorkOrder"] = {
  groupId: sGroup,
  changeSetId: "workOrderChange",
  single: false
};
oModel.setChangeGroups(mChangeGroups);
```

V2 submission uses `submitChanges({groupId, success, error})`; inspect nested batch/change responses because the outer callback can represent a successful batch transport containing failed operations. V2 imperative reads use `read`, and key paths should use the model's `createKey` after metadata is available. V4 uses list/context bindings, `requestObject`, `submitBatch`, and operation bindings instead. Do not wrap a V2 callback API and label it a V4 solution.

V2 `resetChanges` and V4 `resetChanges` differ in parameters and scope. In V2, path-based reset behavior must be checked against the pending entities and any created entries. In V4, group-based reset is meaningful for an application update group, subject to restrictions while requests are running. Neither is a database rollback. Draft discard is a backend operation, not simply resetting a client cache.

## Make business actions explicit

A “Release” action should call the service operation designed for release, not PATCH a status code if that bypasses business checks. In V4 a bound action is represented by a deferred operation binding. The qualified operation name below is hypothetical and must be declared as a bound action in metadata.

```javascript
// Controller method; the view has a bound V4 WorkOrder context.
async function releaseWorkOrder(oContext) {
  const oModel = oContext.getModel();
  const oAction = oModel.bindContext(
    "Maintenance.Release(...)", oContext
  );
  try {
    await oAction.execute("$direct");
    await oContext.requestSideEffects([
      { $PropertyPath: "Status" }
    ]);
  } finally {
    oAction.destroy();
  }
}
```

This example assumes all edits have already been saved successfully and `Status` is the only displayed property requiring refresh. The action's result contract might instead return a new entity or operation identifier. `$direct` sends immediately, so never use this example to bypass pending edits unintentionally. Keep an action-specific busy guard, handle rejection, and verify status before allowing a second attempt.

If release has irreversible consequences, confirmation should describe those consequences and identify the object. Confirmation is not authorization or idempotency. An uncertain timeout still requires reconciliation. Refresh only known affected properties or bindings when supported; a global model refresh can discard useful list position and issue unrelated requests.

## Resolve concurrency without silent overwrites

ETags enable optimistic concurrency where the service supports them. If another user changes the order, a stale update may receive HTTP 412. Show that the data changed, retain the user's proposed values where safe, and offer a deliberate comparison or reload path. Do not retry with a wildcard match header merely to make Save succeed.

A lock and an ETag solve different problems. A lock can reserve editing rights; an ETag detects a version mismatch. Lock expiry, abandoned sessions, and background backend changes still require clear handling. Never use an application's edit flag as proof that the backend lock remains valid.

For a detached edit buffer, retain the original key and concurrency token as part of the editor's context, not as editable fields. On conflict, distinguish the original snapshot, the user's proposal, and the current server value. A silent merge is acceptable only for fields whose merge semantics have been explicitly defined.

## Design bulk operations for partial results

Bulk “Approve selected” requires a result model keyed by business identity. Capture the selected contexts at the start and explain whether the operation applies only to loaded rows or to all rows matching a server query. A visual select-all checkbox must not imply that only the currently rendered page is the whole dataset.

Show progress and per-object outcomes for independent operations. If the service guarantees one atomic bulk transaction, say so and present one outcome; otherwise expose successful, rejected, and unknown items separately. Retrying only failed items requires idempotency or reliable reconciliation. Never report overall success because the last promise resolved.

Long-running operations often need an asynchronous backend job rather than an open HTTP request. Persist the job identifier, show where results can be found later, and stop client polling when its owner is destroyed. Closing a dialog should not claim that the backend job was cancelled unless the service actually supports and confirms cancellation.

## Optimize operational responsiveness

Optimize the critical path: identify object, load editable fields, validate, submit, and confirm server state. Use selective `$select` and deliberate expansions rather than retrieving every attachment and history record before the user can edit a description. Lazy-load secondary sections while preserving an understandable loading state.

The official [performance guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/performance-speed-up-your-app-408b40e.md) emphasizes that network and backend configuration can dominate apparent frontend slowness. Measure service latency separately from rendering and module loading. A faster formatter does not repair a server query lacking a useful index.

Prevent repeated submit events through phase guards, not just visual disabling. Debounce remote suggestions, keep stale responses from overwriting current results, and avoid refreshing the master list on every keystroke in detail. A saved status change should update or refresh the affected list deliberately, preserving filters and explaining if the object no longer matches them.

## Pitfalls to reject in review

Reject “Save succeeded” based solely on a resolved batch promise. Reject Cancel implementations that reset every object in a shared model. Reject using an array index as entity identity, replacing decimals with binary floating point, or storing local timestamps where the service expects a date without time. Reject UI-only authorization and action handlers that change status codes directly without the business operation.

Also reject unexplained busy overlays, error messages that disappear automatically, and retry loops for non-idempotent requests. A recoverable operational application makes uncertainty visible. It does not pretend that every failure means nothing happened or that every successful HTTP envelope means the business process completed.

## Define the handover after completion

Completion should tell users what happened, which object changed, and what they can do next. A newly created document may have a backend-assigned number; show that number after creation rather than retaining a temporary client identifier as if it were permanent. For an asynchronous job, show its accepted status and tracking destination instead of a misleading final-success message.

Preserve enough context for the next task without preserving stale edit state. Returning to a filtered queue may legitimately remove the released order from the results. Explain that outcome and choose the next selection deliberately; do not leave an empty detail pane that looks like data loss. Where the process requires an audit trail, rely on backend business records rather than a transient client notification.

## Manual verification checklist

- Verify Save with valid data, invalid local input, server validation failure, and mixed batch outcomes.
- Confirm pending changes remain available after a rejected request and success is not reported prematurely.
- Test Cancel with both model-pending values and invalid control-only text.
- Edit the same object in two sessions and check the conflict explanation.
- Navigate from A to B while dirty, including browser history and direct hashes.
- Disconnect after submission and verify the unknown-outcome recovery path.
- Exercise phone master/detail behavior, keyboard-only operation, and message-to-field navigation.
- Check that busy guards prevent duplicate business actions and that authorization remains enforced by the service.
- Test bulk partial success, stale selection, polling cleanup, and return navigation.

## Cross-references and official sources

Use [transactional patterns](../03-UI-PATTERNS/transactional-patterns.md), [worklist pattern](../03-UI-PATTERNS/worklist-pattern.md), [object page pattern](../03-UI-PATTERNS/object-page-pattern.md), and [authorization](../05-SECURITY/authorization.md) for adjacent concerns. [Freestyle patterns](freestyle-patterns.md) covers request races and lifecycle ownership, while [custom controls](custom-controls.md) keeps widget behavior separate from business transactions.

The SDK source links above are version-tagged for API verification. SAP's conceptual guidance on [reacting to user input](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/reacting-on-user-input-events-c75861e.md) and [accessibility](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/accessibility-03b914b.md) supplies complementary interaction guidance. Consult the exact deployed patch documentation before relying on model-specific behavior.

Research accessed official GitHub documentation and version-tagged source rather than live SDK pages, whose host was unavailable. Consult the [V4 model SDK reference](https://ui5.sap.com/#/api/sap.ui.model.odata.v4.ODataModel) for the deployed version. Snippets received source-based manual review, not execution against a business service.
