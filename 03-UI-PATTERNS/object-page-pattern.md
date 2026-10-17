# Object page pattern

## Scope and prerequisites

An object page presents one business object, its identity, its current state, and the information and operations needed to understand or change it. A maintenance order, supplier invoice, purchase contract, or employee record can all fit this pattern. Its central responsibility is continuity: the user must always know which object is open, which changes are unsaved, and what an action will affect.

This guide primarily implements a **freestyle `sap.uxap.ObjectPageLayout`**. The layout, dynamic header, sections, subsections, and common form controls are available in OpenUI5. Declare `sap.uxap`, `sap.m`, and `sap.ui.layout` in the application manifest. Supply an MVC component, routing, an i18n model, and a service model. Examples assume an OData V4 service with an `Orders` entity set, a string `ID` key, and properties `Title`, `Description`, `Status`, `SupplierName`, `GrossAmount`, and `Currency`.

The implementation uses V4 API update groups. It does not use V2 `read`, `createEntry`, or `submitChanges`. A later metadata example is explicitly **Fiori elements for OData V4**, not configuration for the freestyle layout. Check every API against the runtime version shipped with the application; current repository samples can use features newer than a customer's maintained release.

The [repository README](../README.md) also lists the neighboring [worklist](./worklist-pattern.md), [list report](./list-report-pattern.md), [dashboard](./overview-dashboard.md), and [transactional patterns](./transactional-patterns.md) guides.

## Separate floorplan from implementation technology

“Object page” is a design pattern. `ObjectPageLayout` is a control that supplies its scrolling layout, header behavior, anchor navigation, and content structure. It does not interpret every OData UI annotation, generate business forms, implement authorization, or create a draft protocol. In freestyle development, application code composes controls and owns data access and transaction behavior.

Fiori elements Object Page templates are a different implementation choice. They consume supported metadata, annotations, manifest settings, and extension points to generate and coordinate the page. In SAPUI5, their framework can manage supported edit and draft flows for a compatible service. That framework behavior must not be attributed to a plain OpenUI5 layout. Fiori elements libraries and supported templates are not interchangeable with the OpenUI5 `sap.uxap` library.

Choose freestyle when highly specialized interactions genuinely require control over composition or when the application uses a nonstandard service contract. Choose Fiori elements when the supported service and annotations can express the business task and standard lifecycle behavior is valuable. Do not choose freestyle solely to change a header label; investigate supported metadata and extension options first. Conversely, avoid embedding an entire custom transaction inside an unsupported template-internal control tree.

## Organize sections around decisions

Sections are major business topics; subsections divide a topic into manageable units. “General,” “Items,” “Delivery,” and “History” are usually more useful than a section for each database table. A person deciding whether an order can be released should find identity, blocking conditions, and release-relevant information without visiting every section.

Keep the hierarchy shallow and use stable IDs for sections, subsections, and important fields. Stable IDs support navigation, adaptation where supported, and reliable automated checks. Do not derive IDs from translated titles. A label can change without changing the identity of a section or invalidating stored navigation state.

An `ObjectPageSubSection` contains content in its `blocks` aggregation and can expose secondary content through `moreBlocks`. Ordinary controls can be blocks; `BlockBase` is useful for reusable, separately developed blocks but is not mandatory for a small form. Hiding an important required field in “Show more” creates a confusing save failure. Place essential editable data in the primary content and reserve expansion for secondary detail.

## Build a freestyle page

The following view demonstrates two sections, a subsection action, explicit expanded and snapped headings, header summary content, and a transaction footer. `ui` is a named JSON model initialized by the controller. The default model is V4. Read-only display controls and editable controls use separate visibility expressions so that display mode remains a real reading experience rather than a collection of disabled inputs.

