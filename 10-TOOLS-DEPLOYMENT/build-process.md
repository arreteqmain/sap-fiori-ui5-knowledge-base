# Building UI5 Applications as Verifiable Release Artifacts

## Scope, assumptions, and boundaries

This guide covers a UI5 application build, including preloads, minification, source maps, exclusions, cache behavior, performance decisions, and packaging. It assumes an existing project with a reviewed local UI5 CLI dependency, a committed lockfile, and working application tests. It does not prescribe a framework upgrade or install additional build tools.

The compatibility baseline is the tagged **UI5 CLI 4.0.0** package, which declares Node **`^20.11.0 || >=22.0.0`** and npm **`>=8`**. Use a maintained, organizationally approved Node patch within a compatible major, and verify every extension's requirements. This historical engine declaration is not a claim that an old Node patch is secure today. The exact package was advisory-checked during preparation; no direct advisory was returned, but the complete lockfile still requires assessment. [B1]

Read [UI5 tooling](ui5-tooling.md) for project architecture and extension contracts. The [pipeline guide](cicd-pipeline.md) describes how to retain build evidence, while [deployment targets](deployment-targets.md) explains target-specific publication. Return to the [README](../README.md) for the broader knowledge-base structure.

A build is a transformation from declared source inputs to an inspectable artifact. Deployment is a separate transition that changes a remote system. Keeping those transitions separate permits offline inspection, repeatable testing, and promotion of already-tested bytes rather than a fresh rebuild disguised as a release.

## Establish the input contract first

Identify every input before optimizing anything: application sources, manifest, framework version, reuse libraries, npm lockfile, UI5 configuration, custom task implementation, environment-specific nonsecret configuration, and toolchain image. If a build reads an untracked local file, it is not reproducible from the repository alone.

Do not conflate a reproducible dependency installation with a bit-for-bit reproducible archive. `npm ci` restores a dependency graph, but archive timestamps, generated version metadata, framework cache contents, locale ordering, and custom tasks can still change output. Define the required reproducibility level explicitly. Many teams first require identical application resource contents, then separately normalize packaging metadata.

Record the source commit, CLI version, Node version, npm version, framework version, lockfile digest, and build command. If a private registry or mirror is involved, record its logical identity without exposing credentials. A release record should explain whether framework resources are included in the artifact or supplied by the target.

Avoid arbitrary environment substitution into source JavaScript. Destination names and relative service paths generally belong in target configuration or routing descriptors. Secrets never belong in an application artifact: anything delivered to a browser must be considered readable by the user, regardless of minification or bundling.

## A controlled initial build

The following commands assume the reviewed package files already contain the CLI and any required extensions:

```sh
npm ci
node --version
npm --version
./node_modules/.bin/ui5 --version
./node_modules/.bin/ui5 build --clean-dest
```

`--clean-dest` prevents files left by an older build from contaminating the output. Use a dedicated output directory rather than pointing the builder at a shared working directory containing evidence or credentials. A successful build should produce the application's deployable resources beneath `dist`, but the exact contents depend on project type and configuration.

For an application whose runtime supplies SAPUI5, an application-only build is usually a more appropriate starting point than shipping the entire framework. If the deployment explicitly owns framework delivery, `ui5 build --all --clean-dest` includes dependency builds. Decide based on the hosting contract, not on which command produces the largest directory. The official builder documentation describes type-dependent tasks and build modes. [B2]

Do not use self-contained mode reflexively. Its standalone bundle and bootstrap transformation solve a different delivery problem than component preloads loaded by a launchpad. A standalone page with an owned runtime can benefit; a launchpad-hosted application can instead duplicate resources and violate the shell's runtime expectations.

## What standard tasks actually accomplish

The builder selects tasks based on project type. Applications normally receive minification, descriptor enhancement, component preload generation, and other resource processing. Libraries receive library-specific work, including library preloads and theme processing where applicable. Some tasks are disabled by default and must be enabled through configuration or an explicit inclusion. [B2]

The important distinction is between a task and a processor. A task gathers resources and arranges work; processors perform transformations. An extension that runs after a bundle is generated cannot expect edits to its source resources to retroactively appear inside that bundle. Order therefore forms part of the artifact's correctness contract.

