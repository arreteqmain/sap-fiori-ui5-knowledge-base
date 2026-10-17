# UI5 Tooling: Architecture, Extensions, and Operational Boundaries

## Scope and reading route

This guide explains how UI5 CLI turns a UI5 project into a development server or a reproducible static build. It assumes a JavaScript application with `webapp/Component.js`, `webapp/manifest.json`, a committed npm lockfile, and a local CLI dependency already approved by the organization. Examples are documentation, not files installed by this repository. Adapt namespaces deliberately rather than copying a sample namespace into production.

Start at the [knowledge-base README](../README.md), continue with the [build process](build-process.md), and use the [pipeline guide](cicd-pipeline.md) and [deployment targets](deployment-targets.md) when an artifact must leave the workstation. These are separate concerns: UI5 CLI prepares and serves resources; a deployment tool publishes them to a particular platform.

SAP Business Application Studio is an IDE and development environment. Its terminal can run these tools, and its generators can produce deployment descriptors, but BAS is not the deployment runtime. Likewise, successful `ui5 serve` output does not establish that authentication, routing, launchpad integration, or backend connectivity will work on a deployed target.

## Version policy: verified facts rather than “latest”

The reproducible compatibility reference here is **UI5 CLI 4.0.0**. Its tagged, official `package.json` declares Node.js **`^20.11.0 || >=22.0.0`** and npm **`>=8`**. That is an engine declaration for that release, not a recommendation to run an obsolete Node patch or every subsequently released major. Select a currently maintained Node release that also satisfies every locked extension's engine constraints. Node 22 is a concrete major within that declared CLI range. Record the exact approved patch in the build image and release evidence. [S1]

The official current development documentation has a different prerequisite range: Node **22.22.3 or higher within major 22**, **24.15.0 or higher within major 24**, or **26.0.0 and higher**, excluding majors 23 and 25; npm remains at least 8. This moving development documentation must not silently redefine a CLI 4 project's contract. Check the package metadata of the exact released CLI selected by your project. Do not adopt a development branch merely because a documentation site points at it. [S2]

The historical CLI package above was checked against the GitHub advisory database while preparing this guide; no direct advisory was returned. That is not a transitive dependency audit, an assurance about every extension, or a reason to downgrade an existing patched lockfile. No installation command below introduces a new npm package version.

There are at least four distinct version axes:

* The CLI release determines available commands and extension APIs.
* `specVersion` selects a configuration and extension contract.
* `framework.version` selects the UI5 framework used locally.
* The deployed shell or application router determines the framework actually loaded by users.

A CLI upgrade does not upgrade SAPUI5 in an ABAP system. Changing `specVersion` does not change a browser's JavaScript support. Keep these decisions independently visible in architecture reviews.

## The architecture in a practical mental model

The CLI coordinates project discovery, virtual resources, build tasks, and the development server. The project layer reads configuration and dependency information to construct a graph. The filesystem layer presents project resources through readers and writers rather than requiring every task to manipulate physical files. The builder selects tasks from project type and configuration. The server exposes the resource graph through an HTTP middleware chain. [S3]

This separation explains several otherwise surprising failures. An npm dependency can be installed yet absent from the UI5 resource graph. A file can exist physically but be mapped to the wrong virtual path. A task can generate the correct resource in memory while a later task excludes it. A middleware can fix a development request without changing a single byte in the deployed artifact.

For an application, `webapp` maps to the application's served root. Libraries and components generally use `src` mapped beneath `/resources`, with tests beneath `/test-resources`. UI5 module identifiers are namespace-oriented resource paths, not arbitrary filesystem imports. `acme/orders/controller/List.controller` needs matching casing throughout configuration, resources, and module declarations. Linux CI often reveals a case mismatch concealed by a developer's filesystem. [S3]

Treat the graph as an input to the build rather than a discovery convenience. If a reuse library is required at runtime, declare it explicitly. If a framework library is only needed for a local launchpad sandbox, classify that use accurately instead of shipping an accidental second framework copy. Framework dependencies and ordinary npm packages have different resolution responsibilities.

