# Implementation steps for an ECC-backed Fiori capability

## Scope, prerequisites, and deliverables

This procedure implements a customer-owned Fiori application against ECC after the business and product decision has been made. It is not an installation recipe for an unsupported S/4HANA standard app. If a released ECC standard application meets the need, follow its implementation requirements instead of reconstructing it. Do not copy SAP proprietary application artifacts without verified rights and a supported deployment basis.

Read [S/4HANA-to-ECC guide](./s4hana-to-ecc-guide.md) for the approach decision, [Compatibility matrix](./compatibility-matrix.md) for target prerequisites, and [Technical challenges](./technical-challenges.md) for transaction and data semantics. These filenames match the [repository README](../README.md).

The example capability is an authorized purchase-order item worklist, initially read-only. A separately approved custom note operation illustrates an update contract without pretending that an invented BAPI can change standard purchasing documents. Any later change to an actual SAP purchase order requires its own verified target business API and functional review.

Prerequisites include an authorized development landscape, representative nonproduction data, a named functional owner, a Basis owner, security participation, an agreed transport route, and a supported UI5/Gateway topology. Deliverables are an evidence register, field contract, authorization design, service implementation, UI integration, acceptance evidence, and an operational handover.

## Step 1: Freeze one business slice

Describe the actor, organizational boundaries, document grain, filters, navigation target, and meaning of “overdue.” Decide whether dates come from a schedule line, an item summary, or another approved interpretation. Record handling of completed items, deletions, partial receipt, returns, and missing delivery dates.

Write acceptance statements that a functional consultant can verify independently of the new app. For example: an authorized buyer sees the same eligible item population as the approved reference process for a selected organization and date range. Another statement should cover a buyer who has no authorization for the requested organization.

Separate the optional note operation from business-document maintenance. A customer note does not change delivery dates, release status, account assignment, or procurement commitments. This separation makes the example transaction honest and prevents a harmless demonstration from being misrepresented as a standard business-object adapter.

**Gate 0:** the functional owner approves definitions, representative documents, excluded actions, and a fallback process. Stop when essential behavior has no target ECC equivalent.

## Step 2: Verify product eligibility

Search the SAP Fiori Apps Reference Library for the actual business task and target ERP product version. Capture the application ID, implementation information, required components, backend requirements, UI requirements, and applicable installation documentation. Do not infer support from a similarly named S/4HANA app. [I6]

Record the commercial and support decision. Standard content needs applicable entitlement and supported installation. A custom implementation needs independent ownership and permitted use of selected interfaces. An adaptation mechanism is not general permission to move an app to a different product.

Do not impose EHP7 as a universal Fiori prerequisite. Conversely, do not treat an enhancement-package label as sufficient. Each standard app has a distinct component and support-package baseline. A custom service has its own technical prerequisites, which still need verification.

**Gate 1:** choose released standard ECC, supported extension, custom implementation, or a redesigned process. If the choice is standard, switch to its version-specific implementation guide rather than continuing with unnecessary custom service development.

## Step 3: Capture the actual landscape

Record ERP version, enhancement package, relevant add-ons, SAP_BASIS, SAP_ABA, SAP_UI, Gateway components, support packages, database, Unicode status where relevant, and the actual UI5 resource version. Note development, quality, and production differences.

Draw browser, proxy, launchpad, front-end server, Gateway, system alias, RFC destination where used, ECC system, and client. Specify authentication termination and identity propagation. Verify that the intended named user, not an overprivileged shared identity, reaches business authorization checks.

Check ABAP syntax and CDS features against the target. CDS availability depends on release and constructs; HANA is not a universal prerequisite for ABAP CDS. Conversely, ECC on HANA does not install S/4HANA services or modern view-entity syntax. [I2]

**Evidence:** a version inventory, routing diagram, runtime decision, and list of environment-specific configuration. Do not embed development hosts or credentials in application source.

## Step 4: Specify the service contract

