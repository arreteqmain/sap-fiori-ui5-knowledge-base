# Authentication for SAP Fiori and UI5

## Scope and assumptions

This chapter describes enterprise authentication for browser-based SAP Fiori and UI5 applications. It assumes HTTPS, an organizational identity provider, and a backend that independently protects its APIs. Deployment might use an SAP application router on BTP, a reverse proxy with SAP Gateway, or an ABAP system behind a Fiori launchpad. Those arrangements have different configuration surfaces; none makes a browser a trusted security authority.

Examples are integration designs, not complete identity-provider configurations. Select supported components for the deployed SAP product and release, then review their configuration with the identity team. No example requires copying credentials into application source or adding an unreviewed dependency. Authentication establishes identity; [authorization](authorization.md) decides what that identity can do. Read [data protection](data-protection.md) for handling the resulting identity information.

SAP explicitly describes browsers as untrusted clients and requires server-side validation [S1]. The SAP security integration library documents the application-router, authorization-code, session, and resource-server arrangement [S2]. OWASP authentication, OAuth, and session guidance supplies the protocol-independent controls used below [S3–S5].

## Establish the identity boundary

Draw the actual request chain before choosing a login mechanism: browser, launchpad, application router, identity provider, backend API, and downstream SAP system. Mark where credentials terminate, where sessions exist, and where one principal becomes another. An authentication diagram is incomplete if it stops at the launchpad while the business service accepts direct requests.

Use a stable subject identifier together with its issuer or identity zone. Email addresses and display names can change and are not reliable global identity keys. Two identity providers can issue the same subject string. Similarly, a tenant identifier supplied in a URL is merely a requested routing context until trusted server logic reconciles it with the authenticated principal.

Reject client-controlled identity headers at public ingress. A proxy may establish authenticated headers only when the backend cannot be reached around that proxy and the transport and header-overwrite rules are controlled. Prefer the platform's supported principal propagation or token validation rather than an application-specific header convention.

Separate human login from service-to-service authentication. A background integration needs its own limited technical identity, not an employee's browser session or a shared administrator account. Record the original actor when delegation is supported, without assuming that a technical account automatically preserves that actor's authorizations.

## Choose a browser architecture deliberately

For most enterprise UI5 applications, prefer a backend-for-frontend arrangement. The browser holds an opaque session cookie; the application router or backend holds access and refresh credentials. The browser calls same-origin application routes, and the server forwards appropriately scoped credentials to resource servers. SAP's integration documentation describes this general shape [S2].

A public browser client is a different design. It cannot keep a client secret because every user can inspect downloaded code, configuration, and network traffic. If direct browser OAuth is justified, use authorization code with PKCE through a maintained identity SDK, keep access tokens in memory, and document refresh behavior and XSS exposure. Do not persist access or refresh tokens in localStorage, sessionStorage, IndexedDB, UI5 models, or URL parameters.

Memory storage reduces persistence; it does not defeat script execution inside the page. An XSS attacker may invoke authenticated operations without extracting any credential. The server-session design therefore still needs output safety, CSRF protection, short-lived credentials, and restricted downstream privileges.

Choose one architecture per route family. Mixing cookie-authenticated writes, manually attached bearer headers, and independent login widgets creates ambiguous logout and CSRF behavior. Document whether a request is authenticated by a session, a bearer token, or a platform-managed trust mechanism. Do not introduce a silent fallback from one to another.

## Understand SAML federation

SAML commonly connects an enterprise identity provider to a service provider such as an SAP landscape component. The browser transports protocol messages, but the trusted service provider validates them. UI5 controllers must not parse a SAML assertion and declare the user authenticated.

Use supported federation components to verify signatures against configured identity-provider metadata and trust anchors. Validation includes the intended audience and recipient, destination, assertion time conditions, and transaction correlation when applicable. Replay protection and correct handling of signed elements are important; matching a username inside XML is not validation. Avoid hand-written XML-signature processing. OWASP's SAML guidance details these protocol and signature-validation concerns [S9].

