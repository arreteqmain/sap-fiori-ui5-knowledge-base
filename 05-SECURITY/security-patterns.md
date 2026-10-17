# Secure Application Patterns for SAP Fiori and UI5

## Scope and assumptions

These patterns describe complete security boundaries for browser-based UI5 applications and their SAP business services. They are design and implementation guidance, not a report of discovered vulnerabilities. Each pattern identifies a risk, a preferred design, an integration pitfall, and evidence that the design works.

Assume an organizational identity provider, HTTPS, supported UI5 and backend releases, and separate development and production identities. Backend examples are intentionally contextual: named application adapters are not undocumented SAP APIs. Use existing reviewed platform components rather than introducing packages from an example.

SAP states that browsers are untrusted and that application code must protect custom rendering and server processing [S1]. Its security integration libraries describe trusted token validation [S2]. Official OData and HTML control documentation defines the client APIs used below [S3–S5]. OWASP guidance supplies complementary authorization, XSS, CSRF, SQL injection, and logging principles [S6–S10].

Read [authentication](authentication.md), [authorization](authorization.md), and [data protection](data-protection.md) for the underlying policies. Use the [code review checklist](code-review-checklist.md) to collect release evidence.

## Pattern: browser session with server-held credentials

**Risk.** A browser application that stores bearer credentials persistently gives any successful same-origin script injection a reusable credential. Shipping a confidential client secret in a bundle is worse: every user can retrieve it without exploiting anything.

**Design.** Put the interactive OAuth client in a supported application router or backend-for-frontend. Use authorization code with S256 PKCE where supported by the selected client and identity platform, exact redirect registration, transaction correlation, and OpenID Connect nonce validation where applicable. Store downstream access and refresh credentials only on the trusted server. Give the browser an opaque session cookie with Secure, HttpOnly, and appropriate SameSite attributes.

SAML federation can sit upstream of this flow. The federation component validates signed assertions and establishes identity; UI5 does not inspect SAML XML to authorize requests. MFA and credential recovery remain at the identity provider.

**Pitfall.** A secure router does not protect an independently exposed backend host. Resource servers must validate their accepted credentials and enforce policy even when the normal UI enters through the router.

**Evidence.** Test direct backend access, session fixation resistance, logout, idle and absolute timeout, MFA recovery, and expired-session handling. Confirm that browser storage, client models, URLs, and logs contain no access or refresh credentials. Do not assume HttpOnly prevents malicious scripts from issuing same-origin requests; it prevents reading the cookie, not using the session.

## Pattern: trusted JWT middleware with narrow acceptance

**Risk.** Decoding a JWT and reading a role claim can accept forged, expired, or wrong-audience credentials. Accepting any signing algorithm named by the token can create algorithm-confusion problems.

**Design.** Put supported resource-server middleware before all protected handlers. Configure trusted issuer metadata, permitted audiences, approved algorithms, expiration and not-before validation, bounded skew, and token purpose. Use the SAP integration libraries only within their documented supported environments [S2], or another platform-supported integration approved for the actual backend.

Keep key discovery rooted in trusted configuration. Do not fetch keys from an arbitrary URL supplied in an unvalidated token. Handle signing-key rotation through the library's supported mechanisms and monitor failures without copying tokens into logs.

**Pitfall.** Signature validation alone is not authorization. A correctly signed token for another service or tenant must fail; an accepted token without the relevant business permission must also fail. A client-side JWT check is presentation logic, never a trusted API decision.

**Evidence.** Use authorized test fixtures for invalid signatures, disallowed algorithms, expired tokens, wrong issuers, wrong audiences, and unexpected tenant relationships. Confirm that every rejection occurs before business side effects. Use maintained test utilities rather than implementing production crypto for the sake of testing.

## Pattern: deny-by-default business service

**Risk.** A hidden approval button gives the appearance of control while direct API calls remain possible. A broad “Approver” role may also permit access to documents outside the intended organization.

**Design.** Combine capability checks with object, tenant, organizational, state, amount, and separation-of-duties checks at the backend business boundary. Load sensitive object attributes from trusted persistence, not a request body asserting the object's owner or organization.

The following **server-side JavaScript composition sketch** uses application-owned adapters. `security` delegates authentication and CSRF handling to the selected platform, while `policy` and `repository` implement reviewed business rules and transactional persistence:

