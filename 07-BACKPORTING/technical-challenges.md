# Technical challenges in an ECC-backed Fiori implementation

## Scope and prerequisites

This document explains the engineering problems behind an S/4HANA-inspired user experience on ECC. It assumes the approach and entitlement decision from [S/4HANA-to-ECC guide](./s4hana-to-ecc-guide.md) already exists. It does not authorize redeployment of SAP proprietary applications or imply standard support for a backport. Use [Compatibility matrix](./compatibility-matrix.md) before choosing dependencies and [Implementation steps](./implementation-steps.md) for delivery sequencing. These names match the [repository README](../README.md).

The running example is a customer-owned purchasing worklist with a separately governed change operation. Names beginning with `Z` describe proposed customer objects. They are not SAP-delivered services, BAPIs, or ready-to-install artifacts. Code fragments illustrate contract boundaries; complete transportable implementations require target-release syntax checks, generated Gateway types, local authorization objects, and the actual business interface.

## Separate compatibility from resemblance

A browser component can render successfully while its backend assumptions remain false. An S/4HANA service may combine released CDS entities, new document semantics, workflow services, value helps, and analytical engines unavailable in the target. Matching entity names or creating empty navigation collections does not reproduce that behavior.

Decompose each screen into reads, mutations, navigation, messages, personalization, and lifecycle. Label every dependency as technical, semantic, operational, or commercial. A UI5 control missing from the installed runtime is technical. A purchase-order status without an equivalent target definition is semantic. An unsupported SAP application deployment is not made acceptable by solving either problem.

ECC is neither uniformly transactional nor uniformly incapable of analytics. A database-independent summary query can be feasible while a specific HANA-dependent KPI application is not. ABAP CDS availability likewise depends on release and features, not HANA alone. The safest design starts from the target's proven capabilities rather than a binary “ECC versus S/4HANA” label. [T2]

## Reconcile business models before schemas

S/4HANA's universal journal cannot be mapped mechanically to one ECC table. A requested measure may require multiple sources, ledger rules, reconciliation, document splitting considerations, and currency treatment. Record its business definition and lineage before deciding whether an equivalent implementation exists.

Supplier experiences can assume business-partner roles and relationships that differ from the target vendor model. Define the identifier exposed by the custom service and the mapping used for each task. Do not strip leading zeroes to “normalize” identifiers: that can break keys, navigation, and external references.

Purchase-order header, item, schedule, history, and account-assignment records have different grains. Joining all of them before aggregation can multiply amounts and quantities. Establish one row's grain first; pre-aggregate appropriate child facts before combining them. Check cardinality using documents with multiple schedule lines and account assignments, not only simple demonstration data.

Maintain a field-lineage table containing source, conversion, authorization scope, null handling, derivation, and freshness. Distinguish an absent fact from a zero value. A missing measure should produce an explicit unsupported requirement or documented omission, not a plausible-looking constant.

## Select the ABAP data-access mechanism

ABAP CDS defines semantic models on the ABAP standard database, but individual constructs have release and runtime requirements. Older DDIC-based CDS views and modern CDS view entities are not interchangeable syntax. Modern examples can include features absent on ECC even when a more limited CDS feature set exists. [T2]

Choose portable Open SQL where it provides the required contract on the installed release. Consider CDS when the required syntax, authorization behavior, annotations, and exposure path are available. Treat AMDP, SQLScript, HANA calculation views, and database-specific functions as distinct dependencies. ECC on HANA does not create the S/4HANA virtual data model or upgrade the ABAP compiler.

Prefer explicit projection and typed predicates. Do not build unrestricted dynamic SQL from OData query text. Use an allowlisted mapping between public property names and backend fields; reject unknown expressions. Neither a generated Gateway provider nor a CDS definition automatically guarantees acceptable query cost.

If the implementation reads SAP tables directly, record the maintenance risk and functional validation obligation. “Not a released API” and “technically inaccessible” are different facts. The architecture board must decide whether the dependency is acceptable and how changes will be detected.

## Define the OData contract

For a custom V2 worklist, define `PurchaseOrderItems` with a composite string key such as purchase order and item, plus the fields approved for the task. Keep numeric-looking document identifiers as strings. Expose amount and currency together, quantity and unit together, and dates with explicitly documented semantics.