```xml
<mvc:View
  controllerName="demo.controller.Object"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns:uxap="sap.uxap"
  xmlns:form="sap.ui.layout.form"
  xmlns="sap.m"
  height="100%">
  <uxap:ObjectPageLayout id="objectPage"
    height="100%"
    showFooter="{ui>/editing}"
    upperCaseAnchorBar="false">
    <uxap:headerTitle>
      <uxap:ObjectPageDynamicHeaderTitle>
        <uxap:expandedHeading>
          <Title text="{Title}" level="H1" wrapping="true"/>
        </uxap:expandedHeading>
        <uxap:snappedHeading>
          <Title text="{Title}" level="H1" wrapping="true"/>
        </uxap:snappedHeading>
        <uxap:expandedContent>
          <Text text="{ID}"/>
        </uxap:expandedContent>
        <uxap:snappedContent>
          <ObjectStatus text="{Status}"/>
        </uxap:snappedContent>
        <uxap:actions>
          <Button text="Edit" type="Emphasized"
            visible="{= !${ui>/editing} }"
            enabled="{= !${ui>/busy} }" press=".onEdit"/>
        </uxap:actions>
      </uxap:ObjectPageDynamicHeaderTitle>
    </uxap:headerTitle>
    <uxap:headerContent>
      <VBox class="sapUiSmallMarginEnd">
        <Label text="Supplier"/>
        <Text text="{SupplierName}"/>
      </VBox>
      <VBox>
        <Label text="Status"/>
        <ObjectStatus text="{Status}"/>
      </VBox>
    </uxap:headerContent>
    <uxap:sections>
      <uxap:ObjectPageSection id="general" title="General"
        titleUppercase="false">
        <uxap:subSections>
          <uxap:ObjectPageSubSection id="details" title="Details">
            <uxap:blocks>
              <form:SimpleForm editable="{ui>/editing}"
                layout="ResponsiveGridLayout">
                <form:content>
                  <Label text="Description" labelFor="description"/>
                  <Input id="description" value="{Description}"
                    visible="{ui>/editing}"
                    editable="{= !${ui>/busy} }"/>
                  <Text text="{Description}"
                    visible="{= !${ui>/editing} }"/>
                  <Label text="Supplier"/>
                  <Text text="{SupplierName}"/>
                </form:content>
              </form:SimpleForm>
            </uxap:blocks>
          </uxap:ObjectPageSubSection>
        </uxap:subSections>
      </uxap:ObjectPageSection>
      <uxap:ObjectPageSection id="history" title="History"
        titleUppercase="false">
        <uxap:subSections>
          <uxap:ObjectPageSubSection title="Business events">
            <uxap:actions>
              <Button text="Refresh history" press=".onRefreshHistory"/>
            </uxap:actions>
            <uxap:blocks>
              <List id="events" items="{Events}">
                <StandardListItem title="{EventText}"
                  description="{CreatedAt}"/>
              </List>
            </uxap:blocks>
          </uxap:ObjectPageSubSection>
        </uxap:subSections>
      </uxap:ObjectPageSection>
    </uxap:sections>
    <uxap:footer>
      <OverflowToolbar>
        <ToolbarSpacer/>
        <Button text="Save" type="Emphasized"
          enabled="{= !${ui>/busy} }" press=".onSave"/>
        <Button text="Cancel"
          enabled="{= !${ui>/busy} }" press=".onCancel"/>
      </OverflowToolbar>
    </uxap:footer>
  </uxap:ObjectPageLayout>
</mvc:View>
```

Assume the component hierarchy provides a meaningful full-height container. Avoid placing the layout inside an independently scrolling `sap.m.Page`; competing scroll containers can undermine snapping, anchor offsets, and keyboard focus behavior. If a shell page is required, configure its scrolling ownership deliberately. Format `CreatedAt` using the service's actual OData type and desired timezone rather than relying on the raw string shown in this structural example.

## Design expanded and sticky header behavior

The expanded header introduces identity and a concise summary. The snapped header preserves orientation after scrolling. It should retain the object title or identifier and a useful state, not become a second unrelated toolbar. Put key actions in the dynamic header's public `actions` aggregation so that the framework can handle layout and overflow.

