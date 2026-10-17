# Community Projects: Evaluate Extensions Before Adoption

## Scope and prerequisites

Community projects can improve development feedback, integrate packages, and provide realistic browser testing. They also introduce additional code into a developer workstation, build pipeline, or browser. This guide curates verified projects by their actual responsibilities rather than presenting a list of supposedly complete business applications. No purchase-order repository is recommended without evidence of its existence, maintenance, and service contract.

The projects below are educational and engineering references. Their presence here does not make them production-ready, officially supported by SAP, or automatically compatible with SAP ECC. Even a well-maintained extension can be inappropriate for a regulated deployment or a particular UI5 release.

Prerequisites are familiarity with `package.json`, UI5 configuration, JavaScript modules, and basic browser requests. The inspection examples assume a source tree has already been downloaded. They do not install dependencies, clone repositories, run package lifecycle scripts, or contact SAP systems. JSON inspection requires an existing Node.js runtime; shell discovery commands require ordinary local tools.

Read [official-samples.md](./official-samples.md) for SAP reference applications and [learning-resources.md](./learning-resources.md) for learning order. Related chapters are [UI5 tooling](../10-TOOLS-DEPLOYMENT/ui5-tooling.md), [build process](../10-TOOLS-DEPLOYMENT/build-process.md), [security patterns](../05-SECURITY/security-patterns.md), and [testing strategy](../06-BEST-PRACTICES/testing-strategy.md).

## Evidence and support boundaries