```javascript
function registerApproval(app, security, policy, repository, audit) {
    app.post(
        "/api/orders/:id/approve",
        security.authenticateRequest,
        security.requireCsrfForSession,
        async (req, res, next) => {
            try {
                const result = await repository.inTransaction(async tx => {
                    const order = await tx.loadForApproval(
                        req.principal.tenantId,
                        req.params.id
                    );
                    await policy.requireApproval(req.principal, order);
                    await tx.approveWithConcurrencyCheck(order);
                    await tx.enqueueAudit(audit.approvalEvent(
                        req.principal, order, req.correlationId
                    ));
                    return { id: order.id, status: "Approved" };
                });
                res.set("Cache-Control", "no-store");
                res.json(result);
            } catch (error) {
                next(error);
            }
        }
    );
}
```

The example's error mapper must return controlled responses and sanitized diagnostics. The repository must validate identifier syntax, restrict lookup to the trusted tenant, enforce transaction and concurrency semantics, and avoid unauthorized existence disclosure. The policy must reject missing documents and unmet permissions rather than treating them as ordinary success.

**Pitfall.** Running the authorization check outside a transaction can leave a race between checking an eligible state and saving an approval. The transactional audit enqueue is an application outbox contract, not a claim that an external logger participates in the database transaction.

**Evidence.** Test direct invocation, wrong organization, self-approval, stale ETags or versions, concurrent state changes, and revoked permissions. Inspect database state, emitted events, and audit output, not only HTTP status.

## Pattern: real ABAP authorization objects

**Risk.** A generic sample object copied into every service can create a check that has no relationship to the business operation. Checking only service-start permission can leave document actions unrestricted.

**Design.** Determine actual business authorization objects, fields, activity values, and organizational constraints from the target SAP system and its approved security design. In classic ABAP, coordinate implementation with PFCG role maintainers and authorization proposals. In ABAP Cloud, use the supported IAM and released programming model.

The following is a **classic ABAP template**, not production-ready code. Replace all markers with approved system definitions; the application-specific exception must be mapped safely by the service:

```abap
" Existing document context; lv_org comes from trusted document data.
AUTHORITY-CHECK OBJECT '<SYSTEM_BUSINESS_OBJECT>'
  ID '<ACTIVITY_FIELD>' FIELD '<ACTION_VALUE>'
  ID '<ORGANIZATION_FIELD>' FIELD lv_org.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_app_access_denied.
ENDIF.
```

**Pitfall.** Do not assume direct table reads enforce end-user business authorizations. CDS DCL controls appropriate reads, while RAP authorization behavior and explicit checks protect relevant operations. Access-control annotations without the intended active policy are not complete evidence [S11].

**Evidence.** Verify narrow authorized and denied role fixtures, inspect effective organizational values, and test the actual service endpoint. Review any privileged or local internal execution path separately. Never resolve a failed test by granting broad administrator rights.

## Pattern: plain text by default

**Risk.** Backend comments, supplier names, and error messages can become stored XSS if inserted as HTML. Data from an authenticated SAP backend is not automatically safe markup.

**Design.** Use standard UI5 text properties for ordinary business content. For a custom control that genuinely needs a renderer, use supported structured RenderManager APIs. The following **UI5 module** demonstrates a modern renderer for a text-only control:

```javascript
sap.ui.define(["sap/ui/core/Control"], function (Control) {
    "use strict";
    return Control.extend("example.control.SafeNote", {
        metadata: {
            properties: {
                text: { type: "string", defaultValue: "" }
            }
        },
        renderer: {
            apiVersion: 2,
            render: function (rm, control) {
                rm.openStart("span", control);
                rm.openEnd();
                rm.text(control.getText());
                rm.close("span");
            }
        }
    });
});
```

The text value is passed as text, not concatenated into markup. Prefer an existing standard control when it meets the requirement; a custom control brings lifecycle and accessibility responsibilities.

**Pitfall.** Expression binding is not an XSS sanitizer. Its result still flows into a property whose rendering contract matters. Likewise, encoding for HTML does not make a value safe as JavaScript, CSS, or an arbitrary URL.

**Evidence.** Test comments containing angle brackets, quotes, ampersands, and synthetic markup indicators. Confirm they display as text without executing behavior. Include error views, tooltips, print views, and custom post-render code.

## Pattern: constrained rich content

**Risk.** Replacing a text control with `sap.ui.core.HTML` can silently change a safe field into a raw HTML sink. Its documented automatic sanitization default is false [S5].

**Design.** First challenge whether rich HTML is necessary. For a genuine business requirement, choose the supported rich-content control or sanitizer available in the deployed UI5 release, explicitly activate required sanitization, constrain permitted markup and URL destinations, and preserve the sanitizer's output without unsafe post-processing.

Treat rich-content policy as an interface contract. Decide whether images, external links, tables, and formatting are permitted. Restrict links to approved schemes and destinations. A sanitizer that permits HTTP links does not decide whether the organization should allow external supplier-controlled destinations.