Define a customer-owned V2 service, for example `ZPO_WORKLIST_SRV`. This is a proposed name, not a delivered SAP service. Its read entity `PurchaseOrderItems` has stable string keys `PurchaseOrder` and `Item`, plus explicitly approved fields such as organization, delivery date, open quantity, unit, amount, and currency.

Define supported filters and orderings. Require an organization or another approved selective scope where needed for cost control. Define count semantics, maximum response size, navigation multiplicity, error categories, and whether offset pages represent a changing live dataset.

For an optional note entity, use the purchase-order/item identity plus a defined note identity if multiple notes exist. Include text length, author, change timestamp, and a server-generated revision. Client payloads must not override author, organizational ownership, or revision increments.

Create a field table with backend lineage, conversion, null handling, precision, scale, currency/unit reference, and authorization scope. Document what an amount means before exposing it. The same-looking property in an S/4HANA service is not automatically an equivalent fact.

## Step 5: Implement the authorized read

Choose approved target interfaces or data sources. A portable Open SQL provider can be appropriate; supported ABAP CDS can also be appropriate where its required features exist. Do not assume auto-exposure annotations or SADL behavior exists on every ECC release.

When using SEGW, define the model, generate runtime objects, and implement customer behavior in the supported extension classes rather than modifying generated base classes. Exact generated method signatures depend on the model and release. Keep selection, mapping, and policy logic in testable customer classes where local standards allow.

Parse query options through the supported Gateway interfaces. Allowlist property-to-field mappings; validate operators, types, lengths, and bounds. Return an explicit error for unsupported expressions. Use typed selection values, not concatenated SQL fragments.

Apply business authorization before paging or counting. Reuse the same predicate for the count and row population. A filtered-out row must not remain visible through totals or navigation.

```abap
AUTHORITY-CHECK OBJECT 'ZPO_SCOPE'
  ID 'ACTVT' FIELD '03'
  ID 'EKORG' FIELD lv_requested_ekorg.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_po_forbidden.
ENDIF.
```

This fragment assumes a customer-defined `ZPO_SCOPE` object and `zcx_po_forbidden` exception already exist. It illustrates one policy check, not complete purchasing authorization. Inspect the target standard business checks and organization semantics with security. Map the customer exception to the release-appropriate Gateway business exception. [I3]

For a keyed read, obtain the actual organization through a safe backend lookup and authorize against that persisted fact. Never let a request claim a document belongs to an organization merely to pass the check.

## Step 6: Implement truthful paging

Add unique keys as tie-breakers after approved user sorting. Push authorization predicates, business filters, ordering, and bounded selection down where supported. Do not select the entire history into an internal table for every page.

Validate `$top` and `$skip`, including zero, negative, malformed, and excessively large values. If enforcing a server page limit, provide a supported continuation contract or reject oversized requests. Silently returning fewer rows than requested without continuation can make clients believe the collection has ended.

For V2 inline counts, count the complete authorized filtered population, not the page length. For changing data, document that offset paging can duplicate or miss records between requests. If an audit-quality extract is needed, design it separately using an approved snapshot or extraction facility. The model documentation explicitly identifies the offset-paging risk. [I1]

**Gate 2:** demonstrate first, middle, and final pages, tied sort values, empty scope, unauthorized scope, and unsupported filters before integrating a sophisticated table control.

## Step 7: Isolate the write adapter

For an actual SAP document change, identify the real target interface and inspect documentation, authorization checks, locks, validation, return messages, enhancements, commit behavior, and update-task completion. Record its exact name and target version in the project evidence. No generic BAPI is asserted here to be a drop-in replacement.

For the illustrative customer note operation, define a customer table and customer enqueue object under local standards. Derive organization from the underlying purchase-order item. A note revision can be reliable if all permitted note writers use the same persistence boundary; that does not make it a valid revision for the entire purchase order.

The transaction design should follow this order:

