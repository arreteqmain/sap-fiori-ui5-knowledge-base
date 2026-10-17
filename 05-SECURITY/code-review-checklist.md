# Security Code Review Checklist for SAP Fiori and UI5

## Scope and assumptions

This document is a practical review method for application changes, not a vulnerability report about this repository. It covers UI5 views and controllers, custom controls, application-router integration, OData services, server handlers, and business authorization. Use it with the actual deployment architecture and installed SAP release.

A reviewer needs the change's purpose, affected routes, data classification, trusted authentication component, and expected business policy. A screenshot of a disabled button is not evidence that its endpoint is protected. A scanner result is useful input, not a replacement for understanding the request path.

The review is grounded in SAP's browser and transport security guidance [S1–S2], official UI5 API documentation [S3–S5], and OWASP guidance for XSS, CSRF, SQL injection, authorization, and logging [S6–S10]. For design detail, read [authentication](authentication.md), [authorization](authorization.md), [data protection](data-protection.md), and [security patterns](security-patterns.md).

## Establish the change boundary

Start by identifying what new trust crossing the change introduces. A formatter that now produces HTML, an export endpoint that reads more fields, or a controller that replaces an OData call with fetch can alter the security model even when the business feature looks small.

List the entry points and sinks touched by the change. Entries include route parameters, query options, uploaded files, backend text, translation content from external systems, and cross-window messages. Sinks include HTML, URLs, database queries, logs, file paths, outbound requests, and privileged business actions.

Review generated configuration and effective deployment behavior as well as source. A secure handler behind a public bypass route is not secure in production. Confirm which assumptions are enforced by code and which require infrastructure configuration.

- [ ] State the new capability and business owner.
- [ ] Identify affected services, model versions, and public routes.
- [ ] Classify data read, written, logged, exported, and cached.
- [ ] Locate the trusted principal and each backend enforcement point.
- [ ] Document the deployment assumptions and their owners.

## Review authentication before reading claims

Find the middleware that authenticates every protected route. It should use the deployment's supported SAP or platform integration, not a utility that merely decodes a JWT payload. Verify signature, trusted issuer, audience, approved algorithms, expiration, applicable not-before rules, and bounded clock skew.

Check that authentication runs before business code and cannot be bypassed through another route or direct service host. A header populated by an application router is trustworthy only under controlled transport, ingress, and header-overwrite rules.

Browser code must not contain a client secret or become the authority for validating tokens. Review manifests, build-time substitutions, destination URLs, mock configuration, and source maps. Minification is not protection for credentials.

- [ ] Use authorization code with S256 PKCE and supported transaction validation.
- [ ] Validate exact redirect destinations and issuer trust.
- [ ] Keep access and refresh credentials out of browser storage.
- [ ] Verify Secure, HttpOnly, and appropriate SameSite session cookies.
- [ ] Enforce idle and absolute session expiry on the server.
- [ ] Verify logout, MFA, step-up, and controlled reauthentication behavior.

## Review authorization at each business operation

Trace a protected action from the UI event to the backend mutation. Identify the capability check, authorized resource lookup, organizational constraints, tenant boundary, state validation, and concurrency control. The service must deny direct API invocation when those conditions are not met.

For classic ABAP, verify the actual business authorization object and field values used in the target system. Do not accept a generic invented object or a broad wildcard as evidence. Capture `sy-subrc` immediately and deny on nonzero results. For CDS and RAP, inspect active access-control and authorization artifacts and their documented runtime behavior.

Review property-level writes. A client can submit fields that the UI hides or marks read-only. Ownership, approver, tenant, total, and workflow status often need server control rather than ordinary mass assignment.

- [ ] Test no-role, read-only, and narrowly scoped users.
- [ ] Check object identifiers, organizational transfers, and cross-tenant requests.
- [ ] Cover actions, attachments, exports, `$expand`, `$count`, and batch operations.
- [ ] Confirm denied requests produce no database, event, or downstream side effects.
- [ ] Treat tiles, UI capability models, and visibility bindings as presentation only.

## Trace XSS by source and rendering context

Do not ask only whether a value was “escaped.” Ask where it is rendered. Text, HTML, HTML attributes, URLs, JavaScript, and CSS have different parsing rules. A value safe for a text property is not automatically safe inside an HTML string.

Standard UI5 controls encode ordinary text properties through their renderers, subject to each control's documented contract [S1]. Prefer those controls. Review escape hatches such as `sap.ui.core.HTML`, DOM manipulation, custom renderers, inline event attributes, and HTML assembled by formatters.

The following **UI5 XML view fragment** keeps a comment in a plain-text sink:

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
    <Text text="{order>/Comment}" />
    <Button
        text="Submit"
        enabled="{= ${state>/valid} === true }"
        press=".onSubmit" />
