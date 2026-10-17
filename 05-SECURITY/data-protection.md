# Data Protection for SAP Fiori and UI5

## Scope and assumptions

This chapter addresses personal, financial, operational, and credential-related data moving through UI5 applications and their SAP backends. It assumes that [authentication](authentication.md) establishes identity and [authorization](authorization.md) restricts business access. Data protection adds purpose limitation, minimal disclosure, transport confidentiality, controlled persistence, and auditable handling.

The guidance applies to browser applications, application routers, Gateway or RAP services, BTP services, databases, logs, and generated exports. It is engineering guidance rather than a claim of legal compliance. Retention periods, consent requirements, lawful purposes, and cross-border processing need organizational privacy and legal decisions.

SAP's browser documentation warns against treating local storage as secure [S1], while its transport guidance requires protected communication [S2]. OWASP provides complementary guidance for TLS, cryptographic storage, SQL parameters, and safe logging [S3–S6]. Examples use existing platform capabilities and do not prescribe dependency installation.

## Inventory data before implementing controls

Create a field-level inventory for each business flow. Record the data source, purpose, classification, allowed recipients, storage locations, retention owner, and deletion obligations. “Purchase order data” is too broad: a document number, employee name, supplier bank account, internal comment, and attachment can have different handling requirements.

Follow data through hidden copies. UI5 models, value-help responses, OData expansions, browser caches, proxy logs, analytics events, crash reports, generated spreadsheets, and support screenshots can retain information even when the primary database is protected.

Map technical owners to lifecycle stages. An application developer may control response fields, while a Basis administrator controls ABAP logs and a platform operator controls backup retention. A control is incomplete if its required owner is unspecified.

Use synthetic fixtures outside approved production handling. Copying a production response into a mock JSON file creates a new data repository with different access and retention. Replacing a person's name while retaining unique document identifiers may still permit reidentification.

## Minimize responses at the server

Return only fields needed for the user's authorized task. A list view usually does not need full bank details, personal addresses, or long internal notes. Separate privileged detail endpoints or projections from routine list representations.

OData `$select` can reduce payloads, but a client-controlled selection is not an authorization policy. A malicious caller can request additional fields. The service must enforce field visibility and permissible projections independently of whatever the UI requests.

Similarly, hiding a control or masking a formatter does not protect a raw value already delivered to the browser. If the user must not know the full value, the backend must not return it. Review metadata and error messages as well as successful entity responses.

Do not use a single unrestricted entity merely because different pages need different fields. Shape service contracts around business capabilities and data sensitivity. Verify associations and navigation targets so a restricted projection cannot be bypassed through an alternate path.

## Encrypt every relevant transport hop

Use HTTPS for the application shell, UI5 resources, authentication redirects, APIs, attachments, telemetry, and export downloads. Mixed content can expose information or break execution. Loading the main page through TLS does not secure an HTTP backend hop.

Prefer TLS 1.3 and retain TLS 1.2 only where supported compatibility requires it. Disable obsolete protocols and weak cipher configurations according to organizational standards and the supported platform [S3]. Validate certificates and hostnames on outbound server connections; do not bypass verification to accommodate a misconfigured destination.

Record where TLS terminates and whether traffic is re-encrypted before entering another trust zone. Reverse-proxy headers describing the original scheme are trustworthy only when the application accepts them from controlled proxies. Incorrect trust can cause insecure cookies or redirects.

Use HSTS on appropriate production origins after checking subdomain ownership and rollout implications. HSTS is not a substitute for issuing valid certificates. Assign certificate renewal, expiry monitoring, and incident replacement to named owners.

Test downloads and authentication callbacks early. Those paths often use different hosts or proxy rules from the main OData endpoint.

## Protect data at rest with managed keys

Use supported database, volume, backup, and object-storage encryption according to the threat model. Encryption at rest protects some storage exposures but does not prevent a legitimately connected application from reading excessive data. Keep authorization and minimization controls.

Where selected fields need application-layer encryption, use an approved maintained cryptographic library or managed encryption service. Prefer authenticated encryption through the supported facility, with safe nonce management handled by that facility. Do not implement an encryption format from raw primitives in UI5 or ABAP application examples.

Separate keys from encrypted data. Use a managed key or secret service, restrict decrypt permission to intended workloads, and audit key access. Configuration files beside the database export are not an independent key boundary.

