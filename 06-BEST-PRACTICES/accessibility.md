# Accessibility for SAP Fiori and UI5

## Scope and prerequisites

Accessibility is part of application correctness, not a final visual adjustment. This chapter covers semantics, accessible names, keyboard interaction, focus, screen-reader output, contrast, validation, and practical verification for UI5 applications. It applies to freestyle screens and supported Fiori elements extensions without replacing framework-owned rendering.

Assume a maintained UI5 1.x runtime with standard accessibility behavior enabled, an approved supported theme, and a documented browser and assistive-technology matrix. Check API availability against the deployed SDK. Examples use ordinary XML views and standard `sap.m` controls; they do not require a new package or a custom accessibility framework.

Use WCAG 2.2 Level AA as the technical target where appropriate to product and contractual requirements. That target includes applicable Level A and AA criteria, not only criteria mentioned here. This guide is not a legal compliance certificate, and passing an automated scan does not establish conformance.

## Start with standard controls

Choose a standard UI5 control that matches the intended interaction before adding custom ARIA. A real button already has keyboard behavior, focus handling, role, and disabled-state semantics. A clickable text or icon requires recreating those responsibilities and is easier to get wrong.

The framework provides accessibility infrastructure, but application authors still own meaningful labels, logical structure, correct configuration, and complete workflows. Official UI5 guidance distinguishes framework and application screen-reader responsibilities and states that accessibility mode is enabled by default [1].

Do not override framework-generated roles or states merely because a scanner reports an unfamiliar pattern. Determine the control's documented behavior, inspect the accessibility tree, and reproduce the issue with a supported runtime and assistive-technology combination.

When the standard control cannot meet a requirement, first consider composition. A custom control is justified only with an explicit interaction model, semantic contract, keyboard design, and test plan.

## Define page structure semantically

Use meaningful page titles, section headings, and landmarks supported by the controls in the deployed version. A bold text label is not necessarily a heading. Structure should allow a screen-reader user to understand the page and move between major sections without traversing every field.

Maintain a logical heading hierarchy that describes content rather than typography. Choose the semantic level according to the page structure and use supported styling separately. Avoid multiple unrelated regions with the same generic accessible name.

Consider the hosting shell. An embedded application should not blindly duplicate every global navigation landmark or invent another top-level shell. Inspect the combined page rather than testing only the standalone development launch.

Tables need appropriate column headers and meaningful cell content. Choose a table control whose keyboard and responsive behavior match the dataset. Do not turn a visual grid of containers into a semantic table by adding arbitrary roles to generated DOM.

## Give controls useful accessible names

Every actionable control needs an understandable name in context. Use visible labels where possible. An icon-only “delete” action repeated on many rows may need the relevant object context, but avoid names so verbose that routine navigation becomes exhausting.

Connect input labels through documented control associations. Placeholder text is not a label: it can disappear during entry and often provides poor context. Instructions, units, and validation hints should remain available while the user interacts.

The following is an illustrative XML view fragment for a real application. It assumes an existing JSON view model named `form` and an existing resource model named `i18n`; the resource keys must be supplied by the application. It defines no controller handlers and adds no manual DOM attributes.

```xml
<mvc:View
    xmlns:mvc="sap.ui.core.mvc"
    xmlns="sap.m">
    <VBox>
        <Label
            id="supplierLabel"
            text="{i18n>supplierLabel}"
            labelFor="supplierInput"
            required="true" />
        <Input
            id="supplierInput"
            value="{form>/supplierId}"
            required="true"
            ariaDescribedBy="supplierHint" />
        <Text
            id="supplierHint"
            text="{i18n>supplierHint}" />
    </VBox>
</mvc:View>
```

The visible hint might explain an identifier format, not contain essential information only on hover. Required-state presentation does not implement validation; the application must still validate input and the backend must enforce its own rules.

Prefer `labelFor`, `ariaLabelledBy`, and `ariaDescribedBy` associations where documented for the chosen control. Avoid assigning raw `aria-label` attributes through custom data or post-rendering jQuery. Framework rerendering can remove them, and an incorrect label can replace useful visible naming.

## Use ARIA only to close real gaps

ARIA supplements semantics; it does not supply keyboard behavior or business logic. Giving a container a button role does not make Space and Enter activation correct. Adding `aria-expanded` does not create a functional disclosure interaction.

Before adding ARIA, identify the exact missing information and whether a standard UI5 property or association already expresses it. Inspect the computed accessible name, role, state, and description in the browser accessibility tree.

For custom controls, implement semantics through supported renderer and accessibility APIs for that runtime. Keep names and states synchronized with properties. A checkbox that looks selected but announces unselected is an application defect even when mouse interaction appears correct.

Avoid redundant or conflicting attributes. A name should identify the control, while a description supplies supplementary information. Do not repeat the same paragraph through both mechanisms or attach every form instruction to every field.

## Preserve keyboard operation

All functionality must be operable through a keyboard interface without requiring specific timing, subject to WCAG's narrow path-dependent exception [2]. Approval, filtering, value help, table actions, file interactions, and recovery dialogs all belong in the keyboard test scope.

