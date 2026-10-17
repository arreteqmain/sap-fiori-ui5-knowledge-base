# SAP Fiori and UI5 Glossary

## Scope and usage

This glossary defines vocabulary used throughout the knowledge base. Its
audience includes developers, reviewers, business analysts, and AI agents
assembling implementation guidance. Definitions describe responsibilities and
boundaries, not universal product availability. Always establish the UI5 release,
OData protocol version, installed SAP software components, and deployment target
before translating a term into code.

Terms are grouped by engineering concern so that retrieval can retain useful
context. An acronym alone is not a service contract: for example, “BAPI” does not
prove that a particular function is released for remote use, and “Fiori” does not
prove that an application uses Fiori elements. See the
[framework guide](01-FOUNDATIONS/ui5-framework.md) for the runtime model and the
[compatibility matrix](07-BACKPORTING/compatibility-matrix.md) for platform
qualification.

## Framework, architecture, and rendering

### SAP Fiori

SAP's user experience approach and design system for business applications.
Fiori emphasizes role-based, adaptive, simple, coherent, and delightful experiences.
It is not a JavaScript framework or a synonym for every SAP web application.
A technically functioning UI5 screen still needs design and accessibility review.

### OpenUI5

The open-source UI5 framework distribution, licensed under Apache License 2.0.
It provides controls, models, binding, routing, and modular application
infrastructure. It does not contain every SAPUI5 library. Check the framework
distribution before selecting analytical, enterprise-specific, or specialized
controls for an implementation.

### SAPUI5

SAP's UI5 distribution, which includes OpenUI5 functionality and additional
libraries under applicable SAP terms. Licensing and runtime availability must
be checked separately from API familiarity. A sample using `sap.ui.comp` cannot
automatically be deployed against a runtime containing only OpenUI5 libraries.

### UI5 runtime

The browser-loaded framework resources that create controls, resolve modules,
manage bindings, and render the application. The runtime version is independent
of the Node.js version running a development server. Build tools do not
automatically upgrade the runtime supplied by an on-premise launchpad.

### UI5 Tooling

The development-time project, server, and build tooling ecosystem, commonly
accessed through the UI5 CLI. It resolves resources and framework dependencies,
serves development content, and creates optimized output. Tooling compatibility
and runtime API compatibility are separate questions; record both in project
documentation.

### MVC: Model–View–Controller

An architectural separation in which models expose data, views declare
presentation, and controllers coordinate user interactions. A UI5 controller is
not the correct place for database authorization or accounting rules. Keep
service access and reusable transformations separate from event-handler wiring.

### Component

An encapsulated UI5 application or reusable unit with its own metadata and
lifecycle. A `UIComponent` can own models, a root control, and routing.
Component boundaries help isolate dependencies and ownership; they do not provide
a browser security boundary between untrusted scripts.

### Application descriptor

Usually `manifest.json`, the declarative configuration describing an application's
identity, dependencies, models, data sources, and routing. Descriptor sections
belong to distinct schemas, such as `sap.app` and `sap.ui5`. JSON validity alone
does not establish that the descriptor matches a supported runtime feature.

### Control

A UI5 object that participates in the control tree and renders user-interface
content. Controls expose metadata-defined properties, aggregations, associations,
and events. Prefer supported control APIs over manipulating generated DOM:
rerendering can replace nodes and invalidate manually attached behavior.

### Property

A typed metadata value belonging to a managed object, such as a button's
`enabled` state. Binding a property connects it to model data. Setting a property
through a supported setter allows UI5 to handle validation, change propagation,
and invalidation consistently.

### Aggregation

A managed parent–child relationship used to compose controls or other managed
objects. Aggregations define cardinality and ownership rules, including normal
destruction behavior. Adding a child to an aggregation is different from storing
an arbitrary reference to another control.

### Association

A relationship expressed through the ID of another managed object, without
aggregation ownership. Labels and accessibility relationships often use
associations. An association does not imply the associated object will be
destroyed with the referring object; lifecycle ownership must remain explicit.

### Invalidation

The framework's mechanism for marking rendered content as needing an update.
Changing a relevant property can trigger invalidation and later rendering.
Avoid forcing synchronous rendering repeatedly in loops; optimize the control
structure and binding updates before interfering with normal scheduling.

### Renderer and RenderManager

A renderer describes a control's output using UI5's rendering infrastructure.
`RenderManager` provides structured methods for elements, text, and attributes.
Use escaping-aware APIs rather than inserting untrusted HTML. A custom control
also needs keyboard, focus, and accessibility behavior, not merely visual output.

### Fragment

