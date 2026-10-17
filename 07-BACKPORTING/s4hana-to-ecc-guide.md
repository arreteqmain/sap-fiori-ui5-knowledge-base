# S/4HANA-to-ECC Fiori feasibility and delivery guide

## Scope and reading order

This guide concerns delivering a Fiori user experience against SAP ERP Central Component, commonly called ECC, when the requested behavior was observed in an S/4HANA application. It does not describe converting S/4HANA into ECC, downgrading an installed product, or transferring SAP application ownership. The unit of planning is a business capability and its supported implementation, not a screenshot or a JavaScript component.

Start here for investment decisions. Use [Compatibility matrix](./compatibility-matrix.md) to record landscape evidence, [Technical challenges](./technical-challenges.md) to design semantic and transactional boundaries, and [Implementation steps](./implementation-steps.md) to execute the delivery gates. These filenames match the [repository README](../README.md). Examples throughout this collection are custom application designs, not reconstructed SAP application source or promises of standard-app compatibility.

The intended audience includes business owners, ECC functional consultants, ABAP and UI5 developers, security administrators, Basis teams, and support owners. Each must contribute evidence. A front-end proof alone cannot establish backend correctness; a backend API alone cannot establish entitlement, accessibility, or upgrade support.

## Non-negotiable corrections

ABAP CDS is not synonymous with HANA-only development. Availability depends on the ABAP platform release, support package, CDS artifact type, expression, and runtime feature. Some CDS implementations and database-specific pushdown techniques require HANA; this does not make every ABAP CDS view HANA-only. Modern documentation about CDS view entities must not be read as proof that the same syntax exists in an older ECC system. Consult the target release documentation and validate individual features. [S2, S3]

ECC does not require enhancement package 7 for every Fiori scenario. Delivered applications have their own product, add-on, Gateway, and support-package requirements. A custom UI5 client using an appropriately supported service has a different prerequisite set from a specific delivered ERP application. Conversely, a system being EHP7 or later does not automatically satisfy a particular app's prerequisites.

Do not classify ECC as capable of “only transactional” applications. Read-only worklists, operational summaries, charts, and analytical integration are possible where the required data, execution engine, authorization, and performance exist. Equally, not all analytics require HANA. Distinguish a bounded aggregate implemented over portable SQL from a particular HANA-dependent delivered analytical application.

Database choice is one compatibility dimension. Business content, ABAP features, installed software components, OData protocol, annotations, UI5 libraries, identity propagation, deployment topology, and licensing can independently make an approach infeasible.

Finally, do not download and redeploy SAP proprietary application artifacts merely because browser resources are technically accessible. Entitlement, software use rights, supported deployment, and extension rules require explicit verification. A standard S/4HANA app is not assumed supported on ECC. Prefer a released standard ECC app where suitable; otherwise consider an independently implemented custom application under the customer's applicable rights and governance.

## Define the business capability

Write a capability statement before searching for technical reuse. “Show a buyer the overdue purchase order items for authorized purchasing organizations, then navigate to an existing maintenance transaction” is implementable scope. “Backport the purchasing overview” is not: it hides approval logic, supplier semantics, analytics, notifications, and extension behavior behind one label.

For each task specify actor, business object, organizational scope, trigger, input, result, expected volume, and acceptable staleness. Define whether the user merely observes data, edits a document, executes a business transition, or completes a workflow task. Approval and document maintenance are not interchangeable operations. An ECC release strategy cannot be replaced with an unverified status-field update.

Capture representative documents and expected outcomes with the functional owner. Include cancelled, deleted, blocked, completed, partially processed, and archived cases. Agree whether balances include commitments, taxes, returns, and currency conversion. Record what “open” means on the target process. A polished interface with the wrong open-item definition is a failed implementation even when every HTTP request succeeds.

## Distinguish the four delivery approaches

| Approach | What is delivered | Entry condition | Primary owner |
|---|---|---|---|
| Released ECC standard app | SAP-delivered content for the selected ERP product version | Exact application implementation requirements and entitlement are satisfied | Product and Basis owners |
| Supported standard extension | Approved changes through documented extension mechanisms | Base app is supported on the actual target and extension points cover the need | Application owner |
| Custom ECC application | Independently designed UI and customer-owned service contract | Required ECC process exists and interfaces can be governed | Customer development owner |
| Side-by-side experience | UI or orchestration outside ECC with controlled ECC integration | Identity, connectivity, consistency, operations, and commercial terms are acceptable | Integration owner |

