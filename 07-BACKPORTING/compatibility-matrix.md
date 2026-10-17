# Compatibility matrix for ECC-backed Fiori capabilities

## Scope and interpretation

This matrix supports an evidence-based decision about an ECC-backed Fiori application inspired by S/4HANA behavior. It is not a SAP product availability matrix, installation certification, or statement that a standard S/4HANA app is supported on ECC. Each delivered application must be checked against its exact target product version, implementation information, installation requirements, and applicable entitlement.

Use [S/4HANA-to-ECC guide](./s4hana-to-ecc-guide.md) for approach selection, [Technical challenges](./technical-challenges.md) for semantic and transaction design, and [Implementation steps](./implementation-steps.md) for execution. These relative filenames match the [repository README](../README.md).

**Legend:** “Conditional” means a capability may be feasible after release, feature, content, and performance verification. “Not implied” means the named landscape alone establishes nothing about that dependency. “Specific prerequisite” means the individual application or execution mechanism sets the requirement. No cell should be interpreted as blanket standard-app support.

## Landscape capability matrix

| Dimension | S/4HANA on HANA | ECC on HANA | ECC on anyDB |
|---|---|---|---|
| Business data model | S/4HANA model for the installed release | ECC model; HANA does not transform it | ECC model |
| Standard application eligibility | Exact S/4HANA app/version requirements | Exact ERP/ECC app/version requirements | Exact ERP/ECC app/version requirements |
| SAPUI5 client | Runtime and library dependent | Runtime and library dependent | Runtime and library dependent |
| OData V2 service | Component and implementation dependent | Component and implementation dependent | Component and implementation dependent |
| OData V4 service | Release and exposure-model dependent | Not implied; verify target stack | Not implied; verify target stack |
| ABAP CDS | Artifact and feature dependent | ABAP release and feature dependent | ABAP release and feature dependent; not categorically excluded |
| S/4HANA delivered CDS content | Installed product/version dependent | Not supplied by database choice | Not supplied by database choice |
| HANA SQLScript or calculation-view dependency | HANA present; content and rights still required | HANA present; content and ABAP integration still required | Cannot execute that HANA-specific dependency unchanged |
| Portable SQL aggregation | Conditional on semantics and cost | Conditional on semantics and cost | Conditional on semantics and cost |
| Specific embedded analytical app | App and engine prerequisites | No general equivalence | No general equivalence |
| Draft lifecycle | Service/framework dependent | Not implied | Not implied |
| Business mutations | Released/approved interface and process dependent | Target ECC interface and process dependent | Target ECC interface and process dependent |
| Customer-owned UI/service | Feasible only within its verified contract | Feasible only within its verified contract | Feasible only within its verified contract |

The table intentionally avoids “all supported” and “only transactional” labels. A read-only chart over approved ECC data may be practical on anyDB. A delivered application that depends on a specific HANA engine is different. Neither observation determines support for an unrelated app.

ECC on HANA can remove a particular database restriction without resolving missing ABAP syntax, Gateway functionality, workflow content, or S/4HANA business semantics. Likewise, an anyDB deployment may satisfy a carefully bounded portable-query design while failing its response budget at production volume. Compatibility and capacity both require evidence.

## ABAP and CDS dependency matrix

| Dependency | Evidence to collect | Acceptable alternative | Incorrect inference to avoid |
|---|---|---|---|
| ABAP language syntax | Target SAP_BASIS/ABAP release and compiler result | Equivalent older supported implementation | Modern SAP example compiles everywhere |
| DDIC-based CDS view | Target CDS feature documentation and activation | Approved Open SQL/provider model | Every CDS construct needs HANA |
| CDS view entity | Target support for this artifact type | Older supported model with explicitly mapped behavior | Older CDS availability implies view entities |
| CDS expressions and functions | Individual feature availability and database support | Approved ABAP calculation or simpler expression | One working view proves all features |
| CDS access control | Actual DCL behavior at the consumption boundary | Explicit reviewed authorization predicates | Raw-table reads inherit CDS restrictions |
| Automatic service exposure | Exact exposure mechanism and installed components | Explicit Gateway provider | An annotation automatically supplies an app |
| SADL integration | Release-specific feature and operation support | Handwritten provider for the bounded contract | All SADL versions support the same operations |
| AMDP/database procedures | ABAP integration plus supported database/runtime | Portable implementation where semantically equivalent | HANA installation supplies the required code |
| Released API status | Interface release and use contract for target product | Approved customer dependency with risk ownership | Accessible object means stable public API |