**Pitfall.** A regular expression that removes a few tags is not an HTML parser or a robust sanitizer. Double encoding can corrupt business text without solving the original unsafe sink. Sanitizing HTML also does not authorize the underlying data or make arbitrary embedded URLs appropriate.

**Evidence.** Use a synthetic content corpus that covers disallowed tags, event attributes, malformed markup, URL schemes, and legitimate formatting. Verify behavior on the actual supported UI5 version and target browsers.

## Pattern: model-managed OData security

**Risk.** Manual request construction can bypass the OData model's supported CSRF behavior or mix APIs from different models. Disabling CSRF to make a write succeed removes a protection rather than fixing integration.

**Design.** Use the correct model's documented write mechanism and keep server CSRF enforcement enabled. The following **UI5 OData V2 controller excerpt** assumes `sap/m/MessageBox` is an imported dependency named `MessageBox`, a same-origin configured model, and server-owned enforcement:

```javascript
function createOrderV2(model, validatedDescription) {
    model.create("/Orders", {
        Description: validatedDescription
    }, {
        success: function () {
            MessageBox.success("Order created.");
        },
        error: function () {
            MessageBox.error("The order could not be created.");
        }
    });
}
```

The description is the only client-supplied field in this example. The backend assigns identity, tenant, status, totals, and other controlled properties. Client validation improves feedback but does not replace backend validation.

For **UI5 OData V4**, list-binding creation uses a different API. These functions assume a view-owned V4 list binding for `/Orders`, configured with `$$updateGroupId: "orderWrites"`, and an `orderWrites` application group using API submission mode. `messageBox` is the imported MessageBox module, and backend messages are integrated with normal application message handling. Begin once per draft; save the same transient entity when the user deliberately retries:

```javascript
function beginOrderV4(binding, description) {
    return binding.create({ Description: description });
}

async function saveOrderV4(binding, view, messageBox) {
    view.setBusy(true);
    try {
        await binding.getModel().submitBatch("orderWrites");
        if (binding.hasPendingChanges()) {
            messageBox.error("The order was not saved. Review the messages.");
            return;
        }
        messageBox.success("Order created.");
    } catch (_) {
        messageBox.error("The order could not be submitted.");
    } finally {
        view.setBusy(false);
    }
}
```

Retain the returned transient context for form binding and cancellation, and retain the list binding for the form's lifecycle. The group must contain only this form's intended pending writes. Disable duplicate create and save actions during submission, then support deliberate correction, retry, or cancellation. Clean up an application-owned binding at the end of its lifecycle, not after each failed save.

Do not use the batch promise alone as proof that an individual create succeeded; inspect pending changes and the framework's message state. `Context.created()` does not reject for an ordinary server-side create failure: the transient entity can remain queued for a later submission. SAP documents this retry behavior and the `createSent` and `createCompleted` events [S13]. Reconcile uncertain outcomes before retrying a non-idempotent operation.

**Pitfall.** V2 `refreshSecurityToken` does not exist as the same application API on the V4 model. An OAuth access token is not an OData CSRF token. Custom uploads and fetch requests need their own supported CSRF integration, even when model writes already work.

**Evidence.** Test success, validation failure, missing or expired CSRF tokens, backend session expiration, and batch-level versus operation-level failures. Confirm no uncontrolled replay of a non-idempotent create.

## Pattern: parameterized persistence and bounded queries

**Risk.** Concatenating a search term, sort field, or document identifier into a database query can introduce SQL injection. Even a parameterized query can disclose another tenant's data if its authorization predicate is missing.

**Design.** Bind values through prepared statements or the framework's structured query API. Use a fixed server mapping for external sort choices and projections. Derive tenant and organizational restrictions from the trusted principal and effective authorization context.

For ABAP, use static ABAP SQL with host variables where appropriate and the intended authorized entity. For a Java repository, use JDBC prepared statements or a supported persistence framework. Never concatenate an OData filter string into SQL.

Bound paging, expansion depth, filter complexity, export volume, and execution time. An injection-safe query can still consume excessive resources. Keep counts and aggregation within the same authorized row boundary.

**Pitfall.** Input validation is not a substitute for parameterization. Conversely, parameterization is not a substitute for checking business permission or constraining expensive operations.

**Evidence.** Use synthetic quoted text and boundary-length inputs, verify fixed query structure, and test cross-tenant and unauthorized-organizational reads. Confirm limits work for batch and export paths, not only interactive lists.

## Pattern: minimal responses and safe observability

**Risk.** Masking an account number in a formatter while returning its raw value still discloses it. Logging an entire failed request can replicate that disclosure in a more widely accessible system.

**Design.** Build server responses from explicitly permitted fields. Return masked representations to ordinary users and require separate authorization for full-value reveal. Apply appropriate no-store caching to sensitive session-specific responses.

