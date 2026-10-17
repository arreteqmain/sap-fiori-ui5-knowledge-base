# Custom UI5 Controls

## Scope and prerequisites

A custom control is a reusable UI element with an explicit public API, framework-managed lifecycle, rendering contract, and accessibility behavior. It is not a convenient location for arbitrary application logic. This chapter develops a small composite `ActionPanel` that owns a standard button, accepts additional content, exposes a press event, and references external accessible text. The example is intentionally modest: a small control makes ownership and rendering mistakes easier to see than a large business widget.

Examples target **UI5 1.120** with asynchronous modules, JavaScript, XML views, and the `sap.ui.core` and `sap.m` libraries. The module lives at `webapp/control/ActionPanel.js` in an application named `demo.workorders`. The application namespace must already resolve to `webapp`. No custom library registration is required for this application-local module. A distributable library requires its own library metadata, resource configuration, documentation, and compatibility policy.

Prerequisites are [freestyle development](freestyle-development.md), JavaScript prototypes, binding basics, and familiarity with public control APIs. SAP's official [control metadata guide](https://github.com/SAP-docs/sapui5/blob/main/docs/09_Developing_Controls/defining-the-control-metadata-7b52540.md), [composite control guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/09_Developing_Controls/building-standard-composite-controls-c1512f6.md), and [ManagedObject API comments](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/base/ManagedObject.js) define the mechanisms used below.

## Choose the smallest reuse mechanism

First ask whether a standard control already provides the required behavior. A button with a different label does not justify a subclass. A reusable search dialog tied to one application can often be a fragment plus controller coordination. A whole navigation screen belongs in a view or component, not in a single giant custom control.

| Requirement | Preferred starting point | Why |
| --- | --- | --- |
| Repeated markup with application-specific handlers | XML fragment | Composition without inventing a public widget API |
| Reusable independent behavior and properties | Composite control | Encapsulates state and delegates standard interactions |
| Separate application feature with models and routing | Component | Provides ownership and feature boundaries |
| A genuinely new visual interaction | Custom rendering control | Full rendering and accessibility responsibility |
| Minor styling of an existing control | Public properties and application style class | Avoids dependency on inherited renderer details |

Composition normally survives framework upgrades better than inheriting from a complex standard control and overriding its renderer. Standard controls can depend on internal structure that is not an extension contract. Extending `Control` and owning standard children makes that dependency explicit through public APIs.

Write three consumer examples before implementing the control. If every consumer needs different private hooks, the proposed abstraction is unstable. If the control requires a particular controller instance, backend service URL, or global model name, it is probably an application feature disguised as a general widget.

## Design metadata as the public contract

Properties describe values such as a title or enabled state. Use primitive or registered framework types and meaningful defaults. A property name should describe the consumer-visible concept, not a DOM implementation detail. Generated getters and setters participate in binding and invalidation; bypassing them by writing arbitrary instance fields loses that behavior.

Events describe user intent or a documented state transition. Include only the payload consumers need. A press event should not expose the private button instance or its DOM event as a required integration point. If the event means “request approval,” make clear whether the control has already changed its own state; ambiguous before/after semantics cause duplicate operations.

Aggregations express ownership. A parent aggregation contains child elements, propagates models and binding contexts, participates in invalidation, and normally destroys its children with the parent. A multiple aggregation has a plural name and singular mutator name. Hidden aggregations still establish ownership but do not expose generated public accessors.

Associations express references, usually as IDs, rather than ownership. An associated label can outlive the referring control and is not destroyed when that control is destroyed. Associations do not automatically render referenced objects or supply accessibility semantics. The renderer or a delegated standard control must actually use the reference correctly.

Avoid an `options` property containing an undocumented object of arbitrary settings. It prevents tooling, binding validation, and API discoverability from helping consumers. If an object-valued property is necessary, document its schema and replacement semantics; mutating a nested object in place does not automatically notify the framework.