## Establishing a predictable project

The following is a configuration fragment for an existing application using the CLI 4 configuration contract:

```yaml
specVersion: "4.0"
metadata:
  name: acme.orders
type: application
resources:
  configuration:
    paths:
      webapp: webapp
builder:
  resources:
    excludes:
      - "test/**"
      - "localService/**"
  cachebuster:
    signatureType: hash
```

It intentionally omits `framework`: retain the exact, approved framework block already selected for the application. Do not replace an installed SAPUI5 runtime requirement with an arbitrary sample version. The complete project also needs its package metadata, lockfile, manifest, component, and any required framework libraries. The configuration schema documents path mappings, exclusions, framework declarations, and hash-based cachebuster signatures. [S3]

Use the local binary after dependency restoration:

```sh
node --version
npm --version
npm ci
./node_modules/.bin/ui5 --version
./node_modules/.bin/ui5 --help
./node_modules/.bin/ui5 serve
```

These commands assume dependencies already exist in reviewed package files. `npm ci` restores the lockfile; it is not an instruction to add CLI packages opportunistically. A local binary avoids an unexpected global installation. The official getting-started documentation also explains that a global CLI attempts to delegate to a local one, but an explicit local path makes CI intent easier to inspect. [S2]

Never use a floating framework alias as part of release preparation. Framework availability, licensing, backend compatibility, and extension requirements should be reviewed before the lockfile or framework configuration changes. A network proxy that makes registry access possible is infrastructure configuration, not a reason to disable TLS verification.

## Development middleware: request handling, not deployment

Custom middleware participates in the development server's request chain. It is appropriate for local response headers, a controlled backend proxy, or test-only endpoint behavior. It is not a substitute for production authentication, destination service configuration, or an application router. Middleware configuration in a dependency does not automatically alter the root project's server. Order and mount path are explicit design decisions. [S4]

The following complete multi-document YAML demonstrates an in-project extension without an additional npm dependency:

```yaml
specVersion: "4.0"
type: application
metadata:
  name: acme.orders
server:
  customMiddleware:
    - name: acme-dev-health
      beforeMiddleware: serveResources
      mountPath: /__dev
---
specVersion: "4.0"
kind: extension
type: server-middleware
metadata:
  name: acme-dev-health
middleware:
  path: tooling/dev-health.cjs
```

The referenced implementation would be:

```js
module.exports = function () {
  return function (req, res, next) {
    if (req.method !== "GET" || req.url !== "/health") {
      next();
      return;
    }
    res.statusCode = 200;
    res.setHeader("Content-Type", "application/json");
    res.setHeader("Cache-Control", "no-store");
    res.end(JSON.stringify({ mode: "development" }));
  };
};
```

Because the middleware is mounted at `/__dev`, the handled URL is `/__dev/health` from the browser's perspective; the mounted handler sees the remaining path. Unknown methods and paths continue down the chain. The extension name in `customMiddleware` must exactly match the extension metadata, and the implementation path is relative to the project. The `.cjs` suffix makes CommonJS intent unambiguous even when package metadata uses ECMAScript modules.

This endpoint deliberately exposes no environment variables, filesystem paths, process arguments, backend tokens, or version inventory. Health endpoints tend to grow into accidental information leaks when developers return configuration objects for convenience. A real local proxy also needs a fixed upstream allowlist, deliberate cookie handling, timeouts, and protection against forwarding arbitrary user-selected URLs.

Place a request-transforming middleware before the middleware whose behavior it changes. Place a passive observer where it can observe the desired response lifecycle. An ordering anchor establishes position, not authorization. If two custom extensions depend on each other, define the referenced extension before the referencing entry, as the official extensibility documentation requires. [S4]

For asynchronous work, handle promise rejections and call `next(error)` where appropriate. Do not call `next()` after completing a response. Avoid broad catch blocks that convert every upstream failure into HTTP 200, because they create tests that pass while authentication or service routing is broken.

## Custom tasks: transforming the build resource graph