Create structured audit events with approved actor references, event categories, outcomes, timestamps, and generated correlation identifiers. Map exceptions to controlled reason codes. Do not record full request objects, Authorization headers, cookies, tokens, SQL text, or business payloads.

Client messages should explain the failed operation without including backend stack traces or raw exception text. A support reference can connect the user to restricted server diagnostics without exposing those diagnostics.

**Pitfall.** A structured logger does not automatically redact nested sensitive objects. Error monitoring can silently transmit business data to external collectors. Review actual serialized output and approved destinations.

**Evidence.** Inspect responses, browser models, cache headers, logs, export files, and error telemetry using synthetic data. Verify retention, restricted log access, and the operational response to audit-sink failure.

## Pattern: defense-in-depth browser policies

**Risk.** An unsafe rendering path or unexpected framing context can increase the impact of another defect. Broad CORS and CSP rules can hide integration mistakes while weakening isolation.

**Design.** Deploy a tested Content Security Policy based on actual UI5 release requirements and resource hosts [S12]. Prefer asynchronous loading and avoid inline scripts and dynamic evaluation in application code. Start with report-only observation in a controlled rollout, then enforce the reviewed policy.

Configure clickjacking protection according to legitimate launchpad embedding. Use restricted CORS origins where cross-origin access is genuinely needed. Keep TLS, certificate validation, secure cookies, and appropriate response headers across all routes.

**Pitfall.** CSP is not an XSS sanitizer, CORS is not authentication, and minification is not security. A strict-looking header that breaks required flows encourages unsafe emergency exceptions; test the integrated landscape before rollout.

**Evidence.** Exercise login, launchpad embedding, OData, exports, fonts, attachments, and approved external navigation. Review reports without collecting unnecessary full URLs or personal information.

## Pattern: authorized adversarial validation

**Risk.** UI-only testing misses direct API access, while uncontrolled security testing can damage systems or expose real business data.

**Design.** Define written authorization before penetration testing: systems, tenants, accounts, methods, time window, rate limits, data handling, contacts, stop conditions, and cleanup. Use synthetic users with narrow roles and synthetic documents in distinct organizations.

Build a test matrix across authentication, authorization, input context, session state, and transaction state. Include positive controls so rejection is not confused with a broken service. Observe business outcomes as well as status codes.

Test wrong-resource access, forbidden properties, stale sessions, expired CSRF context, concurrent approval, failed creates, and restricted exports. Preserve only minimal sanitized evidence. If unexpected real data appears, stop the affected activity and follow the agreed reporting procedure rather than exploring further.

**Pitfall.** A penetration test is not permission to scan unrelated SAP hosts or reuse employee sessions. A clean automated scan does not demonstrate organizational authorization or separation-of-duties correctness.

**Evidence.** Retain approved scope, fixture definitions, expected and actual results, sanitized correlation references, remediation ownership, and a targeted retest record.

## Pattern selection checklist

- [ ] Use server sessions and server-held credentials for the ordinary browser flow.
- [ ] Validate JWTs through supported trusted middleware before business routes.
- [ ] Enforce object, action, tenant, organization, and state policy on the backend.
- [ ] Use actual system business authorization definitions for ABAP.
- [ ] Prefer text sinks and explicitly constrain any rich content.
- [ ] Keep OData V2 and V4 APIs separate and CSRF enforcement enabled.
- [ ] Parameterize persistence and bound resource-intensive requests.
- [ ] Minimize responses, cache exposure, and diagnostic content.
- [ ] Test browser policies in the real launchpad deployment.
- [ ] Perform adversarial validation only within written authorization.

## Sources

- **S1 — SAPUI5 Browser Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/browser-security-91f3b66.md
- **S2 — SAP BTP Security Services Integration Libraries:** https://github.com/SAP/cloud-security-services-integration-library/blob/main/README.md
- **S3 — OpenUI5 OData V2 model documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js
- **S4 — OpenUI5 OData V4 model documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js
- **S5 — OpenUI5 HTML control documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/core/HTML.js
- **S6 — OWASP Authorization Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Authorization_Cheat_Sheet.md
- **S7 — OWASP XSS Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.md
- **S8 — OWASP CSRF Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md
- **S9 — OWASP SQL Injection Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.md
- **S10 — OWASP Logging Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Logging_Cheat_Sheet.md
- **S11 — SAP ABAP Authorization Checks:** https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md
- **S12 — SAPUI5 Content Security Policy:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/content-security-policy-fe1a6db.md
- **S13 — SAPUI5 Creating an Entity in a Collection:** https://github.com/SAP-docs/sapui5/blob/main/docs/04_Essentials/creating-an-entity-in-a-collection-c9723f8.md