Declare precision, scale, maximum lengths, nullability, concurrency properties, supported filters, sort orders, and navigation multiplicity. If the UI cannot function without a server feature, make that feature a prerequisite rather than silently simulating it with partial data.

V2 function imports are not interchangeable with V4 actions. V4 binding APIs, draft flows, and aggregation requests require deliberate redesign when the target service is V2. Generated pages also depend on annotation vocabulary support in the exact UI5 runtime. Removing an annotation can change behavior, not just appearance.

Document unsupported query options and return a controlled client error for them. Returning all data when a filter parser cannot understand an expression is both a correctness and a confidentiality failure.

## Enforce authorization before selection

A launchpad role is not row-level security. Check service access, then business authorization for the requested operation and organization. Apply authorized restrictions before rows, counts, aggregates, navigation reads, and expansions. An unauthorized user must not infer another organization's document volume through `$count`.

ABAP SQL does not inherently perform business authorization. CDS access control can constrain qualifying CDS reads, but direct access to underlying tables does not automatically inherit that restriction. Authorization on a reused view also needs inspection at the actual consumption point. [T3]

The following fragment illustrates a deliberately defined customer authorization object. It assumes `ZPO_SCOPE` has fields `ACTVT` and `EKORG`, with role assignments reviewed by security. This does not replace standard purchasing authorizations or checks inside the chosen business API.

```abap
AUTHORITY-CHECK OBJECT 'ZPO_SCOPE'
  ID 'ACTVT' FIELD '03'
  ID 'EKORG' FIELD lv_ekorg.
IF sy-subrc <> 0.
  "Raise a mapped forbidden response; return no rows or counts.
  RAISE EXCEPTION TYPE zcx_po_forbidden.
ENDIF.
```

`zcx_po_forbidden` is a proposed customer exception, not a SAP framework class. In a real provider, map it to the target Gateway release's documented exception mechanism. Derive `lv_ekorg` from a validated request scope for collection reads and from the persisted document for keyed reads and writes. A client-supplied organization cannot establish authority over a different stored document.

## Make paging and filtering truthful

Process requests in this conceptual order: authenticate, validate query syntax, determine authorized scope, apply business filters, compute any requested count, sort deterministically, then page. Validate nonnegative paging values and cap response size through a documented server-paging contract. If a request exceeds the permitted page size, reject it or provide a supported continuation mechanism; do not truncate silently.

```text
allowed = resolve_authorized_organizations(current_user)
query   = parse_allowlisted_filter(request.filter)
scope   = intersect(allowed, query.organization_constraints)
rows    = select_matching_items(scope, query.business_predicates)
count   = count_same_authorized_population_if_requested()
order   = validated_sort + missing_unique_key_tiebreakers
page    = apply_documented_paging(rows, order, request)
```

This is an algorithm, not an instruction to materialize all matching rows in memory. Push safe predicates, ordering, and limits down through the target-supported mechanism. If the older SQL dialect cannot efficiently implement the required offset behavior, consider a framework-supported alternative or change the contract.

Offset paging over a changing dataset can skip or duplicate records even with deterministic ordering. State that consistency model explicitly. For a critical reconciliation extract, prefer an approved snapshot or continuation design rather than treating UI scrolling as an audit-quality export. The UI5 V2 model documentation itself warns about changes between `$skip` and `$top` requests. [T1]

## Preserve decimal, unit, and date semantics

Use `Edm.Decimal` with deliberate precision and scale. V2 JSON decimal values should stay strings across the JavaScript boundary. `parseFloat`, `Number`, and arithmetic on IEEE-754 values can lose precision before the backend sees the request. Formatting for display is not validation for posting.

An approved amount-editing field can use the UI5 OData decimal type with a string model representation, subject to the installed runtime's type API. SAP's type documentation explicitly describes string representation and precision/scale constraints. [T7] Currency-specific decimal rules still require domain handling: a fixed service scale alone does not establish the correct display or business rounding rule for every currency.

```xml
<Input xmlns="sap.m"
  value="{
    path: 'Amount',
    type: 'sap.ui.model.odata.type.Decimal',
    constraints: {precision: 15, scale: 2}
  }" />
```

