# XML views: syntax, layout, responsiveness, and composition

## Scope and prerequisites

This chapter uses UI5 **1.120** public APIs and ordinary XML views. Select a maintained patch of that line or verify compatibility with your newer runtime. Everything demonstrated uses libraries available in **OpenUI5 and SAPUI5**: `sap.ui.core`, `sap.m`, and, in the optional form example, `sap.ui.layout`. SAPUI5-only controls such as `sap.ui.comp` smart controls are not prerequisites. A freestyle XML view is not a Fiori elements page definition, even when both eventually render familiar controls.

The namespace is `com.acme.orders`, mapped to `webapp`. The complete local view below pairs with `webapp/controller/Orders.controller.js` from [controllers.md](./controllers.md#complete-local-crud-controller). That controller creates named JSON models `orders` and `view`, so no manifest, resource bundle, router, or backend is hidden behind the first example. Local messages are intentionally English. The localization section explains the explicit additional model needed to replace them.

Orders have a string `OrderID`, customer text, decimal-string `Amount`, currency code, status code, and ISO calendar-date `OrderDate`. The first view displays only fields it can display without custom formatter dependencies. In particular it does not convert financial strings into binary floating-point numbers or parse business keys as integers.

Related references are [models.md](./models.md), [services.md](./services.md), and [utilities.md](./utilities.md). The directory and file names match the repository [README](../README.md#repository-structure). All paths in snippets refer to files you create in the consuming application; this documentation does not install a sample application automatically.

## Minimal hosting integration

For a standalone local demonstration, serve `webapp` over HTTP with UI5 resources available at `resources/`. That resource location must be mapped by your existing UI5 development server or hosting environment. Do not open the HTML directly through `file://`. Put the following content in the consuming application's `webapp/index.html`. There are no external script dependencies beyond the selected UI5 runtime.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Local orders</title>
  <script
    id="sap-ui-bootstrap"
    src="resources/sap-ui-core.js"
    data-sap-ui-theme="sap_horizon"
    data-sap-ui-libs="sap.m,sap.ui.core,sap.ui.layout"
    data-sap-ui-async="true"
    data-sap-ui-compat-version="edge"
    data-sap-ui-resource-roots='{"com.acme.orders":"./"}'
    data-sap-ui-on-init="module:com/acme/orders/bootstrap">
  </script>
</head>
<body class="sapUiBody">
  <div id="content"></div>
</body>
</html>
```

Create `webapp/bootstrap.js` with this module. `XMLView.create` is asynchronous; rendering must wait for its promise. The catch handles initialization failure with a fixed message rather than writing exception contents into the document.

```javascript
sap.ui.define([
  "sap/ui/core/mvc/XMLView",
  "sap/m/MessageBox"
], function (XMLView, MessageBox) {
  "use strict";
  XMLView.create({
    id: "ordersView",
    viewName: "com.acme.orders.view.Orders"
  }).then(function (view) {
    view.placeAt("content");
  }).catch(function () {
    MessageBox.error("The application could not be started.");
  });
});
```

A production component application should let its component and manifest own root-view creation rather than running this bootstrap alongside a ComponentContainer. Creating the same root view twice causes duplicate IDs, duplicate models, and confusing event behavior. This standalone host exists to make the local example's integration boundary explicit, not to recommend bypassing component architecture for a deployed business application.

## Complete local orders view

Place this at `webapp/view/Orders.view.xml`. Use the complete local CRUD controller from [controllers.md](./controllers.md#complete-local-crud-controller), not the remote alternative. Each handler referenced here is implemented there. `CustomerName` is an editable draft, not a direct binding to the selected entity. Selection is optional because an empty selection means the next Save creates a new order.

```xml
<mvc:View
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  controllerName="com.acme.orders.controller.Orders"
  displayBlock="true">
  <App>
    <Page title="Orders">
      <content>
        <VBox class="sapUiResponsiveContentPadding">
          <Title id="ordersHeading" text="Available orders" level="H2"/>
          <Table
            id="ordersTable"
            ariaLabelledBy="ordersHeading"
            mode="SingleSelectMaster"
            selectionChange=".onSelect"
            growing="true"
            growingThreshold="20"
            items="{orders>/OrderSet}">
            <columns>
              <Column importance="High">
                <Text text="Order"/>
              </Column>
              <Column
                minScreenWidth="Tablet"
                demandPopin="true"
                importance="High">
                <Text text="Customer"/>
              </Column>
              <Column
                minScreenWidth="Desktop"
                demandPopin="true"
                importance="Low">
                <Text text="Status code"/>
              </Column>
            </columns>
            <items>
              <ColumnListItem>
                <cells>
                  <ObjectIdentifier title="{orders>OrderID}"/>
                  <Text text="{orders>CustomerName}"/>
                  <Text text="{orders>Status}"/>
                </cells>
              </ColumnListItem>
            </items>
          </Table>
          <Title text="Customer editor" level="H2"
            class="sapUiMediumMarginTop"/>
          <Label text="Customer name" labelFor="customerName"/>
          <Input
            id="customerName"
            value="{view>/draft/CustomerName}"
            maxLength="80"
            liveChange=".onDraftChange"/>
          <OverflowToolbar>
            <Button text="New" press=".onNew"/>
            <ToolbarSpacer/>
            <Button
              text="Delete selected"
              enabled="{= !!${view>/selectedId} }"
              press=".onDelete"/>
            <Button
              text="Save"
              type="Emphasized"
              enabled="{view>/canSave}"
              press=".onSave"/>
          </OverflowToolbar>
        </VBox>
      </content>
    </Page>
  </App>
</mvc:View>
```

The three table columns correspond to the three cells in exactly the same order. A responsive table does not match cells to columns by property name. Changing the order of one aggregation without the other produces misleading output, particularly when columns become pop-ins. Keep the identifier visible at narrow sizes so every row still has an intelligible identity.

The customer column becomes a pop-in below the tablet breakpoint, and status does so below the desktop breakpoint. `demandPopin` retains information instead of simply hiding it. Breakpoints are design choices, not a guarantee that content fits: test long customer names, translated headers, zoom, and increased font sizes. Avoid a fixed-width layout that merely happens to fit your laptop.

`growing` limits the initial number of rendered list items and adds more on demand. With the local JSON array it does not reduce the amount of data downloaded because all data is already in memory. With an OData list binding it can cooperate with server paging. Do not confuse it with a grid table's row virtualization or assume it makes an unbounded dataset inexpensive.

## XML names, aggregations, and escaping

The default namespace maps unprefixed elements to `sap.m`. The `mvc` prefix maps `mvc:View` to `sap.ui.core.mvc.View` handling. Add a prefix such as `xmlns:layout="sap.ui.layout"` when using controls from another library. A namespace declaration identifies classes; the application must still make the corresponding library available through its bootstrap or manifest dependencies.

Control properties appear as attributes, while child controls belong to aggregations. Many controls have a default aggregation, but explicit `<content>`, `<items>`, and `<columns>` wrappers are helpful when learning or reviewing a complex tree. A property expects a scalar or binding value; an aggregation expects controls or an aggregation binding. Putting a Button inside an attribute is neither XML syntax nor UI5 composition.

XML rules apply before UI5 evaluates a binding. Write `&amp;` for an ampersand and `&lt;` for a less-than character inside an attribute. For example, an expression containing logical AND needs `&amp;&amp;`. Quoting a JavaScript expression inside XML does not exempt it from XML entity rules. When an error points to XML parsing, check basic well-formedness before investigating model values.

A binding such as `{orders>CustomerName}` is relative. It inherits the row's context from the table's `items` binding. `{orders>/OrderSet}` is absolute within the named model. Leaving off `orders>` would select the default model instead, which is a common reason for a blank cell even though the expected object exists elsewhere in the model inspector.

Use expression binding for short presentation decisions such as whether a selection exists. Business validation, date conversion, currency rules, and network work do not belong in increasingly elaborate XML expressions. Long expressions are difficult to localize, review, and debug. Move reusable presentation transformations into a module, or expose a deliberate state property maintained by the controller.

## Responsive form alternative

This complete alternative view can replace `Orders.view.xml` while keeping the same local controller. It exposes create and draft-save behavior only; the list and selection actions are intentionally absent. If the controller's `onNew` is used here, first remove its table-selection line because this alternative has no `ordersTable`. The view therefore does not bind a New handler and does not introduce an undefined table dependency. The Save handler currently calls `onNew`, so retain the table in a combined view or use the tiny subclass below for this standalone alternative.

For the standalone form, create `webapp/controller/OrderForm.controller.js`:

```javascript
sap.ui.define([
  "com/acme/orders/controller/Orders.controller"
], function (OrdersController) {
  "use strict";
  return OrdersController.extend("com.acme.orders.controller.OrderForm", {
    onNew: function () {
      this._viewState.setProperty("/selectedId", "");
      this._viewState.setProperty("/draft", { CustomerName: "" });
      this._viewState.setProperty("/canSave", false);
    }
  });
});
```

The superclass is the complete local controller already provided. Its inherited initialization and cleanup still execute; the subclass replaces only the table-dependent reset method. In a real project prefer extracting shared editing behavior into a small module over creating many inheritance layers for minor layout differences.

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  xmlns:form="sap.ui.layout.form"
  controllerName="com.acme.orders.controller.OrderForm"
  displayBlock="true">
  <App>
    <Page title="Create order">
      <content>
        <form:SimpleForm
          editable="true"
          layout="ResponsiveGridLayout"
          labelSpanXL="3"
          labelSpanL="3"
          labelSpanM="4"
          labelSpanS="12"
          columnsXL="2"
          columnsL="1"
          columnsM="1">
          <form:content>
            <Label text="Customer name" labelFor="formCustomer"/>
            <Input id="formCustomer"
              value="{view>/draft/CustomerName}"
              maxLength="80"
              liveChange=".onDraftChange"/>
          </form:content>
        </form:SimpleForm>
      </content>
      <footer>
        <OverflowToolbar>
          <Button text="Clear" press=".onNew"/>
          <ToolbarSpacer/>
          <Button text="Save" type="Emphasized"
            enabled="{view>/canSave}" press=".onSave"/>
        </OverflowToolbar>
      </footer>
    </Page>
  </App>
</mvc:View>
```

A SimpleForm interprets labels and following fields as form content. It is not a general-purpose arbitrary grid. Use form groups for related fields, and avoid inserting unrelated tables between a label and its input. At phone sizes labels should remain associated with controls rather than becoming visually detached captions.

Responsive spacing classes express intent without manipulating generated DOM. They complement, rather than replace, responsive control behavior. A VBox is useful for vertical composition; an HBox with inflexible children can still overflow. Prefer wrapping, flexible widths, and OverflowToolbar for actions that cannot all remain visible on small screens.

## Fragment composition with explicit ownership

A fragment is a reusable control-tree definition, not a view with its own controller. This separate runnable composition demonstration uses `view/Popup.view.xml`, `controller/Popup.controller.js`, and `view/Help.fragment.xml`. To host it, change the bootstrap's `viewName` to `com.acme.orders.view.Popup`. It requires only `sap.m` and core.

```xml
<!-- webapp/view/Popup.view.xml -->
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  controllerName="com.acme.orders.controller.Popup">
  <Page title="Order help">
    <content>
      <Button text="Show help" press=".onOpenHelp"/>
    </content>
  </Page>
</mvc:View>
```

```xml
<!-- webapp/view/Help.fragment.xml -->
<core:FragmentDefinition xmlns:core="sap.ui.core" xmlns="sap.m">
  <Dialog id="helpDialog" title="Order identifiers">
    <content>
      <Text class="sapUiSmallMargin"
        text="Order identifiers are text. Keep leading zeros."/>
    </content>
    <endButton>
      <Button text="Close" press=".onCloseHelp"/>
    </endButton>
  </Dialog>
</core:FragmentDefinition>
```

```javascript
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/core/Fragment",
  "sap/m/MessageBox"
], function (Controller, Fragment, MessageBox) {
  "use strict";
  return Controller.extend("com.acme.orders.controller.Popup", {
    onInit: function () {
      this._alive = true;
      this._dialogPromise = null;
      this._dialog = null;
    },
    onOpenHelp: function () {
      if (!this._dialogPromise) {
        this._dialogPromise = Fragment.load({
          id: this.getView().getId(),
          name: "com.acme.orders.view.Help",
          controller: this
        }).then(function (dialog) {
          if (!this._alive) {
            dialog.destroy();
            return null;
          }
          this.getView().addDependent(dialog);
          this._dialog = dialog;
          return dialog;
        }.bind(this)).catch(function () {
          this._dialogPromise = null;
          if (this._alive) {
            MessageBox.error("Help could not be opened.");
          }
          return null;
        }.bind(this));
      }
      return this._dialogPromise.then(function (dialog) {
        if (dialog && this._alive) { dialog.open(); }
      }.bind(this));
    },
    onCloseHelp: function () {
      if (this._dialog) { this._dialog.close(); }
    },
    onExit: function () {
      this._alive = false;
      this._dialog = null;
    }
  });
});
```

The cached promise prevents rapid clicks from constructing duplicate dialogs with the same ID. Adding the loaded dialog as a view dependent propagates models and ensures destruction with the view. The late-resolution branch destroys a dialog created after exit, because it never became a dependent of a living view. Clearing the cached promise on failure allows a deliberate later retry.

Passing the view ID as the fragment prefix makes `this.byId("helpDialog")` resolve correctly. Without an explicit prefix, repeated fragment instances can collide. For multiple simultaneous instances, choose distinct prefixes and use `Fragment.byId` with the matching prefix. A fragment's local ID is not automatically a global application ID.

## Typed values, accessibility, and localization

Prefer a binding type for editable values because it supplies parsing and validation as well as formatting. A formatter returns display text and is usually one-way; it does not tell UI5 how to turn localized input back into a decimal string. [utilities.md](./utilities.md) provides financial and calendar-date examples with explicit representation boundaries.

Bind key fields to text-capable controls. An order identifier is not a numeric quantity, even if its characters are digits. Similarly, display status codes through translated texts and explicit semantic states; color alone is not an accessible explanation. Unknown codes need a safe fallback rather than a blank label or an invented success state.

Labels should target their input through `labelFor`. Table headings need a meaningful title or `ariaLabelledBy`. Icons used without text need an accessible name, and validation needs visible feedback associated with the field. A disabled Save button is not enough to explain what the user must correct.

To localize this view, install the `i18n` ResourceModel from [models.md](./models.md#resource-model-and-supported-locales), then replace literals with bindings such as `{i18n>ordersTitle}`. Supply every referenced key in the base bundle. Declare supported locales honestly: listing `de` without providing the German file does not translate the interface. English fallback must be configured consistently with the actual files.

Dates should follow the user's locale for display, with the business meaning of a calendar date kept separate from an instant. Test near midnight and in negative and positive time zones. A fixed `MM/dd/yyyy` string is not a locale-aware solution, and converting every date through `new Date(isoDate)` can shift the visible calendar day.

## Binding state, empty results, and interaction feedback

A view needs to distinguish initial loading, a legitimate empty result, and a failed request. Showing “No orders” immediately while a slow first request is pending falsely suggests a successful empty response. Bind busy state to a view model and choose empty-state text that reflects whether loading has completed. On failure, preserve an intelligible error state rather than presenting an empty table as if it were authoritative.

For a filtered collection, explain that an empty result belongs to the current filter. Keep the query visible and provide a clear way to remove it. If the service returns only the first fifty matches, do not label the resulting array length as the total count. An OData count, when supported and requested, has different semantics from the number of currently loaded contexts.

Bindings can temporarily evaluate with undefined values while metadata, contexts, or resources are loading. A formatter should tolerate that state without turning it into `"undefined"` text. Avoid truthiness checks that accidentally hide valid zero values. For example, a numeric zero quantity and an empty string are not interchangeable, while an empty selected-ID string can deliberately represent no selection.

Control visibility and enablement are interaction conveniences, not security boundaries. A hidden approval button does not prevent an unauthorized caller from invoking the endpoint. The backend still validates permissions and business transitions. Bind actions to known capabilities or application state, but never describe those bindings as access-control enforcement.

## Composition boundaries and stable identifiers

Choose composition by lifecycle, not by arbitrary file length. A fragment is useful when a control tree belongs to an existing view and controller. A child XML view can have its own controller and lifecycle when it represents a more independent region. A component is appropriate for a genuinely reusable application unit with its own models, routing boundary, or manifest configuration. These are different ownership structures, not interchangeable ways to split XML.

Stable local IDs are valuable for accessibility associations, adaptation, and integration checks. Avoid manually constructing global IDs from guessed separators; use view and fragment APIs. Do not put an entity key directly into a control ID merely to recover it later. The binding context already supplies entity identity without conflating business data with control-tree identity.

When reusing a fragment twice, check both ID uniqueness and model context. A correctly prefixed control can still show the wrong order if it inherits an unintended context. Set or bind the intended context explicitly at the composition boundary. Conversely, avoid assigning a shared model separately to every leaf control when model propagation from the owning view already supplies it.

Keep layout changes separate from business behavior where possible. Moving the customer editor from a VBox to a form should not change how order keys are generated or how Save handles a conflict. The complete form alternative above makes its one table-dependent reset difference explicit; larger applications should extract that editing state so layout-specific controls do not leak into shared business helpers.

## Pitfalls and manual verification

- Open the local demo and verify every referenced handler exists and fires once.
- Resize through phone, tablet, and desktop widths; verify pop-ins retain labels and values.
- Check that column order still matches cell order after any modification.
- Use keyboard selection, tab navigation, and browser zoom, not only mouse clicks.
- Enter text containing ampersands and angle brackets; ordinary Text controls must render text, not markup.
- Try long translations and right-to-left layout with a supported locale.
- Create two view instances; local IDs must receive distinct view prefixes.
- Open help repeatedly, then navigate away while the fragment is loading.
- Simulate a missing fragment and confirm a safe message and possible later retry.
- Confirm dates, decimals, and identifiers retain their intended representations.

These checks describe manual acceptance work in a consuming application. They do not assert that rendering or accessibility audits have already been performed against a deployed UI.

## Official API references

Research access: the SDK host could not be resolved during research. API behavior was checked through SAP's official OpenUI5 GitHub sources instead. The [XMLView SDK reference](https://ui5.sap.com/#/api/sap.ui.core.mvc.XMLView) is a navigation link, not a claim of successful SDK access; select your deployed version when consulting it.

- [XMLView creation and XML processing](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/mvc/XMLView.js).
- [sap.m.Table column/cell ordering and responsive behavior](https://github.com/SAP/openui5/blob/1.120.0/src/sap.m/src/sap/m/Table.js).
- [Fragment loading, IDs, and controller association](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/Fragment.js).
- [ResourceModel bundle configuration](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/resource/ResourceModel.js).
- [DateFormat locale and timezone semantics](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/format/DateFormat.js).
