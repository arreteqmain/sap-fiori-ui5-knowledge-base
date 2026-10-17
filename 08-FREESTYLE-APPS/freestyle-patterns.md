# Freestyle Patterns and Antipatterns

## Scope and prerequisites

This chapter collects implementation patterns for maintainable freestyle applications: state ownership, request races, fragments, navigation, performance, error handling, and feature organization. Each pattern addresses a failure mode, not a preferred aesthetic. Apply it when its assumptions match the task and verify the resulting behavior under interruption, repeated navigation, and realistic data volume.

Examples assume **UI5 1.120**, asynchronous modules, JavaScript controllers, XML views, `sap.ui.core`, and `sap.m`. Examples involving `sap.f` require that library in the manifest. OData references distinguish V2 and V4 explicitly; JSON models in this chapter hold UI state or illustrative search results, not an automatic replacement for the OData cache.

Start with [freestyle development](freestyle-development.md) for application and control selection, [operational applications](operational-apps.md) for transactions, and [custom controls](custom-controls.md) for rendering and aggregation ownership. SAP's official [performance guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/performance-speed-up-your-app-408b40e.md) and [folder-structure guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/folder-structure-where-to-put-your-files-003f755.md) provide the framework context behind these application-level recommendations.

## Give every state value an owner

A useful state inventory records the value, owner, persistence, and reset rule. “Selected order key: router, bookmarkable, replaced on navigation” is more precise than “store selection globally.” “Dialog busy: dialog controller, transient, cleared after completion or destruction” makes cleanup and testing straightforward.

Keep service-backed entities in their OData model. Keep view presentation state in a named JSON model. Keep shareable identity and layout in routes. Keep durable personalization in the platform's supported persistence mechanism. A model named `global` containing all of these categories encourages accidental coupling and makes Cancel dangerous.

Do not infer one state category from another without a documented rule. A selected row does not prove that the detail route is still current. A zero message count does not prove the backend saved successfully. A hidden button does not prove the user lacks authorization. Derive visible state from explicit contracts and verify service outcomes separately.

For a small editor, an explicit `phase` with values such as `Loading`, `Editing`, and `Saving` is often clearer than five booleans. For independent regions, use independent phases rather than one global busy flag. A long-running attachment upload need not freeze a read-only work-order header.

## Bind the view instead of synchronizing it manually

Use property bindings for displayed values and enabled states, aggregation bindings for repeated items, and element/context binding for selected business objects. Manual loops that call `setText` on every field after every request duplicate the binding system and easily miss refreshes, locale changes, or error recovery.

The following XML excerpt assumes a `ui` JSON model with `phase` and `canEdit`, plus an ordinary business context with `Description`. `%{}` requests an uncoerced embedded binding value, useful when expression binding might otherwise convert a value to the target property type.

```xml
<VBox xmlns="sap.m">
  <Text text="{Description}" />
  <Button
    text="{i18n>edit}"
    enabled="{= %{ui>/canEdit} &amp;&amp; %{ui>/phase} === 'Display' }"
    press=".onEdit" />
</VBox>
```

XML requires escaping the ampersands even inside a binding expression. Keep expressions short and side-effect free. A complex authorization decision or localized business calculation belongs in a focused model/helper, not in a long nested expression that nobody can test in isolation.

Formatters should transform inputs into presentation values. They should not fetch data, mutate models, log business events, or decide whether a transaction is committed. UI5 may call a formatter more often than expected. Prefer OData types for parsing and validation, especially for decimal, date, and time semantics.

## Reject stale asynchronous completions

Search A can complete after search B even if A started first. Debouncing reduces requests but does not solve ordering. Use a generation token, cancel an obsolete request where supported, and apply results only when they still match the current input and live owner.

The controller excerpt below assumes a view-local `ui` model with `/query`, `/loading`, `/results`, and `/errorText`; `_searchService.search` returns a promise and accepts an `AbortSignal`. This is an application service adapter, not an ODataModel API. The component injects `_searchService` before handlers run, and `_searchFailedText` is a localized string.

