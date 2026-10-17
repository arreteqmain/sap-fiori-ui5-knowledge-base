# UI5 Development Environment

**Scope:** selecting and operating a secure, repeatable UI5 workspace using SAP Business Application Studio or Visual Studio Code, approved tooling, and an HTTP development server. **Prerequisites:** basic terminal use, Git, JavaScript, and access to the intended runtime and backend. **Runtime assumptions:** browser-based SAPUI5 or OpenUI5, asynchronous module loading, and a reviewed project-local toolchain. The examples describe commands for an already provisioned application; no dependencies were installed and no commands shown below were executed as application verification.

## 1. Separate workspace, server, and runtime

The editor helps write source. Node runs development tools. The development server exposes application and framework resources. The browser executes UI5. The backend processes business requests. Keeping these roles separate makes diagnosis much faster.

A Node upgrade cannot make an older deployed SAPUI5 runtime support a newer browser API. An editor extension cannot repair a missing service authorization. A local proxy can make development connectivity possible, but it does not establish production routing or grant backend access.

Create a small environment record for the team: operating system, Node and package-manager versions, UI5 CLI version, UI5 framework delivery and patch, browser targets, authentication approach, and deployment host. Store only nonsensitive configuration in source control.

## 2. Choose SAPUI5 or OpenUI5 independently of the editor

Both deliveries can be developed using suitable local tooling. OpenUI5 provides the shared framework core and many controls under Apache licensing. SAPUI5 includes additional libraries and product integration under SAP terms. An editor's “SAP Fiori” label does not turn an OpenUI5 project into a licensed SAPUI5 application.

Check which libraries the application actually needs. A simple `sap.m` application is different from a Fiori elements application requiring SAPUI5-specific libraries. Select framework resources through the project configuration and target hosting environment, not by whichever SDK URL a developer last opened.