A reusable UI definition without a controller lifecycle of its own. Fragments
can reuse an owning view's controller and require careful ID handling.
Dialogs loaded from fragments should have explicit ownership and cleanup.
Fragments are not independently deployable components.

### AMD and module ID

Asynchronous Module Definition is the dependency-oriented model used by
`sap.ui.define` and `sap.ui.require`. A module ID uses slash-separated names,
such as `sap/ui/model/json/JSONModel`. Declaring dependencies explicitly enables
asynchronous loading and avoids reliance on globals created by accidental load
order.

### Namespace and resource root

A namespace organizes application identifiers; a resource root maps module
prefixes to resource locations. Their relationship must be consistent with the
folder structure, component name, and loader configuration. Renaming only a
directory can leave controllers and views resolving under obsolete module IDs.

## Models, protocols, and transactions

### JSONModel

A UI5 client-side model for JavaScript data, commonly used for view state,
selection summaries, or small local datasets. It does not automatically persist
changes to a backend. Avoid copying entire sensitive server datasets into it
merely to simplify presentation logic.

### ResourceModel

A model exposing localized text resources, typically from properties files.
It separates interface strings from application code. Translation keys should
describe meaning rather than a particular English sentence, and parameterized
texts must account for language-specific ordering and grammar.

### Binding context

The data location against which relative bindings resolve. An object-page view
can receive a context so child controls bind to fields without repeating the
absolute entity path. A route parameter is not itself a binding context and
must be validated and resolved appropriately.

### Property, context, and list binding

Property bindings address individual values, context bindings address objects,
and list bindings address collections. Their available operations depend on the
model implementation. Treat the binding's protocol-specific API as part of
the contract rather than assuming all models expose interchangeable methods.

### One-way and two-way binding

One-way binding propagates model changes to controls; two-way binding additionally
allows supported control edits to update model values. Neither mode guarantees
business validity or persistence. A two-way edit may remain pending locally
until a service-specific submission operation succeeds.

### Formatter and expression binding

A formatter transforms model values for display; expression binding performs
supported declarative calculations in binding syntax. Neither is a general
HTML sanitizer or an authorization mechanism. Output safety depends on the
receiving control property and the rendering or URL context.

### OData

A standardized protocol for describing and interacting with data services.
Metadata, entity identity, query options, and operations form a discoverable
contract. OData does not remove the need for business authorization, validation,
transaction design, and protection against expensive or abusive queries.

### OData V2 versus V4

Distinct protocol versions with different UI5 model implementations. The V2
model's `read` and `submitChanges` patterns are not V4 APIs. V4 commonly uses
bindings, contexts, and `submitBatch`. Choose examples matching both the service
version and the model class.

### Entity, entity set, and key

An entity is an identifiable service object; an entity set is its addressable
collection. Keys follow the service metadata and can be composite. Preserve
business identifiers as strings when appropriate: numeric conversion can discard
leading zeros or exceed JavaScript's safe integer precision.

### Navigation property

A metadata-defined relationship from one entity to related entities.
Navigation is not arbitrary path concatenation. Use supported binding paths and
service query mechanisms, and confirm which relationships are actually exposed.
An underlying database relationship need not be part of the public service.

### `$select` and `$expand`

OData query options for choosing properties and requesting related data.
They can reduce requests but can also create excessive payloads or costly
joins. Select only necessary information, review authorization at each exposed
entity, and measure the service's actual execution behavior.

### Batch request and change set

A batch packages multiple OData operations into one transport request.
Transport grouping does not mean every operation is one atomic transaction.
V2 change sets and V4 atomicity mechanisms require protocol- and
service-specific understanding; inspect individual operation results.

### ETag and optimistic concurrency

An ETag identifies a representation version. Conditional changes let a service
reject updates made against stale versions, commonly through `If-Match`.
Handle conflicts by refreshing and reconciling user intent; do not suppress
conflict protection by routinely overwriting other users' changes.

### Draft

A backend-supported editing lifecycle that separates incomplete work from active
business data. Draft ownership, activation, validation, and cleanup are service
capabilities. Adding an Edit button or storing a temporary JSON copy does not
implement SAP draft semantics.

### Idempotency

The property that repeating an operation has the same intended business effect
as performing it once. A timed-out mutation may already have committed.
Use service-supported idempotency keys or reconciliation; disabling double-clicks
in the UI alone cannot prevent duplicate postings.

### LUW: Logical Unit of Work

A backend transaction boundary organizing business changes and commit or rollback.
An HTTP request, batch, and SAP LUW are not automatically equivalent.
Wrappers around business APIs must preserve the documented transaction and
message-handling contract rather than issuing arbitrary commits.

## Experience, navigation, and accessibility

### Freestyle and Fiori elements