This illustration assumes the service explicitly defines a scale-two `Amount`. It is not a universal monetary editor. Preserve the currency property and validate its relationship to the amount. Reject overflow and excess scale in the service; avoid trusting a localized string submitted directly from an input control.

For dates, distinguish date-only business values from UTC instants and local wall-clock times. Test midnight boundaries, daylight-saving changes, and browsers in different time zones. A delivery date should not shift because a browser converts an assumed UTC midnight to the previous local day.

## Assign LUW ownership

One business command needs a documented transaction owner. A Gateway changeset may coordinate operations, but individual APIs can contain internal commits, update-task behavior, or remote calls that invalidate naive atomicity assumptions. The SAP LUW documentation explains why implicit and explicit commits matter. [T4]

Do not insert `BAPI_TRANSACTION_COMMIT` after each wrapper call and assume that the surrounding changeset can still roll everything back. Inspect the actual business interface's transaction contract. Where a locally called interface can participate in one LUW, coordinate validation and commit through the selected Gateway processing model. Where it cannot, reject the claimed atomic design or create an explicitly asynchronous command process with reconciliation.

```text
command coordinator:
  validate payload and identity
  acquire documented business-object lock
  reread authoritative state under that lock
  authorize against persisted organization and current state
  compare supplied version with authoritative version
  invoke the reviewed target business interface
  map all business messages, including errors and aborts
  complete the transaction through its designated owner
  release locks according to the real interface contract
```

The diagram deliberately does not name a generic “change document” BAPI. Select and verify the actual target API, its parameters, enhancement behavior, release status, authorization coverage, and LUW obligations. If the interface requires an incompatible commit model, a wrapper cannot make it atomic by naming it an adapter.

## Combine ETags with locks correctly

An ETag detects stale representations; an enqueue lock protects a critical update section. Read an authoritative revision, compare `If-Match`, then perform the change under appropriate locking. Return a conflict or precondition failure according to the service contract rather than overwriting another user's work.

The version source must account for all relevant writers, including SAP GUI, batch programs, and integrations. A custom revision column updated only by the new service fails this requirement for shared standard documents. Hashing a representation is not sufficient unless the locked re-read and hash calculation cover the same business state changed by every writer.

The UI5 V2 model can use cached ETags or an explicit `eTag` parameter for updates. It does not verify the backend's revision algorithm. Do not replace a stale ETag with `*` to “fix” a save failure. Refresh the authoritative state, explain the conflict, and require a deliberate user decision. [T1]

For create or approval commands, ETags alone may not prevent duplicates after a lost response. Use a supported idempotency contract where possible. Otherwise reconcile by a reliable business identifier before retrying; the client must not assume that a network error means no commit occurred.

## Preserve CSRF and session behavior

The V2 model manages CSRF tokens for protected writes and offers `refreshSecurityToken` when a refresh is needed. Keep the model's token handling enabled and preserve cookies through the approved same-origin proxy path. Do not inject tokens into URLs or disable server protection to resolve routing problems. [T1]

Distinguish authentication expiry from CSRF failure and business authorization denial. A login page returned as HTML to an OData request is not valid service data. Proxy caching must not serve another user's token-bearing service response. Do not log token headers, cookies, or authorization credentials in client diagnostics.

For a token-related retry, confirm the original request failed specifically before business execution. Never apply a generic retry interceptor to every failed POST. An ambiguous timeout after a commit requires reconciliation, not another blind execution.

## Adapt UI5 without hiding backend gaps

Use a small service adapter between controllers and backend operations. The adapter maps error categories and field names, while the OData model continues to own metadata, serialization, CSRF, and HTTP interactions. Avoid duplicating OData state in an untyped JSON cache unless its lifecycle and concurrency rules are explicit.

Use `createKey` after metadata is available to construct keyed paths instead of concatenating document numbers and quotes. Pass filter objects and sorter objects to the model rather than concatenating user input into URLs. Preserve backend messages sufficiently for the user to correct data without disclosing implementation details.

For batch requests, an outer success callback can accompany failed inner operations. Inspect changeset and operation responses; report success only when the required business command has succeeded. One UI batch group is not automatically one atomic transaction. The V2 documentation distinguishes batch groups from changesets and explains their configuration. [T1]

## Control analytical cost