A custom build task operates on build resources, not HTTP requests. Use it for deterministic generation, validation, transformations, or target packaging that cannot be expressed through standard tasks. Keep publication out of the ordinary build path whenever possible: a developer should be able to build without holding deployment credentials or mutating a remote system. [S5]

This example validates application-owned descriptor data before minification:

```yaml
specVersion: "4.0"
type: application
metadata:
  name: acme.orders
builder:
  customTasks:
    - name: acme-check-descriptor
      beforeTask: minify
---
specVersion: "4.0"
kind: extension
type: task
metadata:
  name: acme-check-descriptor
task:
  path: tooling/check-descriptor.cjs
```

```js
module.exports = async function ({ workspace, log }) {
  const manifests = await workspace.byGlob("**/manifest.json");
  if (manifests.length !== 1) {
    throw new Error("Expected exactly one application manifest");
  }
  const descriptor = JSON.parse(await manifests[0].getString());
  if (descriptor["sap.app"]?.id !== "acme.orders") {
    throw new Error("Unexpected application identity");
  }
  log.info("Application identity verified");
};
```

The single-manifest restriction is an intentional application policy, not a general rule for libraries or multi-component projects. JSON parsing errors fail the build naturally. Because this task only reads root project resources, it does not ask the builder to build dependencies unnecessarily.

For extensions using specification version 3.0 and later, the absence of `determineRequiredDependencies` means that dependency resources are not requested and the `dependencies` parameter can be omitted. Tasks that genuinely inspect libraries must export that callback and return the required dependency names. This is important both for correctness and scheduling: asking for every dependency creates avoidable work; assuming the reader always exists can crash an otherwise valid build. [S5]

When generating resources, use the supported resource factory exposed through the task utility and write into `workspace`. Do not write directly into `dist` during task execution: the output directory is a publication destination for the builder, not the authoritative workspace. A later write phase or clean operation could overwrite such physical files.

Transformation order is semantic. A TypeScript transformation belongs before minification and preload creation, with valid input source maps. A metadata validation task can run early. A checksum manifest belongs after all byte-changing operations. A deployment task belongs in an explicit deployment configuration or, preferably, an independent deployment command. Referencing a standard task that is disabled can still establish an ordering position; it does not enable that task. [S5]

## Choosing and governing plugins

“Plugin” can mean a UI5 extension, an npm package, an IDE extension, or a Cloud Foundry CLI plugin. State which one you mean in change requests. UI5 build and server extensions execute code with the user's permissions, so an innocuous-sounding task name does not create a security boundary.

Before approving a package, identify its maintainer, license, release cadence, supported CLI specification, Node requirements, filesystem access, network behavior, and transitive dependencies. Review whether install scripts execute. Pin the selected release through the lockfile, and evaluate advisories for the exact versions being introduced. A registry download count is not evidence that the extension is compatible with your build graph.

Prefer a small in-project extension when its behavior is both narrow and easy to own, such as the descriptor assertion above. Prefer an established package for complex protocols or transformations rather than maintaining a private implementation of authentication or module conversion. Both choices require ownership: a local extension without tests can be riskier than a well-maintained external package.

Do not turn package installation into a production deployment step. Approved build images should contain platform CLIs and the expected trust store; npm dependencies should come from the project's reviewed lockfile. Separate credentials for package restoration from credentials for deployment, and scope registry access to the specific package feed.

## Debugging failures by layer

If the CLI cannot discover a project, inspect YAML parsing, `specVersion`, project type, and metadata first. Do not start changing runtime module paths before the project graph exists. If a module is missing during development, compare physical casing, virtual resource paths, component namespace, and manifest library dependencies. If development works but the build fails, inspect exclusions and custom-task ordering.

If a generated file appears in `dist` but is never requested, inspect preload membership and the runtime URL base. If a request works only through local proxy middleware, reproduce the deployed routing design rather than adding a second local workaround. A successful local proxy proves connectivity from the developer environment, not connectivity from a managed application router.