“Backport” is an informal umbrella, not a fifth support model. A custom implementation can resemble a Fiori interaction without being a supported port of the original SAP app. Record that distinction in architecture decisions, procurement documents, and help-desk runbooks.

An adaptation project changes an eligible base application through supported mechanisms; it does not supply missing ERP business content. SAP's Fiori tools documentation explicitly directs developers to the app's implementation and extensibility information. The cited page concerns particular cloud environments, not a general ECC extension entitlement. [S6]

## Apply the decision tree

```text
Is the requested ECC business process present and approved?
  No -> change scope, retain the existing process, or migrate the capability.
  Yes -> is an appropriate standard ECC app released for the target version?
    Yes -> verify implementation requirements, license, and extension needs.
      All satisfied -> configure standard; extend only within supported rules.
      Not satisfied -> remediate prerequisites or evaluate custom delivery.
    No -> can an independently built service implement the required semantics?
      No -> stop or redesign the business requirement.
      Yes -> can authorization, LUW, performance, and operations be proven?
        No -> narrow scope or choose another platform/process.
        Yes -> compare custom ECC and side-by-side lifecycle costs.
```

Run this tree for each capability, not once for an entire launchpad. A read-only search might proceed while an approval operation remains blocked. Do not allow a successful read demonstration to bypass the write feasibility gate. Every “yes” must link to an evidence item; undocumented confidence is an unresolved risk, not approval.

## Establish landscape facts

Record the full ERP product version and enhancement package, installed business add-ons, activated business functions, SAP_BASIS and SAP_ABA releases, relevant support packages, SAP_GWFND or applicable Gateway components, SAP_UI, and deployed application components. Record the database and its supported version separately. Component combinations matter more than a friendly landscape name such as “ECC on HANA.”

Map browser-to-backend routing: reverse proxy, authentication endpoint, launchpad host, UI5 resource host, Gateway front end, system alias, RFC destination if applicable, and ECC client. Decide whether Gateway is embedded or deployed as a hub. Verify where business authorization runs and which identity reaches ECC. A hub with a powerful fixed technical user can erase the intended per-user authorization boundary.

Record the exact UI5 runtime used by the deployed launchpad. A project manifest minimum does not install that runtime. Libraries available from a public documentation site are not necessarily installed, licensed, or supported on the corporate front-end server. Identify browser and accessibility requirements as landscape inputs, including corporate devices that cannot accept the latest browser.

## Build a dependency inventory

Inspect the requested capability through documented interfaces and implementation information rather than copying the source application. List entity sets, keys, navigation properties, value helps, action contracts, annotation vocabularies, draft expectations, attachments, workflow integration, intent navigation, and analytical requests. Capture observations only from systems the team is authorized to inspect.

Separate technical and semantic dependencies. A missing `CompanyCode` property might be a simple projection gap; a missing universal-journal accounting dimension is a business-model gap. A UI expecting a draft administrative entity requires a draft lifecycle, not just an extra entity set. A chart expecting grouped and filtered measures requires an aggregation service, not a list of raw documents.

Tag each dependency as available unchanged, available with configuration, adaptable through a custom contract, replaceable with altered UX, or blocking. Assign an owner and a proof method. “Adaptable” requires a credible mapping and a test, not a suggestion that ABAP can probably calculate the field.

## Perform a capability feasibility matrix

| Capability example | S/4HANA baseline concern | ECC feasibility question | Preferred initial decision |
|---|---|---|---|
| Purchase order worklist | Released CDS and service entities | Are status and authorization semantics reproducible from approved ECC sources? | Standard ECC app first; bounded custom read otherwise |
| Document change | Business object validation and save behavior | Is a suitable interface present with documented update and locking behavior? | Prototype one complete transaction |
| Release approval | Workflow and approval model | Does the target approval process expose the required task semantics? | Preserve target process; do not imitate an unrelated action |
| Embedded analytical KPI | Analytical engine and semantic measures | Can the same measure be computed with acceptable meaning and cost? | Redesign or use an established reporting service |
| Journal-line exploration | Universal journal and dimensions | Do equivalent facts exist without lossy joins or false lineage? | Reject direct entity substitution |
| Supplier overview | Business partner relationships | Can vendor and partner mappings be defined for this task? | Explicit adapter with stable identifiers |
| Attachments | Repository and object linkage | Are document types, access checks, retention, and scanning available? | Separate integration workstream |
| Draft editing | Draft engine and generated services | Does target infrastructure support the required draft contract? | Avoid claiming draft support; redesign save flow |