Consider a generated formatter module. If it is written before minification and preload generation, it can be minified, mapped, and included consistently. If it appears after preload generation, the application might load the old bundled version while a direct module request shows the new version. This is an especially confusing failure because both files appear plausible in isolation.

Document custom task anchors and expected outputs alongside the task implementation. A reviewer should know whether a task changes executable code, generates metadata, excludes resources, or packages an archive. Avoid a catch-all “prepare” task that secretly performs all four.

## Bundles and preloads are not interchangeable

`Component-preload.js` aggregates component resources so the framework can register them with fewer network round trips. A library preload serves a corresponding library purpose. Neither is a generic replacement for every runtime request: remote service calls, dynamically constructed resource names, external assets, and deliberately lazy functionality can remain outside the preload.

A preload must preserve UI5 module identifiers. Renaming a file or namespace without updating imports can yield a bundle that exists and parses but never satisfies the requested module. Test with case-sensitive paths and a deployment-like URL prefix, not only a root-mounted development server.

Use the default component preload before introducing custom bundle definitions. Custom bundles increase ownership: filters, dependency resolution, execution order, and future modules must remain coherent. A giant bundle can reduce request count while increasing initial download, parse time, and cache invalidation frequency. Measure the complete user journey.

The following application fragment excludes development-only resources from the build and a large help area from the component preload:

```yaml
specVersion: "4.0"
metadata:
  name: acme.orders
type: application
builder:
  resources:
    excludes:
      - "test/**"
      - "localService/**"
  componentPreload:
    excludes:
      - "acme/orders/help/**"
  cachebuster:
    signatureType: hash
```

Build exclusions and preload exclusions have different effects. A build exclusion removes resources from the build's resource processing. A preload exclusion leaves a resource available separately while omitting it from that bundle. Here the help area remains deployable and can load on demand. The namespace-based preload pattern is not the same path convention as an application-relative resource exclusion. Consult the configuration reference when adapting patterns. [B3]

Verify exclusions with the actual output tree. A pattern that looks reasonable in a review can silently match nothing because it targets a physical path instead of the virtual resource path. Conversely, an overly broad pattern can remove a runtime fragment that was never loaded by the happy-path test.

## Minification, debug variants, and maps

UI5's standard `minify` task compresses JavaScript while retaining original code as debug variants, such as `Controller-dbg.js`. It also creates source maps. Bundling can incorporate these maps so a browser displays the original module locations while executing optimized content. These mechanisms support debugging; they do not hide business logic. [B2]

Treat the optimized file, its map, and its referenced sources as a related set. A map from a different build may point to the wrong line while looking superficially valid. Uploading only a new map after a production incident cannot reconstruct a missing original artifact.

For transpiled applications, the transpiler must emit valid input maps and references before UI5 minification. Official documentation notes that input-map support was introduced in CLI 3.7. If an earlier task changes JavaScript without updating its map, the minifier may ignore that input map to avoid corrupt mappings. Version and copyright replacement can matter here. [B2]

Choose a publication policy deliberately. Public maps improve supportability but may reveal original source comments or embedded paths. Private maps retained in an access-controlled artifact store support incident investigation without public distribution. Removing maps does not make browser code secret, and it should not become a substitute for code review or authorization.

When maps are withheld, retain the complete original build internally. Do not regenerate them later from a nominally identical commit: toolchain and dependency changes can shift offsets. If a packaging step strips maps, verify that it also handles map references according to the team's policy and that preload behavior still works.

## Cachebuster information and cache layers

Cachebuster metadata is not automatically an HTTP caching policy. The builder can generate cachebuster information when the appropriate task is included, but the target runtime must consume it correctly. For a build that needs this output:

```sh
./node_modules/.bin/ui5 build \
  --clean-dest \
  --include-task generateCachebusterInfo
```

The `hash` signature setting in the earlier YAML makes signatures content-based rather than timestamp-based. This is useful when fresh workspaces change file modification times without changing resource content. It improves stability but does not sign or authenticate the artifact. [B3]

Think in layers: browser cache, framework cachebuster resolution, reverse proxy or CDN, application-router behavior, and ABAP application index. Invalidating one does not necessarily refresh the others. A user seeing an old screen after deployment might have stale shell metadata or a cached preload, not an unsuccessful upload.