Define rotation and recovery before production. Record how encrypted data identifies its key version, how old versions remain available for legitimate reads, and when they can be retired. Test restoring backups with the required keys under controlled access. Losing every key can make retained business records permanently unreadable.

Do not confuse signing with confidentiality. A signed JWT is normally readable, and base64 encoding is not encryption. Browser-side encryption with a key shipped in the same JavaScript bundle does not hide data from the browser user.

## Mask without creating false confidence

Masking changes presentation; redaction removes information from a delivered representation. A safe ordinary-user response might contain a masked account label and an opaque server identifier, not the full account with an instruction to hide it.

The following server-side JavaScript function illustrates response shaping after authorization. `bankRecord` is trusted backend data, and `policy` is a server-computed decision, never a request boolean:

```javascript
function bankAccountView(bankRecord, policy) {
    const view = {
        accountId: bankRecord.id,
        accountLabel: bankRecord.maskedDisplayLabel
    };
    if (policy.mayReadFullBankAccount === true) {
        view.fullAccount = bankRecord.accountNumber;
    }
    return view;
}
```

The mask should be generated according to the data type and privacy policy; it should not accidentally disclose an entire short value. The identifier still needs object-level authorization when used by another endpoint.

Render the label in a standard text property:

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m">
    <Text text="{account>/accountLabel}" />