## Implement a composite control with API version 2

The following complete module creates its private button once in `init`. The renderer outputs one root element, an optional title, the private button, and public content children. Public setters propagate relevant properties to the button while retaining the wrapper's own state and normal invalidation. Associated descriptions are applied to the group, not copied into the button's accessible name.

```javascript
sap.ui.define([
  "sap/ui/core/Control",
  "sap/m/Button"
], function (Control, Button) {
  "use strict";

  return Control.extend("demo.workorders.control.ActionPanel", {
    metadata: {
      properties: {
        title: { type: "string", defaultValue: "" },
        actionText: { type: "string", defaultValue: "" },
        enabled: { type: "boolean", defaultValue: true }
      },
      defaultAggregation: "content",
      aggregations: {
        content: {
          type: "sap.ui.core.Control",
          multiple: true,
          singularName: "content"
        },
        _action: {
          type: "sap.m.Button",
          multiple: false,
          visibility: "hidden"
        }
      },
      associations: {
        ariaDescribedBy: {
          type: "sap.ui.core.Control",
          multiple: true,
          singularName: "ariaDescribedBy"
        }
      },
      events: {
        press: {
          parameters: {
            actionText: { type: "string" }
          }
        }
      }
    },

    init: function () {
      if (Control.prototype.init) {
        Control.prototype.init.apply(this, arguments);
      }
      this.setAggregation("_action", new Button({
        press: function () {
          if (this.getEnabled()) {
            this.firePress({ actionText: this.getActionText() });
          }
        }.bind(this)
      }));
    },

    setActionText: function (sText) {
      this.setProperty("actionText", sText);
      this.getAggregation("_action").setText(this.getActionText());
      return this;
    },

    setEnabled: function (bEnabled) {
      this.setProperty("enabled", bEnabled);
      this.getAggregation("_action").setEnabled(this.getEnabled());
      return this;
    },

    renderer: {
      apiVersion: 2,
      render: function (oRm, oPanel) {
        const sTitle = oPanel.getTitle();
        const aDescriptions = oPanel.getAriaDescribedBy();
        const mA11y = { role: "group" };
        if (sTitle) {
          mA11y.labelledby = oPanel.getId() + "-title";
        }
        if (aDescriptions.length) {
          mA11y.describedby = aDescriptions.join(" ");
        }
        oRm.openStart("div", oPanel);
        oRm.class("demoActionPanel");
        oRm.accessibilityState(oPanel, mA11y);
        oRm.openEnd();
        if (sTitle) {
          oRm.openStart("div", oPanel.getId() + "-title");
          oRm.class("demoActionPanelTitle");
          oRm.openEnd();
          oRm.text(sTitle);
          oRm.close("div");
        }
        oRm.renderControl(oPanel.getAggregation("_action"));
        oPanel.getContent().forEach(function (oChild) {
          oRm.renderControl(oChild);
        });
        oRm.close("div");
      }
    }
  });
});
```

The wrapper's `enabled` property controls the action only. It does not recursively disable arbitrary content children; document this distinction in the API. If consumers need a fully disabled subtree, design that behavior explicitly rather than traversing children and guessing which ones have an `enabled` property. The title labels a semantic group; it is not automatically a document heading. Use a suitable standard title control if page heading hierarchy requires one.

This example has no external subscriptions, timers, or unmanaged objects, so it needs no custom `exit`. The framework destroys the private button and public content through their aggregations. Adding an `exit` that separately destroys the same children is unnecessary and can mask ownership mistakes. A future superclass lifecycle implementation is respected by the guarded `init` call.

## Understand the renderer contract