Never manually delete broad backend caches as the first response. Establish which URL returned which resource and compare it with the retained artifact. Then use the target's supported cache and application-index procedures with the platform owner.

## Optimization based on evidence

Begin with a cold navigation and record transferred bytes, request count, time to usable screen, and backend latency. Repeat with a warm browser cache. Separate framework delivery from application delivery. A large OData response can dominate startup even after every JavaScript optimization succeeds.

Optimize at the largest measured bottleneck. For excessive application requests, verify preload loading and synchronous imports. For oversized preloads, identify rarely used areas and lazy-load them through supported UI5 mechanisms. For slow backend calls, reduce unnecessary initial bindings or request only needed properties where the service contract permits. For theme delays, check which theme resources are actually requested.

Compression belongs to the serving infrastructure unless a target specifically consumes precompressed artifacts. Minified bytes on disk are not equal to compressed transfer bytes. Test the deployed response headers and content encoding rather than assuming a local development server represents the production network.

Set performance budgets with a stable test scenario. A budget such as “initial application payload must not grow beyond the approved threshold” should specify whether it includes maps, debug files, framework assets, fonts, and translations. Otherwise teams can meet the number by changing measurement boundaries instead of improving users' experience.

## Packaging without rebuilding the application

After verification, package the exact build output. For Cloud Foundry MTAs, the Cloud MTA Build Tool reads `mta.yaml` and produces an `.mtar`. Inspect module build commands carefully: if they invoke `npm run build`, that is another build unless the MTA packaging stage is defined as the authoritative build. Choose one authority and retain its resulting artifact. [B4]

For ABAP publication, the deploy tool can consume an explicit archive or folder. That permits uploading the already-approved output without invoking a deployment-bound UI5 task as part of every ordinary build. The deploy-tooling documentation describes `--archive-path` and `--archive-folder`; see the target guide for the surrounding configuration and transport requirements. [B5]

Do not inject environment-specific credentials while packaging. A deployment extension descriptor may identify a target-specific service name, but its digest and approval should accompany the release record. An artifact digest alone is insufficient if unreviewed deployment configuration can redirect traffic or change authorization.

## Failure analysis and acceptance

If a build passes but runtime modules fail, compare preload identifiers, resource case, manifest dependencies, and deployed URL prefixes. If a map opens the wrong source, compare artifact identity before editing the map configuration. If clean builds differ, inspect timestamps and generated metadata before assuming dependency corruption.

If a bundle grows unexpectedly, inspect the added resource set, not just the compressed total. A broad inclusion can accidentally pull fixtures or translations into startup. If a mock service appears in production output, correct the resource policy and add a targeted artifact assertion; deleting that file manually after every build is not a reliable control.

Use this release acceptance checklist:

* Source commit, lockfile, toolchain, and framework inputs are recorded.
* A clean output contains the expected manifest, component, views, and translations.
* Preload resources resolve under deployment-like paths and Linux casing.
* Optimized code and retained maps belong to the same artifact.
* Mock data, test pages, private configuration, and credentials are absent.
* Cachebuster generation matches the target's consumption model.
* Browser tests exercise optimized output, not only development sources.
* Performance measurements use an agreed cold and warm scenario.
* The final archive has a recorded digest and retention policy.
* Promotion uses that archive rather than quietly rebuilding it.

## Official sources

Consulted on 2026-10-17 through official GitHub sources where necessary:

* **[B1]** SAP, [CLI 4.0.0 package metadata](https://github.com/SAP/ui5-cli/blob/v4.0.0/package.json), for exact Node and npm engine requirements.
* **[B2]** UI5, [Builder documentation source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/Builder.md), for task selection, preloads, minification, debug variants, and source-map behavior.
* **[B3]** UI5, [Configuration documentation source](https://github.com/UI5/cli/blob/main/internal/documentation/docs/pages/Configuration.md), for exclusion semantics, preload filters, and cachebuster signatures.
* **[B4]** SAP, [Cloud MTA Build Tool README](https://github.com/SAP/cloud-mta-build-tool/blob/master/README.md), for the distinction between project descriptors and deployment-ready MTAR archives.
* **[B5]** SAP, [ABAP deploy-tooling README](https://github.com/SAP/open-ux-tools/blob/main/packages/deploy-tooling/README.md), for deployment independent of UI5 build and archive/folder inputs.
