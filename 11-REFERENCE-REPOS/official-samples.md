# Official SAP Samples: Selecting and Inspecting References

## Scope and prerequisites

This guide curates public source repositories for learning SAP Fiori, UI5, and CAP. It distinguishes application examples from framework implementation and development tooling. The examples are educational references, not production templates, contractual support commitments, or proof that an application works on SAP ECC. Before transferring a pattern, establish the target frontend runtime, backend service contract, authentication mechanism, and deployment platform.

You should understand JSON, JavaScript modules, HTTP, and basic OData concepts. Reading CDS annotations is useful for the CAP examples. Repository inspection requires only an already-downloaded source tree and ordinary text tools; the optional JSON examples require an existing Node.js installation. No dependency installation, repository cloning, deployment, or service access is necessary for the inspection exercises below.

Use [learning-resources.md](./learning-resources.md) for a sequenced learning path and [community-projects.md](./community-projects.md) when considering third-party tooling. Related knowledge-base chapters include [UI5 framework](../01-FOUNDATIONS/ui5-framework.md), [data binding](../02-ARCHITECTURE-PATTERNS/data-binding.md), [testing strategy](../06-BEST-PRACTICES/testing-strategy.md), and [compatibility matrix](../07-BACKPORTING/compatibility-matrix.md).

## Source verification and changing repository locations

The repository READMEs linked here were retrieved through GitHub during preparation on 2026-10-17. They establish project purpose, documented entry points, and important limitations. Main branches change; reproduce a study using the commit actually downloaded rather than assuming that today's README describes an older archive.