```text
validate keys and allowed fields
resolve persisted business-object organization
check business and note-change authorization
acquire the defined note/business lock
reread current note and relevant business state
compare supplied revision against authoritative revision
validate text and current business eligibility
stage the note update and revision increment in one LUW
let the designated transaction coordinator finish the LUW
return the committed representation and new concurrency token
```

Do not use direct SQL updates for standard SAP purchasing tables. Direct persistence in this illustration applies only to the explicitly designed customer note store.

## Step 8: Prove LUW and concurrency behavior

Choose one transaction owner and verify it against the Gateway release and processing model. Default handlers, deferred changeset processing, and custom coordination have different responsibilities. Do not scatter `COMMIT WORK` or `BAPI_TRANSACTION_COMMIT` through entity methods.

For a multi-operation changeset, establish whether all participating APIs can share the required LUW. If an interface commits internally or uses remote work with incompatible transactional semantics, reject the atomicity claim. A UI batch group alone does not establish rollback capability. [I4]

Define ETag generation and comparison. Missing `If-Match` must be handled according to the documented service policy; stale versions must not overwrite current data. Compare the revision after locking and rereading. A purchase-order ETag must observe relevant external writers; the custom-note ETag only covers its own explicitly scoped data.

Simulate two users reading one note, the first saving, and the second saving the stale revision. Verify rejection of the stale update and preservation of the first result. Also verify lock contention, validation failure, and rollback when a later participating operation fails.

## Step 9: Register and route the service

For a conventional Gateway V2 deployment, use the target-supported service maintenance process, commonly `/IWFND/MAINT_SERVICE`, to add the approved service and select the correct system alias. Verify metadata and service document using an authorized account. Transaction names and options vary with installed components; follow local Basis procedures.

Check ICF activation for only the required service path through the approved administration process. Do not activate broad trees indiscriminately. For a hub topology, verify provider registration, alias routing, RFC trust or logon configuration, and backend service implementation.

Exercise the endpoint through the same proxy path the browser will use. A direct Gateway test can pass while the corporate route strips cookies, redirects to HTML, or points to the wrong ECC client. Confirm TLS, principal identity, language, and client context.

**Evidence:** sanitized metadata, service registration, alias mapping, successful authorized read, forbidden read, and a route diagram. Do not place session cookies, passwords, or token values in the evidence register.

## Step 10: Configure transport and roles

Assign backend artifacts to transportable packages and the correct route. Separate customer tables, lock objects, authorization objects, classes, Gateway model/provider artifacts, and necessary configuration according to the landscape's transport rules.

Create least-privilege service-start and business roles. Include the required launchpad content only for eligible users. Maintain authorization proposals and role generation through local standards. Test with business users, not only developers with broad access.

Keep aliases, destinations, identity trust, and credentials under Basis/security ownership. Verify which settings are transported and which require documented per-system configuration. Record import order and prerequisites so the UI cannot be exposed before its service and roles are ready.

**Gate 3:** the service works correctly under the intended identity through the production-like route, with complete authorized semantics and no dependence on developer privileges.

## Step 11: Create the UI5 integration

Select the runtime from the landscape inventory, not from whichever SDK is newest. For a V2 service use `sap.ui.model.odata.v2.ODataModel`. Keep the service URI relative to the approved application route.

```javascript
sap.ui.define([
  "sap/ui/model/odata/v2/ODataModel"
], function (ODataModel) {
  "use strict";
  return function createModel() {
    return new ODataModel(
      "/sap/opu/odata/sap/ZPO_WORKLIST_SRV/",
      { useBatch: true, defaultBindingMode: "OneWay" }
    );
  };
});
```

This example establishes a read-oriented model, not a complete application component. In an actual application, configure model lifecycle through the manifest or component consistently; do not instantiate competing default models.

Wait for metadata before operations that need key metadata. Handle metadata failure explicitly, including authentication redirects. The V2 model loads metadata asynchronously. Do not dereference metadata immediately after construction and assume it exists. [I1]

## Step 12: Bind filters, sorting, and decimals