SAP's CDS guide describes modeling on the ABAP standard database and points to feature tables. Its contemporary view-entity examples do not establish a historical ECC baseline. [C2] Use the target system's release-specific documentation; do not replace feature analysis with a database-only rule.

Do not publish a universal minimum EHP for Fiori. Enhancement packages, technical components, application add-ons, business functions, and support packages interact. A delivered application can have narrower requirements than another application bearing the same business-area label. A customer UI may use a supported service without sharing the requirements of either delivered application.

## Gateway and protocol dependency matrix

| Requirement | What to verify | Failure consequence | Delivery decision |
|---|---|---|---|
| Gateway V2 runtime | Installed components, support package, provider registration | No usable service endpoint | Install/configure only through supported planning |
| Hub or embedded topology | System aliases, routing, provider location | Wrong system/client or failed execution | Assign Basis ownership and prove route |
| Identity propagation | Named user and trusted connection behavior | Excessive access or failed authorization | Block release until principal is correct |
| `$filter` | Supported operators, types, and validation | Wrong or overly broad population | Narrow contract or implement missing behavior |
| `$orderby` | Supported fields and unique tie-breakers | Unstable pages | Specify deterministic order |
| Paging/count | Authorized population, limits, continuation | Data leaks or incomplete lists | Prove server contract before UI scale-up |
| `$expand` | Relationship authorization and bounded cost | Leaks or explosive payloads | Limit or replace with explicit navigation |
| V2 changesets | Provider processing and participating API LUWs | Partial commit despite apparent batch | Reject unsupported atomicity claims |
| ETags | Authoritative version across relevant writers | Lost updates | Block editing without a valid design |
| CSRF | Token handling, cookies, proxy cache behavior | Rejected writes or weakened protection | Preserve framework security |
| V4 semantics | Actual service and client model support | Broken actions, bindings, or drafts | Deliberate redesign, not URL replacement |

The UI5 V2 model supports batching, ETag use, and CSRF handling, but these client features do not implement backend atomicity or business authorization. [C1] Check the exact Gateway documentation and provider behavior on the target. A service document that returns HTTP 200 is only one small part of this matrix.

## UI5 and annotation dependency matrix

| UI concern | Dependency evidence | Safe response to a gap |
|---|---|---|
| Core runtime | Deployed version, maintenance status, browser support | Select a supported runtime through landscape governance |
| Library availability | Installed and entitled SAPUI5 libraries | Replace control or adjust scope; do not assume OpenUI5 parity |
| V2 model | Required API availability in deployed version | Use supported V2 APIs |
| V4 model | Protocol and binding behavior match backend | Rebuild against V2 or provide a genuine supported V4 service |
| Fiori elements template | Template/version and service contract | Use a compatible template or freestyle UI |
| UI annotations | Vocabulary, term, qualifier, and consumer support | Supply supported local annotations or redesign |
| Value help | Annotation plus search/filter service semantics | Implement bounded explicit selection |
| Draft | Full draft state, locking, activation, cleanup | Use explicit nondraft edit/save flow |
| Analytics | Measure metadata and aggregation service | Use approved custom reporting contract |
| Launchpad navigation | Intent, target mapping, parameters, roles | Define supported navigation targets |
| Personalization | Supported persistence and schema lifecycle | Constrain personalization to stable fields |
| Decimal binding | String representation and correct type constraints | Preserve decimal strings; avoid floating-point coercion |

Metadata and annotations have different roles. Metadata describes types and relationships; annotations can guide UI behavior, value help, and interpretation. Local annotations cannot manufacture a missing backend capability. Advertising an action or draft support that the server does not implement creates a misleading client.

Check annotations at their intended target and qualifier. A term ignored by the installed runtime might remove a mandatory interaction rather than merely alter a label. Demonstrate the actual generated page before approving a Fiori elements approach.

The SAP UI5 decimal type documents string representation and precision/scale constraints. [C5] Apply the metadata contract rather than a universal scale-two assumption. Currency-specific display and posting rules remain business responsibilities even when the type validates a decimal string correctly.

## Business capability dependency matrix