Use this matrix as a discussion template, not as a SAP release certificate. The [compatibility matrix](./compatibility-matrix.md) adds infrastructure and protocol dimensions. A capability can be technically feasible but economically irrational if its backend semantics must be re-created and supported for only a short remaining ECC lifetime.

## Design the target service boundary

Favor a narrow customer-owned OData contract whose names describe the ECC business meaning. For example, `PurchaseOrderItems` can expose document number, item, purchasing organization, delivery date, open quantity, unit, amount, and currency with documented definitions. Avoid naming the service like an S/4HANA API merely to satisfy an existing UI. Similar names can conceal different authorization and state behavior.

Choose a read implementation based on the target release and approved sources: portable Open SQL, supported ABAP CDS where available, an existing released interface, or an established reporting layer. Do not assume every direct table read is forbidden or every CDS entity is a released API. Review the stability and maintenance obligations of each selected dependency.

For writes, use the target business object's documented interface where available and appropriate. Validate the actual function module, class, API, enhancement behavior, and update-task contract in the target. There is no generic purchase-order or approval BAPI that is automatically a drop-in substitute for every S/4HANA operation. Never update SAP business tables directly to mimic a save button.

## Preserve data semantics

Define canonical identifiers without destroying target distinctions. Document numbers with leading zeroes remain strings. Composite keys must contain every necessary business dimension. Client handling belongs to the trusted backend context, not a freely supplied browser field. Map vendor and business-partner identifiers through an approved relationship, not a numeric formatting assumption.

Carry currency and unit with amounts and quantities. Define precision, scale, conversion dates, rounding policy, and whether an amount is document, local, or group currency. OData V2 decimal values should remain decimal strings through the JavaScript boundary; coercing them to floating-point numbers can change financial values. The backend validates scale and business rules independently.

Define date-only fields separately from instants. ECC business dates do not become local midnight timestamps merely because a browser uses `Date`. Specify time zones for actual events, blank values, historical dates, and inclusive interval boundaries. These rules belong in the contract and acceptance cases, not hidden in controller formatters.

## Choose UI technology deliberately

A freestyle SAPUI5 app is often a reasonable choice when the target service is narrow or its behavior does not match a generated page contract. Fiori elements can reduce custom UI work where the exact runtime, OData protocol, templates, and annotation support meet the need. Neither option is inherently more compatible with ECC; their dependency surfaces differ.

For OData V2, use `sap.ui.model.odata.v2.ODataModel`, not the deprecated original model. SAP's UI5 documentation explains server-side requests, metadata loading, batching, CSRF token handling, and ETag behavior. Those features reduce client plumbing but do not implement missing backend authorization or transactions. [S1]

Do not translate an OData V4 application by changing only the service URL. V4 actions, binding APIs, draft conventions, query behavior, and annotations differ from V2. Select a target protocol early. If a UI must be rebuilt, preserve the approved task flow rather than reproducing unsupported controls or template internals.

## Establish security boundaries

Authentication identifies the principal; launchpad visibility is navigation configuration; service-start authorization permits endpoint use; business authorization limits actual data and actions. Treat them as separate controls. A hidden tile does not secure an entity set. A successful service call does not establish permission to read every purchasing organization.

Apply authorization before paging, counting, aggregation, and expansion. Otherwise counts and summaries can disclose records that ordinary rows hide. Recheck permissions for writes using the persisted object's organization and current state; never rely on an organization submitted by the client. Explicit ABAP checks and CDS access control have different application points. Selecting raw tables does not automatically reproduce CDS access control. [S4]

Use TLS and approved identity propagation. Preserve CSRF protection and session cookies for writes; do not disable protection to make an adapter work. Restrict logs to necessary technical evidence, redact tokens, and avoid storing complete business payloads without a retention and access policy.

## Establish transaction boundaries

Document what one user action commits. One save can contain multiple technical changes, but it needs one accountable transaction coordinator. A Gateway changeset is not proof that every called interface can participate atomically. Hidden commits, remote calls, update tasks, and external side effects can break the intended boundary. [S5]

For a custom write, authorize, acquire the correct application lock, reread current state, compare the concurrency version, validate, invoke the vetted interface, and let the designated transaction owner complete the LUW. Never insert `COMMIT WORK` into each item handler as a universal fix. Verify the exact Gateway release and change-processing implementation before relying on multi-operation changesets.