Define each measure's grain, additive dimensions, currency, unit, and freshness. A chart summarizing one loaded page is not a total over all authorized matching documents. Mixed-currency sums are invalid without grouping or an approved conversion policy.

Choose live aggregation only when the target query plan and volume support the response budget. A scheduled aggregate can be acceptable if its timestamp and reconciliation rules are visible. Existing BW or other established reporting integration may be preferable to reproducing an S/4HANA analytical engine inside an ECC dialog request.

On anyDB, portable SQL can still aggregate effectively with selective predicates and appropriate indexing. On HANA, poor cardinality and excessive result transfer can still be expensive. Measure both backend execution and payload size, and avoid making database brand a proxy for proven performance.

## Diagnose failures at the correct layer

| Symptom | Likely investigation | Unsafe shortcut |
|---|---|---|
| Tile missing | Catalog, target mapping, role, launchpad configuration | Grant broad backend authorization |
| Metadata unavailable | Routing, alias, service registration, authentication | Replace service URL with a hard-coded host |
| List count differs from rows | Authorization and predicate alignment | Hide the count without investigating |
| Save reports success but data unchanged | Inner batch responses, update completion, business messages | Add unconditional commits |
| Duplicate documents after timeout | Commit status and retry policy | Retry every failed POST |
| Stale update overwrites data | Revision source and locked comparison | Use wildcard `If-Match` |
| Totals inflate after join | Fact grain and child multiplicity | Apply `DISTINCT` until numbers look plausible |
| Date shifts by one day | Business-date representation and time zones | Add a fixed browser offset |

Capture sanitized request identifiers, operation names, timings, and backend correlation references. Diagnostic payloads should follow access and retention rules. An error log is not a reason to expose purchasing prices or personal data to every front-end support user.

## Technical acceptance gates

Before UI acceptance, require a contract review showing every property, action, authorization boundary, and transaction assumption. Before performance acceptance, require representative volume and concurrent access. Before release, demonstrate negative authorization cases and failure recovery, not just a successful administrator session.

- [ ] Missing fields have explicit business decisions rather than fabricated defaults.
- [ ] ABAP and CDS features are verified against the actual target release.
- [ ] Filters, counts, paging, expansions, and totals share authorized semantics.
- [ ] Decimal strings, units, currencies, and dates survive round trips.
- [ ] The actual write interface and LUW owner are documented.
- [ ] ETags detect changes from every relevant writer or limitations block editing.
- [ ] CSRF and identity propagation remain enabled and correctly routed.
- [ ] Batch errors, lock conflicts, and uncertain writes have safe UX paths.
- [ ] Unsupported standard backport assumptions have been removed.

## Evidence register and sources

Research date: 2026-10-17. The following SAP-maintained GitHub sources were retrieved. They establish general platform behavior, not app-specific ECC support. SAP Help and Apps Reference Library retrieval failed DNS resolution in this environment; obtain release-specific evidence during project feasibility.

| ID | Source | Supports | Required local evidence |
|---|---|---|---|
| T1 | [SAP UI5 OData V2 Model](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md) | CSRF, ETags, metadata, batching, paging | Exact runtime APIs and request traces |
| T2 | [SAP ABAP CDS guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/15_CDS_View_Entities.md) | CDS on the ABAP standard database; feature references | Target compiler and database-feature checks |
| T3 | [SAP authorization guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md) | Explicit checks and CDS access-control distinction | ECC authorization objects, roles, negative cases |
| T4 | [SAP LUW guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/17_SAP_LUW.md) | Commit and transaction consistency concepts | Actual API and Gateway transaction contract |
| T5 | [SAP Help CDS feature tables](https://help.sap.com/docs/abap-cloud/abap-keyword/abap-cds-feature-tables) | Feature-verification destination, not a legacy baseline | Release-specific documentation |
| T6 | [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/) | App-specific implementation-information destination | Target product/version requirements and entitlement |
| T7 | [SAP UI5 Decimal API/source](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/type/Decimal.js) | String representation and precision/scale constraints | Installed-version type options and currency rules |

For each local claim, record owner, source version, observation date, representative document, expected result, actual result, and unresolved limitation. Mark a design unproven when evidence is missing. Architectural plausibility is not a substitute for a verified authorization, transaction, or business-semantic boundary.