```javascript
onInit: function () {
  this._searchGeneration = 0;
  this._searchTimer = null;
  this._searchAbort = null;
  this._disposed = false;
},

onLiveSearch: function (oEvent) {
  const sQuery = (oEvent.getParameter("newValue") || "").trim();
  const oUi = this.getView().getModel("ui");
  const iGeneration = ++this._searchGeneration;
  clearTimeout(this._searchTimer);
  if (this._searchAbort) {
    this._searchAbort.abort();
  }
  oUi.setProperty("/query", sQuery);
  oUi.setProperty("/results", []);
  oUi.setProperty("/errorText", "");
  oUi.setProperty("/loading", false);
  if (sQuery.length < 3) {
    return;
  }
  this._searchTimer = setTimeout(async () => {
    const oAbort = new AbortController();
    this._searchAbort = oAbort;
    oUi.setProperty("/loading", true);
    try {
      const aResults = await this._searchService.search(sQuery, {
        signal: oAbort.signal
      });
      if (!this._disposed && iGeneration === this._searchGeneration) {
        oUi.setProperty("/results", aResults);
      }
    } catch (oError) {
      if (!this._disposed && iGeneration === this._searchGeneration
          && oError.name !== "AbortError") {
        oUi.setProperty("/errorText", this._searchFailedText);
      }
    } finally {
      if (!this._disposed && iGeneration === this._searchGeneration) {
        oUi.setProperty("/loading", false);
        this._searchAbort = null;
      }
    }
  }, 250);
},

onExit: function () {
  this._disposed = true;
  this._searchGeneration += 1;
  clearTimeout(this._searchTimer);
  if (this._searchAbort) {
    this._searchAbort.abort();
  }
}
```

The generation increments immediately when input changes, not only when the debounce timer fires. Otherwise an older result could populate the screen during the debounce interval. The completion also checks whether the owner was destroyed, preventing writes into a dead view model.

For normal OData filtering, prefer the collection binding's filtering and request lifecycle instead of introducing a parallel search cache. V2 request handles and V4 binding/context APIs have different cancellation semantics; do not assume either accepts an `AbortSignal`. The adapter pattern above applies only where the adapter contract actually supports it.

## Cache fragment promises, not accidental duplicates

A fast double-click can start two fragment loads before either completes. Cache the promise immediately, not only the resolved dialog. Use the controller's `loadFragment` helper for view ownership and ID prefixing. Avoid global lookups of fragment-local IDs.

```javascript
// Methods inside a Controller subclass; import sap/m/MessageBox as MessageBox.
onOpenHelp: async function () {
  if (!this._helpPromise) {
    this._helpPromise = this.loadFragment({
      name: "demo.workorders.view.Help"
    }).catch((oError) => {
      this._helpPromise = null;
      throw oError;
    });
  }
  try {
    const oDialog = await this._helpPromise;
    if (!this._disposed && !oDialog.isDestroyed()) {
      oDialog.open();
    }
  } catch (oError) {
    if (!this._disposed) {
      const oBundle = this.getView().getModel("i18n").getResourceBundle();
      MessageBox.error(oBundle.getText("helpLoadFailed"));
    }
  }
},

onCloseHelp: function () {
  this.byId("helpDialog").close();
},

onExit: function () {
  this._disposed = true;
}
```

The fragment root must be a `sap.m.Dialog` with local ID `helpDialog`, and the application bundle must define `helpLoadFailed`. This excerpt's `onExit` must be merged with any existing cleanup, not pasted over it. A failed load clears the cached promise so a later interaction can retry. `loadFragment` defaults to adding loaded content as a view dependent and participates in owner destruction; verify those defaults if overriding options.

An alternative using `Fragment.load` is valid, but then the code must explicitly handle owner component context, view ID prefixing, `addDependent`, and late completion after destruction. Mixing the two approaches often creates duplicate dependents or inconsistent IDs. Use one pattern consistently per application.