The root READMEs for [ui5-ecosystem-showcase](https://github.com/ui5-community/ui5-ecosystem-showcase/blob/main/README.md) and [wdi5](https://github.com/ui5-community/wdi5/blob/main/README.md) were retrieved through GitHub on 2026-10-17. Package-specific proxy documentation was also inspected. These are substantive sources describing purpose, support channels, examples, and licensing, not inferred recommendations based on repository names.

The ecosystem showcase explicitly identifies itself as community-maintained and not officially supported by SAP. Individual maintainers may work for SAP, but that does not convert a community extension into a supported SAP product. The wdi5 README distinguishes community support from separately offered commercial support. Review any commercial agreement independently; a repository link does not define service levels.

Avoid popularity shortcuts. A package with many users can still have unsuitable defaults, weak compatibility documentation, or no maintainer for your version. More useful evidence includes relevant issue responses, documented release changes, automated checks, and an understandable implementation that your team can maintain if necessary.

## Decision map

| Need | Candidate | What it does not establish |
|---|---|---|
| Discover UI5 development extensions | Ecosystem showcase and its extension catalog | Suitability of every extension for your environment |
| Consume an npm library from UI5 modules | `ui5-tooling-modules` in the showcase | Compatibility of arbitrary browser packages |
| Forward local development requests | `ui5-middleware-simpleproxy` | Production identity propagation or authorization |
| Explore CAP/UI5 development integration | Showcase CAP-related packages | Deployment architecture or backend business correctness |
| Test UI5 interactions in a browser | wdi5 | Exhaustive service, security, or accessibility coverage |

Choose one candidate for one measurable problem. “Make development modern” is too vague; “remove manual page refresh while editing a local view” is testable. Record the baseline inconvenience, proposed extension, new risks, and a removal strategy before adoption.

## UI5 ecosystem showcase

### Purpose and layout

The [repository](https://github.com/ui5-community/ui5-ecosystem-showcase) combines independently published packages with showcase applications. Its README separates `packages/` from `showcases/`, making an important distinction between an extension's implementation and an application demonstrating consumption. A showcase root dependency graph may include many unrelated tools that your application does not need.

The catalog includes tasks, middleware, and broader tooling extensions. A task participates in build processing; middleware participates in request handling during serving. Some packages provide both. This distinction determines whether behavior appears only during local development, changes generated distribution files, or requires configuration in multiple phases.

Use the repository's [extension catalog](https://ui5-community.github.io/ui5-ecosystem-showcase/extensions/) and [selection guidance](https://ui5-community.github.io/ui5-ecosystem-showcase/selecting/) as navigation, then read the selected package's own README. Do not treat a catalog description as a substitute for version-specific options, compatibility notes, and examples.

### Runtime and OData considerations

These are primarily Node.js-side development tools, although their output can affect browser runtime code. Their existence does not select OData V2 or V4. A proxy can forward either protocol; the application's model class and the backend's metadata define the service contract.

Inspect the downloaded revision's package manager declaration, engine requirements, workspace configuration, and lockfile. A monorepo may require a different development setup from the published package consumed by a small application. Never generalize the repository root's prerequisites to every historical release of every package.

The root README describes compatibility with UI5 CLI extension specifications. Verify the exact package version and selected CLI version together. A newer specification number in an example is not an instruction to change an established application's toolchain without checking its other extensions.

### Read-only inventory

These commands inspect the already-downloaded repository:

```sh
cd ./ui5-ecosystem-showcase
git rev-parse HEAD
find packages -maxdepth 2 -name package.json -print
find showcases -name 'ui5*.yaml' -print
find packages -maxdepth 2 -name README.md -print
```

Choose one package directory from the inventory. Read its README and compare a consuming showcase's configuration with the extension implementation. Establish what inputs it reads, what resources it writes, and whether it opens network connections. This is especially important for tasks processing all application files or middleware accepting arbitrary requests.

## npm package consumption with ui5-tooling-modules

### Purpose and useful questions

The [package README](https://github.com/ui5-community/ui5-ecosystem-showcase/blob/main/packages/ui5-tooling-modules/README.md) describes middleware and a task allowing npm package names in UI5 module dependencies. It is useful when an application needs a browser-compatible library not distributed as a UI5 library.

Before selecting it, ask whether the dependency is genuinely needed, whether an existing platform API already solves the problem, and whether the package works in browsers. A Node-only package can assume filesystem access, native bindings, or process behavior unavailable in a browser. A transformation tool cannot make those assumptions disappear safely.

### Inspect before adapting

Locate the consuming JavaScript import or UI5 dependency declaration, the npm dependency declaration, and the tooling configuration. Then trace how the package is transformed and included in the built result. Development serving can mask missing build configuration: a package may load locally while its transformed resources are absent from the deployment artifact.

Compare development and build paths explicitly. Identify whether source maps reveal application source, whether output contains duplicated dependencies, and whether the chosen package brings additional license obligations. Check that the intended deployment's content security policy permits the generated execution pattern.

Do not copy a whole showcase dependency list. Select the package needed for the demonstrated behavior and document why each transitive runtime dependency is acceptable. Retaining unrelated plugins increases review effort and can make an apparently simple update unexpectedly disruptive.

## Local connectivity with ui5-middleware-simpleproxy

### Verified behavior

The [simpleproxy README](https://github.com/ui5-community/ui5-ecosystem-showcase/blob/main/packages/ui5-middleware-simpleproxy/README.md) describes proxy middleware and documents a target URI, SSL verification behavior, optional authentication inputs, header/query overrides, and request inclusion or exclusion controls. It notes that the mount path is not automatically incorporated into the configured base URI.

That path rule is a practical debugging clue. If the browser requests `/odata/Products`, inspect how the middleware combines the mounted route and target address. A duplicated or missing service segment can produce misleading 404 responses even though the backend itself is healthy.

### Security and protocol boundaries

A same-origin development proxy can avoid browser cross-origin restrictions; it does not grant backend permissions. Distinguish routing failure, authentication failure, authorization rejection, CSRF failure, and OData query incompatibility. They require different fixes and should not all be treated as “CORS problems.”

Keep TLS verification enabled. Documentation showing how to disable verification explains a capability, not a recommended baseline. Establish the correct trust chain for the approved backend instead of normalizing certificate bypasses. Avoid placing passwords in `ui5.yaml`, examples, screenshots, or Git history.

Environment configuration is convenient but still sensitive. A developer proxy may hold credentials, log request data, or access systems reachable only from the workstation. Bind it narrowly, use nonproduction data, and avoid publicly exposing it. Production routing needs an independently reviewed architecture for identity, session handling, destinations, and monitoring.

### A bounded investigation

Search the consuming application configuration rather than editing options experimentally:

```sh
grep -R -n --include='ui5*.yaml' \
  'simpleproxy\|mountPath\|baseUri\|strictSSL' showcases
```

Record the browser-facing path, target service root, authentication source, and expected metadata URL. For later authorized execution, verify both a successful read and an unauthorized request. Ensure failure responses remain visible; a proxy that hides backend errors can make troubleshooting and tests misleading.

## CAP-related development integration

The ecosystem README lists CAP integration packages including `cds-plugin-ui5` and `ui5-middleware-cap`. Their direction of integration differs: embedding UI5 serving in a CDS server is not the same arrangement as embedding CAP middleware in a UI5 development server. The repository's [backend connectivity comparison](https://ui5-community.github.io/ui5-ecosystem-showcase/backend-connectivity/) is the authoritative starting point for comparing these options.

Choose according to the development process you need to simplify. If the lesson is service annotations, embedding everything under one local origin may reduce setup distractions. If the lesson is production identity propagation, that simplification can hide exactly the boundary you need to understand.

Compare local configuration with the actual deployment topology. Separate processes, application routers, destination services, and backend identity checks may reappear after deployment. A working embedded example does not establish that a cloud router forwards the same paths or authentication context.

For an ECC-facing application, verify the real Gateway service and frontend constraints independently. A CAP development service is not a drop-in replacement for existing ECC transaction behavior, locking, or authorization. Keep the distinction explicit in architecture notes and acceptance criteria.

## wdi5

### Purpose and runtime

The [wdi5 README](https://github.com/ui5-community/wdi5/blob/main/README.md) describes a WebdriverIO service using UI5's test API for end-to-end testing of UI5 web applications. Consult the [project documentation](https://ui5-community.github.io/wdi5/) for version-specific setup and selectors. The project belongs in a test architecture discussion, not in a list of backend business services.

wdi5 does not itself choose OData V2 or V4. It interacts with the browser application; that application's model and backend determine the protocol. Test scenarios must still verify the behavior affected by the selected protocol, such as batching, draft flows, server messages, or stale-data handling.

### How to inspect existing tests

Find the repository's configuration and sample tests. Read how the application is started, how the base URL is supplied, what browser capabilities are declared, and how UI5 controls are selected. Determine whether a test uses stable IDs, control properties, or brittle rendered DOM details.

Prefer selectors tied to stable application semantics. Avoid using an entire generated DOM path merely because it currently locates the button. Dynamic IDs, responsive layout changes, translations, and framework updates can invalidate selectors without changing the user-visible behavior.

Review synchronization carefully. UI5-aware waiting can help with framework processing, but an external identity provider, file download, or backend asynchronous job may require its own observable completion condition. Fixed sleeps hide race conditions and extend execution time without proving readiness.

### What to test

A strong first scenario opens a list, selects an entity, performs a bounded action, and verifies a persisted result. Include one rejection case, such as insufficient authorization or invalid input. Keep data setup isolated and explain cleanup ownership. Do not let a shared demonstration account modify uncontrolled business data.

Separate assertions about UI state from assertions about server truth. A success toast can appear while a subsequent refresh exposes an error. Where appropriate, confirm the resulting state after reload or through an approved service-level check. Browser automation complements service tests rather than replacing them.

### Licensing and support

The README describes Apache 2.0 together with a derived Beer-ware alternative, identifying Apache 2.0 as the official license. Review the actual license files and choose a clear compliance basis with your organization. Separate browser driver, WebdriverIO, hosted browser service, and application runtime terms may also apply.

## Package inspection as a study artifact

Run the following from an already-downloaded ecosystem showcase root. It prints declared metadata without importing extension implementation or executing package scripts:

```sh
node - ./packages/ui5-middleware-simpleproxy/package.json <<'JS'
const fs = require("node:fs");
const p = JSON.parse(fs.readFileSync(process.argv[2], "utf8"));
console.log(JSON.stringify({
  name: p.name,
  version: p.version,
  license: p.license,
  engines: p.engines,
  peerDependencies: p.peerDependencies,
  scripts: p.scripts,
  dependencies: Object.keys(p.dependencies || {})
}, null, 2));
JS
```

Treat the output as declared evidence, not a security audit or proof of compatibility. Review scripts before any later installation, because package lifecycle execution is a separate trust decision. Inspect lockfiles and direct implementation dependencies to understand what code would actually be introduced.

## Maintenance and reproducibility

Record the repository commit, published package version if relevant, package path, CLI version, application runtime, and selected configuration. Link the exact README revision used. Distinguish “observed in source” from “executed in our environment.” This allows another engineer to reproduce the reasoning without repeating an uncontrolled setup.

Before upgrades, read release notes and examine changes affecting the configuration you use. Compare the built application as well as the development experience. A middleware fix may have no production effect, whereas a transformation task can change every deployed JavaScript module.

Define an internal owner and exit plan. If an extension becomes unsuitable, can standard tooling replace it? Can the team maintain a small fork within its license obligations? Does removing it require rewriting application imports? These questions are more useful than an unsupported promise that community maintenance will continue indefinitely.

## Adoption verification checklist

- [ ] The original repository and selected package README have been read.
- [ ] The problem solved is narrow and measurable.
- [ ] Runtime-side effects are distinguished from development and build behavior.
- [ ] CLI compatibility, engine requirements, and lockfile are recorded.
- [ ] OData behavior is verified in the application and backend, not inferred from middleware.
- [ ] TLS, credentials, request forwarding, and logging have been reviewed.
- [ ] Licenses cover copied source, dependencies, and generated artifacts.
- [ ] Existing tests and relevant negative cases are understood.
- [ ] No unperformed installation or execution is described as successful.
- [ ] Community support limitations, internal ownership, and removal strategy are explicit.