An ETag is an optimistic concurrency token, not a replacement for business locking. Its generation must observe relevant changes from SAP GUI and background processing, not only writes from the new app. If the object has no dependable version source, narrow the scope or choose another concurrency design; do not advertise protection using a timestamp updated only by the custom service.

## Establish performance boundaries

Define expected users, request frequency, maximum result size, response-time percentiles, expensive filters, and representative production data volume. Separate first render from backend query time and write completion time. A HANA database can improve some workloads but does not repair an unbounded join, missing authorization predicate, or chatty client.

Require server-side filtering, stable sorting, and bounded paging. Add a unique tie-breaker to user sort criteria. Apply the identical authorized predicate to rows and counts. Decide whether changing datasets permit best-effort offset paging or require a supported continuation/snapshot mechanism. The UI5 V2 documentation explicitly warns that intervening changes can make `$skip` and `$top` pages inconsistent. [S1]

Avoid selecting every document into ABAP and then trimming a page. Likewise, avoid aggregate totals calculated from just the visible page. A genuine analytical total must specify the full filtered population, currency grouping, unit treatment, and freshness. If that cannot meet the response budget, use a reporting service or redefine the requirement.

## Compare lifecycle economics

Estimate standard activation, prerequisite upgrades, custom service development, UX work, authorization design, functional validation, operations, and future migration. Include regression after support packages and dependence on unreleased internals. A screen that takes two weeks to draw can still require months of semantic validation.

Prefer limited, replaceable adapters over a broad imitation of the S/4HANA virtual data model. Put ECC-specific mappings behind customer-owned interfaces; keep UI controllers unaware of table names and backend release checks. This reduces later migration effort without pretending that ECC and S/4HANA already share the same business model.

Time-box the feasibility spike around the riskiest capability. If approval concurrency is the major uncertainty, do not spend the spike polishing a read-only dashboard. Define a stop condition such as absence of a supportable update interface, inability to preserve authorization, or unacceptable query cost at representative volume.

## Use explicit migration gates

| Gate | Required evidence | Stop condition |
|---|---|---|
| G0: Scope | Approved task and ECC process mapping | Essential target process is absent |
| G1: Eligibility | Version inventory, app requirements, entitlement decision | Unsupported standard deployment is the only proposal |
| G2: Contract | Field lineage, protocol, authorization and concurrency design | Semantics or isolation cannot be established |
| G3: Backend proof | Representative reads and one complete write where applicable | Hidden commits, leaks, or wrong totals remain |
| G4: UX integration | Runtime-compatible controls, navigation, error paths | Unsupported runtime assumptions remain |
| G5: Release readiness | Functional, security, volume, recovery, and operational evidence | No safe rollback or ownership |
| G6: Transition | Migration mapping and retirement criteria | Temporary adapter has become an unowned permanent dependency |

Gate approval is conditional on the recorded landscape. A front-end server upgrade, new enhancement, changed workflow, or database migration can invalidate earlier evidence. Reopen only affected decisions, but do not silently assume earlier test results cover changed behavior.

## Plan deployment and rollback

Transport backend dependencies before service activation and UI exposure. Separate workbench artifacts, landscape-specific aliases, authorization roles, launchpad content, and business configuration according to local change management. Avoid hard-coded system IDs and credentials. A manifest should use an approved relative route rather than an ECC development hostname.

Rollback must address data effects as well as UI removal. Disabling a tile does not reverse posted documents or stop queued work. Prefer a documented fallback to the existing supported business transaction, with reconciliation for in-flight operations. Never promise rollback of already committed business events unless the domain provides an authorized reversal process.

Operations need correlation between browser-visible errors, Gateway diagnostics, application logs, and business documents. Define who handles authorization failures, stale ETags, update-task failures, slow queries, and expired sessions. Make retry guidance operation-specific: a failed read is often safe to repeat; an uncertain write requires status reconciliation first.

## Worked decision: procurement worklist

Suppose buyers request an S/4HANA-like overdue-item experience against ECC. First ask whether the appropriate released ECC application covers the workflow. If it does, compare prerequisites and extension limits before funding custom code. If it does not, define a custom read-only service containing only approved fields and navigation to the existing maintenance process.

