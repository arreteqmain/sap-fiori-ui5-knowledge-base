# Deployment Targets: Cloud Foundry, HTML5 Repository, and ABAP

## Scope and target selection

This guide follows a built UI5 application into three related but distinct hosting arrangements: a Cloud Foundry application, SAP BTP HTML5 Application Repository content, and an on-premise SAPUI5 ABAP repository. It assumes an approved application build exists and that platform administrators have provisioned accounts, trust, network routes, and authorizations.

SAP Business Application Studio is an IDE, not a production hosting target. A BAS deployment wizard invokes tools against a selected runtime. The runtime still needs its own identity provider, services, destinations, roles, and lifecycle ownership.

Read the [build guide](build-process.md) before packaging and the [pipeline guide](cicd-pipeline.md) before automating credentials or promotion. [UI5 tooling](ui5-tooling.md) distinguishes local middleware from production routing. The [README](../README.md) links the surrounding knowledge base.

For compatibility discussions, the verified historical baseline is **UI5 CLI 4.0.0**, with Node **`^20.11.0 || >=22.0.0`** and npm **`>=8`** according to its tagged metadata. Select a maintained patch and approved extensions rather than installing this old baseline automatically. A direct advisory check of that package returned no advisory during preparation; transitive dependencies and existing project versions need separate review. [D1]

Choose based on ownership. A standalone Cloud Foundry application router provides application-specific routing and extension control but consumes runtime resources and requires maintenance. The HTML5 repository stores static application content; a managed or standalone application router serves it with authentication and routing. ABAP hosting integrates with an existing ABAP front-end server and transport landscape. None automatically grants backend authorization.

## Common release prerequisites

Before touching a target, record the artifact digest, application identity, application version, expected framework runtime, and supported backend service contract. Confirm that development mock data and credentials are absent. Keep a previous approved artifact and document whether a rollback requires a corresponding backend change.

Use target-specific service identities. A developer's browser SSO session is not a durable CI credential. Prefer short-lived federation or broker-issued sessions where supported. Otherwise use a dedicated, least-privileged technical identity managed through the platform's approved secret mechanism. Never store passwords, service keys, destination credentials, or CF CLI session files in Git.

Preserve TLS validation. Install enterprise certificate authorities in the approved trust store rather than copying examples that disable certificate checks. A reachable HTTPS endpoint with a certificate error is an infrastructure issue to resolve, not an instruction to weaken every subsequent deployment.

Separate content changes from infrastructure changes. Creating an identity service, changing a destination's authentication method, or replacing a shared repository host can affect applications beyond the one being released. Those changes need their own review, impact assessment, and ownership.

## Cloud Foundry: prepare the organization and space

For the MTA procedure, install approved platform tooling through the controlled workstation or runner image: Cloud Foundry CLI **v8**, the MultiApps CF CLI plugin, the Cloud MTA Build Tool, and GNU Make. The MultiApps README explicitly requires CF CLI v8; MBT's usage documentation identifies GNU Make 4.2.1 and module-specific build tools as prerequisites. Record exact installed versions in release evidence. [D2][D3]

An administrator must enable Cloud Foundry in the BTP subaccount, allocate entitlements and quota, create the target space, and grant deployment rights to the release identity. Check availability of the services named by the MTA descriptor in the selected region. A service plan available in one subaccount is not guaranteed in another.

For an interactive operator, the normal login path is:

```sh
cf api "$CF_API"
cf login --sso
cf target -o "$CF_ORG" -s "$CF_SPACE"
cf target
cf plugins
cf marketplace
```

Set the nonsecret variables to the approved API endpoint, organization, and space. SSO is suitable for an operator, not an unattended pipeline. CI should obtain a short-lived session using its approved authentication integration, as discussed in the pipeline guide. Inspect target output before deploying; an inherited CLI session can point at the wrong space.

## Cloud Foundry: deploy an MTA

Use a reviewed `mta.yaml` generated or maintained for the application's architecture. Its modules describe UI content, content deployment, an optional standalone application router, and any owned backend modules. Its resources describe service instances and bindings. Do not paste a service resource into an existing landscape without deciding whether that service is application-owned or shared.

The documented build and deployment sequence is:

```sh
npm ci
mbt build -p cf --mtar orders.mtar
sha256sum mta_archives/orders.mtar
cf deploy mta_archives/orders.mtar
cf mtas
cf mta acme-orders
```

The final identifier `acme-orders` must match the MTA's actual `ID`, not necessarily the UI5 namespace or CF application name. MBT produces the archive; the MultiApps plugin deploys it to the platform. There is no generic `ui5 deploy --target cloud-foundry` command in this procedure. [D2][D3]

For environment-specific MTA settings, the plugin supports a reviewed extension descriptor:

```sh
cf deploy mta_archives/orders.mtar -e config/production.mtaext
```

Retain that descriptor's digest alongside the MTAR digest. Do not embed secrets in a checked-in extension file. Promotion should reuse the approved archive while changing only explicitly reviewed target configuration.