Treat metadata and signing-certificate rotation as operational changes. Test overlap periods, expiry alerts, and emergency certificate replacement. A valid identity-provider certificate authenticates a federation message; it does not grant purchasing or financial privileges. Map incoming identities through controlled server-side rules, then apply backend authorizations.

Prefer service-provider-initiated login where the deployment supports it and document any accepted identity-provider-initiated flow. Unsolicited assertions have different transaction-correlation properties. Review relay-state and return destinations so federation cannot become an open redirect. Federation logout also needs explicit design: terminating one service-provider session does not necessarily terminate every upstream session.

## Apply authorization code with PKCE

OAuth authorizes access to resources; OpenID Connect adds an identity layer. An access token is not automatically a login proof for the UI, and an ID token is not a substitute for an API access token. Use the correct token at each interface.

The intended flow is:

1. A trusted client component creates a one-time login transaction bound to the initiating browser session.
2. It redirects to the configured authorization endpoint with an exact registered redirect URI and narrowly selected scopes.
3. PKCE uses a fresh verifier and an S256 challenge; the verifier is not placed in the authorization request.
4. The callback validates transaction correlation, expected issuer, and applicable OpenID Connect nonce requirements.
5. The trusted client redeems the code once, using the matching verifier and any server-side client authentication required for a confidential client.
6. After successful validation, the server rotates the session identifier and sends the browser only its session cookie.

Use supported libraries for these steps. PKCE reduces authorization-code interception and injection risks; it does not encrypt access tokens or make redirect validation optional. Follow OWASP's OAuth guidance on mix-up prevention, transaction binding, and refresh-token protection [S4].

Do not use the implicit flow for a new implementation. Do not collect corporate passwords in a custom UI5 form to perform a password grant. Delegate interactive authentication to the organizational identity provider, where MFA, credential policy, recovery, and risk decisions can be consistently enforced.

## Validate JWTs only at trusted servers

A JWT payload can be decoded without any secret. Decoding shows what a token says, not whether it is authentic. A browser-side check may improve presentation, but must never establish API trust or authorize a business transaction.

Use supported resource-server middleware, such as an appropriate SAP security integration library for its documented BTP environment [S2]. Configure it from trusted service bindings and approved issuer metadata. Do not copy cryptographic verification code into controllers, implement signature algorithms manually, or trust a key URL supplied by an unvalidated token.

The middleware configuration must enforce the following acceptance contract:

| Check | Required behavior |
| --- | --- |
| Signature | Verify using the trusted issuer's authorized keys and supported library |
| Algorithm | Accept only the explicitly approved algorithm set; reject unsigned tokens |
| Issuer | Match the configured trusted issuer, including approved tenant relationships |
| Audience | Require this resource server's intended audience |
| Time | Enforce expiration and applicable not-before rules with bounded clock skew |
| Token purpose | Distinguish access tokens from ID tokens and other JWT types |
| Principal | Construct identity only from validated claims and trusted mappings |
| Privileges | Apply required scopes and separate business authorization checks |

Do not select verification behavior merely from the token's algorithm header. Key rotation requires controlled key discovery, bounded caching, and failure handling. An unknown key identifier is not permission to accept the token without verification. Monitor clock drift rather than increasing permitted skew indefinitely.

A valid signature with the wrong audience must fail. A valid token from an unapproved tenant must fail. A correctly authenticated user without the necessary business authorization must still be denied.

## Configure concrete SAP Spring resource-server middleware

The official repository cited in this chapter provides Java integration, not the Node.js `@sap/xssec` API. Its current sample uses Spring's `SecurityFilterChain` configuration [S10–S11]. The following concrete configuration adapts that supported integration without copying its sample-only debug setting or implementing a JWT verifier.

**Prerequisites:** an existing, approved SAP BTP Cloud Foundry or Kyma Spring Boot 4 application, Java 17 or newer as supported by that application, Spring Security 7, and the compatible SAP `resourceserver-security-spring-boot-starter` 4.x already present. Use the organization's reviewed maintenance versions. This example assumes one correctly configured XSUAA service binding, SAP auto-configuration enabled, and the SAP-provided `JwtDecoder` bean. It is not a dependency-installation or identity-service-provisioning recipe. Spring Boot 3 applications need SAP's separate compatibility module and their supported configuration, not an unreviewed version substitution.