The functional owner defines overdue using the target schedule and completion semantics. The ABAP owner identifies stable sources and tests partial receipts, returns, deletion indicators, and delivery completion. The security owner defines purchasing-organization visibility. The service rejects unsupported filter expressions instead of silently broadening the result. The UI displays amounts with currency and quantities with unit.

A subsequent request to change delivery dates is a new feasibility slice. It needs a vetted target interface, document locking, revision checks, validation, and recovery evidence. It is not enabled merely because the original worklist had an editable date column. Approval remains outside scope unless its distinct ECC release process is mapped and proven.

## Pitfalls and corrective decisions

| Pitfall | Why it fails | Corrective decision |
|---|---|---|
| Copy the S/4HANA UI and replace its URL | Service and business contracts differ; rights may prohibit reuse | Select standard ECC or independently build the approved task |
| Require HANA for every CDS definition | Conflates ABAP language infrastructure with database-specific execution | Check exact CDS features and target documentation |
| Use EHP level as the sole app criterion | Ignores components, support packages, and app version | Capture complete implementation requirements |
| Filter only in the browser | Leaks data and corrupts paging and totals | Enforce authorized predicates on the server |
| Accept any positive HTTP response as save success | Batch envelopes and asynchronous processing can hide failures | Inspect operation results and reconcile committed state |
| Retry an uncertain write automatically | May duplicate a business event | Use supported idempotency or business-status reconciliation |
| Treat custom rebuild as standard support | Misleads operations and commercial planning | Declare customer ownership and dependency risks |

## Evidence register and approval checklist

Maintain one register row per claim with claim text, source title, product version, retrieval date, local verification, owner, and decision. Example: “The worklist supports descending delivery-date sorting” needs a contract reference plus target results for duplicate dates and tied keys. “The app is supported on ECC” needs the actual app implementation record and installation requirements, not this guide.

- [ ] Business owner approved precise read and write semantics.
- [ ] Basis recorded component, database, topology, and runtime facts.
- [ ] Product owner selected standard, supported extension, custom, or side-by-side delivery.
- [ ] Commercial owner verified applicable use rights and support boundaries.
- [ ] Backend owner mapped every field and identified the real update interface.
- [ ] Security owner verified direct URLs, counts, expansions, and actions.
- [ ] Concurrency, LUW, paging, decimals, and error behavior have evidence.
- [ ] Operations accepted monitoring, reconciliation, fallback, and retirement.

## Sources and verification limits

Research date: 2026-10-17. SAP-maintained GitHub documentation was retrieved directly. SAP Help and the Apps Reference Library URLs could not be fetched in this environment because DNS resolution failed; they are authoritative release-verification destinations, not inspected proof of any particular app's compatibility. No application-specific release claim is made here.

- **S1 — SAP UI5, OData V2 Model:** [SAP-maintained documentation source](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/odata-v2-model-6c47b2b.md). Retrieved; supports model behavior, asynchronous metadata, paging caveats, CSRF, batching, and ETags. Match APIs against the deployed UI5 version.
- **S2 — SAP ABAP CDS overview:** [SAP-maintained CDS guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/15_CDS_View_Entities.md). Retrieved; describes CDS over the ABAP standard database. Its modern examples are not an ECC feature baseline.
- **S3 — Release-specific ABAP CDS:** [ABAP 7.40 documentation entry](https://help.sap.com/doc/abapdocu_740_index_htm/7.40/en-US/abencds.htm) and [current feature tables](https://help.sap.com/docs/abap-cloud/abap-keyword/abap-cds-feature-tables). Help retrieval unavailable; verify exact release and feature locally.
- **S4 — SAP authorization checks:** [SAP-maintained guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md). Retrieved; supports explicit checks, SQL limitations, and CDS access-control distinction, not ECC role-maintenance instructions.
- **S5 — SAP LUW:** [SAP-maintained transaction guide](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/17_SAP_LUW.md). Retrieved; supports transaction boundaries and commit pitfalls. Gateway-specific behavior still requires target documentation.
- **S6 — SAP Fiori tools extensibility:** [SAP-maintained prerequisites](https://github.com/SAP-docs/btp-fiori-tools/blob/main/docs/extending-an-sap-fiori-application-for-sap-s-4hana-cloud-public-edition-and-sap-btp-abap-f4881a9.md). Retrieved through GitHub search; points to application-specific extensibility information.
- **S7 — SAP Fiori Apps Reference Library:** [implementation-information entry point](https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/). Retrieval unavailable; obtain the exact target product/application record during G1.