Use Tab and Shift+Tab to move between appropriate interactive elements. Composite controls may use arrow keys internally rather than placing every child in the tab sequence. Follow the standard control's documented keyboard model instead of forcing an extra tab stop onto every cell.

Avoid positive `tabindex` values. They create a separate ordering scheme that quickly diverges from document structure. Do not capture global shortcuts that conflict with browser, operating-system, screen-reader, or shell commands.

Provide a non-drag alternative for operations such as reordering where required. A pointer gesture must not be the only way to accomplish a business task. Keyboard alternatives should be discoverable and should preserve meaningful feedback.

## Manage focus through transitions

Define where focus goes when a dialog opens, closes, or fails to load. Standard dialogs manage much of this behavior; application code should avoid immediately overriding it. After dismissal, return focus to the initiating action when it still exists, or to a meaningful alternative.

When navigation changes the main content, use supported application or shell mechanisms to establish a sensible destination. Do not focus an arbitrary hidden element simply to satisfy a test. Preserve context so users understand which object or section is now active.

If a row is deleted, the focused action may disappear. Move focus to a nearby remaining item or another logical control, and announce the outcome appropriately. A destructive action should not strand focus on the document body.

Keep focus visibly identifiable. Never remove outlines without an accessible replacement. Do not use CSS to hide a required focus target, and do not leave invisible interactive elements in the normal tab sequence. Use supported control visibility and interaction properties so rendered and semantic state remain aligned.

WCAG 2.2 Focus Not Obscured (Minimum), at Level AA, requires that a focused component is not entirely hidden by author-created content [3]. Check sticky headers, bottom action bars, dialogs, and overlays. This criterion is not identical to the stronger AAA requirement that no part be hidden.

## Support screen-reader workflows

Choose supported browser and screen-reader combinations based on the product's actual users and framework support information. Record versions in test evidence. A result from one screen reader does not automatically predict all others.

Test tasks, not just individual labels. A user should be able to identify the page, locate a field, understand its constraints, enter a value, encounter an error, correct it, and complete the action.

Inspect the accessibility tree when spoken output seems wrong. Separate missing application semantics from screen-reader verbosity preferences or platform behavior. Reproduce framework issues with a minimal standard-control example before adding a workaround.

Do not depend on reading visual punctuation or color to convey state. “Overdue” text, meaningful status labels, and correctly associated messages communicate more reliably than an unexplained red icon.

## Announce asynchronous changes deliberately

Dynamic results need appropriate feedback without flooding speech output. A filter that updates a result count may require a concise status announcement. A background refresh should not repeatedly interrupt a user entering a form.

Use standard UI5 messaging patterns first. When a live announcement is necessary beyond those patterns, use the documented runtime facility and select interruption priority based on urgency. Do not create several competing live regions to ensure that “something is read.”

Do not put confidential details into announcements that might be spoken aloud in a shared environment unless they are essential to the task. Status summaries can often communicate success without reading an entire document identifier or personal record.

Avoid making a disappearing toast the only location of essential recovery instructions. Errors requiring action need a durable, discoverable representation. A toast can complement a confirmed state change but should not be the sole evidence that a critical transaction completed.

## Make validation actionable

Identify the field, explain the problem, and provide a correction where possible. “Invalid input” is rarely sufficient. Associate error state and explanatory text through supported control and message mechanisms.

For multi-field submission errors, provide an accessible summary or message collection with navigation to affected fields where supported. Do not automatically move focus after every keystroke or announce every intermediate invalid value during typing.

Preserve input when submission fails. Resetting the form forces keyboard and screen-reader users to repeat work and can make correction impossible when the original data is no longer available.

Distinguish client formatting checks from server business rules. A backend rejection must still become an understandable application message. Do not expose technical stack traces, internal authorization policy, or raw response bodies to explain a failure.

## Verify contrast and non-color cues

For WCAG 2.2 Level AA, normal text generally needs at least 4.5:1 contrast, while qualifying large text needs at least 3:1, subject to the criterion's exceptions [4]. Large text is based on size and weight, not a developer's impression that a label looks prominent.

Necessary non-text visual information such as control boundaries and states is covered by the separate non-text contrast criterion, generally requiring 3:1 against adjacent colors where applicable. Test focus indicators and selected states, not only body text.

Use supported themes and theme-aware values instead of hardcoded brand colors for essential information. Validate the application's custom styles in standard, high-contrast, and supported dark themes. Theme support in the framework does not guarantee that application overrides remain usable [5].

Never communicate error, success, or selection by color alone. Pair color with text, shape, icon meaning, or another understandable cue. Decorative icons should not add redundant speech; meaningful icons need an accessible interpretation.

## Handle zoom, reflow, and touch

Test browser zoom and enlarged text with realistic translations. Labels, instructions, buttons, and validation messages must remain available. Truncating an essential action to an ambiguous fragment is not a satisfactory responsive strategy.

WCAG reflow testing commonly uses a viewport equivalent to 320 CSS pixels wide for vertically scrolling content, with exceptions for content that requires two-dimensional layout. Evaluate those exceptions narrowly; an ordinary form should not inherit a table's exception.