Use typed UI5 filters and sorters instead of building query strings from user text. Keep keys and required amount/unit properties in any `$select` projection. Reset or submit edits before changing list filters when using two-way pending changes, in accordance with the model's behavior.

Use a decimal type for decimal model values and retain string representation. This example assumes metadata specifies precision fifteen and scale two; it is not a universal rule for all currencies:

```xml
<Input xmlns="sap.m"
  value="{
    path: 'Amount',
    type: 'sap.ui.model.odata.type.Decimal',
    constraints: {precision: 15, scale: 2}
  }" />
```

The backend remains responsible for authorization, scale, range, currency, and business validation. The SAP UI5 decimal type represents values as strings and exposes precision and scale constraints. Match options to the installed UI5 version rather than copying newly added API options. [I5]

Keep date-only display separate from event timestamps. Verify locale parsing with approved languages. Include leading-zero keys, long decimals, zero values, negative adjustments where permitted, and currencies with different decimal conventions in the manual acceptance dataset.

## Step 13: Add a scoped update adapter

The following customer adapter illustrates updating one existing note with an explicit ETag. It assumes the note entity and concurrency contract have already been implemented and verified. `loadNote` and error presentation belong to the application, not to a delivered SAP API.

```javascript
function updateNote(oModel, sPath, sText, sETag, onSaved, onFailed) {
  if (!sETag || sETag === "*") {
    throw new Error("Read the current note before saving.");
  }
  oModel.update(sPath, { Text: sText }, {
    groupId: "noteSave",
    eTag: sETag,
    success: onSaved,
    error: onFailed
  });
}
```

Construct `sPath` using `oModel.createKey` with metadata-defined keys after metadata is available. Derive the ETag from the actual read response or cached representation; do not invent it. Ensure `noteSave` is an immediate group for this example, meaning it has not been added to the deferred groups. Confirm the update method and payload semantics against the implemented provider.

The model handles CSRF tokens; do not set a hard-coded token. On an HTTP precondition failure, reread and let the user reconcile. On an uncertain network outcome, check the current note before retrying. For multi-operation batching, inspect inner operation responses as well as the outer batch result; adapt success handling to that operation structure. [I1]

## Step 14: Adapt the user experience

Decide whether freestyle controls or a Fiori elements template best fits the verified service. A generated page requires compatible annotations and template capabilities, not merely OData metadata. A V4 draft design cannot be preserved on a V2 service by renaming properties.

Implement loading, empty, forbidden, validation, stale-state, lock-conflict, and session-expiry states. Do not present all failures as “technical error.” Preserve meaningful business messages without exposing stack traces or sensitive backend details.

Configure semantic navigation to the approved existing application or transaction using target mappings. Validate parameters and preserve authorization at the destination. Do not hard-code SAP GUI transaction URLs as a universal cross-landscape solution.

Check keyboard operation, focus after errors, accessible labels, responsive layout, and locale-specific formatting. A visual resemblance to the S/4HANA app is less important than completing the approved task safely on the supported runtime.

## Step 15: Execute acceptance checks

Run the project's existing backend and UI validation practices; do not introduce a new test framework merely for this adaptation. The following is the required evidence plan, not a claim that these checks have been executed in this documentation repository.

| Area | Representative check | Acceptance evidence |
|---|---|---|
| Functional | Partial receipt, completion, return, deletion | Approved reference population and values |
| Authorization | Direct URL and foreign organization | No unauthorized rows, counts, or writes |
| Query | Combined filters, tied sorts, page boundaries | Stable contract-compliant results |
| Numeric | Large decimals and currency/unit combinations | No floating-point or conversion loss |
| Concurrency | Two writers and stale ETag | No lost update |
| LUW | Failure after a staged change | No partial commit within promised boundary |
| Security | CSRF rejection and session expiry | Protection preserved, safe recovery |
| Performance | Production-like volume and concurrent users | Agreed percentile and resource budget |
| Recovery | Timeout after possible commit | Reconciliation without blind duplication |
| UX | Keyboard, language, empty states | Business-user acceptance |