</mvc:View>
```

Do not attach the unmasked value to custom data, tooltips, accessibility labels, hidden controls, or export models. An authorized reveal action should be separately checked and audited, with a response that is not shared-cacheable.

## Validate parameters without constructing queries

Validate types, lengths, ranges, allowed properties, and business relationships on the server. Browser validation provides immediate feedback but can be bypassed. Do not accept tenant, actor, approval status, or calculated totals merely because they appear in a typed UI5 model.

Use parameterized database APIs. The following **server-side Java JDBC example** assumes an existing reviewed `DataSource`, an authenticated principal, and a repository query that returns only authorized order summaries. Column names describe an application schema, not standard SAP tables:

```java
String sql = """
    SELECT order_id, status
      FROM app_order_summary
     WHERE tenant_id = ?
       AND purchasing_org = ?
       AND order_id = ?
    """;
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(sql)) {
    statement.setString(1, principalTenant);
    statement.setString(2, authorizedPurchasingOrg);
    statement.setString(3, validatedOrderId);
    try (var result = statement.executeQuery()) {
        while (result.next()) {
            // Map only the response fields approved for this capability.
        }
    }
}
```

The principal tenant and organization must be derived from trusted authorization logic. Parameters prevent values from becoming SQL syntax; they do not establish business permission. Dynamic sort columns and table names cannot simply be substituted as value parameters: map external choices to a fixed server allowlist.

For ABAP SQL, use the release's static query syntax and host variables rather than concatenating dynamic WHERE fragments. Ordinary table access does not automatically enforce end-user business authorizations; select through the intended protected model and apply necessary checks [S7].

## Keep browser persistence deliberately small

Do not store access tokens, refresh tokens, full business records, or sensitive drafts in localStorage. Switching to sessionStorage does not make those values confidential; same-origin scripts can read it. Browser storage is not an approved replacement for a server session or protected draft service.

If the UI needs non-sensitive preferences, review whether even those preferences reveal personal or business context. Store only the approved minimum, version the format, and clear user-specific state when accounts change. Do not let one person's remembered filters expose another person's customer names on a shared device.

Use server-supported drafts when persistence is required. Drafts need access control, expiry, ownership handling, and deletion just like active records. An application should not silently recreate deleted sensitive data from an old local cache.

Classify offline requirements separately. A managed mobile offline solution needs device controls, supported encrypted storage, synchronization authorization, remote lifecycle handling, and explicit residual-risk approval. Browser application convenience is not equivalent to that managed environment.

## Control caches and exports

For sensitive session-specific responses, configure `Cache-Control: no-store` at the serving boundary. Test the actual response headers after proxies and routers have processed them. Do not assume a frontend model option controls intermediary caching.

Avoid placing personal data in URLs because URLs can appear in history, referrers, monitoring, and access logs. Use business identifiers only where their exposure is acceptable, minimize free-text searches in logged query strings, and configure appropriate referrer policy.

Export is a separate disclosure event. Apply the same row and field authorization as interactive reads, plus any export-specific permission. Enforce size limits and retention, protect download routes, and delete abandoned files.

Spreadsheet exports also need format-aware handling of cells that could be interpreted as formulas. Use the existing reviewed export implementation and verify its protection for text fields beginning with formula-significant characters. Do not assume CSV quoting alone prevents spreadsheet execution.

Do not email raw sensitive files as a workaround for download authorization. Prefer a short-lived authenticated retrieval flow, and reassess permissions when the user retrieves an asynchronously generated artifact.

## Log security events without logging data

Separate business audit events from verbose diagnostics. Audit events explain who attempted which capability, on what approved resource reference, with what outcome and time. Diagnostics help operators identify failures. Neither needs complete OData bodies or authentication headers.

Use controlled event fields, generated correlation identifiers, and internal reason codes. The following **server-side sketch** assumes an existing structured logger and trusted request context:

```javascript
function recordDeniedExport(logger, context) {
    logger.warn({
        event: "export.denied",
        correlationId: context.correlationId,
        actorRef: context.auditActorRef,
        reasonCode: "MISSING_EXPORT_PERMISSION"
    });
}
```

The example intentionally excludes raw exception objects, query strings, cookies, tokens, account numbers, and document contents. Validate and length-limit externally sourced log fields to prevent control-character injection and oversized events. Do not assume structured logging automatically redacts arbitrary nested objects.

Restrict log access, monitor export of logs, define retention, and protect transport to the collector. Audit administrators should not be able to silently rewrite the evidence they are responsible for reviewing. Document what happens when an audit sink fails: critical business events may require durable queuing or transaction-aware handling, while blindly failing every request can create an availability incident.

## Return safe, useful errors

Give users a controlled business message and an opaque support reference. Keep SQL text, database hostnames, stack traces, filesystem paths, and identity-provider diagnostics out of the response. A backend exception may contain the submitted value, so do not render its message verbatim.

Client-side logging should record a fixed failure category, not an entire request or response object. A browser console is visible to the user and may be harvested by monitoring tools. It is not a secure diagnostic sink.

When a message includes a document reference, verify that the user is authorized to know that reference. Authentication failures and resource-not-found decisions must not become an account or document enumeration interface.

Use standard text controls for messages. Error handling must not create an HTML injection path when the ordinary data view is safe.

## Manage retention and deletion

Set retention by data purpose and legal obligations, not by whichever storage service has the longest default. Cover active records, drafts, exports, temporary uploads, integration queues, support captures, logs, and backups.

Deletion may require coordinated lifecycle procedures rather than an immediate database DELETE. Preserve legally required financial records while removing unnecessary duplicates and derived artifacts according to approved policy. Record holds and exceptions with owners and expiry review.

Backups need controlled access, encryption, restore procedures, and finite retention. Document how deleted information is treated after a restore so old snapshots do not silently repopulate data that should remain removed.

Verify that monitoring and analytics vendors receive only approved fields through approved channels. Do not send real business payloads to an external error-reporting service during troubleshooting without the necessary organizational authorization.

## Verification checklist

- [ ] Maintain a field inventory with purpose, classification, owners, and retention.
- [ ] Prove that unauthorized fields never reach the browser, including hidden representations.
- [ ] Verify TLS and certificate validation across every hop and download path.
- [ ] Review encryption boundaries, managed keys, rotation, and backup recovery.
- [ ] Use parameterized database values and allowlisted dynamic identifiers.
- [ ] Keep tokens and sensitive business data out of browser persistence.
- [ ] Check cache headers, referrers, exports, and spreadsheet handling.
- [ ] Inspect sanitized logs and error messages for indirect disclosure.
- [ ] Exercise deletion, draft expiry, export cleanup, and restore handling.
- [ ] Record evidence with synthetic data and approved access.

Use the [code review checklist](code-review-checklist.md) to attach these checks to a change, and [security patterns](security-patterns.md) for complete workflows.

## Sources

- **S1 — SAPUI5 Browser Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/browser-security-91f3b66.md
- **S2 — SAPUI5 Transport Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/transport-security-91f3e60.md
- **S3 — OWASP Transport Layer Security Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Transport_Layer_Security_Cheat_Sheet.md
- **S4 — OWASP Cryptographic Storage Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cryptographic_Storage_Cheat_Sheet.md
- **S5 — OWASP SQL Injection Prevention Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.md
- **S6 — OWASP Logging Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Logging_Cheat_Sheet.md
- **S7 — SAP ABAP Authorization Checks:** https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md