Freestyle applications assemble UI5 controls and application logic explicitly.
Fiori elements provides metadata-driven application patterns with supported
extension mechanisms. Neither approach eliminates backend responsibilities.
Choose based on workflow fit, metadata quality, supported release capabilities,
and the long-term cost of custom interaction behavior.

### Worklist, list report, and object page

A worklist focuses on actionable items; a list report provides structured
discovery and filtering; an object page presents one object's details and actions.
These are interaction patterns, not guarantees about particular controls.
Freestyle and Fiori elements implementations have different configuration models.

### Router, route, and target

A router interprets application navigation state. A route matches a pattern,
while a target describes content to display. Deep-linking requires correct
state reconstruction when no previous screen exists. Avoid depending on a
selected list item being present after a direct URL entry.

### Semantic object and intent

Launchpad navigation can identify a business concept and action through an
intent, resolved by target mappings. Intent parameters should not contain secrets.
The same intent may resolve differently for different roles or landscapes;
it is not necessarily a fixed application URL.

### Responsive versus adaptive

Responsive behavior adjusts layout to available space; adaptive behavior can
change interaction or information presentation for the device context.
A table that technically fits is not necessarily usable on a handheld scanner.
Validate representative workflows, input methods, and interruption scenarios.

### WCAG, ARIA, and accessible name

WCAG describes accessibility success criteria. ARIA conveys semantics and state
to assistive technology when used correctly. An accessible name identifies a
control for a user who may not see it. Prefer standard UI5 semantics and
explicit labels over adding conflicting ARIA manually.

### Focus management

The deliberate placement and restoration of keyboard focus during interactions.
Dialogs, errors, navigation, and asynchronous content changes need predictable
behavior. A visually prominent notification is insufficient if keyboard or
screen-reader users cannot discover it or return to their previous task.

### Theme and design token

A theme supplies coordinated visual values; design tokens represent semantic
choices such as colors or spacing. Use supported theming parameters rather than
hardcoded colors that may fail high-contrast or dark themes. Visual customization
still requires contrast and usability checks.

## Identity, security, and operations

### Authentication and authorization

Authentication establishes who or what is acting; authorization determines
whether that principal may perform a specific operation on a specific object.
Hiding a control can improve usability but does not enforce authorization.
Every backend operation must validate the caller independently.

### SAML and identity provider

SAML supports federated authentication through signed assertions exchanged by
trusted parties. An identity provider authenticates users; a service provider
consumes the identity result. Applications should use the platform's established
integration rather than parsing assertions or implementing signature checks in
browser code.

### OAuth, OIDC, and PKCE

OAuth delegates access through tokens and scopes. OpenID Connect adds an
identity layer. PKCE binds an authorization-code exchange to its initiating
client. These mechanisms solve different problems; an OAuth access token is
not automatically an application session or proof of every business permission.

### JWT

A structured token format that can carry signed claims. Decoding a JWT is not
validation. A trusted server must verify the applicable signature, allowed
algorithm, issuer, audience, validity interval, and other required claims using
maintained platform libraries and configured trust.

### Session and secure cookie

A session associates requests with authenticated server-side state or validated
credentials. Cookies carrying session identifiers should use appropriate
`Secure`, `HttpOnly`, and `SameSite` settings. Logout, expiration, and reauthentication
must account for identity-provider and application-session behavior separately.

### CSRF

Cross-Site Request Forgery causes a browser to send an unwanted authenticated
request. Cookie-based applications need suitable defenses, including the
platform's CSRF-token handling where applicable. A CSRF token is not a bearer
access token, and protection must not be disabled to simplify development.

### XSS and output encoding

Cross-Site Scripting executes attacker-controlled content in a trusted origin.
Output encoding must match the destination, such as text, attributes, or URLs.
Standard text controls are preferable to raw HTML rendering. Content Security
Policy adds defense in depth but does not excuse unsafe rendering.

### CORS and CSP

Cross-Origin Resource Sharing controls which browser origins may read selected
cross-origin responses. Content Security Policy constrains resource loading and
execution in a document. Neither replaces backend authentication or authorization,
and permissive development settings should not become production defaults.

### PFCG role and authorization object

PFCG manages ABAP authorization roles and associated profiles. Authorization
objects group fields checked by backend authorization logic. Application tiles,
service access, and business-object permissions require coordinated configuration;
granting a launchpad entry does not establish permission to post a document.

### Least privilege and segregation of duties

Least privilege grants only necessary access. Segregation of duties separates
conflicting business responsibilities, such as vendor maintenance and payment
approval. Design these controls with business owners and enforce them in the
backend, including technical users and integration paths.

### Audit trail and correlation ID