Keep diagnostics proportionate. Verbose CLI output can reveal paths and configuration, while deployment debug output can expose request headers. Capture a normal log first, reproduce with a minimal configuration, and sanitize any expanded logs before attaching them to tickets. Never paste a complete environment dump into an issue.

When isolating an extension, temporarily compare a clean build without that extension in a disposable working copy. Record exactly which behavior disappears. Do not disable every standard task simultaneously, because that destroys the ability to identify the failing transformation. Preserve the original artifact so source-map and preload differences can be compared.

## Keeping development and production behavior comparable

Maintain a short inventory of development conveniences: proxy rewrites, mocked users, injected headers, local framework delivery, and sandbox launchpad services. For each convenience, identify the production component that supplies the equivalent behavior or explicitly record that no equivalent is intended. This inventory is especially valuable when a team changes hosting models.

For example, a developer proxy can remove a path prefix while forwarding requests. If the deployed application router preserves that prefix, the same manifest service URI reaches a different endpoint. Resolve that discrepancy in routing design rather than hardcoding a second backend URL in controller code. Similarly, local anonymous access must never become evidence that the deployed destination should use an unauthenticated technical route.

## Planning an upgrade without losing the baseline

Upgrade the CLI and its extensions in a dedicated change rather than folding the upgrade into an urgent feature delivery. Capture the current artifact inventory, startup network trace, representative source-map breakpoint, and runtime framework selection first. Those observations become the comparison baseline, especially when a new builder changes output ordering or descriptor enhancement.

Read the selected release's package engines and migration notes together. An engine range answers whether Node can run the package; migration guidance answers whether existing configuration and extensions still behave correctly. Update the lockfile through the approved package workflow, inspect dependency changes, and verify each extension against the new API contract. A successful installation alone proves very little.

Compare meaningful behavior, not only filenames. Generated bundles can differ while remaining functionally equivalent, whereas identical file counts can conceal a missing translation or broken lazy-loaded fragment. Exercise the optimized application, development middleware, and any custom task that reads dependencies. Retain the previous lockfile and build image identity so an upgrade can be reversed without reconstructing an old global tool installation.

## Operational verification checklist

Before approving a tooling change, a reviewer should be able to answer all of the following without reading a developer's private IDE settings:

* Is the CLI locally resolved, and are the exact Node and npm versions recorded?
* Is the configuration specification supported by the selected release?
* Are the framework version and deployed runtime relationship documented?
* Does each custom extension have an owner and a narrow purpose?
* Are middleware-only behaviors absent from production assumptions?
* Do custom tasks request only the dependencies they actually read?
* Are generated resources written through the build workspace?
* Are transforms ordered before minification, maps, and bundles that depend on them?
* Can the normal build run without remote deployment credentials?
* Are registry access, backend access, and deployment identities separated?
* Does the deployed artifact pass the checks in [build-process.md](build-process.md)?

Finish by recording the expected output, not merely the successful command. For example, state that the descriptor identity was validated, the component preload exists, no local mock data was packaged, and the target runtime can resolve the chosen libraries. Those observations make a later incident diagnosable even after the original workstation is gone.

## Official sources and research notes

Sources were consulted on 2026-10-17. Where a documentation hostname could not be reached, the corresponding official GitHub source was read instead. Moving `main` documentation is explanatory material; the tagged package below is the compatibility evidence for CLI 4.

* **[S1]** SAP, [UI5 CLI 4.0.0 package metadata](https://github.com/SAP/ui5-cli/blob/v4.0.0/package.json): exact Node/npm engines and CLI identity.
* **[S2]** UI5, [Getting Started source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/GettingStarted.md): current development prerequisites, local/global behavior, and build/server separation.
* **[S3]** UI5, [Configuration source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/Configuration.md): project types, framework configuration, path mapping, exclusions, and cachebuster settings.
* **[S4]** UI5, [Custom Server Middleware source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/extensibility/CustomServerMiddleware.md): extension declarations, mounting, ordering, and API contracts.
* **[S5]** UI5, [Custom Tasks source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/extensibility/CustomTasks.md): workspace access, task ordering, dependency declarations, and extension trust.