`apiVersion: 2` declares a renderer contract; it is not merely a performance switch. Use the semantic RenderManager APIs consistently and maintain one valid root node associated with the control. The official [renderer object documentation](https://github.com/SAP-docs/sapui5/blob/main/docs/09_Developing_Controls/renderer-object-c9ab345.md) explains this contract, while the versioned [RenderManager implementation and API comments](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/RenderManager.js) define individual methods.

`openStart("div", oControl)` writes the control identity and framework data. Passing only a handcrafted ID would omit part of that framework integration. Child IDs such as `getId() + "-title"` must be unique within the control and must not collide with internal child-control IDs. Do not invent identifiers based on translated text or array positions.

`text` escapes text content and `attr` escapes attribute values. `class` and `style` express classes and styles through the renderer API. These methods do not validate business meaning or make arbitrary URLs safe. A user-supplied URL still needs an allowed scheme and destination policy. Avoid raw HTML insertion, legacy string-building renderer methods, and `unsafeHtml` for ordinary application values.

Rendering must be a deterministic description of current control state. Do not create children, attach handlers, change models, or fire business events while rendering. The framework may render repeatedly, and such side effects multiply or create invalidation loops. The shared renderer object must not store per-instance mutable state; state belongs to the control instance.

Never patch a standard child's private DOM to implement a property. The next rendering can erase the change, while accessibility and framework state remain wrong. Use the child's public setter. For this panel, changing action text updates `sap.m.Button`; it does not search for an internal button span.

## Consume the control in XML

This view assumes the application already supplies an `i18n` model, a `ui` JSON model containing `canRelease`, and a business context with `Status`. The controller's `onReleaseRequested` coordinates the backend action. The control only reports intent.

```xml
<mvc:View
  controllerName="demo.workorders.controller.Detail"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  xmlns:core="sap.ui.core"
  xmlns:local="demo.workorders.control">
  <VBox>
    <core:InvisibleText
      id="releaseHelp"
      text="{i18n>releaseConsequences}" />
    <local:ActionPanel
      id="releasePanel"
      title="{i18n>releaseSection}"
      actionText="{i18n>release}"
      enabled="{ui>/canRelease}"
      ariaDescribedBy="releaseHelp"
      press=".onReleaseRequested">
      <local:content>
        <ObjectStatus text="{Status}" />
        <Text text="{i18n>reviewBeforeRelease}" />
      </local:content>
    </local:ActionPanel>
  </VBox>
</mvc:View>
```

The XML view resolves the local association ID. The `InvisibleText` must exist in the rendered accessibility tree to be referenced; merely knowing its ID is insufficient. Destroying the panel must not destroy `releaseHelp`, because the association is not ownership. Destroying the surrounding `VBox` destroys both through their actual aggregations.

All visible text comes from bindings or consumers. The control contains no hardcoded English action label and no knowledge of work orders. If it becomes a reusable library control with intrinsic text, provide a library resource bundle and document fallback behavior. Empty action text is technically supported here but should fail the application's accessibility acceptance check.

## Keep aggregation ownership unambiguous

A UI5 element has one parent. Inserting the same content control into two panels does not make two visual copies; it reparents that control. Create distinct instances or bind an aggregation template where repeated content is required. Parent identity matters for model propagation, invalidation, event bubbling, and destruction.

`removeContent` removes a child from the aggregation but does not imply destruction. The caller must decide whether to reuse or destroy the returned child. `destroyContent` destroys the aggregated children. Code that repeatedly removes content and drops its references can leak controls even though nothing remains visible.

Private child controls belong in hidden aggregations, not only in fields such as `this._button`. An instance field is useful as a convenience reference but does not establish framework ownership. Similarly, a popup should be placed in an appropriate dependent aggregation when it needs the view's lifecycle and models but is not rendered inline.

Forwarding aggregations can be appropriate for a mature composite control, but it changes the ownership and API design discussion. Use the documented forwarding mechanism rather than overriding every generated aggregation mutator ad hoc. Review binding, cloning, and removal behavior before publishing that API.

## Treat lifecycle hooks as resource boundaries

`init` runs during construction; settings are applied afterward, so getters can still hold defaults there. This is why the panel creates children in `init` and propagates consumer values through setters. Never assume a constructor setting has already been applied when creating a child in `init`.

`onBeforeRendering` is useful for tearing down DOM-dependent registrations from a previous rendering. `onAfterRendering` can establish a registration that genuinely requires the rendered element. Neither should be needed simply to style a standard button or attach its public press event. Recreating children after every rendering loses focus, bindings, and state.

`exit` releases resources not already owned through aggregations. A control using `sap/ui/core/ResizeHandler` should register only when it needs measurement and deregister its registration before another registration or destruction. The following lifecycle excerpt illustrates resource cleanup for a separate measuring control; it is not required by `ActionPanel`.

```javascript
// ResizeHandler is imported from "sap/ui/core/ResizeHandler".
onBeforeRendering: function () {
  if (this._resizeId) {
    ResizeHandler.deregister(this._resizeId);
    this._resizeId = null;
  }
},

onAfterRendering: function () {
  this._resizeId = ResizeHandler.register(
    this, this._onResize.bind(this)
  );
},

exit: function () {
  if (this._resizeId) {
    ResizeHandler.deregister(this._resizeId);
    this._resizeId = null;
  }
  if (Control.prototype.exit) {
    Control.prototype.exit.apply(this, arguments);
  }
}
```

For that measuring control, `_onResize` must avoid causing an endless measure/invalidate loop; compare meaningful size changes and use a design that converges. In a subclass of another control, preserve that superclass's lifecycle hooks as documented. SAP's [exit method guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/09_Developing_Controls/exit-method-d4ac0ed.md) explains why replacing inherited cleanup without calling it breaks lifecycle guarantees.

External event sources require explicit detachment with the original handler identity and listener context. Save bound functions when attaching to an event bus or browser event; calling `.bind(this)` again during detach creates a different function. Clear timers and invalidate pending asynchronous completion tokens so a destroyed control cannot later update itself.

## Build accessibility into the API

An accessible custom control requires an understandable name, role, state, keyboard operation, focus behavior, and visual adaptation. Adding `role="button"` to a `div` does not implement a button. Prefer a standard `sap.m.Button` for activation; it already implements keyboard and disabled behavior, as the example does.

Do not make the outer panel focusable merely because it contains an action. That creates an extra tab stop without an additional operation. Let focus move to the standard button and any interactive content. If the control removes or replaces the focused child, define where focus goes afterward using public focus APIs.

Associations should express labeling or description relationships in terms consumers can use. The panel's description explains the group; the action's visible text remains its name. If every button requires a different warning, add a documented button-level description API rather than forcing consumers to discover the hidden aggregation.

Verify high-contrast themes, zoom, right-to-left layout, long translations, and reduced-motion expectations. Use theme parameters and logical spacing instead of fixed colors or left-only positioning. Color must not be the sole indicator of state. SAP's [accessibility aspects for controls](https://github.com/SAP-docs/sapui5/blob/main/docs/09_Developing_Controls/accessibility-aspects-694b356.md) links the relevant keyboard, screen-reader, and theme guidance.

## Keep business operations outside the widget

The control's press event is an intent signal. A controller or application service decides whether to release an order, which update group to use, and how to display backend messages. Calling OData directly inside the renderer or assuming a globally named service model couples a reusable control to one application and makes failure handling opaque.

This boundary is especially important for transactions. Disabling the button prevents duplicate local input; it does not guarantee backend idempotency or authorization. The application must distinguish UI state from business completion and use the correct V2 or V4 API. See [operational applications](operational-apps.md) for explicit save groups, conflicts, and uncertain results.

If asynchronous work genuinely belongs inside a control, such as loading a private visualization resource, define cancellation and destruction behavior. Check that the completion still belongs to the current instance and input generation. Never write into another control's DOM as a workaround for a stale result.

## Optimize only after measuring

Normal property invalidation is the safe starting point. Suppressing invalidation can leave the rendered wrapper stale even if an inner control updates correctly. The panel keeps normal invalidation because its simplicity outweighs speculative micro-optimization. A proven optimization needs a documented invariant and verification for initial rendering, updates, and destruction.

Avoid creating a control for every decorative character in a dense visualization. At the same time, replacing standard controls with handwritten interactive markup transfers accessibility and keyboard costs to the application. Measure real task data and retained objects after repeated open/close cycles before deciding where that tradeoff is justified.

A reusable control should not perform synchronous module loading, trigger network requests during property formatting, or subscribe globally without cleanup. Cache immutable resources carefully, not live child instances shared between parents. Stable API boundaries and ordinary framework rendering usually outperform clever DOM manipulation when long-term maintenance is included.

## Publish an evolution policy before sharing

Once several features consume a control, its metadata becomes a compatibility boundary. Renaming a property, changing an event's timing, or switching an aggregation from owned content to referenced content can break consumers even when the screenshot looks identical. Describe deprecated settings and provide a migration path rather than silently repurposing them.

Separate public behavior checks from implementation checks. Consumers should verify the panel's press event and enabled behavior without reaching into `_action`. A control's own focused checks may inspect its composition, but they should not make an undocumented standard-control DOM structure part of the public contract. This separation lets a future implementation replace the private button composition without rewriting every consuming screen.

Cloning deserves deliberate review. Hidden aggregations are implementation details created by the control lifecycle, not public configuration that consumers should clone manually. Do not assume that copying a reference to a child creates a fresh instance. Verify that cloned public content has independent identity and that associations point to the intended external objects in the clone's usage context.

For a shared library, document supported UI5 versions, dependent libraries, theme expectations, accessibility limitations, and module paths. A dependency on `sap.m` must be declared even when the control module imports its button asynchronously. Library consumers should not discover missing resources only when a rarely used action panel first becomes visible.

## Make visual styling a bounded extension point

The example intentionally omits custom CSS because its purpose is ownership and behavior. If a production panel adds spacing or borders, scope those rules to its own root class and use theme-aware values. Do not select the private button's internal descendants to change its semantics or reconstruct its disabled appearance.

Document whether consumers may add a style class and what that class can reasonably affect. A public customization point should survive internal composition changes. Prefer a small property with defined semantics when consumers need a stable density or emphasis choice; unlimited CSS overrides turn every consumer into a collaborator in maintaining the renderer.

## Manual verification checklist

- Create two instances and verify independent properties, events, IDs, and content ownership.
- Bind and update every public property; check defaults and setter chaining.
- Supply malicious-looking text containing angle brackets and quotes; verify it remains text.
- Add, remove, reuse, and destroy content; verify the expected parent and lifetime.
- Destroy the panel and confirm associated external labels remain alive.
- Navigate by keyboard and check action naming, group description, and focus order.
- Repeat rendering and open/close cycles; confirm no duplicated listeners or retained registrations.
- Check disabled behavior without assuming content children are disabled.
- Verify API version 2 rendering uses only semantic RenderManager APIs and renders children through `renderControl`.
- Review high contrast, narrow width, right-to-left layout, and long translated labels.

## Cross-references

See [module structure](../02-ARCHITECTURE-PATTERNS/module-structure.md), [accessibility](../06-BEST-PRACTICES/accessibility.md), [testing strategy](../06-BEST-PRACTICES/testing-strategy.md), and [freestyle patterns](freestyle-patterns.md). These are the exact companion paths requested by the repository README. Official source links in this chapter distinguish moving conceptual documentation from the version-tagged 1.120 API baseline; verify newer guidance before using newer methods.

Research accessed the cited official GitHub sources; direct SDK access was unavailable. The [RenderManager SDK reference](https://ui5.sap.com/#/api/sap.ui.core.RenderManager) is supplied for deployment-version follow-up, not as a page inspected during preparation. Manual API review does not replace keyboard, screen-reader, or runtime verification of a consuming application.