When deployment fails, inspect `cf mta-ops` and use the plugin's operation-log download capability rather than immediately deleting services and retrying. Server-side operations can outlive the client process. Deleting a shared destination or repository host to clear an error can remove unrelated applications.

An ordinary `cf push` is appropriate only when the deployable unit is an actual CF application with a valid manifest or buildpack contract. A UI5 `dist` folder is not automatically a runnable Node application. With a standalone router, deploy the router application and bind the required services; with repository content, use the content deployment or repository upload path.

## HTML5 Application Repository: storage and serving are separate

The repository's `app-host` plan stores application content. A standalone router typically consumes the `app-runtime` plan to retrieve it. A managed application router instead relies on its managed runtime integration and destination configuration. Do not provision `app-runtime` for every application merely because a standalone-router sample contains it. SAP's official samples explain these two serving models. [D4]

For managed runtime deployment, use an appropriate generated MTA that includes a content module targeting the repository host and, when needed, a destination-content module. SAP's basic managed-runtime sample defines repository host, destination, and XSUAA resources, together with destination content linking their service keys. Those relationships are the operational substance of the example, not optional decoration. [D5]

The key repository content structure is illustrated by this MTA fragment:

```yaml
modules:
  - name: orders-content
    type: com.sap.application.content
    path: .
    requires:
      - name: orders-repo-host
        parameters:
          content-target: true
    build-parameters:
      build-result: resources
      requires:
        - name: orders-ui
          artifacts:
            - orders-ui-content.zip
          target-path: resources/
resources:
  - name: orders-repo-host
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-host
```

This is deliberately a **fragment**, not a complete runnable MTA. A complete descriptor also defines `orders-ui`, its build command and output, descriptor header and version, and the selected serving model's authentication and destination resources. The ZIP name must match the artifact actually produced by that module. Inspect the archive before deployment rather than assuming MBT will infer a missing UI module.

At application level, the repository content needs a correct manifest identity and application version. A minimal routing descriptor for an authenticated application whose content is served from the repository looks like this:

```json
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^(.*)$",
      "target": "$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```

This mirrors the repository-serving route in SAP's managed-runtime sample. It assumes the selected runtime integration supplies the corresponding service and XSUAA context. It does not create an identity service, a business destination, or backend scopes. Add narrower backend routes before a catch-all route and review them with the security owner. [D5]

## HTML5 repository: direct upload and verification

For controlled development or a lifecycle explicitly based on repository uploads, SAP provides the CF HTML5 Applications Repository CLI plugin. Have the approved plugin preinstalled, a valid CF session targeting the intended space, and an existing host instance. The upload folder must contain `manifest.json` and `xs-app.json`. [D6]

```sh
cf target -o "$CF_ORG" -s "$CF_SPACE"
cf html5-info -n acme-orders-host
cf html5-push -n acme-orders-host dist
cf html5-list
```

The `-n` option selects the host instance name; `dist` is the already-built application folder. The plugin can create services in some flows, but an explicit existing host is easier to govern. Avoid mixing direct uploads and MTA-managed content updates for the same application without a documented ownership policy.

After upload, verify application identity and version in the repository inventory and inspect the HTML5 Applications view in the BTP cockpit where available for the chosen managed integration. If the application is absent from the managed view, inspect destination exposure, `sap.cloud.service` consistency, and runtime enablement instead of repeatedly uploading identical content.

Open the application through its intended runtime URL. Confirm the identity-provider redirect, successful return, static resource loading, translations, backend calls, and access denial for an unauthorized user. Repository inventory proves storage, not correct serving or business authorization.

For an on-premise backend consumed from BTP, connectivity may require a configured Cloud Connector mapping, Connectivity service, and an `OnPremise` destination. Cloud Connector establishes controlled reachability; it does not replace business-user authentication. Principal propagation and technical-user destinations have different authorization consequences and must not be switched casually during troubleshooting.

## On-premise ABAP and Gateway: establish system capabilities

The modern SAP deployment tooling requires **SAP_UI 7.53 or higher**, an enabled and reachable repository service, and appropriate development authorization, including `S_DEVELOP` for repository operations according to its official README. A system with SAP Gateway alone does not necessarily meet those prerequisites. [D7]

Identify the actual front-end repository system. In a hub deployment, the UI repository and Gateway runtime may be separate from the business backend. Record the development client, transport route, transportable package, BSP/application name, and service alias arrangement. The UI5 application ID, repository name, and launchpad semantic object are related configuration, not interchangeable identifiers.

Ask the ABAP administrator to verify the SAPUI5 repository OData service, normally `UI5/ABAP_REPOSITORY_SRV` under `/sap/opu/odata/UI5/ABAP_REPOSITORY_SRV`, and required ICF activation and authorizations for that release. The deployment identity should be able to change the intended package and transport, not develop arbitrarily across the system.

For older systems below the tool's prerequisite, use the SAP-supported repository upload mechanism available for that system, commonly the `/UI5/UI5_REPOSITORY_LOAD` report with a Basis-controlled procedure. Do not force a modern deployment command against an unsupported endpoint or assume every ECC landscape has the same SAP_UI level.