Sticky behavior belongs to the layout and dynamic header mechanism. Do not add `position: fixed` to generated header elements or copy DOM nodes into a separate floating toolbar. Such customizations commonly overlap anchor navigation, break on zoom, and depend on private rendering details. Configure supported header properties for the runtime and verify their behavior with real section heights.

Not all expanded content remains visible in the snapped state. If a blocked status is safety-critical, include it in the compact representation or in a persistent message area rather than relying on a large header facet. On small screens, prioritize identity and the primary action; secondary actions can overflow, but their labels must remain understandable.

Preserving header expansion across object navigation is a user-experience decision, not business state. Some teams restore the prior preference, while others expand each newly opened object to establish context. Do not reset header state on every property update or save response. That creates unexpected scroll movement while the user is reading.

## Understand header facets

In freestyle discussions, “header facet” often means a compact summary block in `headerContent`: supplier, contact, amount, progress, or status. These are ordinary controls laid out by the application. There is no `UI.HeaderFacets` interpretation merely because such blocks are visually facet-like.

In **Fiori elements for OData V4**, `UI.HeaderFacets` is an annotation collection of `UI.ReferenceFacet` records. Each record points to a supported annotation, such as a qualified data point. The official guide explicitly describes this metadata contract and different desktop and mobile layouts [3]. A minimal service annotation looks like this:

```xml
<Annotations Target="Demo.Order"
  xmlns="http://docs.oasis-open.org/odata/ns/edm">
  <Annotation Term="com.sap.vocabularies.UI.v1.DataPoint"
    Qualifier="GrossAmount">
    <Record Type="com.sap.vocabularies.UI.v1.DataPointType">
      <PropertyValue Property="Title" String="Gross amount"/>
      <PropertyValue Property="Value" Path="GrossAmount"/>
    </Record>
  </Annotation>
  <Annotation Term="com.sap.vocabularies.UI.v1.HeaderFacets">
    <Collection>
      <Record Type="com.sap.vocabularies.UI.v1.ReferenceFacet">
        <PropertyValue Property="ID" String="GrossAmountFacet"/>
        <PropertyValue Property="Target"
          AnnotationPath="@com.sap.vocabularies.UI.v1.DataPoint#GrossAmount"/>
      </Record>
    </Collection>
  </Annotation>
</Annotations>
```

This fragment assumes a surrounding annotation document with references to the UI vocabulary and a real `Demo.Order` entity type. Add a currency annotation on the amount property in the service metadata. It does not create a section in a freestyle XML view. Body facets use `UI.Facets`, while header facets use `UI.HeaderFacets`; annotations, supported targets, and extension behavior must be selected for the specific Fiori elements generation and OData version.

## Bind the object independently of its source list

