# UI5 Project Setup

**Scope:** establishing a small, component-based freestyle application with a clear folder layout, descriptor, runtime selection, and dependency policy. **Prerequisites:** JavaScript, XML, JSON, an approved Node environment, and familiarity with HTTP resource loading. **Runtime assumptions:** asynchronous UI5 loading and an already provisioned, reviewed UI5 CLI. The example selects OpenUI5 and only shared libraries. Its version numbers illustrate configuration relationships, not current support or security recommendations. No snippets were executed, and no application files or dependencies are created by this guide.

## 1. Decide the target before generating files

Record whether the application will run standalone, in an SAP Fiori launchpad, or in another component host. Record the backend protocol, authentication mechanism, deployment platform, and available framework version. These decisions constrain the scaffold more than a preferred editor does.

Choose OpenUI5 when its libraries and license suit the application. Choose SAPUI5 when the required SAP product integration or libraries demand it, subject to entitlement. SAP Fiori elements, smart-control scenarios, and certain analytical libraries cannot be assumed available in OpenUI5. Consult the official [distribution comparison](https://github.com/SAP/openui5-docs/blob/main/docs/02_Read-Me-First/sapui5-vs-openui5-5982a97.md).

For a generated application, use a reviewed generator supplied through the organization's tool catalog. Inspect the resulting descriptor, dependencies, server configuration, and scripts before accepting it. A generator accelerates setup but does not decide authorization, service contracts, or whether its runtime baseline matches the deployment system.

## 2. Choose a stable application identity

Use a unique reverse-domain namespace such as `org.example.orders`. Keep that identity consistent across `sap.app/id`, the component's class name, controller names, and resource-root mappings. A namespace is not a network hostname and should not contain credentials or environment-specific paths.

Separate application identity from business display text. Users may see “Request Review” in several languages while the technical ID remains stable. Renaming a technical ID later can affect navigation targets, persisted personalization, adaptation artifacts, and deployment registration.

The package name can use npm-compatible lowercase hyphenated syntax, while UI5 uses dotted namespace syntax. Do not assume those strings must be identical. Explicit configuration connects the package, UI5 project metadata, and runtime resource namespace.

## 3. Establish the folder contract

A conventional minimal structure is:

```text
orders/
├── package.json
├── package-lock.json
├── ui5.yaml
└── webapp/
    ├── index.html
    ├── Component.js
    ├── manifest.json
    ├── controller/
    │   └── App.controller.js
    ├── view/
    │   └── App.view.xml
    ├── i18n/
    │   └── i18n.properties
    ├── model/
    └── util/
```

The lockfile exists only after the approved dependency provisioning process. Empty `model` and `util` directories are optional; create modules there when needed instead of manufacturing placeholder abstractions.

`webapp` contains browser-delivered resources. Keep server credentials, private keys, deployment secrets, and internal diagnostic exports outside it. Files in that directory may become publicly retrievable after deployment even when no application code references them.

Generated output typically belongs in a separate build directory such as `dist`, not mixed back into source. Do not edit preload artifacts by hand. A source change must be reproducible through the established build process.

## 4. Keep three dependency systems distinct

`package.json` describes development tooling and any npm-managed project packages. `ui5.yaml` identifies the UI5 project and its framework resources for UI5 Tooling. `manifest.json` declares runtime application metadata and libraries required by the component.

Adding a library in only one place is not always enough. Declaring `sap.m` in the descriptor tells the runtime about a library dependency, while the tooling configuration makes the corresponding framework library available to the local resource graph. npm dependency resolution does not automatically change the runtime installed in a launchpad.

A small package file can expose scripts without prescribing an unreviewed dependency version:

```json
{
  "name": "org-example-orders",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start": "ui5 serve --port 8080",
    "build": "ui5 build --all"
  }
}
```

This is the script portion of a project after tool provisioning, not a self-contained installation recipe. An approved project-local `@ui5/cli` must also be recorded in development dependencies and the lockfile through the team's reviewed process. npm scripts resolve local executables; that avoids accidental use of an unrelated global CLI.

## 5. Pin a deliberate framework version

The following `ui5.yaml` is an illustrative OpenUI5 configuration. Version `1.120.0` is used to make relationships concrete; replace it with an approved exact release and patch before actual deployment. Verify maintenance status rather than assuming this historical example is appropriate today.

```yaml
specVersion: "3.0"
metadata:
  name: org.example.orders
type: application
framework:
  name: OpenUI5
  version: "1.120.0"
  libraries:
    - name: sap.ui.core
    - name: sap.m
    - name: themelib_sap_horizon
```

This uses a project specification supported by suitable modern UI5 CLI releases. The project specification is not the framework version. The theme library provides the Horizon theme resources expected by the standalone bootstrap.

If switching to SAPUI5, change the delivery deliberately, verify license entitlement, and review every additional library against the selected framework version. OpenUI5 and SAPUI5 patch numbers need not always match. Do not mix unrelated framework patch versions in one runtime resource set.

## 6. Understand descriptor version fields

The descriptor's `_version` identifies the descriptor schema level. `sap.app/applicationVersion/version` identifies the application release. `sap.ui5/dependencies/minUI5Version` documents the minimum framework capability expected by the application. None of these installs a runtime.

A `minUI5Version` value is not a guarantee that an incompatible environment will fail safely before every unsupported API runs. Align it with actual API usage and verify against the deployed framework. Keep the tooling version pin distinct from the descriptor's minimum: one selects local resources, the other communicates a compatibility floor.

Treat the application version as release metadata. Increment it according to the team's release policy, but do not use it as a substitute for artifact provenance, source revision, or transport tracking.

## 7. Define a complete minimal descriptor

Save the following as `webapp/manifest.json` in the example project. It is intentionally a single-view application without routing or a backend. The selected descriptor schema and features fit the illustrative baseline:

```json
{
  "_version": "1.58.0",
  "sap.app": {
    "id": "org.example.orders",
    "type": "application",
    "i18n": "i18n/i18n.properties",
    "title": "{{appTitle}}",
    "description": "{{appDescription}}",
    "applicationVersion": {
      "version": "0.1.0"
    }
  },
  "sap.ui": {
    "technology": "UI5",
    "deviceTypes": {
      "desktop": true,
      "tablet": true,
      "phone": true
    }
  },
  "sap.ui5": {
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": {
        "sap.ui.core": {},
        "sap.m": {}
      }
    },
    "rootView": {
      "viewName": "org.example.orders.view.App",
      "type": "XML",
      "id": "app",
      "async": true
    },
    "models": {
      "i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "settings": {
          "bundleName": "org.example.orders.i18n.i18n",
          "supportedLocales": [""],
          "fallbackLocale": ""
        }
      }
    }
  }
}
```

Descriptor placeholders use double braces; ordinary XML model bindings use single braces. The example declares only a root resource bundle. When adding translations, list the actual supported locales and choose a fallback consistent with the files provided.

Declaring phone support is not proof that the design works on a phone. It expresses an intended device scope that must be verified with representative tasks and layouts.

## 8. Create the owning component

For `webapp/Component.js`, create one component-owned local model. This example intentionally has no router; adding `getRouter().initialize()` without routing configuration would be cargo-cult setup.

```javascript
sap.ui.define([
    "sap/ui/core/UIComponent",
    "sap/ui/model/json/JSONModel"
], function (UIComponent, JSONModel) {
    "use strict";

    return UIComponent.extend("org.example.orders.Component", {
        metadata: {
            manifest: "json"
        },

        init: function () {
            UIComponent.prototype.init.apply(this, arguments);
            this._oViewModel = new JSONModel({
                reference: "",
                previewVisible: false
            });
            this.setModel(this._oViewModel, "view");
        },

        destroy: function () {
            if (this._oViewModel) {
                this._oViewModel.destroy();
                this._oViewModel = null;
            }
            UIComponent.prototype.destroy.apply(this, arguments);
        }
    });
});
```

The component creates this model, so it explicitly destroys it. This policy does not authorize destroying models injected by a host or shared across components. Distinguish ownership from merely having a reference.

The model represents local UI state, not saved orders. Its name makes that separation visible in bindings. Production applications should not present local JSON changes as persisted business transactions.

## 9. Add a view and controller together

Save this as `webapp/view/App.view.xml`:

```xml
<mvc:View
    controllerName="org.example.orders.controller.App"
    xmlns:mvc="sap.ui.core.mvc"
    xmlns="sap.m">
    <App>
        <pages>
            <Page title="{i18n>appTitle}">
                <content>
                    <VBox class="sapUiSmallMargin">
                        <Label
                            text="{i18n>reference}"
                            labelFor="referenceInput"/>
                        <Input
                            id="referenceInput"
                            value="{view>/reference}"
                            maxLength="40"/>
                        <Button
                            text="{i18n>preview}"
                            press=".onPreview"/>
                        <Text
                            text="{view>/reference}"
                            visible="{view>/previewVisible}"/>
                    </VBox>
                </content>
            </Page>
        </pages>
    </App>
</mvc:View>
```

Save the corresponding module as `webapp/controller/App.controller.js`:

```javascript
sap.ui.define([
    "sap/ui/core/mvc/Controller"
], function (Controller) {
    "use strict";

    return Controller.extend("org.example.orders.controller.App", {
        onPreview: function () {
            this.getView().getModel("view")
                .setProperty("/previewVisible", true);
        }
    });
});
```

The preview is deliberately local and has no service side effects. Text is rendered through a standard control rather than injected as HTML. The input length limit improves interaction but is not a security validation strategy for a future backend.

Add `webapp/i18n/i18n.properties`:

```properties
appTitle=Request Preview
appDescription=Preview a locally entered request reference
reference=Request reference
preview=Show preview
```

All view keys now have definitions. Avoid concatenating translated fragments in controllers; future messages should use complete resource-bundle strings with placeholders.

## 10. Provide a standalone bootstrap

For `webapp/index.html`, use external UI5 initialization rather than an inline application script:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Request Preview</title>
    <script
        id="sap-ui-bootstrap"
        src="resources/sap-ui-core.js"
        data-sap-ui-theme="sap_horizon"
        data-sap-ui-async="true"
        data-sap-ui-on-init="module:sap/ui/core/ComponentSupport"
        data-sap-ui-resourceroots='{"org.example.orders":"./"}'>
    </script>
</head>
<body class="sapUiBody">
    <div
        data-sap-ui-component
        data-name="org.example.orders"
        data-id="ordersContainer"
        data-settings='{"id":"ordersComponent"}'>
    </div>
</body>
</html>
```

This expects the development server to expose UI5 framework resources under `resources/`. It will not work merely by opening the file through `file://`. In a launchpad deployment, the shell normally provides bootstrap and component hosting; this HTML page remains a standalone development entry.

The relative resource URL avoids prescribing an unversioned CDN dependency. If a deployment uses an approved CDN, pin an appropriate release and configure content security policy for trusted origins. Do not insert remote scripts from arbitrary hosts to repair a missing library.

## 11. Add services through explicit contracts

For a real service, declare a data source in `sap.app/dataSources` and reference it from a model in `sap.ui5/models`. Record the OData protocol, authentication path, and submission behavior. Do not choose V2 or V4 solely from the application generator's default.

A service URI such as `/api/orders/` assumes a same-origin route or reviewed proxy that actually exists. The descriptor does not create that route. Relative paths also depend on deployment location, so verify them under the application repository or application router used in production.

Never put usernames, passwords, client secrets, or bearer tokens into a descriptor. Browser-delivered files cannot keep secrets. Authentication, token exchange, and protected backend connectivity belong in the hosting architecture and approved destination or proxy configuration.

## 12. Add routing only when the application needs it

When introducing list-to-detail navigation, declare routes and targets in the descriptor, provide a host aggregation such as an `sap.m.App` page aggregation, and initialize the router in the component. Align target view names with namespace-relative paths.

Define a startup route, object route, and meaningful not-found behavior. Route parameters should identify objects rather than contain full business records. Cached views require route-aware rebinding, because initialization alone does not run every time an object changes.

Avoid mixing hash manipulation, manual page creation, and descriptor routing without a clear ownership strategy. Two navigation systems competing for one container produce duplicated pages and unpredictable Back behavior.

## 13. Make dependencies reproducible and reviewable

Pin framework resources and record the reviewed development toolchain in a lockfile. Tooling and generators can execute code during installation or through lifecycle scripts, so provenance and transitive dependencies matter even in a frontend project.

Do not use an unqualified `npx` command as a harmless version check: it may resolve and download a package. Use already provisioned project scripts or the known local executable. Avoid globally installed “latest” tools in repeatable team workflows.

Review dependency changes separately from application feature changes where practical. Check package ownership, licensing, advisory status, engine requirements, and the exact resolved graph. This guide intentionally supplies no commands that fetch or install unreviewed dependencies.

## 14. Verify the initial slice manually

After provisioning approved tools, the application's `start` script should expose the standalone page. Inspect the first failing resource if startup fails. A 404 for `Component.js` suggests namespace or path mapping; a 404 for `sap-ui-core.js` suggests the server's framework-resource configuration.

Confirm the page title, input label, and button come from the expected resource bundle. Enter a reference, choose Show preview, and inspect whether the text appears. This demonstrates local binding only; it does not validate backend connectivity or persistence.

Compare the loaded runtime version with the selected configuration. In a hosted environment, confirm which shell or platform provides UI5. A successful local run does not prove deployment compatibility.

## Common setup pitfalls

Case mismatches often survive on one filesystem and fail on another. Keep `App.view.xml`, its view name, and `App.controller.js` aligned exactly. Do not repair those errors by adding duplicate files with different capitalization.

A descriptor library declaration cannot compensate for missing runtime resources. A tooling dependency cannot compensate for a misspelled XML namespace. Distinguish configuration layers before changing them.

Generated credentials, local proxy files, and development-only mock configuration should never silently become production assets. Review the deployment artifact's contents as well as the source tree.

## Verification checklist

- [ ] Record delivery, exact framework patch, hosting model, and service protocol.
- [ ] Confirm technical identity across namespace, descriptor, component, and paths.
- [ ] Review package scripts and provisioned tool versions without implicit downloads.
- [ ] Keep framework, descriptor schema, and application versions distinct.
- [ ] Check that every referenced view, controller, resource key, and library exists.
- [ ] Confirm bootstrap resources and theme resources come from approved locations.
- [ ] Define ownership and destruction for custom-created models.
- [ ] Keep credentials and private configuration out of browser resources.
- [ ] Verify local startup and target-host compatibility as separate activities.

These are proposed verification steps, not executed test results.

## Official references and related foundations

- [UI5 CLI project configuration](https://ui5.github.io/cli/stable/pages/Configuration/)
- [Official application file overview](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/app-overview-the-basic-files-of-your-app-28b59ca.md)
- [Official folder structure guidance](https://github.com/SAP/openui5-docs/blob/main/docs/05_Developing_Apps/folder-structure-where-to-put-your-files-003f755.md)
- [OpenUI5 project and tooling overview](https://github.com/SAP/openui5)
- [SAP-maintained generator building blocks](https://github.com/SAP/open-ux-tools)

Continue with [framework concepts](./ui5-framework.md), [Fiori design](./fiori-design-system.md), and [development environment](./development-environment.md), the requested sibling paths listed in the [repository README](../README.md).