The [official distribution comparison](https://github.com/SAP/openui5-docs/blob/main/docs/02_Read-Me-First/sapui5-vs-openui5-5982a97.md) and each release's API reference are the starting points for library availability. Use a versioned reference when investigating a compatibility issue.

## 3. Provision a compatible toolchain

Use an organization-approved Node release supported by the exact tooling packages. Do not equate “newest Node” with “supported.” UI5 CLI, generators, middleware, and TypeScript transformations may have different engine constraints.

For example, the official [`@ui5/cli` package metadata](https://github.com/SAP/ui5-cli/blob/main/package.json) inspected for this guide identifies CLI `4.0.71` with Node engine range `^20.11.0 || >=22.0.0` and npm `>=8`. This is a source observation, not a recommendation to install that exact version or use an end-of-life Node release. Recheck package metadata and platform support before provisioning.

Keep project-local tooling and the lockfile authoritative. Different global CLIs can interpret project configuration differently. A working command on one workstation may merely be using a stale global installation that another developer lacks.

Install or restore dependencies only through the team's reviewed process after checking package provenance, advisory status, license, and lifecycle scripts. This guide intentionally avoids installation commands and implicit package downloads.

## 4. Inspect the environment without downloading tools

In an existing application workspace, these commands inspect locally available tools:

```sh
node --version
npm --version
git --version
npm ls --depth=0
```

`npm ls` can return a nonzero status for missing or invalid packages; that is useful evidence rather than a reason to fetch random replacements. Compare its output with the project manifest and lockfile.

Do not use an unqualified `npx` invocation to discover whether a tool exists. Depending on configuration, it may download and execute a package. Prefer a reviewed npm script or the known local executable.

Check corporate certificate requirements, outbound network restrictions, and registry access before troubleshooting the application. An intercepted TLS connection or blocked registry may be an environment problem rather than a broken dependency declaration.

## 5. Decide between BAS and local VS Code

SAP Business Application Studio is an SAP BTP service providing cloud-hosted development workspaces. It can simplify access to SAP-oriented extensions, destinations, and guided development. It still needs account permissions, suitable dev-space configuration, and access to the required services.

Visual Studio Code provides a local or remotely hosted editor with extension support. It gives direct control over the workstation toolchain and can suit offline or tightly managed environments. Developers must provision compatible tools and connectivity through organizational policy.

Choose based on identity integration, network access, team support, compute needs, and data policy. Do not choose BAS expecting it to bypass a private backend's network restrictions, or choose local development expecting browser CORS rules to disappear.

## 6. Establish a BAS workspace deliberately

Obtain the required BTP entitlement, subscription, and user role assignments through administrators. Create an appropriate SAP Fiori development dev space and select only necessary capabilities. Product offerings, role collections, and dev-space limits are release- and subscription-sensitive; follow the current service documentation.

After opening the workspace, clone the approved repository through the organization's authentication method. Avoid embedding access tokens in Git remote URLs or copying credentials into terminal commands that become shell history.

Inspect preinstalled tool versions instead of assuming they match the project. Managed workspace updates can change the available Node or generator version. Prefer the project's recorded toolchain where the workspace supports it.

The official [BAS development overview](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/app-development-using-sap-business-application-studio-6bbad66.md) describes the service relationship and links to onboarding guidance. It does not eliminate the need to verify the current tenant's capabilities and access controls.

## 7. Connect BAS to services safely

A destination describes approved connectivity to a target system and can support authentication patterns appropriate to that landscape. It is not simply a URL copied into frontend JavaScript. A private on-premises backend may additionally require an approved connectivity path such as configured cloud connectivity.

Verify that the development identity can discover the intended destination and that the target service is reachable. Distinguish connection failure from service authorization failure. The latter may require business roles in the backend even when the destination itself is visible.

Do not export destination secrets into application source to “make local preview work.” Preserve the server-side trust boundary. Use development systems and synthetic or approved test data; a conveniently reachable production endpoint is not automatically authorized for experimentation.

Record the required destination name and purpose without recording credentials. A new developer should understand what access to request without receiving someone else's secret values.

## 8. Configure VS Code with trusted extensions

Open the application root containing `package.json` and `ui5.yaml`, not just the `webapp` directory. Tooling and language services often need the full project context to resolve namespaces, configuration, and scripts.

Use official or organization-approved extensions, including SAP Fiori tools where appropriate. SAP's [Open UX Tools repository](https://github.com/SAP/open-ux-tools) describes reusable modules maintained by the Fiori tools team; individual extension capabilities and supported project types still require their own current documentation.

Treat workspace recommendations as suggestions, not permission to install arbitrary extensions. Extensions can execute code and access project files. Review publisher identity, workspace trust prompts, and remote execution implications.

Share nonsensitive editor settings only when they improve consistency. Do not commit machine-specific SDK paths, user home directories, proxy passwords, or personal authentication configuration. Keep executable debug tasks readable so reviewers can see what they launch.

## 9. Use an HTTP server, not file URLs

UI5 applications load modules, views, descriptors, resource bundles, and service metadata over HTTP. Opening `index.html` through `file://` bypasses the expected origin and resource-serving model and commonly fails.

With the approved local UI5 CLI already provisioned, the `start` script from [project setup](./project-setup.md) can be invoked:

```sh
npm run start
```

That script resolves to `ui5 serve --port 8080`. It expects a valid UI5 application configuration and framework resources. Keep the process running in its terminal, then open the server's reported URL and application entry page.

Do not assume any server is private merely because it is called a development server. Check its configured bind address, operating-system firewall, and workspace port visibility. A forwarded BAS port has a different exposure model from a local loopback connection.

## 10. Verify the server layer before debugging UI code

For an already running local server, these HTTP checks inspect expected resources:

```sh
curl --fail --head http://localhost:8080/index.html
curl --fail --head http://localhost:8080/resources/sap-ui-core.js
```

A successful header request shows resource reachability, not that the application rendered or that authentication works. Some servers handle HEAD differently from GET; when necessary, inspect the browser's actual GET request before concluding the resource is unavailable.

Check the server terminal for configuration errors and port conflicts. If another process owns the port, identify it specifically or choose a documented alternate port. Do not terminate unrelated processes by name.

Check the browser network panel next. A missing `Component.js` usually points to resource-root or namespace mapping. A missing framework file points to server or framework configuration. A service returning HTML rather than JSON may indicate a login redirect or incorrect proxy route.

## 11. Treat proxies as development infrastructure

The standard UI5 server provides development resource serving; it is not automatically a production authentication gateway or a universal backend proxy. Middleware adds capabilities and also expands the dependency and trust surface.

Use only reviewed middleware with a documented purpose. Define which paths it proxies, which hosts are allowed, how credentials are obtained, and what logs it emits. Avoid a catch-all proxy that forwards arbitrary user-controlled URLs.

CORS restrictions are browser security boundaries. Resolve them through approved same-origin routing or backend CORS configuration, not by disabling browser security or installing a random extension. Do not set wildcard origins and credential sharing indiscriminately.

Never normalize bypassing certificate validation. Obtain the proper trust chain and fix hostname or certificate configuration. Development exceptions can become production defaults when copied into shared configuration.

## 12. Distinguish mock mode from live mode

Mock data is useful for layout, navigation, and predictable error scenarios. It must have clearly synthetic records and a visibly distinguishable operating mode. A successful mock approval proves nothing about backend authorization or transactional semantics.

Live mode requires a reviewed service path, identity, and nonproduction environment. Keep those settings separate so a routine preview cannot accidentally mutate production data. Do not hide environment switching in an obscure query parameter that a user could misunderstand.

Model failure cases deliberately: empty lists, forbidden objects, validation messages, slow requests, and conflicting edits. Mock behavior should support design exploration without implying that every backend responds identically.

Document what mock mode omits, including persistence durability, concurrency checks, batch semantics, and server-side paging. That boundary prevents a polished demo from being mistaken for integration readiness.

## 13. Debug browser behavior methodically

Start with the earliest meaningful console error. Later errors often result from the first failed module or invalid descriptor. Preserve the failing request URL, status, and response type before changing configuration.

Use breakpoints in the controller handler for the interaction under investigation. Inspect the event source, binding context, model name, and current route. Avoid dumping entire models into logs when they can contain personal or commercially sensitive information.

Browser caching can hide source changes. Compare cold and warm loads and understand the deployment cache strategy. Debug flags or support tools can affect request patterns and performance, so do not use a heavily instrumented session as production performance evidence.

The framework's technical information and support facilities vary by release. Consult the exact SDK rather than relying on a remembered keyboard shortcut or private global object.

## 14. Diagnose bindings before adding refresh calls

If a control is empty, inspect whether its model exists and whether the binding path is absolute or relative. Confirm the binding context points at the expected object. A model named `view` is not the default model.

If a value changes but the display does not, check whether code used the model's supported update API rather than mutating a nested plain object. For remote models, distinguish a pending edit from persisted data.

Avoid using repeated model refreshes as a general repair. They can discard context, create unnecessary requests, and conceal race conditions. A slow service response may be updating an object that the user already navigated away from.

For OData issues, identify V2 versus V4 first. Their APIs, submission groups, and event patterns differ. Instructions for one model are not automatically valid for the other.

## 15. Make translation and accessibility feedback available

Provide a quick way to inspect supported locales, long text, right-to-left layout, and relevant themes in the development preview. Do not rely exclusively on the developer's default language and high-resolution display.

Use browser accessibility inspection to check control names and relationships. Navigate with the keyboard and confirm focus remains meaningful after dialogs, validation, and list updates. Automated inspection cannot replace completing the task with assistive technology where required.

For themes, ensure all required library resources are available. Modern APIs such as `sap/ui/core/Theming` are release-sensitive; verify availability before adding a settings screen copied from newer documentation.

Keep test preview configuration out of normal user-facing production behavior. Developers need switches for investigation; production users need approved, understandable preferences.

## 16. Keep source, generated artifacts, and local state separate

Commit source and reproducibility metadata according to repository policy. Exclude dependency directories, generated build output when appropriate, personal workspace state, credentials, and local diagnostic exports.

Do not commit exported browser network archives without inspection. They can contain authorization headers, session cookies, request bodies, and personal data. Redact at the source where possible and use an approved support channel.

Local caches can be useful but are not the source of truth. A project that works only because of an unrecorded cached framework library is not reproducible. Document required resource access and use approved caching or artifact mirrors.

Be careful with cleanup commands. Removing the entire workspace to solve one cache issue risks deleting uncommitted work. Identify the specific generated directory and confirm it contains no source before removal.

## 17. Plan for launchpad and deployment differences

A standalone `index.html` controls bootstrap settings during development. A launchpad typically controls runtime loading, theme, user context, and navigation services. Keep application behavior in the component and descriptor so it does not depend on standalone-only globals.

Relative service paths may resolve differently after deployment under an application repository path. Validate the application router or platform route, authentication redirect, and resource origins in the target environment.

Check the deployed framework version rather than inferring it from local `ui5.yaml`. An ABAP landscape may supply a centrally installed runtime with a different maintenance cycle from a cloud-hosted development workspace.

Separate “works in local preview,” “works with live development service,” and “works in target shell” as distinct readiness milestones. Each verifies a different integration boundary.

## Verification checklist

- [ ] Record approved Node, package-manager, CLI, and runtime versions.
- [ ] Confirm SAPUI5/OpenUI5 library availability and licensing assumptions.
- [ ] Review workspace extensions, tasks, and package scripts before execution.
- [ ] Verify server reachability and framework resources independently.
- [ ] Check bind address, forwarded-port visibility, and authentication.
- [ ] Keep credentials outside frontend source and diagnostic exports.
- [ ] Separate mock mode, live development mode, and production routing.
- [ ] Inspect namespace mapping, model names, and relative contexts.
- [ ] Review keyboard interaction, locale growth, zoom, and themes.
- [ ] Recheck the application under its intended deployment host.

This is an operational checklist, not a record of completed execution.

## Official references and related foundations

- [UI5 CLI documentation](https://ui5.github.io/cli/stable/)
- [UI5 CLI project source and release guidance](https://github.com/UI5/cli)
- [CLI package engine metadata](https://github.com/SAP/ui5-cli/blob/main/package.json)
- [SAP Business Application Studio product documentation](https://help.sap.com/docs/bas)
- [Official BAS and UI5 development overview](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/app-development-using-sap-business-application-studio-6bbad66.md)
- [SAP Open UX Tools](https://github.com/SAP/open-ux-tools)
- [VS Code workspace trust](https://code.visualstudio.com/docs/editing/workspaces/workspace-trust)

Continue with [project setup](./project-setup.md), [UI5 framework](./ui5-framework.md), and [Fiori design](./fiori-design-system.md). These are the requested sibling foundation paths listed in the [repository README](../README.md).