| Capability | Required semantic evidence | Typical target option | Blocking condition |
|---|---|---|---|
| Purchase-order worklist | Item grain, schedules, completion, receipts, authorization | Standard ECC app or bounded custom read | No agreed definition of eligibility |
| Document maintenance | Validated target API, locks, messages, LUW | Approved ECC business interface | Only direct standard-table updates proposed |
| Approval | Target release/workflow semantics and task authorization | Existing ECC process integration | Unrelated status toggle masquerades as approval |
| Financial exploration | Ledger, line-item, currency, reconciliation rules | Established reporting or explicit custom scope | Universal-journal equivalence is assumed |
| Supplier overview | Vendor/partner relationships and stable identifiers | Explicit task-specific mapping | Keys are inferred from formatting alone |
| KPI/chart | Grain, aggregation, units, currency, freshness | Portable aggregate or reporting integration | Totals cannot be reconciled |
| Attachments | Repository linkage, retention, scanning, access checks | Approved document integration | Download URL bypasses object authorization |
| Search | Authorized fields, supported matching, cost | Bounded target search | Full-text behavior assumed from UI alone |

A custom rebuild is justified only when the target process can be implemented honestly and maintained. The lack of a matching standard app does not itself prove a custom implementation is feasible. It may reveal that the requested business capability belongs to a different product model.

## Approach decision matrix

| Situation | Preferred approach | Evidence required before commitment |
|---|---|---|
| Released ECC app satisfies task and prerequisites | Configure standard content | Exact app record, installed components, entitlement |
| Supported base app needs a documented extension | Use approved extension points | Extensibility documentation for that app/version |
| No standard fit; ECC facts and interfaces are adequate | Independently build a custom UI/service | Contract, authorization, LUW, performance, ownership |
| Integration spans systems and has acceptable consistency model | Evaluate side-by-side delivery | Identity, connectivity, retries, operations, rights |
| Required facts or process do not exist on ECC | Redefine scope or migrate capability | Business decision, not a cosmetic prototype |
| Proposal depends on copied proprietary S/4HANA artifacts | Stop pending rights and support review | Explicit permitted use and supported deployment basis |

SAP's adaptation-project documentation directs developers to application-specific extensibility information. [C6] Its cloud-specific prerequisites cannot be generalized into ECC eligibility. Treat standard extension and independent custom development as different ownership and support arrangements.

When scoring alternatives, use separate business, technical, support, and cost columns. A high score for visual reuse must not compensate for a failed authorization or licensing gate. Mark those as hard constraints, not weighted preferences.

## Release verification worksheet

For every selected capability, complete these fields:

| Evidence field | Required content |
|---|---|
| Capability and owner | Precise user task and accountable business owner |
| Target baseline | Product, EHP, components, support packages, database, runtime |
| Standard-app record | App ID, selected product/version, requirements, retrieval date |
| Data contract | Keys, lineage, units, currencies, dates, freshness |
| Interface contract | Actual read/write APIs, release status, LUW behavior |
| Security | Principal, organization checks, counts/expands, negative evidence |
| Concurrency | Lock object or interface, version source, stale-write policy |
| Performance | Data volume, selectivity, page size, percentile budget |
| Commercial/support | Entitlement decision and custom ownership boundaries |
| Operational exit | Fallback, reconciliation, monitoring, retirement owner |

Use the statuses **unknown**, **documented**, **configured**, **verified**, and **rejected**. “Documented” is not “verified.” A source may establish a feature exists while local configuration or a custom provider still behaves incorrectly.

Store release-specific evidence with its date. Current documentation can change independently of an old ECC installation. Reopen relevant rows after changes to support packages, enhancements, identity routing, UI5 runtime, or database execution.

### Example: machine-readable capability evidence

The following JSON is an illustrative evidence-record structure, not an observed SAP landscape or a supported component combination. Replace every illustrative value with the actual system inventory. A populated version field does not establish eligibility: verification remains false until an accountable owner attaches the required evidence.