## Keep navigation and unsaved changes coherent

Routes carry stable business identity, not binding indexes. Derive a detail binding from validated route arguments and handle invalid or unavailable objects intentionally. Preserve meaningful filters and list position when returning from detail. Do not reconstruct the entire master view just to show a changed status.

Centralize the save/discard/stay policy so toolbar Back, list selection, and route-driven transitions use the same rules. Delay rebinding until the user has decided and the selected operation has completed. A global reset on every route change is an antipattern when multiple editors share one model.

Flexible column layouts add layout state to this problem. The official [FlexibleColumnLayout usage source](https://github.com/SAP/openui5/blob/1.120.0/src/sap.f/src/sap/f/FlexibleColumnLayout.js) describes column pages and layout changes. Keep the route and visual layout synchronized without adding a new history entry for every responsive size transition. Check phone behavior explicitly; a hidden column is not necessarily a destroyed page.

## Scope submissions and messages

V4 application update groups provide explicit submission boundaries; V2 uses deferred groups and change-group mappings. Neither creates business atomicity across unrelated services. Give an editor exclusive ownership of its pending scope or design a coordinator for shared edits. A dialog should not silently submit another screen's pending data.

After a V4 `submitBatch`, check the operation's result and retained pending changes, not just promise resolution. For V2, inspect nested batch responses in addition to transport callbacks. Backend actions, draft activation, and ordinary field updates have different success contracts. See [operational applications](operational-apps.md) for concrete examples.

Message ownership must be as explicit as data ownership. Client validation, backend field messages, and global technical failures can coexist. Removing all messages after one successful request can hide unresolved problems elsewhere. Keep persistent errors available and provide a message summary with navigation to relevant fields when appropriate.

## Organize by feature without hiding dependencies

Keep a feature's controller, view, fragments, and focused helpers close enough that a change can be understood locally. Shared service adapters should expose business intentions and typed contracts rather than generic string-based “execute anything” methods. Pure utilities should remain independent of views and components.

A base controller can hold a small stable convenience such as obtaining a router. It should not become a hidden service locator, authorization engine, global state store, and message manager. Prefer explicit imports and injected collaborators where a test needs to replace a dependency.

Use `sap.ui.define` and declare all module dependencies. Avoid synchronous loading inside event handlers and undeclared global constructors. Keep application IDs stable, use view-local lookup through `byId`, and avoid module-level mutable selections. Multiple instances of the same component can run together in a launchpad.

## Measure performance by task phase

Measure application bootstrap, first usable content, query response, interaction latency, submission, and retained memory separately. A single “page load” number hides the cause. The official [performance recommendations](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/performance-speed-up-your-app-408b40e.md) distinguish framework configuration from network and server effects and recommend asynchronous loading and manifest dependency declarations.

Reduce unnecessary data first. Select required properties, expand relationships deliberately, page large collections, and postpone heavy secondary content. A large attachment history need not block editing a short description. Do not remove required identity or concurrency information merely to minimize response bytes.

Reduce unnecessary controls next. A responsive table creates item controls and is not a virtualized desktop grid. Growing limits incremental rendering but can still accumulate a large control tree. The official [`sap.m.Table` guidance](https://github.com/SAP/openui5/blob/1.120.0/src/sap.m/src/sap/m/Table.js) documents responsive constraints and mobile volume considerations. Choose a different interaction or table where the task requires genuinely dense data work.

Do not suppress invalidation broadly or modify private DOM to make measurements look better. Such optimizations can produce stale accessibility state and fragile patch dependencies. Profile repeated open/close cycles for leaks; a fast first visit does not prove a long operational session remains healthy.

## Preserve accessibility through repeated interaction

Standard controls provide a strong starting point, but composition can still fail. Unlabeled inputs, duplicate headings, focus trapped behind an overlay, or messages announced without object identity are application defects. Associate labels with inputs and make the action's visible name describe its purpose.

When asynchronous content replaces a focused region, preserve focus where possible or move it deliberately to a meaningful control. Do not focus the first input after every data refresh; that interrupts users who are already working elsewhere. Announcements should be useful and restrained, particularly for rapidly updating status regions.

Use SAP's official [accessibility guidance](https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/accessibility-03b914b.md) and control-specific design semantics. A custom renderer inherits additional responsibility, described in [custom controls](custom-controls.md). Test keyboard, zoom, high contrast, long translations, and right-to-left layout before freezing the component contract.

## Recognize recurring antipatterns

| Antipattern | Consequence | Better pattern |
| --- | --- | --- |
| One global mutable application model | Navigation and cancellation affect unrelated screens | Explicit state owners and scoped models |
| Fetching in formatters | Duplicate requests and unstable rendering | Load through bindings or a coordinated adapter |
| Recreating dialogs on every click | Duplicate IDs, listeners, and memory | Promise-cached owned fragments |
| “Latest started wins” without a completion guard | Stale results overwrite current input | Generation token plus cancellation where supported |
| Global model refresh after every save | Extra requests and lost context | Refresh affected bindings or side effects |
| Direct status PATCH for a business action | Bypassed business invariants | Service-defined action with explicit outcome |
| Global message clearing | Hidden unresolved failures | Scoped message ownership |
| Styling private control DOM | Fragile upgrades and incorrect states | Public properties, classes, and composition |

An antipattern sometimes begins as a local shortcut that appears harmless with one screen and one request. Review its behavior with two component instances, two simultaneous requests, and repeated navigation. These small adversarial scenarios expose hidden shared state earlier than a large end-to-end scenario.

## Design observability without collecting business secrets

Use correlation identifiers to connect a failed UI operation with service diagnostics. Record the operation category, duration, outcome, and safe technical identifier; do not log entire entity payloads or user-entered notes. Client logs can be visible to support tools and browser extensions, so treat them as another data disclosure surface.

Separate expected business rejection from technical failure in operational metrics. A rejected release because a required inspection is missing is different from a gateway timeout, even if both prevent completion. This distinction helps teams improve either the workflow or the infrastructure instead of optimizing the wrong layer. Keep diagnostic recording outside formatters and renderers so repeated binding updates do not inflate failure counts.

## Manual verification checklist

- Identify every model's owner, persistence rule, and reset boundary.
- Reverse search response order and verify that only current results appear.
- Destroy a view while search or fragment loading is pending.
- Open the same dialog repeatedly and verify one instance and stable IDs.
- Exercise route navigation while editing, including browser history.
- Confirm V2/V4 APIs match the configured model and backend protocol.
- Inspect partial failures and persistent message ownership.
- Measure realistic row volume and repeated-session retained objects.
- Verify keyboard focus, descriptions, narrow layout, zoom, and translations.
- Check that event subscriptions, timers, and external registrations are removed by their owner.

## Cross-references and source interpretation

Continue with [code quality](../06-BEST-PRACTICES/code-quality.md), [performance optimization](../06-BEST-PRACTICES/performance-optimization.md), [testing strategy](../06-BEST-PRACTICES/testing-strategy.md), [module structure](../02-ARCHITECTURE-PATTERNS/module-structure.md), and [routing and navigation](../02-ARCHITECTURE-PATTERNS/routing-navigation.md). These paths match the repository README's structure.

Conceptual SAP documentation links may evolve after this chapter's baseline. Version-tagged OpenUI5 sources establish the concrete API assumptions. Treat the patterns as design guidance, verify the target patch, and adapt backend-specific behavior from an explicit service contract rather than copying an isolated snippet.

Research used the official GitHub sources cited here because direct SDK/design-host access was unavailable. The [Controller SDK reference](https://ui5.sap.com/#/api/sap.ui.core.mvc.Controller) is an additional deployment-version reference, not a live page inspected during preparation. The checklists describe downstream application verification; no application tests were executed for this documentation.