Place this configuration in the application's component-scan package. It deliberately exposes only a GET authentication probe and denies every other route; extend the explicit allowlist only after designing each route's authorization:

```java
import com.sap.cloud.security.spring.config.IdentityServicesPropertySourceFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@PropertySource(
    factory = IdentityServicesPropertySourceFactory.class,
    ignoreResourceNotFound = true,
    value = {""}
)
class ApiSecurityConfiguration {
    @Bean
    SecurityFilterChain apiSecurity(HttpSecurity http, JwtDecoder decoder)
            throws Exception {
        http.sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        http.authorizeHttpRequests(routes -> routes
            .requestMatchers(HttpMethod.GET, "/api/authentication-check")
                .authenticated()
            .anyRequest().denyAll());
        http.oauth2ResourceServer(resource -> resource
            .jwt(jwt -> jwt.decoder(decoder)));
        return http.build();
    }
}
```

The endpoint implementation can return an empty success response without exposing identity claims:

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class AuthenticationCheckController {
    @GetMapping("/api/authentication-check")
    ResponseEntity<Void> check() {
        return ResponseEntity.noContent()
            .header("Cache-Control", "no-store")
            .build();
    }
}
```

Here `JwtDecoder` is supplied by SAP's starter from trusted service-binding configuration; it must not be replaced with a custom payload decoder or an unrelated permissive bean. Verify the actual selected bean and issuer/audience configuration in the deployed application. The library documents RS256 support for this integration [S10]; accepting arbitrary token-declared algorithms is not part of this configuration.

The router forwards the access token server-to-server to this resource server. The browser continues using its router session, while this backend does not create its own login session. CSRF defaults remain enabled: this GET-only example introduces no write endpoint and no CSRF-disable flag. Adding cookie-authenticated writes requires the appropriate supported server-side CSRF integration.

Within an authorized environment, an accepted access token should reach the controller and receive 204; absent or invalid authentication should be rejected by the filter chain, normally with 401. Every non-allowlisted route remains denied. Business routes additionally need scope mapping and the object-level policy described in [authorization](authorization.md); successful authentication alone never grants approval rights. These are expected integration checks, not a claim that this documentation was executed against a provisioned SAP service.

## Integrate application policy after middleware

The following server-side sketch is intentionally framework-neutral JavaScript. Unlike the concrete Java filter-chain example above, it illustrates ordering rather than providing a middleware implementation. The named adapters are application interfaces, not package APIs. Their implementation must delegate authentication to supported platform middleware, not decode a JWT locally:

```javascript
// Server composition sketch; adapters are supplied by the deployment.
function registerProtectedRoutes(app, security, orders) {
    app.use("/api", security.authenticateRequest);
    app.use("/api", security.requireAuthenticatedPrincipal);
    app.get("/api/session-summary", (req, res) => {
        res.set("Cache-Control", "no-store");
        res.json({
            displayName: req.principal.displayName,
            sessionExpiresAt: req.session.absoluteExpiresAt
        });
    });
    app.post(
        "/api/orders/:id/approve",
        security.requireCsrfForSession,
        orders.approveWithBusinessAuthorization
    );
}
```

The middleware must stop processing on authentication failure and must establish the server-owned principal before any protected route executes. The approval handler must obtain the principal from that trusted context, load the order in the principal's authorized tenant, and check action and organizational permissions.

Do not return a token, complete claim set, or backend credentials from the session-summary endpoint. Display names are still untrusted text at the rendering boundary. An authentication endpoint also needs request limits and safe error handling; it is not harmless merely because its response is small.

Verify direct access to service hosts. Router authentication is not sufficient when the backend also has a publicly reachable route that skips equivalent middleware.

## Protect the server session

Configure session cookies as Secure and HttpOnly, with the narrowest practical host and path scope. Select SameSite behavior according to the federation flow and browser requirements. A cross-site POST-based federation callback can require a different arrangement from a same-site API call; test it rather than weakening every cookie globally.

Avoid broad parent-domain cookies when sibling applications are not equally trusted. Rotate session identifiers after login and privilege changes. Never accept a session identifier supplied in a navigation URL. Prevent session fixation by ensuring an anonymous session cannot simply become an authenticated session with an unchanged attacker-known identifier.

Define idle timeout and absolute lifetime separately. Idle timeout limits inactivity; absolute lifetime ends a session even if requests continue. Set both on the server. A UI5 timer is only a warning mechanism and cannot enforce expiration against a modified client.

Decide which requests count as meaningful activity. Automatic polling must not keep a privileged session alive forever. Coordinate router, identity-provider, backend, and refresh-token lifetimes so the visible warning reflects actual enforcement. Document whether an identity-provider SSO session may immediately reauthenticate a user after the application session ends.

Use a shared session store or a supported stateless design appropriate to scaling. Restarts, failover, and routing changes must not restore expired sessions or inconsistently apply logout.

## Make timeout behavior safe for users

Distinguish authentication loss from missing authorization. An unauthenticated API request normally receives 401; a known user denied a protected action normally receives 403. Federation navigation may use redirects, but an OData request unexpectedly receiving an HTML login page is an integration failure, not a successful data response.

When the session expires, stop new business writes, explain that authentication is required, and provide a deliberate way to continue through the configured login entry point. Do not repeatedly retry a failed approval or payment. The first request might already have committed before a connection failure became visible.

Unsaved data needs a classification-aware policy. Do not automatically save an entire model to browser storage as a timeout workaround. Prefer a server-side draft with its own authorization, retention, and cleanup rules where supported. Otherwise warn early and discard sensitive local state on logout.

Use accessible warnings with enough time for users to respond, but never let the client extend the server's absolute maximum. Test multiple tabs, suspended devices, expired backend sessions, and long-running requests. A tab waking from sleep must not assume its displayed countdown remains accurate.

## Require MFA and deliberate step-up

Enforce MFA at the identity provider rather than inventing a second-factor dialog inside UI5. Prefer phishing-resistant authenticators where available and appropriate. Enrollment, recovery, replacement, and administrator reset procedures are part of authentication security, not optional help-desk details [S3].

A sensitive transaction can require recent or stronger authentication. The backend must decide whether the authenticated context satisfies the required assurance and freshness, using supported identity-provider claims or transaction mechanisms. A client boolean such as `mfaComplete` is not evidence.

Bind step-up to the intended transaction where the platform supports it. Reauthentication for an unrelated page should not silently authorize changing bank details indefinitely. After successful step-up, the business service must still perform authorization, state validation, and separation-of-duties checks.

Avoid unnecessarily frequent prompts that train users to approve unexpected challenges. Use risk and business impact to define policy. Monitor suspicious challenge patterns, recovery activity, and repeated login failures without recording passwords, one-time codes, or authenticator secrets.

## Keep token and secret storage out of UI5

Application source, manifests, preload bundles, and source maps are public to application users. A value injected during a frontend build remains public after deployment. Minification is a performance technique, not a security boundary or a way to conceal a client secret.

Keep confidential client credentials in platform-managed server bindings or an approved secret store. Restrict which workloads can read them and define rotation procedures. Never send them to the browser, include them in diagnostics, or put them in destination URLs.

Do not place tokens in localStorage. Do not switch to sessionStorage and call the problem solved: both are JavaScript-readable. Do not use a UI5 JSONModel as a credential vault. With server sessions, the browser should not need to inspect backend access tokens at all.

If a public-client exception is approved, use short-lived in-memory access tokens and the provider's supported refresh protections. Reassess browser extensions, third-party scripts, shared devices, and reload behavior. A storage exception requires a documented threat model, not a convenience argument.

## Integrate OData without mixing model APIs

The UI5 OData V2 and V4 models manage supported CSRF-token interactions for their own OData requests; that is separate from OAuth authentication [S6–S8]. A CSRF token is neither a login credential nor a role assignment.

For V2, use the documented V2 request methods and callbacks. Its `refreshSecurityToken` API belongs to that model and is not a generic authentication-renewal function. For V4, use bindings, contexts, and operation bindings; do not copy V2 `read`, `create`, or token-refresh calls into V4 code.

Keep the OData service URL same-origin where the deployment architecture permits. Let the configured router and session mechanism carry identity. Do not manually copy browser cookies into Authorization headers or attach access tokens retrieved from a session-summary response.

Custom fetch calls, uploads, and non-OData endpoints need their own documented CSRF integration. Never disable CSRF enforcement to solve an expired-token error. Investigate token-cookie correlation, service roots, proxy behavior, and session expiration instead.

## Log authentication outcomes safely

Create structured server events for login success, failure categories, session expiry, logout, step-up, and unusual refresh behavior. Use a generated correlation identifier, a controlled event name, and a minimal stable subject reference where policy permits.

Do not log Authorization headers, cookies, full federation messages, authorization codes, refresh credentials, or complete error objects. Identity libraries may include request details in exceptions. Map them to approved categories before logging, and keep necessary diagnostic traces in restricted facilities with short retention.

Return a generic, useful message to users rather than revealing whether an account exists or which claim failed. Rate-limit and monitor authentication entry points using platform capabilities. Ensure operational dashboards do not become an alternative repository of personal data.

## Verification and operational checklist

- [ ] Identify the authentication mechanism and owner for every public and backend route.
- [ ] Verify exact redirects, issuer trust, code transaction binding, and S256 PKCE.
- [ ] Confirm browser resources contain no confidential client credentials.
- [ ] Demonstrate rejection of invalid signatures, wrong audiences, wrong issuers, and expired tokens using authorized test fixtures.
- [ ] Verify server-enforced idle and absolute expiration, identifier rotation, and logout.
- [ ] Exercise MFA recovery and transaction step-up with approved test accounts.
- [ ] Confirm no tokens or sensitive drafts persist in browser storage or logs.
- [ ] Test OData authentication loss without replaying non-idempotent writes.
- [ ] Prove the backend cannot be reached through an unprotected alternate route.
- [ ] Assign owners for trust metadata, certificates, library updates, and incident revocation.

Run protocol-negative testing only in approved environments with synthetic accounts. Record expected status and absence of business side effects, not token contents. For a repeatable evidence process, use the [code review checklist](code-review-checklist.md); for end-to-end designs, see [security patterns](security-patterns.md).

## Sources

Sources were consulted through official GitHub-hosted documentation and source comments. Match APIs and configuration to the release actually deployed.

- **S1 — SAPUI5 Browser Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/browser-security-91f3b66.md
- **S2 — SAP BTP Security Services Integration Libraries:** https://github.com/SAP/cloud-security-services-integration-library/blob/main/README.md
- **S3 — OWASP Authentication Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Authentication_Cheat_Sheet.md
- **S4 — OWASP OAuth 2.0 Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/OAuth2_Cheat_Sheet.md
- **S5 — OWASP Session Management Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Session_Management_Cheat_Sheet.md
- **S6 — SAPUI5 Transport Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/transport-security-91f3e60.md
- **S7 — OpenUI5 OData V2 model API documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v2/ODataModel.js
- **S8 — OpenUI5 OData V4 model API documentation:** https://github.com/SAP/openui5/blob/master/src/sap.ui.core/src/sap/ui/model/odata/v4/ODataModel.js
- **S9 — OWASP SAML Security Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/SAML_Security_Cheat_Sheet.md
- **S10 — SAP BTP Spring Security integration and prerequisites:** https://github.com/SAP/cloud-security-services-integration-library/blob/main/spring-security/README.md
- **S11 — SAP current SecurityFilterChain sample:** https://github.com/SAP/cloud-security-services-integration-library/blob/main/samples/spring-security-hybrid-usage/src/main/java/sample/spring/security/SecurityConfiguration.java
