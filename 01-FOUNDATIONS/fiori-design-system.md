# SAP Fiori Design System

**Scope:** translating SAP Fiori principles into practical enterprise application decisions, with freestyle UI5 examples. **Prerequisites:** familiarity with business tasks, basic UI5 XML views, and browser interaction. **Runtime assumptions:** supported desktop and mobile browsers and an explicitly selected SAPUI5 or OpenUI5 release. Examples using `sap.m` work within both deliveries when their APIs are available; SAP Fiori elements requires the appropriate SAPUI5 libraries. Snippets are illustrative and were not executed here.

## 1. Treat Fiori as a design system

SAP Fiori is more than a theme applied to a collection of controls. It joins interaction patterns, information hierarchy, visual language, accessibility expectations, and business-task conventions. A UI can use UI5 controls and still violate Fiori principles through confusing navigation, excessive information, or unpredictable saving.

Start with the person and task, not the available component catalog. Describe the trigger, required information, decision, resulting business state, and recovery options. For an approval task, the essential question may be whether an exception justifies approval, not whether every field from the purchase-order database is visible.

The [SAP Fiori design portal](https://www.sap.com/design-system/fiori-design-web/) is the official design reference. Guidelines evolve; names, recommended floorplans, and product-specific restrictions can change. Match design guidance to the chosen SAP product and runtime rather than assuming every current example exists in an older deployment.

## 2. Apply the five principles as decisions

**Role-based** means exposing the information and actions needed for a responsibility. A buyer, warehouse operator, and financial approver may work on related objects while needing very different initial screens. Roles influence defaults and navigation, but they do not replace backend authorization.

**Adaptive** means supporting the device, interaction method, and context. A responsive layout alone does not make a dense approval table practical on a phone. Adapt visible information, action placement, and navigation while preserving the task's meaning.

**Simple** means reducing unnecessary decisions and complexity. It does not mean removing important evidence. Put frequent decisions first and disclose specialist detail when needed. Avoid asking users to choose technical service parameters they cannot reasonably understand.

**Coherent** means similar actions and states behave consistently across applications. If Save persists edits in one application, it should not merely advance to another unannounced editing stage in another. Align labels, keyboard behavior, navigation, and message severity.

**Delightful** means enabling confident, efficient completion. Predictable feedback, useful defaults, good performance, and clear recovery are more valuable than decorative animation. A restrained screen that explains an exception can be more satisfying than an impressive dashboard that obscures the next action.

## 3. Model the task before choosing a floorplan

Write a short task scenario: “An approver reviews overdue requests, examines budget exceptions, and approves eligible requests.” Identify frequency, expected volume, urgency, and consequences of mistakes. Then decide what deserves a list, object detail, comparison, or guided sequence.

Distinguish exploration from execution. An analyst exploring trends may need charts and flexible dimensions. A stock operator executing a movement needs scanning, immediate validation, and a clear confirmation. Combining both into a single overloaded page typically harms both.

Create a content inventory with required, optional, derived, and restricted information. Remove duplicate fields with different technical names. Decide whether the user needs a code, a description, or both. If a field is read-only because of workflow state, explain that state rather than presenting a mysteriously disabled form.

Use realistic data during design reviews. Long supplier names, mixed currencies, missing addresses, and restricted objects reveal weaknesses that pristine sample records hide.

## 4. Select list and object patterns intentionally

A list report supports finding and acting on business objects through filters, search, tables, and navigation to details. An object page organizes one object's identity, status, sections, and actions. A worklist focuses on a bounded queue of items requiring attention.

These are interaction patterns, not just class names. Freestyle UI5 can implement the intent, while SAP Fiori elements supplies supported metadata-driven floorplans with defined extension contracts. Do not claim an OpenUI5 application can instantiate SAPUI5-only Fiori elements libraries simply because its custom page looks similar.

For a list, prioritize the columns that support selection and decision. Put the object's identity first, add status and key evidence, and move infrequent details to the object view. Keep filtering distinct from free-text search and explain whether counts represent the complete result set.

For an object, establish a clear header: identity, meaningful title, business status, and high-value attributes. Organize sections by user reasoning rather than database tables. Separate object-level actions from row-level actions so users understand what will change.

## 5. Use flexible columns and guided flows carefully

A flexible column layout can keep a list and selected detail visible together. It is useful when users repeatedly compare or process objects without losing their place. On narrow screens, preserve the navigation model even though fewer columns remain visible.

Column layout is application state. Deep links and Back navigation should restore an understandable selection and layout, not merely a technically valid view. Avoid stacking complex nested pages that each contain independent scroll areas and competing headers.

A wizard is appropriate when completion genuinely has ordered dependencies. It is not a general-purpose replacement for a long form. Keep steps meaningful, allow review, and define what happens when users return to change an earlier answer.

For occasional confirmation or small contextual tasks, use a dialog or popover instead of navigating to an entirely new workflow. Do not put a full application inside a modal dialog. Complex editing, repeated navigation, or extensive tables usually deserve a page.

## 6. Design navigation around business identity

Use recognizable object titles and preserve user orientation. A link labeled only “Details” repeated across many rows is less informative than an object identifier combined with context. Avoid exposing internal database keys when a business key exists.

Define the distinction between selecting a row, opening the object, and performing an action. A selection checkbox should not unexpectedly navigate. A row action should make its scope clear before changing business data.

Unsaved edits need an explicit navigation policy. Decide whether the system uses drafts, a local edit buffer, automatic persistence, or explicit Save and Cancel. Do not display an ambiguous warning for every harmless navigation simply because the application cannot distinguish dirty and clean state.

Cross-application navigation should preserve relevant context without leaking confidential information into URLs. Use supported shell navigation services when hosted in a launchpad. Never treat a hidden navigation entry as a security boundary.

## 7. Make action hierarchy visible

Identify the primary action for each state, such as Create, Save, or Approve. Secondary actions remain available but should not compete visually. Destructive actions require clear wording and appropriate safeguards, especially when they cannot be reversed.

Button text should describe the result. “Approve request” is clearer than “OK” when a business decision is being recorded. If an action opens a review step rather than executing immediately, its label and feedback should communicate that difference.

For mass actions, explain selection scope: selected visible rows, all loaded rows, or every matching server-side result are different sets. Summarize partial failures and retain enough context for retry without unintentionally repeating successful operations.

Do not make every action emphasized or semantic red. Emphasis guides attention; semantic colors communicate meaning. Reusing success styling for a preferred but non-success-related action weakens the meaning of genuine success states.

## 8. Establish a complete state vocabulary

Design loading, empty, no-results, error, read-only, and success states alongside the normal data state. An empty collection can mean no business objects exist, the current filters exclude everything, or data could not be loaded. Those conditions require different explanations and next steps.

Use busy states at an appropriate scope. A local table refresh should not necessarily block the whole application. Prevent duplicate submissions while requests are pending, but distinguish waiting from failure and offer recovery when a request times out.

Transient success feedback can acknowledge a completed noncritical operation. Important failures must remain discoverable; a disappearing toast is insufficient for an error that prevents task completion. For mixed results, explain what succeeded and what requires attention.

Avoid false certainty. A background request being accepted is not necessarily equivalent to the business process finishing. Label queued or processing states accurately, and provide a route to check progress where the service supports it.

## 9. Use semantic messages with recovery guidance

A useful message states what happened, its scope, and what the user can do next. “Request could not be saved because its approval status changed. Reload the request and review your edits” is more actionable than “Backend error.”

Separate validation messages from service outages. Place field-specific feedback near the relevant field and expose an accessible message summary for complex forms. Do not erase all entered data after one invalid field.

Keep technical correlation identifiers available for support without exposing stack traces, tokens, personal records, or internal hostnames in ordinary messages. Translate user-facing explanations while preserving a stable support identifier.

Message severity should reflect the business situation. An overdue deadline is not automatically a technical error. A warning may permit continuation; an error may block it. The application and backend must agree about which actions remain valid.

## 10. Accessibility is an application responsibility

Standard UI5 controls provide substantial keyboard and accessibility behavior, but application composition can undermine it. Missing labels, poor focus movement, inaccessible custom content, or contradictory headings remain application defects.

The official [UI5 accessibility guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/accessibility-03b914b.md) explicitly treats accessibility as a continuing development responsibility. Do not describe a screen as compliant merely because its controls come from UI5.

Plan accessibility acceptance criteria for each task. A keyboard-only user should be able to locate a request, understand its status, change valid fields, discover errors, and complete or cancel the action. A screen-reader user needs meaningful names, relationships, and timely feedback.

Consult organizational accessibility requirements and applicable standards. Automated checks help find structural issues but cannot prove that a business workflow is understandable or that its focus behavior is sensible.

## 11. Label fields and describe relationships

Use visible labels connected to inputs. Placeholder text disappears during entry and is not a durable label. Explain unusual formats or constraints before the user fails, and associate supplementary instructions with the control.

The following XML belongs inside the content of a view using `sap.m` and `sap.ui.core`. It assumes named `edit` and `i18n` models. It provides a label and description without duplicating ARIA markup manually:

```xml
<mvc:View
    xmlns:mvc="sap.ui.core.mvc"
    xmlns:core="sap.ui.core"
    xmlns="sap.m">
    <VBox class="sapUiSmallMargin">
        <Label text="{i18n>reference}" labelFor="referenceInput"/>
        <Input
            id="referenceInput"
            value="{edit>/reference}"
            ariaDescribedBy="referenceHelp"/>
        <Text id="referenceHelp" text="{i18n>referenceHelp}"/>
        <ObjectStatus
            text="{i18n>approvalPending}"
            state="Warning"/>
        <core:InvisibleText
            id="requestListLabel"
            text="{i18n>requestList}"/>
        <List
            ariaLabelledBy="requestListLabel"
            noDataText="{i18n>noRequests}"/>
    </VBox>
</mvc:View>
```

Define every resource key and load the required libraries in the component descriptor. The empty list is intentional here; a real list additionally needs an items binding and template. The visible status text communicates meaning independently of its warning color.

For icon-only buttons, supply an accessible name, commonly through the documented tooltip and labeling facilities. Prefer visible action text where space permits. Do not attach a tooltip to every visible label; redundant or conflicting names can make screen-reader output worse.

## 12. Preserve keyboard and focus behavior

Use standard interactive controls instead of clickable generic containers. A `Text` with a press-like DOM listener is not automatically a keyboard-operable button. Native UI5 controls already implement many expected Enter, Space, arrow-key, and focus conventions.

When opening a dialog, choose a useful initial focus target. When closing it, return focus to the triggering context where possible. If the triggering row disappeared after deletion, choose a predictable remaining target rather than leaving focus on the page body.

Do not move focus on every refresh or status update. Unexpected movement disrupts typing and screen-reader reading. For asynchronous results, use supported announcements where appropriate, reserving assertive interruption for genuinely urgent information.

Keep a logical tab order. Do not add positive `tabindex` values to compensate for an incoherent visual layout. Keyboard shortcuts should complement standard interaction, avoid common browser conflicts, and be documented.

## 13. Support zoom, reflow, and content growth

Design for long translations, larger text, browser zoom, and right-to-left languages. Fixed pixel heights and clipped labels often fail before an application reaches production because they were evaluated only with short English text.

Use responsive layouts, relative units where custom styling is necessary, and controls designed to wrap or reveal content. Preserve access to primary actions when the viewport shrinks. Check dialogs as well as full pages; a modal can become unusable when its footer falls outside the viewport.

Tables require explicit prioritization. Responsive tables can move selected columns into pop-ins, but not every column should be hidden indiscriminately. Users still need enough identity and evidence to understand each row.

Content density is another variable. Compact density can improve mouse-and-keyboard productivity, while touch interaction needs adequate targets. Do not apply conflicting compact and cozy classes at nested levels or force a desktop density merely because a screen contains many fields.

## 14. Use themes as semantic contracts

Themes control colors, typography, spacing, and states across the supported libraries. Choose a theme family supported by the application's runtime and required libraries. Horizon variants are examples, not a guarantee that a particular historical installation contains them.

Prefer control properties and documented style classes over overriding internal selectors. Hardcoded white backgrounds or gray text can destroy contrast in dark or high-contrast themes. A corporate brand color is not automatically an accessible text color.

The official [colors and theming guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/colors-and-theming-086c41c.md) recommends theme-aware styling and checking custom styles across themes. A custom theme must be evaluated, not assumed to retain the accessibility characteristics of its source theme.

In a launchpad, user theme selection is normally managed by the host. A standalone application must decide how to expose supported theme choices. Do not silently override the user's shell preference each time a view is opened.

## 15. Change themes through supported APIs

Modern UI5 releases expose `sap/ui/core/Theming`; older baselines may require different documented APIs. Check the module's availability in the actual target SDK before using this example. It belongs in a standalone application's controller that offers an explicitly labeled theme choice, not in a launchpad startup routine:

```javascript
sap.ui.define([
    "sap/ui/core/mvc/Controller",
    "sap/ui/core/Theming"
], function (Controller, Theming) {
    "use strict";

    return Controller.extend("org.example.orders.controller.Settings", {
        onUseHighContrast: function () {
            Theming.setTheme("sap_horizon_hcb");
        }
    });
});
```

Ensure the selected theme resources are served for all required libraries. Do not accept arbitrary user-supplied theme resource URLs. Theme changes can load external resources; restrict configuration to approved themes and trusted origins.

For custom visualizations, map semantic values to documented theme parameters rather than copying computed colors once at startup. Account for theme changes and release any theme listeners your integration registers.

## 16. Internationalization affects interaction design

Put user-facing strings in resource bundles rather than controllers or backend error-code maps scattered through the application. Provide complete messages with placeholders instead of concatenating fragments that assume English word order.

Format dates, numbers, and currencies using locale-aware types or formatters. Always display the relevant currency or unit when ambiguity could affect a decision. A number labeled only “Amount” is dangerous in a mixed-currency approval queue.

Right-to-left support includes alignment, reading order, mixed-direction identifiers, and directional icons. Let standard controls handle supported directionality rather than forcing CSS `direction` globally.

Time zones need a business decision. A date-only posting date differs from an instant such as a shipment timestamp. Displaying both through the same JavaScript date conversion can move a business date to the previous day for some users.

## 17. Design privacy and trust into the interface

Only display information needed for the task. Masking data visually does not remove it from the network response, and hiding a field does not make it unavailable to browser tools. Coordinate data minimization with the service.

Keep sensitive values out of URLs, analytics events, local storage, and generic error messages. Explain consequential actions before execution. Avoid confirmation fatigue by reserving interruptions for situations with material risk or unclear scope.

If a session expires during editing, provide a safe recovery path without persisting confidential form data in an unapproved location. State what was saved, what was not saved, and whether retry is safe.

## 18. Review with realistic failure scenarios

Walk through the task with keyboard only, high contrast, a narrow viewport, long translations, slow responses, and partial service failures. Include an unauthorized user and a user who has access to only some objects.

Inspect reading order, labels, focus return, and message discovery rather than reviewing screenshots alone. Ask participants to explain what will happen before they activate a button; mismatched expectations reveal action-label and hierarchy problems.

Record design decisions and unresolved tradeoffs with acceptance criteria. “Responsive” is too broad; “the approver can review exception reason and approve using keyboard navigation at the supported zoom level” is testable.

## Verification checklist

- [ ] Identify the role, task, decision evidence, and expected outcome.
- [ ] Choose a floorplan based on workflow rather than habit.
- [ ] Define edit, save, cancel, loading, empty, and failure states.
- [ ] Verify action scope, labels, and mass-operation feedback.
- [ ] Check accessible names, headings, keyboard flow, and focus recovery.
- [ ] Review high contrast, zoom, long translations, and narrow screens.
- [ ] Confirm theme and API availability in the selected runtime.
- [ ] Enforce authorization and data minimization in services.

This checklist describes review work to perform; it is not an execution record.

## Official references and sibling documents

- [SAP Fiori design guidelines](https://www.sap.com/design-system/fiori-design-web/)
- [Official labeling and tooltip guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/labeling-and-tooltips-329a029.md)
- [Official keyboard guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/keyboard-handling-e303820.md)
- [Official text size and font guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/text-size-and-fonts-a3465c8.md)
- [SAPUI5 versus OpenUI5](https://github.com/SAP/openui5-docs/blob/main/docs/02_Read-Me-First/sapui5-vs-openui5-5982a97.md)

Read [UI5 framework](./ui5-framework.md) for control and lifecycle mechanics, [project setup](./project-setup.md) for descriptor integration, and [development environment](./development-environment.md) for local feedback workflows. These are the sibling foundation paths listed in the [README](../README.md).