</mvc:View>
```

The expression binding controls a boolean. It is not an XSS sanitizer. Binding the same comment to an arbitrary HTML property changes the threat model completely.

If rich HTML is a genuine requirement, identify the supported sanitization path, its allowed elements and URLs, and the exact deployed version. The HTML control's documentation states that automatic sanitization is not enabled by default [S3]. Setting the appropriate supported sanitization property can be part of a solution, but the allowed markup still needs review and testing.

- [ ] Prefer text controls and safe renderer text APIs.
- [ ] Do not insert untrusted values through `innerHTML` or HTML concatenation.
- [ ] Review URL protocols and destinations independently of output encoding.
- [ ] Test stored, reflected, and DOM-derived content with inert synthetic fixtures.
- [ ] Verify errors, tooltips, accessibility text, and exports as well as main views.

## Review custom controls and renderers

A custom control can undo the protections of standard UI5 controls. For modern RenderManager implementations, use documented structured APIs and `text` for untrusted text rather than writing raw markup strings. Validate URL-valued properties according to their intended destinations; HTML attribute encoding alone does not prevent navigation to an unwanted scheme or host.

Check lifecycle hooks for post-render DOM modification. A safe renderer followed by an unsafe `afterRendering` assignment is still unsafe. Similarly, a sanitizer's output must not be modified afterward in ways that reintroduce executable markup.

Avoid evaluating strings as code. Search relevant changed paths for dynamic code construction, string-based timers, and inline event handlers. If a framework or legacy library requires a CSP exception, document it and evaluate the installed release rather than adding broad exceptions blindly.

Verify asynchronous loading and CSP behavior with the real launchpad and resource hosts. A strict policy that breaks authentication or essential UI is not an acceptable release artifact; a permissive policy that allows everything is not meaningful defense.

## Review CSRF at the receiving server

Determine how the browser authenticates the request. Cookie-authenticated state-changing operations need the platform's supported CSRF protections. Do not confuse OAuth `state`, an access token, an OData CSRF token, and a session cookie: they protect different boundaries.

UI5 OData models manage supported CSRF behavior for their own requests [S2, S4, S5]. Custom fetch calls, file uploads, bespoke endpoints, and router-local services need their own integration. Verify which server issues the token and which validates it.

Review missing, stale, and mismatched token behavior. The server should reject before changing business state. Keep state changes out of GET requests. SameSite cookies, origin validation, and Fetch Metadata can add defense, but use a coherent supported policy instead of assuming any one header solves every case [S7].

- [ ] Keep CSRF enforcement enabled throughout the request chain.
- [ ] Ensure token and cookie belong to the same session and service context.
- [ ] Test direct writes and operations inside `$batch`.
- [ ] Check uploads and non-OData endpoints separately.
- [ ] Do not automatically replay non-idempotent requests after authentication loss.

## Keep OData V2 and V4 APIs distinct

Identify the imported model class, not just the service protocol named in a ticket. V2 model request APIs use methods such as `read`, `create`, and `update` with their documented callbacks. V4 uses bindings and contexts, including list-binding creation and operation bindings.

Do not copy V2 `refreshSecurityToken` calls into V4 code. Do not assume a V2 batch callback means every embedded operation succeeded. In V4, submitting a group and resolving a batch-level promise does not necessarily mean each business operation succeeded; inspect the documented operation or context result and message handling.

Review error paths for rejected operations, stale contexts, cancellation, and session loss. Avoid logging full error objects, which can contain server messages or request details. Return a controlled user message and preserve only an approved correlation reference.

Version differences matter. Check documentation for the deployed UI5 release before approving an API copied from a newer example.

## Review SQL injection and dynamic query construction

Trace all externally influenced values into database APIs. Parameterized values should remain data, never SQL syntax. Validation is useful but not an alternative to parameterization. Replacing apostrophes by hand is not a general defense [S8].

For Java JDBC, look for prepared statements and bound values. For ABAP SQL, prefer static statements with host variables and the intended authorized read model. For framework query builders, distinguish structured query expressions from raw query fragments.

Dynamic identifiers require a fixed mapping from external options to approved columns or operations. A parameter placeholder cannot safely choose an arbitrary table name. Also review wildcard search semantics and expensive filters: injection prevention does not prevent a user from requesting an unbounded expensive query.

- [ ] Parameterize all database values.
- [ ] Allowlist dynamic sort and projection identifiers.
- [ ] Enforce row and tenant restrictions independently.
- [ ] Limit paging, expansion depth, and costly query shapes.
- [ ] Reject unknown or privilege-bearing write properties.

## Review validation and encoding separately

Validation establishes that a value is acceptable for the business operation. Encoding ensures it remains data in a particular output context. Sanitization transforms permitted rich content. These are separate controls and should not be collapsed into one “sanitize input” helper.

Check server-side type, length, range, enum, precision, scale, and cross-field rules. Validate currency and amount together. Check dates against business calendars and time-zone rules. Normalize only according to a documented policy; inconsistent normalization can produce ambiguous identifiers.

Do not blacklist a few suspicious substrings as the main XSS or SQL injection defense. Legitimate business text can contain apostrophes and angle brackets. Keep such text safe through parameterized persistence and context-appropriate rendering.

For file inputs, validate authorization, size, filename handling, content type, and actual content according to the supported upload service. Store files outside executable paths and apply the organization's scanning and quarantine process. A browser-provided MIME type is not evidence of safe content.

## Review data exposure and diagnostics

Inspect actual service response contracts. Sensitive fields must be absent for unauthorized users, not merely hidden by view logic. Review expanded entities, search suggestions, count endpoints, generated files, and exception messages.

Check browser persistence and caching. Tokens must not appear in localStorage, sessionStorage, URL fragments, query parameters, or client models. Protected drafts need a deliberate server-side lifecycle. Sensitive API responses should have the intended cache headers after proxy processing.

Logs need fixed event categories, generated correlation identifiers, and minimal permitted context. Never approve logging of cookies, Authorization headers, passwords, full JWTs, complete request bodies, or raw error objects as routine diagnostics [S10].

- [ ] Inspect success and failure responses for unauthorized fields.
- [ ] Check exports, cache keys, browser storage, and support captures.
- [ ] Validate and length-limit externally influenced log fields.
- [ ] Confirm restricted log access, retention, and incident retrieval procedures.

## Review browser and platform hardening

Check HTTPS across the whole chain, outbound certificate validation, cookie flags, security headers, and restricted CORS. CORS controls browser response access; it is not authentication or business authorization. A non-browser caller does not need a CORS permission to send a request.

Review clickjacking protection with the real launchpad embedding requirements. A blanket frame prohibition can break legitimate hosting, while unrestricted framing exposes sensitive interaction. Use the supported platform controls and a deliberate ancestor allowlist.

Review dependencies through the existing organizational process. Confirm supported versions, vulnerability response ownership, lockfile handling where applicable, and trusted artifact sources. Do not treat minification, obfuscation, or a private repository as substitutes for security controls.

Exclude operational secrets from frontend builds and diagnostics. A build pipeline can accidentally turn a protected environment variable into public JavaScript.

## Define evidence and authorized testing

For each material control, capture the reviewed code location, the deployment assumption, and a reproducible authorized test. Evidence should use synthetic accounts and data and contain expected outcomes, not actual credentials.

Negative tests should demonstrate that a request was rejected and that no business side effect occurred. Positive control cases prove that the test fixture can perform the intended action when genuinely authorized. Cover direct API calls instead of relying exclusively on UI automation.

Penetration testing requires written scope, approved environments, named contacts, test accounts, timing, rate limits, data-handling rules, and stop conditions. Do not scan unrelated hosts, reuse production employee sessions, or attempt destructive payloads without explicit authorization. If unexpected sensitive data is encountered, stop, preserve only minimal approved evidence, and follow the reporting procedure.

## Release decision checklist

- [ ] No unresolved authentication or authorization bypass remains.
- [ ] XSS sinks, CSRF integration, and database parameters have explicit review evidence.
- [ ] UI5 model APIs match the deployed model class and release.
- [ ] Sensitive data is minimized in responses, storage, logs, and exports.
- [ ] Error handling is safe and business retries are deliberate.
- [ ] Infrastructure assumptions have owners and effective configuration checks.
- [ ] Authorized negative and positive cases cover the changed behavior.
- [ ] Any exception has an owner, expiry, compensating control, and approved risk decision.

Record unresolved uncertainty rather than checking a box without evidence. A review should explain why the changed request path remains safe and what deployment conditions must continue to hold.

## Sources

- **S1 — SAPUI5 Browser Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/browser-security-91f3b66.md
- **S2 — SAPUI5 Transport Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/transport-security-91f3e60.md
- **S3 — OpenUI5 HTML control documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/core/HTML.js
- **S4 — OpenUI5 OData V2 model documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js
- **S5 — OpenUI5 OData V4 model documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js
- **S6 — OWASP XSS Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.md
- **S7 — OWASP CSRF Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md
- **S8 — OWASP SQL Injection Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.md
- **S9 — OWASP Authorization Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Authorization_Cheat_Sheet.md
- **S10 — OWASP Logging Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Logging_Cheat_Sheet.md
- **S11 — SAPUI5 Content Security Policy:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/content-security-policy-fe1a6db.md

S11 supports the CSP review guidance; start with report-only observation in the target landscape, then enforce an appropriately tested policy.