A deep link must work without a worklist binding context. Resolve a validated route key into an entity path and bind the view to that object. The following V4 controller excerpt assumes simple string keys. It escapes OData string literals and URL-encodes the result. Composite keys, GUID keys, and alternate keys require the service's corresponding key syntax and should use a centralized, verified key-path utility.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox",
  "sap/m/MessageToast"
], function (Controller, JSONModel, MessageBox, MessageToast) {
  "use strict";

  return Controller.extend("demo.controller.Object", {
    onInit: function () {
      this.getView().setModel(new JSONModel({
        editing: false, busy: false
      }), "ui");
    },

    bindOrder: async function (id) {
      const literal = encodeURIComponent(id.replace(/'/g, "''"));
      this.getView().bindElement({
        path: "/Orders('" + literal + "')",
        parameters: { $$updateGroupId: "objectEdit" }
      });
      await this.getView().getBindingContext().requestObject();
    },

    onEdit: function () {
      this.getView().getModel("ui").setProperty("/editing", true);
    },

    onSave: async function () {
      const view = this.getView();
      const ui = view.getModel("ui");
      if (ui.getProperty("/busy")) {
        return;
      }
      ui.setProperty("/busy", true);
      try {
        await view.getModel().submitBatch("objectEdit");
        if (view.getElementBinding().hasPendingChanges()) {
          MessageBox.error("Changes were not saved. Review the messages.");
          return;
        }
        ui.setProperty("/editing", false);
        MessageToast.show("Saved");
      } catch (error) {
        MessageBox.error("Saving failed. Your changes remain available.");
      } finally {
        ui.setProperty("/busy", false);
      }
    },

    onCancel: function () {
      this.getView().getElementBinding().resetChanges();
      this.getView().getModel("ui").setProperty("/editing", false);
    },

    onRefreshHistory: function () {
      this.byId("events").getBinding("items").refresh();
    }
  });
});
```

The route handler calls `bindOrder()` only after any unsaved-change decision, catches its rejection, and presents a recoverable not-found or access error. The excerpt does not implement route guards, full message presentation, or client validation; those are required integrations, not features supplied by the layout. Keep the `objectEdit` group dedicated to this page's transaction. Its default API submission mode queues updates until Save.

For a real edit screen, validate registered controls and inspect relevant message targets before submission. A parse error can leave invalid text in an input while the old valid value remains in the model. Pending-change checks alone cannot catch that condition. The save code demonstrates V4 submission semantics, not a replacement for the validation pipeline described in [transactional patterns](./transactional-patterns.md).

## Define edit state as a lifecycle

Separate business status from view mode. An order can be “Released” while the page is in display mode, or “Open” while changes are being saved. Useful UI phases include loading, display, editing, saving, conflict, and unavailable. A single `/editable` boolean becomes inadequate when navigation, actions, and request failures interact.

Entering edit mode does not automatically acquire a lock or create a server draft. In the simple non-draft example, it only changes control visibility. Two-way binding queues property updates in `objectEdit`. Leaving edit mode after successful save is a deliberate state transition; leaving after an individual failed PATCH would hide unresolved changes.

The V4 batch guide warns that `submitBatch()` rejects for failure of the whole batch request, not necessarily for each contained operation [4]. Check pending changes and the application's message model before reporting success. For actions, inspect their individual promises and business result contracts. An HTTP 200 envelope is not evidence that every requested update succeeded.

## Implement cancel and navigation guards

Cancel should discard only the current transaction. Binding-scoped `resetChanges()` is preferable when unrelated page bindings share a model. Model-wide reset can unexpectedly erase edits elsewhere. In this example, the dedicated group and page binding make the transaction boundary explicit.

Resetting pending property changes is not the same as undoing an already executed action. The official V4 documentation states that reset does not remove queued explicit operation invocations as if they were ordinary property edits [4]. Do not queue an irreversible release action in a normal edit group and assume Cancel can remove it. Invoke consequential actions through a deliberately controlled flow.

Before internal navigation, ask whether dirty work should be saved, discarded, or retained on the current page. Await save completion before navigating. Browser closing warnings are limited by browser behavior and cannot guarantee recovery. Do not promise that refresh restores a non-draft edit, and do not persist sensitive business payloads to browser storage without an explicit security and lifecycle design.

## Integrate draft without inventing it

Draft is a service-backed lifecycle with defined create, edit, activation, discard, ownership, and concurrency behavior. A V4 model, two-way binding, or JSON copy is not automatically a draft implementation. Draft requires a compatible backend protocol and application orchestration, or supported Fiori elements handling for that protocol.

When choosing draft, document which entity is active, which is the draft, how identifiers and navigation change, who owns the draft, and what happens when another user has unsaved work. Distinguish “saved to draft” from “activated as business data.” Make abandoned-draft cleanup and server timeouts part of the contract.

Draft does not imply offline capability. Losing connectivity can prevent saving even when fields remain visible. An offline application additionally needs durable local storage, synchronization, authorization expiry handling, conflict policy, and appropriate data protection. Do not describe queued in-memory changes as offline persistence.

## Place actions by scope

Header actions operate on the entire object: Edit, Release, Cancel Order, or Copy. Subsection actions operate on the displayed topic: add a delivery address, refresh history, or attach a document. Row actions operate on individual child objects. A transaction footer contains Save and Cancel for the current edit session.

Use one clear primary action at a time. In display mode that may be Edit or Release; in edit mode it is normally Save. Prevent repeated submissions while a command is pending and retain meaningful labels when actions move to overflow. An icon-only destructive action is difficult to understand and easy to trigger accidentally.

The backend must authorize every action and evaluate current state. A status returned at page load can be stale by the time Release is pressed. Use optimistic concurrency, normally through ETags where the service supports them, and provide conflict recovery rather than unconditional overwrite. For destructive operations, explain the affected object and consequences in a confirmation dialog.

## Refresh dependent state safely

A change can affect totals, status, messages, child collections, and action availability. Refresh only the affected binding paths or request service-defined side effects when supported. Broad refresh during editing can overwrite the user's working values or trigger unnecessary requests.

In freestyle, explicitly coordinate dependent reads with the completion of writes. Do not infer a new gross amount by summing formatted currency strings. Prefer authoritative backend values, especially when pricing, tax, or rounding rules are involved. Ensure a late response for object A cannot update the screen after navigation to object B.

Retain user orientation after a save. Avoid rebinding the entire page unless necessary, since that can reset header expansion, selected section, and child-table position. If an operation changes the object's key or creates a successor object, navigate deliberately and explain the new identity.

## Manage state and performance

Store selected section, header preference, and navigation origin separately from business data. Persist only what is useful and permitted. Do not serialize runtime contexts, control instances, or raw pending-change internals. Restored state should reference stable IDs or business keys.

Large sections should load according to task priority. Lazy content strategies can reduce startup cost, but deferred rendering does not necessarily eliminate backend requests if the bindings already exist. Inspect actual requests. Avoid expanding every child association in the root object request when most users only need General and Items.

Nested tables need a clear sizing strategy. A growing responsive table inside the object page usually fits mobile tasks better than an independently scrolling grid table. If a dense desktop table is essential, allocate a deliberate height and test scrolling and focus transitions. Do not assume the object page makes a desktop-only control responsive.

## Navigate to errors without losing orientation

An error overview should lead to the affected field, including fields in a subsection that is not currently visible. Resolve the message target to a stable control or binding path, reveal the containing section through the layout's public navigation API, and focus the field after its content is available. Avoid guessing generated DOM IDs or scrolling to a hard-coded pixel offset.

If a field lives in deferred content, coordinate content creation before focusing it. An attempted focus call against a control that has not been created cannot help the user correct the error. Keep the message selected in the overview until navigation succeeds, and retain an accessible summary when the field cannot be resolved.

Cross-field errors may belong to a section rather than a single input. For example, a delivery schedule total can disagree with the order quantity even though each individual number is valid. Explain the relationship and provide navigation to the relevant subsection. Do not attach every backend validation error arbitrarily to the first visible input.

After Save fails, leave the page in edit mode and preserve the user's location when possible. Automatically jumping to the top for every warning can be disruptive, especially when the user is correcting a lower section. Prefer a deliberate initial focus on the message summary or first blocking error, with subsequent navigation controlled by the user.

## Secure attachments and related content

Attachments, notes, and embedded links are part of the object's security boundary even when they appear in a secondary section. Authorize attachment upload and download independently on the server. Validate file size, media type, and content according to the backend policy; a filename extension and a frontend accept filter are not sufficient controls.

Do not render untrusted note content as arbitrary HTML. Prefer text controls or an explicitly sanitized, supported rich-content path. Links supplied by a service need an allowed-protocol policy and a clear external-navigation experience. Avoid exposing internal attachment storage URLs when the service offers an authorized download endpoint instead.

Define how attachments participate in the edit lifecycle. An upload may be persisted before the main form is saved, or it may belong to a draft or temporary upload session. Cancel cannot truthfully promise to undo an already permanent upload unless the backend supplies cleanup behavior. The UI should communicate this boundary and recover from orphaned temporary content safely.

## Design child-object editing boundaries

An item table on an order page does not automatically belong to the same edit transaction as the order header. Establish whether adding an item, editing its quantity, and saving the header must be atomic. If they must, the service must provide a compatible transaction contract, and all relevant bindings must use the intended update group and changeset behavior.

If child editing opens a separate object page or dialog, state whether its Save persists independently. A parent Cancel that leaves independently saved child changes in place is legitimate only when that boundary is understandable. Avoid using identical Save and Cancel labels for different scopes without contextual headings.

New child rows also need lifecycle ownership. A transient row can remain after a failed create, while an already persisted row requires a real delete operation to remove it. Distinguish correction, local cancellation, and deletion in both code and messaging. Preserve the parent key association using the supported service relationship rather than copying a display identifier into an arbitrary payload field.

Finally, prevent unrelated background refreshes from replacing active child edits. If the server changes a total or availability value while the user edits an item, coordinate side-effect reads with pending updates and present any resulting conflict. A consistent-looking total is not worth silently discarding the user's intended quantity.

## Verify the lifecycle with representative scenarios

Review a complete user journey, not just the first render. Open a deep link in a fresh browser session, edit a field, trigger a business validation error, correct it, save, and return to the source list. Repeat with a concurrent server change and a network interruption after submission.

Also inspect a long object with many child records, a restricted user, and a translated title that wraps in the snapped header. These scenarios reveal different failure classes: binding ownership, authorization assumptions, pending-change handling, sizing, and focus management. Record the intended result of each scenario before implementation so that a visually correct header does not conceal an incomplete transaction design.

## Verification checklist

- [ ] Deep links bind the object without a previously loaded source list.
- [ ] Expanded and snapped headers preserve identity and important status.
- [ ] Sections and subsections have stable IDs and task-oriented titles.
- [ ] Required editable information is not hidden behind secondary expansion.
- [ ] Display mode reads cleanly without misleading disabled inputs.
- [ ] Validation blocks saving invalid text that never reached the model.
- [ ] Save checks individual outcomes, messages, and pending changes.
- [ ] Cancel resets only the intended transaction and cannot undo executed actions.
- [ ] Navigation guards await the user's save or discard decision.
- [ ] ETag conflicts preserve recoverable work and explain the next step.
- [ ] Draft, if present, is a verified backend lifecycle rather than a UI flag.
- [ ] Keyboard focus, zoom, phone layout, and overflow actions remain usable.
- [ ] Dependent totals and status refresh without resetting page orientation.
- [ ] Backend authorization is enforced independently of action visibility.

## Official sources

1. [OpenUI5 ObjectPage XML sample](https://github.com/SAP/openui5/blob/master/src/sap.uxap/test/sap/uxap/demokit/sample/ObjectPageXML/view/ObjectPageXML.view.xml): sections, subsections, blocks, more blocks, and subsection actions.
2. [OpenUI5 ObjectPage state sample](https://github.com/SAP/openui5/blob/master/src/sap.uxap/test/sap/uxap/demokit/sample/ObjectPageState/view/ObjectPageState.view.xml): dynamic header expanded/snapped aggregations, header content, and section navigation.
3. [SAPUI5 Fiori elements V4 header facets](https://github.com/SAP-docs/sapui5/blob/main/docs/06_SAP_Fiori_Elements/header-facets-17dbd5b.md): the `UI.HeaderFacets` metadata contract and its responsive presentation.
4. [SAPUI5 OData V4 batch control](https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/batch-control-74142a3.md): API groups, reset boundaries, pending changes, and batch failure semantics.

Examples are original compositions informed by these official sources, not a claim that the samples implement the complete business lifecycle described here.

SDK reference: [`sap.uxap.ObjectPageLayout`](https://ui5.sap.com/#/api/sap.uxap.ObjectPageLayout). Research used successfully retrieved official GitHub files; direct SDK access failed DNS resolution.