Inspect Gateway diagnostics using the landscape's approved tools, commonly the relevant front-end and backend error logs. Correlate failures with sanitized request identifiers. Limit tracing duration and data access; production payloads can contain commercially sensitive and personal information.

**Gate 4:** business, security, performance, and operations owners sign the evidence appropriate to their responsibilities. A successful administrator demo is not release acceptance.

## Step 16: Deploy in dependency order

Import backend data definitions and supporting objects before provider classes, service configuration, roles, UI content, and launchpad exposure, according to the actual dependency graph. Coordinate configuration that is not transported. Verify aliases and destinations in quality before production.

Deploy only customer-owned UI artifacts or properly entitled standard content through the supported target deployment mechanism. Confirm the deployed UI5 version and cache behavior. Clear or invalidate caches only through approved procedures when metadata or UI content changes require it.

Perform a limited smoke check with a representative business user: launch, selective read, navigation, approved note change if enabled, and logout/session behavior. Verify the deployed artifact and service contract versions match the release record.

Do not enable all users until monitoring and fallback are ready. Pilot scope can be restricted by roles and organizational assignments rather than hidden controller flags.

## Step 17: Handover and migration exit

Document application ownership, service ownership, dependency inventory, versioned contract, support contacts, and known limitations. Include actions for authorization failure, slow queries, lock conflict, stale ETag, token/session problems, and uncertain write completion.

Rollback means disabling new entry points and restoring the approved existing process, not deleting already committed business effects. Customer notes may need retention; standard business documents require authorized reversals where applicable. Document in-flight request reconciliation before removing service access.

For future S/4HANA migration, preserve capability acceptance criteria and replace the ECC adapter behind a reviewed service boundary. Reevaluate business semantics rather than assuming the custom contract is already an S/4HANA API. Set retirement criteria and an owner so a temporary bridge does not become an unmaintained integration.

## Evidence register and completion checklist

Record each claim with product version, source, owner, verification date, observed result, and limitation. Keep “designed,” “implemented,” “verified,” and “supported” as separate statuses.

- [ ] Exact target and app eligibility captured; rights decision recorded.
- [ ] Read population, amount semantics, and key lineage approved.
- [ ] Authorized paging, counts, navigation, and errors verified.
- [ ] Actual business API reviewed for any standard-document mutation.
- [ ] Customer note storage is not misrepresented as standard document editing.
- [ ] LUW, locks, ETags, CSRF, and retry behavior evidenced.
- [ ] UI runtime, annotations, roles, routes, and imports verified.
- [ ] Operational fallback and migration exit accepted.

## Sources and release-verification limits

Research date: 2026-10-17. SAP-maintained GitHub sources below were retrieved. SAP Help and the Apps Reference Library could not be fetched because DNS resolution failed; their links identify required release-verification destinations rather than evidence of a specific supported app.

- **I1:** [SAP UI5 OData V2 Model](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md): asynchronous metadata, CRUD, keys, paging, CSRF, ETags, and batch versus changeset behavior. Verify APIs against the deployed version.
- **I2:** [SAP ABAP CDS guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/15_CDS_View_Entities.md): standard-database modeling and links to feature tables; modern examples do not establish legacy syntax availability.
- **I3:** [SAP authorization checks](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md): explicit checks and SQL/CDS access-control distinctions. Local ECC role administration is separate.
- **I4:** [SAP LUW guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/17_SAP_LUW.md): commits, update tasks, and consistency; actual Gateway and business-interface coordination requires target verification.
- **I5:** [SAP UI5 Decimal implementation and API documentation](https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/type/Decimal.js): string representation, precision, and scale constraints. The main development branch is not the installed runtime baseline.
- **I6:** [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/): application-specific product requirements and implementation information; retrieval unavailable.
- **I7:** [SAP Help ABAP CDS feature tables](https://help.sap.com/docs/abap-cloud/abap-keyword/abap-cds-feature-tables): feature-verification entry point; retrieval unavailable and current cloud documentation is not an ECC support statement.