```json
{
  "recordId": "EXAMPLE-PO-WORKLIST-001",
  "illustrativeOnly": true,
  "capability": {
    "name": "Authorized overdue purchase-order item worklist",
    "deliveryApproach": "customer-owned custom application",
    "mutationsIncluded": false
  },
  "target": {
    "product": "ECC — illustrative, exact product version not recorded",
    "database": "anyDB — illustrative, actual platform not recorded",
    "softwareComponents": [
      {
        "name": "SAP_BASIS",
        "release": "NOT_CAPTURED",
        "supportPackage": "NOT_CAPTURED"
      },
      {
        "name": "SAP_GWFND",
        "release": "NOT_CAPTURED",
        "supportPackage": "NOT_CAPTURED"
      }
    ],
    "ui5": {
      "version": "1.120.0",
      "versionIsIllustrative": true,
      "deployedRuntimeConfirmed": false
    }
  },
  "service": {
    "name": "ZPO_WORKLIST_SRV",
    "protocol": "OData V2",
    "entitySet": "PurchaseOrderItems",
    "authorizedCountRequired": true,
    "stablePagingRequired": true
  },
  "verification": {
    "status": "unknown",
    "verified": false,
    "owner": "ASSIGN_BACKEND_AND_BASIS_OWNERS",
    "evidenceReferences": [],
    "blockingItems": [
      "Capture exact installed component versions",
      "Confirm deployed UI5 runtime and maintenance eligibility",
      "Verify row and count authorization with a restricted user",
      "Verify paging and query cost at representative volume"
    ]
  }
}
```

The illustrative UI5 version is neither a recommendation nor a minimum ECC prerequisite. Likewise, listing `SAP_GWFND` is not an assertion that every legacy Gateway installation uses that component arrangement; record the applicable installed Gateway components. Transition `verified` to true only after the stated capability has evidence, and retain separate app-eligibility and entitlement records for any standard content.

## Migration gates and manual checks

At feasibility, require all hard constraints to be documented and unresolved dependencies explicitly costed. At backend proof, require authoritative field lineage, authorized queries, actual write-interface behavior, and transaction evidence. At UI integration, require runtime-compatible APIs, annotations, navigation, and errors.

Before release, verify counts and expansions cannot expose unauthorized data; paging is deterministic within its stated consistency model; decimal strings and dates survive round trips; CSRF remains enabled; ETags observe relevant writers; and batch errors cannot be reported as success.

- [ ] No cell has been converted into a universal SAP support claim.
- [ ] No universal HANA requirement has been attached to ABAP CDS.
- [ ] No universal EHP7 requirement has been attached to all Fiori.
- [ ] ECC analytics are evaluated by semantics and engine, not excluded categorically.
- [ ] Standard S/4HANA backport support is not presumed.
- [ ] Rights and supported deployment are explicit.
- [ ] Every conditional requirement has an owner and a local proof.
- [ ] Migration and retirement decisions preserve business meaning.

If a hard requirement remains unknown, the release decision is blocked even if a demonstration works. Record the reason and the narrowest remediation. An unsupported annotation may justify a smaller UX change; an unprovable financial measure may require abandoning the capability.

## Evidence register and sources

Research date: 2026-10-17. Retrieved SAP-maintained GitHub sources support general technical distinctions. SAP Help and the Apps Reference Library could not be fetched because DNS resolution failed; no specific standard-app release or installation prerequisite is claimed to have been verified.

- **C1 — UI5 service behavior:** [SAP OData V2 Model documentation](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md), retrieved. Supports client metadata, batching, paging, ETag, and CSRF behavior. Check the deployed UI5 version separately.
- **C2 — CDS model:** [SAP ABAP CDS guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/15_CDS_View_Entities.md), retrieved. Describes the ABAP standard-database model and feature references. Modern examples are not an ECC compatibility certificate.
- **C3 — Authorization:** [SAP authorization checks](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md), retrieved. Supports explicit checks and the distinction between SQL and CDS access control.
- **C4 — Transaction model:** [SAP LUW guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/17_SAP_LUW.md), retrieved. Supports commit and consistency analysis, not a universal Gateway changeset recipe.
- **C5 — Decimal type:** [SAP UI5 Decimal API/source](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/type/Decimal.js), retrieved. Confirms string representation and constraints; development-branch options require release verification.
- **C6 — Extensibility:** [SAP Fiori tools adaptation prerequisites](https://github.com/SAP-docs/btp-fiori-tools/blob/main/docs/extending-an-sap-fiori-application-for-sap-s-4hana-cloud-public-edition-and-sap-btp-abap-f4881a9.md), retrieved. Illustrates application-specific extensibility checks in its stated environments.
- **C7 — App eligibility:** [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/), retrieval unavailable. Obtain exact product/version implementation information before approving standard content.
- **C8 — Release features:** [SAP Help ABAP CDS feature tables](https://help.sap.com/docs/abap-cloud/abap-keyword/abap-cds-feature-tables), retrieval unavailable. Follow release-specific references rather than treating current cloud documentation as a legacy baseline.