## ABAP: configure and perform publication

The following is an example `ui5-deploy.yaml` for an existing project with approved SAP deploy tooling:

```yaml
specVersion: "4.0"
metadata:
  name: acme.orders
type: application
builder:
  customTasks:
    - name: abap-deploy-task
      afterTask: generateCachebusterInfo
      configuration:
        target:
          url: https://abap-dev.example.invalid
          client: "100"
        app:
          name: Z_ACME_ORDERS
          description: Orders application
          package: Z_ACME_UI
          transport: DEVK900123
        exclude:
          - /test/
          - /localService/
```

Replace the placeholder endpoint, client, package, and transport with approved development values. The transport is illustrative and must be an existing suitable modifiable workbench request assigned according to local policy. Do not use `$TMP` for a production-bound release: local objects do not provide the intended transport lifecycle.

The standalone deploy executable allows publication independently of rebuilding:

```sh
./node_modules/.bin/deploy --help
./node_modules/.bin/deploy \
  --config ui5-deploy.yaml \
  --archive-folder dist \
  --test
./node_modules/.bin/deploy \
  --config ui5-deploy.yaml \
  --archive-folder dist \
  --yes --no-retry
```

A base UI5 configuration is still required by the tool. The first deployment command uses documented test mode to obtain backend validation without uploading changes. The second performs publication and disables interactive retry behavior for automation. Verify the options against the already-approved installed version; this guide does not install a new npm package. [D7]

Authentication must already be available through the selected SAP Fiori tools secure-storage or destination mechanism, or supplied through the organization's documented automation integration. Do not assume that arbitrary environment variable names are automatically recognized by the deploy tool. Its documented username/password flags can be bound from a secret manager on an isolated worker, but command arguments can be visible to local processes; prefer secure storage or supported destination authentication where possible.

## Gateway, launchpad, and transport verification

After upload, verify the repository application directly under its approved `/sap/bc/ui5_ui5/sap/...` path where enabled. Then verify the application through the launchpad, including target mapping, catalog or space/page assignment, role assignment, and application index state. A working direct URL does not prove launchpad resolution.

Treat business OData service activation separately. Validate service registration and system aliases in the correct Gateway system using its supported administration tools. Investigate metadata or authorization failures with the Gateway administrator; rebuilding the JavaScript cannot repair an incorrect alias or missing backend authorization.

Record the successful deployment with the artifact digest and workbench request. Release and import that transport through QA and production using CTS or the approved change-management process. Coordinate repository content, backend services, authorization roles, and launchpad configuration as related but distinct transport concerns.

For recovery, redeploy an approved earlier artifact into a new corrective transport according to local policy. Importing old transports out of sequence can overwrite unrelated fixes. A frontend rollback also cannot undo business data changes performed while the newer version was active.

## Final operational checklist

Confirm target identity, entitlements, supported tool versions, certificate trust, least-privileged credentials, artifact digest, service ownership, routing order, runtime framework compatibility, authenticated smoke results, authorization-denial behavior, previous-release retention, and promotion evidence. For ABAP, additionally confirm package, client, transport, Gateway aliases, and launchpad content. For the repository, confirm host identity, application version, and managed-runtime exposure. Close the release only when user-facing behavior is verified, not merely when an upload command exits successfully.

## Official sources

Consulted on 2026-10-17; official SAP GitHub sources were used where documentation hosts were unreachable:

* **[D1]** SAP, [CLI 4.0.0 metadata](https://github.com/SAP/ui5-cli/blob/v4.0.0/package.json).
* **[D2]** Cloud Foundry MultiApps, [plugin README](https://github.com/cloudfoundry/multiapps-cli-plugin/blob/master/README.md), for CF CLI v8, MTA commands, extension descriptors, and operations.
* **[D3]** SAP, [MBT usage](https://github.com/SAP/cloud-mta-build-tool/blob/master/docs/docs/usage.md), for build prerequisites and archive options.
* **[D4]** SAP, [HTML5 application architecture samples](https://github.com/SAP-samples/multi-cloud-html5-apps-samples), including links to official SAP Help deployment documentation. Older sample Node prerequisites are not adopted here.
* **[D5]** SAP, [managed-runtime MTA](https://github.com/SAP-samples/multi-cloud-html5-apps-samples/blob/main/managed-html5-runtime-basic-mta/mta.yaml) and [application routing descriptor](https://github.com/SAP-samples/multi-cloud-html5-apps-samples/blob/main/managed-html5-runtime-basic-mta/HTML5Module/xs-app.json).
* **[D6]** SAP, [HTML5 repository CLI plugin README](https://github.com/SAP/cf-html5-apps-repo-cli-plugin/blob/master/README.md), for upload prerequisites and `html5-push`, `html5-list`, and `html5-info` syntax.
* **[D7]** SAP, [ABAP deployment tooling README](https://github.com/SAP/open-ux-tools/blob/main/packages/deploy-tooling/README.md), for SAP_UI prerequisites, repository authorization, configuration, archive input, and test/no-retry options.