Two requested historical addresses deserve attention. [SAP/openui5](https://github.com/SAP/openui5) currently exposes OpenUI5 documentation identifying the UI5 organization. [SAP/ui5-tooling](https://github.com/SAP/ui5-tooling) resolves to documentation for UI5 CLI, whose canonical project links identify [UI5/cli](https://github.com/UI5/cli). Follow repository redirects, record the resolved location, and do not treat an organization move as a different framework or an automatic compatibility guarantee.

## Decision map

| Question you need to answer | Start here | Boundary to verify |
|---|---|---|
| How does a specific Fiori tools workflow work? | `SAP-samples/fiori-tools-samples` | Individual example README and V2/V4 category |
| Which annotations configure a V4 List Report feature? | `SAP-samples/fiori-elements-feature-showcase` | SAPUI5 version and feature prerequisites |
| How do UI, service model, and business actions connect? | `SAP-samples/cap-sflight` | Selected backend stack and application manifest |
| How does a UI5 control or binding implementation behave? | `SAP/openui5` | Public API versus internal implementation |
| How are dependencies served and application resources built? | `SAP/ui5-tooling` / `UI5/cli` | Stable release documentation versus development branch |

Start with the narrowest example that answers your question. A full travel application is unnecessarily complex for learning a single manifest setting. Conversely, a standalone annotation fragment does not explain transaction consistency or authorization across a complete request.

## SAP Fiori tools samples

### Purpose and runtime

The [repository README](https://github.com/SAP-samples/fiori-tools-samples/blob/main/README.md) describes a collection organized into `V2/`, `V4/`, `cap/`, third-party library examples, generator extensions, migration content, and miscellaneous tooling scenarios. Its [sample catalog](https://github.com/SAP-samples/fiori-tools-samples/blob/main/SAMPLES_INDEX.md) is the appropriate discovery entry point. This organization matters: there is no single root application, backend, or OData version that characterizes every example.

Choose a V2 example when studying a V2 service and its matching Fiori elements generation. Choose V4 when investigating V4-specific templates or extension APIs. A directory name is useful classification, but inspect the selected application's `manifest.json` and metadata before making a compatibility decision. Tooling can generate a client configuration; it cannot add unsupported query capabilities to an existing service.

### How to inspect one example

Read the example's README before the root package scripts. Identify whether it assumes a live SAP system, local mock data, CAP, or a particular destination. Then inspect its manifest, UI5 configuration, local annotations, and package scripts in that order. A sample involving an on-premise destination may require account and network configuration even when its UI files look self-contained.

For an already-downloaded tree, these commands list likely entry points without executing project code:

```sh
cd ./fiori-tools-samples
git rev-parse HEAD
find V2 V4 cap -name README.md -o -name manifest.json
find . -maxdepth 3 -name 'ui5*.yaml' -print
```

Select one path from the output instead of recursively studying everything. Write a short evidence note: “This example demonstrates local annotation editing against a V2 model; it does not establish deployment compatibility with our frontend server.” That statement prevents an attractive demonstration from becoming an unjustified architecture recommendation.

### Reuse and licensing

The root README identifies Apache 2.0 licensing with exceptions as noted in repository licensing material. Check the selected files, notices, embedded assets, and third-party dependencies before redistribution. Generated applications may reference SAPUI5 libraries whose availability and licensing are separate from the sample's source license. Preserve required notices and document substantive modifications rather than presenting borrowed examples as independently authored work.

## Fiori elements feature showcase

### What it demonstrates

The [feature showcase README](https://github.com/SAP-samples/fiori-elements-feature-showcase/blob/main/README.md) explicitly presents SAP Fiori elements for OData V4, particularly List Report and Object Page capabilities. It supplies topic-specific search terms for locating corresponding implementation snippets. This makes it a feature index rather than a recommended combination of every demonstrated option.

The README warns that some combinations are shown for demonstration even when they are not intended by the Fiori UX guidelines. Therefore, finding a configuration in this repository proves that it is illustrated, not that it fits your users' workflow. Evaluate table density, navigation, action placement, responsive behavior, and accessibility against the actual task.

### Runtime and service boundary

The inspected [package manifest](https://github.com/SAP-samples/fiori-elements-feature-showcase/blob/main/package.json) describes a CAP project and declares Apache 2.0 licensing. It separates development and production authentication/database settings. Treat those profiles as different operating contexts. Dummy development authentication must never be mistaken for a production authorization design.

The README describes a local CAP launchpad sandbox. That sandbox is a convenience for exploring applications; it does not demonstrate your organization's launchpad content deployment, identity trust, or role assignment. Likewise, a sample CDS annotation can be educational for an ABAP service without being directly transferable to its implementation language or metadata generation pipeline.

### A bounded annotation investigation

Suppose a business user wants dependent value help. Find the README's dependent-filtering topic and use its search term to locate the annotation. Trace the entity property, value-list target, parameter mapping, and any server implementation. Determine which parameter supplies context and which returns the selected value.

```sh
cd ./fiori-elements-feature-showcase
grep -R -n --include='*.cds' 'ValueList' app srv
grep -R -n --include='*.cds' 'SideEffects' app srv
find app -name manifest.json -print
```

Inspect the matching source, not just the first search result. Two similarly named fields can belong to different projections. A value help may appear correct with sample data while failing when the controlling field is empty or a previously selected dependent value becomes invalid.

A useful outcome is a four-part note: the annotation path, the service property it targets, the UI behavior expected, and the negative case to verify. Keep the example small enough to explain without the rest of the showcase.

## CAP SFlight

### Purpose and stack choices

The [CAP SFlight README](https://github.com/SAP-samples/cap-sflight/blob/main/README.md) describes a travel reference scenario built with CAP and Fiori elements. Its purposes include demonstrating annotations, comparing features across stacks, and running UI test suites. It documents Node.js and Java backend workflows; do not infer that both workflows produce identical artifacts or require the same toolchain.

The inspected [travel processor manifest](https://github.com/SAP-samples/cap-sflight/blob/main/app/travel_processor/webapp/manifest.json) declares OData `4.0` and Fiori elements List Report/Object Page targets. This is stronger protocol evidence than the word “CAP” alone. The manifest also links routing and navigation to travel and booking entities, which makes it useful for tracing multi-page interactions.

### Read the warnings as design evidence

The README acknowledges workarounds and simplified business calculations, including total-price calculation that ignores currencies. Do not adapt that calculation into a financial or procurement application. A realistic currency workflow needs explicit currency consistency, rounding policy, exchange-rate handling when applicable, and server-side validation.

It also documents a parser-related analytical limitation. Treat such warnings as properties of the inspected revision, not timeless statements about every CAP release. Confirm the exact runtime dependency and matching release documentation before relying on an analytical query feature or carrying an experimental flag into another project.

### Trace one action end to end

Choose one travel action and follow it from the UI annotation to the service operation and handler. Record whether it is bound to an entity, which parameters it accepts, and what the UI refreshes afterward. Then look for authorization, validation, and concurrency handling. The presence of an action button is not evidence that every caller is authorized to execute the operation.

Read the existing integration tests to understand stable selectors and intended user behavior. The README exposes an OPA test entry point and references `sap.fe.test`; those are learning leads, not evidence that tests were executed successfully in your environment. Distinguish a source-level walkthrough from a recorded runtime result.

### Deployment is a separate study

The README contains BTP deployment material, role collection guidance, managed application router context, and separate deployment configuration considerations. Study those only after understanding the local application. Cloud deployment introduces identity, entitlements, database services, routing, and operational ownership absent from a static code reading.

Configuration containing local service bindings or credentials must remain outside reusable documentation and commits. A sanitized explanation of required bindings is more useful than a copied environment file.

## OpenUI5

### Framework source, not a Fiori elements replacement

The [OpenUI5 README](https://github.com/SAP/openui5/blob/master/README.md) describes a JavaScript UI framework with controls and JSON, XML, and OData model support. It links the SDK, sample applications, public API reference, and contributor development documentation. Use it to study common UI5 fundamentals, control behavior, binding, and framework contribution.

OpenUI5 is not interchangeable with every SAPUI5 distribution. A Fiori elements sample can require libraries unavailable in OpenUI5. Compare the sample manifest's library dependencies with the target distribution before assuming a runtime substitution works. Also distinguish OData V2 and V4 model APIs; support for both protocols in a framework does not make their methods interchangeable.

### Inspection strategy

Start from the public SDK documentation for the API being used, then locate its implementation and tests in the downloaded source. A private helper can explain behavior but should not become your application dependency. Prefer a documented public extension point even when an internal call looks shorter.

The README identifies Apache 2.0 and separately calls out third-party modules and license information. Review `LICENSE.txt`, `THIRDPARTY.txt`, and applicable REUSE metadata. Framework source availability does not automatically license unrelated SAP products, trademarks, datasets, or backend services.

## UI5 Tooling and UI5 CLI

### What to learn here

The [tooling README](https://github.com/SAP/ui5-tooling/blob/main/README.md) describes a modular toolchain for dependency management, serving, building, and resource handling. Its current development branch distinguishes upcoming CLI development from the stable documentation stream. Follow the [stable CLI documentation](https://ui5.github.io/cli/stable/) when selecting a released toolchain; read development source only when investigating that development version.

Build tooling does not determine the OData version of an application. It may serve mock or proxied requests while the browser model and backend define the protocol contract. Similarly, producing an optimized resource bundle does not certify business authorization, backend correctness, or transport compatibility.

Inspect `package.json` engine requirements and the selected release documentation. Requirements on a development branch can be stricter than those of the version already used by an application. Avoid copying root maintainer scripts into an application project: maintaining the CLI monorepo and consuming the CLI are different activities.

## Read-only JSON inspection artifact

After selecting an application directory, inspect its manifest as data. This command reads JSON rather than requiring project modules or invoking package scripts:

```sh
node - ./app/travel_processor/webapp/manifest.json <<'JS'
const fs = require("node:fs");
const m = JSON.parse(fs.readFileSync(process.argv[2], "utf8"));
console.log(JSON.stringify({
  id: m["sap.app"]?.id,
  sources: m["sap.app"]?.dataSources,
  minimumUI5: m["sap.ui5"]?.dependencies?.minUI5Version,
  libraries: Object.keys(m["sap.ui5"]?.dependencies?.libs || {}),
  models: m["sap.ui5"]?.models
}, null, 2));
JS
```

Run it from an already-downloaded CAP SFlight root, or replace the argument with a discovered manifest path. The output establishes declared configuration, not actual backend reachability or the runtime loaded by a deployed shell. Compare it later with browser network evidence and target-system documentation when an authorized runtime exercise is available.

## Reproducibility and testing plan

Record repository URL, commit, selected application path, lockfile presence, manifest requirements, and the question investigated. Note whether evidence came from README text, implementation, tests, or an actual execution. An archive without Git metadata needs an independently recorded source revision and archive identity.

For later execution in a properly provisioned environment, choose the smallest existing test covering the adapted behavior. Annotation changes need checks against served metadata and UI behavior; action changes need service authorization and validation checks; routing changes need deep-link and browser-back checks. Add empty, unauthorized, stale, and invalid-data cases to the verification plan. Do not call a source reading a passing test.

For maintenance, compare upstream changes in the files actually reused. Track public API deprecations, security advisories, engine requirements, and lockfile changes. A working sample from an old revision is reproducible evidence, but it is not necessarily an appropriate baseline for new deployment.

## Adoption verification checklist

- [ ] The selected repository and application solve a specific, written question.
- [ ] Original README, source revision, and relevant implementation paths are recorded.
- [ ] OData protocol, model class, annotations, and backend capabilities agree.
- [ ] Required SAPUI5 libraries and supported target runtime are available.
- [ ] License, notices, third-party assets, and redistribution obligations are reviewed.
- [ ] Mock identities, simplified calculations, and demonstration workarounds are excluded from production assumptions.
- [ ] Existing tests and missing negative cases are identified without claiming unperformed execution.
- [ ] Backend authorization, concurrency, and validation have explicit owners.
- [ ] Deployment and ECC compatibility are assessed independently of the sample.
- [ ] The reusable outcome is a documented pattern with bounded assumptions, not an unexplained copy of the entire application.