An audit trail records accountable business or security events according to
retention and integrity requirements. A correlation ID connects related
technical events without exposing secrets. Diagnostic logs are not automatically
an adequate audit record and should avoid unnecessary personal or financial data.

## SAP business systems and delivery

### ECC and S/4HANA

ECC is the SAP ERP Central Component product family; S/4HANA is a different
ERP product generation with changed capabilities and data models.
A similar screen does not imply equivalent service contracts. Establish the
installed component versions and supported application variant before planning
reuse or backporting.

### ABAP CDS and HANA

ABAP Core Data Services defines semantic data models on supported ABAP platforms.
Available features depend on release and implementation; CDS availability is
not determined solely by using HANA. Database-specific functions, analytical
capabilities, and released consumption models require separate qualification.

### SAP Gateway and SEGW

SAP Gateway provides OData-related infrastructure in relevant ABAP landscapes.
SEGW is a service-development tool commonly associated with classic OData
implementations. Gateway can be deployed in different topologies; service
registration, routing, system aliases, and backend permissions must match the
chosen landscape.

### BAPI, RAP, and CAP

BAPIs expose defined SAP business operations where released and applicable.
RAP is the ABAP RESTful Application Programming Model. CAP is the SAP Cloud
Application Programming Model. These are different backend approaches, not
interchangeable endpoints. Choose an integration contract that the target system
actually supports.

### BTP, Cloud Foundry, and HTML5 Application Repository

SAP Business Technology Platform hosts services and application capabilities.
Cloud Foundry is one runtime environment. The HTML5 Application Repository
stores application content for supported delivery arrangements; routing,
destinations, identity, and backend connectivity remain separate concerns.
Business Application Studio is a development environment, not the production target.

### Destination and approuter

A destination describes connectivity and authentication settings for a remote
system in supported BTP scenarios. An application router routes requests and
integrates platform authentication. Keep credentials in approved server-side
configuration and validate user propagation rather than embedding backend
passwords in a browser application.

### CI/CD and artifact promotion

Continuous integration checks changes; continuous delivery prepares repeatable
releases with appropriate approval gates. Artifact promotion moves the same
verified build between environments. Rebuilding independently for production
can invalidate earlier evidence; track source revision, dependency lock,
configuration, approvals, and rollback procedures.

### QUnit and OPA5

QUnit supports JavaScript unit testing. OPA5 supports UI5 integration journeys
using asynchronous waits and application interactions. Unit checks cannot prove
backend authorization, and UI journeys cannot replace focused business tests.
See the [testing strategy](06-BEST-PRACTICES/testing-strategy.md) for layering.

## Applying the vocabulary

An illustrative review note might read: “This freestyle component binds an
object page to an OData V2 entity context. Save submits pending changes with
optimistic concurrency. Gateway enforces authorization and the backend controls
the LUW.” Each sentence identifies a responsibility that can be verified.
Avoid replacing this precision with “the Fiori app handles everything.”

```javascript
sap.ui.define(["sap/ui/model/json/JSONModel"], function (JSONModel) {
    "use strict";
    return function createViewState() {
        return new JSONModel({
            busy: false,
            editing: false,
            validationMessage: ""
        });
    };
});
```

This complete factory module creates local presentation state. It does not
authenticate users, implement drafts, write business records, or establish
authorization. Import it through a component or controller, assign the returned
model with an explicit name, and destroy it with its owner. Distinguishing what
a snippet does **not** provide is essential when an AI agent assembles examples.

## Authoritative references and related guides

- [OpenUI5 project and distribution](https://github.com/SAP/openui5): framework
  scope, licensing, and links to the developer guide and API reference.
- [OpenUI5 API reference](https://sdk.openui5.org/api): check the target release's
  classes, methods, events, and metadata.
- [UI5 CLI documentation](https://ui5.github.io/cli/): project and tooling concepts.
- [SAP Fiori design guidelines](https://experience.sap.com/fiori-design-web/):
  interaction patterns and design rationale.
- [SAP Fiori Apps Reference Library](https://fioriappslibrary.hana.ondemand.com/):
  application variants and implementation prerequisites.
- [OData documentation](https://www.odata.org/documentation/): protocol vocabulary.
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/): accessibility criteria.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/): security guidance.

For implementation detail, continue with
[data binding](02-ARCHITECTURE-PATTERNS/data-binding.md),
[authentication](05-SECURITY/authentication.md),
[authorization](05-SECURITY/authorization.md), and
[deployment targets](10-TOOLS-DEPLOYMENT/deployment-targets.md).
Official references describe evolving products; definitions here do not replace
release-specific documentation, licensed product terms, or business process
approval.