Check sticky content at high zoom. Headers and footers that occupy most of the viewport can obscure focused controls and leave little usable space. Test popovers and value helps near viewport edges.

Consider WCAG 2.2 target-size requirements and applicable exceptions for pointer input. A dense enterprise table still needs usable interaction. Prefer supported density modes and controls over reducing targets through custom CSS.

## Combine automated and manual testing

Use the project's existing automated accessibility checks if available. They can detect missing names, certain contrast failures, and structural problems, but cannot establish whether instructions are understandable or a complex workflow is keyboard-complete.

Use QUnit for custom control semantics and state changes where appropriate, and OPA5 for important interaction flows. Keep selectors based on stable IDs and supported control contracts. Do not assert raw generated DOM markup unless that markup is the custom control's deliberate public rendering contract.

The official OPA5 cookbook explains synchronization behavior that matters for dynamic status and busy-state checks [6]. An automated test that waits for all activity to settle may miss a transient announcement; choose observations according to the actual user requirement.

Manual verification should include keyboard-only operation, supported screen-reader tasks, zoom/reflow, theme variants, and error recovery. Record expected and observed behavior rather than merely checking “accessibility tested.”

## Triage defects with reproducible evidence

Describe the route, synthetic data, control, input method, browser, assistive technology, runtime, and theme. Include the sequence needed to reproduce the defect and the user impact.

Separate application configuration errors from framework defects and third-party limitations. A missing label association is normally application-owned. An incorrect role emitted by an unchanged standard control may require a supported framework update or vendor issue.

Keep workarounds narrow, documented, and removable. A post-rendering patch applied to every input can introduce more defects than it fixes and may fail after any rerender. Prefer an approved framework patch or supported extension point.

Sanitize screenshots, recordings, speech logs, and traces. Accessibility evidence can include entered personal data or confidential business context just like other test artifacts.

## Integrate accessibility into delivery

Add acceptance criteria during design: page structure, label wording, focus destination, keyboard alternatives, status feedback, and error recovery. Review these before implementation so they are not reconstructed after visual approval.

Include accessibility checks in pull-request review and release verification. A library upgrade, custom theme change, or table replacement can affect every workflow even without obvious visual differences.

Maintain a small set of representative tasks covering the application's most important patterns. Rotate broader manual coverage over time, but never omit the only route for a critical task. Track unresolved issues with impact, owner, and remediation plan rather than treating scanner scores as a release argument.

## Verification checklist

- [ ] Standard controls and supported semantics are used before manual ARIA.
- [ ] Headings, labels, instructions, and names make sense in the combined shell.
- [ ] Every task and recovery path is keyboard-operable.
- [ ] Focus remains visible, meaningful, and not entirely obscured.
- [ ] Screen-reader users receive correct names, states, errors, and status changes.
- [ ] Text and non-text contrast meet applicable criteria without color-only meaning.
- [ ] Zoom, reflow, long translations, and supported themes remain usable.
- [ ] Automated checks are supplemented by manual task-based evidence.
- [ ] Workarounds have owners and do not mutate framework DOM indiscriminately.
- [ ] Test artifacts and announcements avoid unnecessary sensitive information.

## Related paths

The [repository README](../README.md) lists these exact companion paths.

- [Code quality](./code-quality.md): `06-BEST-PRACTICES/code-quality.md`.
- [Testing strategy](./testing-strategy.md): `06-BEST-PRACTICES/testing-strategy.md`.
- [Git workflow](./git-workflow.md): `06-BEST-PRACTICES/git-workflow.md`.
- [Performance optimization](./performance-optimization.md): `06-BEST-PRACTICES/performance-optimization.md`.

## Official references

Research access: the official UI5 GitHub documentation and W3C criterion source files below were retrieved successfully. The hosted SDK and published WCAG site were not directly accessible; the Recommendation URL is supplied as the normative reference, not as a claim of successful retrieval.

1. [OpenUI5: Screen Reader Support for Controls](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/screen-reader-support-for-openui5-controls-656e825.md), consulted 2026-10-17.
2. [WCAG criterion source: Keyboard](https://github.com/w3c/wcag/blob/main/guidelines/sc/20/keyboard.html), consulted 2026-10-17; [WCAG 2.2 Recommendation](https://www.w3.org/TR/WCAG22/) is the normative published specification.
3. [WCAG criterion source: Focus Not Obscured (Minimum)](https://github.com/w3c/wcag/blob/main/guidelines/sc/22/focus-not-obscured-minimum.html), consulted 2026-10-17.
4. [WCAG criterion source: Contrast (Minimum)](https://github.com/w3c/wcag/blob/main/guidelines/sc/20/contrast-minimum.html), consulted 2026-10-17.
5. [OpenUI5 Accessibility Features](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/openui5-accessibility-features-7048ca9.md), consulted 2026-10-17. Links keyboard, visualization, high-contrast, messaging, and support-history guidance.
6. [OpenUI5: Cookbook for OPA5](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/cookbook-for-opa5-ce4b180.md), consulted 2026-10-17.

Source files on moving branches are research references, not replacements for the published WCAG Recommendation or the SDK documentation matching a deployed UI5 version.
