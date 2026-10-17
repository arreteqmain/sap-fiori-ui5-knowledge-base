# Authorization for SAP Fiori and UI5

## Scope and assumptions

This chapter covers business authorization across UI5 applications, SAP Gateway services, ABAP implementations, and BTP services. It assumes an already authenticated principal established by trusted middleware or the SAP runtime. The identity boundary is described in [authentication](authentication.md). Here, the question is whether that principal may perform this action on this object, in this organizational and tenant context, now.

Classic on-premise ABAP commonly uses PFCG roles and authorization objects. ABAP Cloud uses its supported IAM artifacts and business-role arrangements; BTP applications can use scopes, role templates, and role collections. These mechanisms are related but not interchangeable configuration formats. Select the model supported by the actual product, release, and deployment.

SAP's ABAP authorization documentation distinguishes explicit checks, CDS read access control, and RAP authorization [S1]. SAP's browser guidance establishes that the client cannot be trusted [S2]. OWASP recommends least privilege, deny by default, and checks on every request [S3]. The practical controls below combine these principles without assuming that launchpad content assignments secure backend data.

## Define policy as a business decision

Begin with business capabilities rather than technical endpoint names. “Approve purchase order” is a useful capability; “call POST” is not. Define the relevant company code, purchasing organization, plant, amount threshold, document status, ownership, and separation-of-duties conditions. Different domains use different dimensions, so do not attach all possible dimensions to every action.

A useful policy statement identifies a subject category, an action, a resource class, and constraints. For example, a purchasing approver may approve an eligible order in an assigned purchasing organization, subject to configured value limits and no self-approval. The actual permission derives from controlled business configuration, not a role name invented inside JavaScript.

Build a permission matrix with the business owner:

| Capability | Resource boundary | Additional condition | Expected denial |
| --- | --- | --- | --- |
| Read order | Authorized organization | Data classification permits fields | No unauthorized rows |
| Change order | Authorized organization | Editable business state | No state change |
| Approve order | Authorized organization | Threshold and separation of duties | No approval or event |
| Export orders | Authorized result set | Separate export permission | No export artifact |
| Manage roles | Approved administrative scope | Controlled assignment workflow | No privilege change |

Include exceptional administration, emergency access, and temporary assignments. A policy is not complete until someone owns its approval and revocation process.

## Separate the authorization layers

Launchpad catalogs, spaces, pages, and target mappings determine discoverability and navigation according to the launchpad configuration. They do not prove that a directly invoked OData operation is authorized. An absent tile is not a backend denial.

Application-router scopes can restrict route access. Service middleware can require API scopes. Those checks establish a coarse permission to enter a service or invoke a capability, but often lack the document's organization, amount, or current workflow state.

The business service must therefore enforce object and action authorization after establishing the trusted principal and locating the relevant resource. Database constraints and restricted technical identities provide additional defense, but a shared database connection does not automatically impose end-user business permissions.

Keep a responsibility map for each layer. If a route relies on a backend check, name that check and show its evidence. If a gateway performs a coarse access check, prove that direct backend invocation cannot bypass the remaining policy. Avoid the phrase “security handled by SAP” without identifying the runtime, artifact, and request path.

## Model RBAC without losing object constraints

Role-based access control assigns capabilities to roles and roles to principals. It works well for stable responsibilities such as buyer, approver, and auditor. It becomes dangerous when a role such as “Approver” is interpreted as permission over every order in every organization.

Combine role or scope checks with server-derived attributes and relationships. These might include assigned purchasing organizations, authenticated tenant, document owner, approval hierarchy, and current state. Organizational grants should come from trusted authorization configuration or backend APIs, not a list returned by the browser.

Apply least privilege vertically and horizontally. A reader should not gain approval rights, and an approver for one organization should not gain access to another organization's documents. Separate administrative role assignment from daily transactional roles.

Do not infer privileges from editable profile fields, email domains, URL segments, or application preferences. An identity-provider group can participate in a controlled mapping, but the mapping must be reviewed for issuer trust, naming collisions, tenant boundaries, and removal behavior. A group label alone does not specify the complete business policy.

## Design classic PFCG roles deliberately

In classic ABAP landscapes, PFCG roles contain authorization data and can support launchpad content assignments. Authorization objects group fields whose values constrain permitted activities and organizational data. The business service must execute the appropriate checks; adding an object to a role does not cause arbitrary custom code to enforce it.

Determine the applicable standard business authorization objects from the transaction, service implementation, system documentation, authorization proposals, and authorized traces. Coordinate with functional security specialists. There is no universal “Fiori authorization object” that grants or denies all business operations.

Maintain organizational levels and field values narrowly. A wildcard used to make a test pass can create broad production access. Review generated authorization profiles and user comparison according to the system's administrative procedure. Transporting a role definition is not proof that the correct users have the intended effective authorizations.

Use approved troubleshooting tools such as SU53 or an appropriately scoped authorization trace where available. A trace from one successful UI path is not a complete policy specification; it may omit negative cases, exports, alternate operations, and exceptional states. Trace access itself needs authorization because traces can reveal sensitive operational details.

Do not solve a denied business action by granting a broad administrative role. Identify the missing object and field values, confirm the requested capability, and obtain business approval for the narrow change.

## Use contextual ABAP checks

The following example is a **classic ABAP template**, not executable universal policy. Angle-bracket values inside literals are replacement markers. Replace the object, field, and activity with actual entries from the target system's business authorization design before adapting the code. The exception class is application-defined and must map to a controlled service error.

```abap
" Context: classic ABAP service after trusted document lookup.
" Replace every placeholder with system-approved business definitions.
AUTHORITY-CHECK OBJECT '<BUSINESS_OBJECT>'
  ID '<ACTIVITY_FIELD>' FIELD '<REQUIRED_ACTIVITY>'
  ID '<ORG_FIELD>'      FIELD lv_document_org.

IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_app_access_denied.
ENDIF.
```

Capture `sy-subrc` immediately; another statement can overwrite it. Treat any nonzero result as denial. Review whether system settings or authorization proposals can suppress a check; a zero result alone is not proof that the intended policy was configured and exercised. Do not use `DUMMY` for a meaningful organizational dimension merely to avoid supplying its value.

Derive `lv_document_org` from the trusted persisted object, not from a request parameter asserting where the object belongs. For creation, validate the proposed organization and check create authority before saving. For a transfer between organizations, check the policy for both the original and destination contexts.

In ABAP Cloud, use the permitted syntax, released APIs, and supported authorization artifacts for that environment. Do not assume classic dynamic authorization-object techniques or administrative transactions are available.

## Place checks at the business boundary

For an existing document, use this conceptual sequence:

1. Establish the authenticated principal and approved tenant context.
2. Validate the identifier's syntax without trusting its meaning.
3. Load the minimum resource information through an authorized lookup path.
4. Check action, organization, ownership, and state constraints.
5. Execute the mutation within the appropriate transaction and concurrency controls.
6. Record a minimal audit outcome and return only permitted response fields.

Avoid exposing object details while deciding whether access is allowed. A policy may deliberately use 404 instead of 403 to conceal resource existence; apply that consistently, including timing and error shape where practical. Do not return document descriptions in denied-operation messages.

State and authorization can change between lookup and save. Recheck critical conditions under the framework's transaction and locking model, and use optimistic concurrency where appropriate. An ETag prevents some lost updates; it does not replace the authorization decision.

Check the complete requested action, not only generic change permission. Approval, release, cancellation, posting, and downloading attachments can have separate consequences and require separate authorization.

## Secure CDS read access

CDS access control can restrict reads on supported CDS entities through DCL. It is valuable because collection requests, paging, and navigation should operate on an authorized result set. SAP's authorization material explains how CDS access conditions relate to authorization objects [S1].

Do not assume an annotation alone defines a policy. An access-control check annotation without the intended active DCL may produce warnings rather than enforce the expected restrictions, depending on the release and setting. Inspect the actual access-control artifact, its activation, and the entity being queried.

Understand where access control is applied and where it is bypassed. Reading an underlying table directly is not the same as reading an access-controlled CDS entity. Privileged access constructs, privileged execution contexts, and alternate projections require explicit justification and review.

Read restrictions do not automatically implement every write or action authorization. A user who can see an order might still be prohibited from changing its bank details or releasing it. Use the relevant transactional framework's authorization mechanisms and business checks for those operations.

Test joins, associations, navigation, counts, aggregations, and value helps. A filtered main list is insufficient if an expanded association exposes unauthorized information or a count reveals the size of another organization's workload.

## Integrate RAP authorization correctly

RAP distinguishes authorization responsibilities such as global capability decisions and instance-dependent decisions. Global authorization can answer whether the user may generally perform an operation; instance authorization evaluates permissions that depend on a particular business object.

Implement the authorization behavior required by the behavior definition and the deployed release. Feature control can improve interaction by making actions unavailable, but it should not be mistaken for a complete security policy. Confirm which framework checks actually enforce rejection at the service boundary.

Use the generated method contracts and documented failure reporting rather than copying a generic Gateway exception into a RAP handler. Local-mode or privileged internal operations can have different semantics; review internal consumers and do not expose a bypass path through a custom action.

For draft-enabled applications, evaluate draft creation, editing, activation, discard, and ownership rules. A user's permission at draft creation may not still be valid at activation. Revalidate sensitive business conditions when committing the active document.

Framework adoption does not remove the need for object-specific tests. Include the actual action bindings and transactional requests used by the application, not just a handler-level unit test.

## Apply BTP scopes and role collections

For BTP services, define scopes and role templates around business capabilities. Administrators assign appropriate role collections according to the product's authorization model. Resource-server middleware validates tokens and exposes trusted claims; application code uses the framework's supported authorization facilities.

The role collection's administrative label is not necessarily the scope string presented to an application. Verify the effective token and service mapping in a controlled environment without copying token contents into tickets. Namespace scopes and isolate environments so a development grant is not accidentally accepted in production.

Require the correct audience and tenant relationship as part of authentication before interpreting scopes. A same-named scope inside a token issued for another service is not sufficient. Map coarse scopes to object-level checks instead of storing every document entitlement in a long-lived token.

Handle revocation explicitly. Effective privileges can remain in an existing session or token until refresh or expiry, depending on the platform. Set acceptable revocation latency, shorten privileged sessions where necessary, and use supported termination or invalidation mechanisms for urgent removal.

## Distinguish visibility from enforcement

UI5 should reflect permissions to avoid confusing or unusable workflows. A server endpoint may return minimal capability hints after applying its trusted policy. Bind them to control properties, but treat them as presentation data:

```xml
<mvc:View
    xmlns:mvc="sap.ui.core.mvc"
    xmlns="sap.m">
    <Button
        text="Approve"
        visible="{permissions>/maySeeApproval}"
        enabled="{= ${permissions>/mayApprove} === true }"
        press=".onApprove" />
</mvc:View>
```

The expression binding evaluates an enabled-state condition. It is not an XSS sanitizer and does not enforce authorization. A user can modify the model, trigger the event handler, or call the API without rendering this view.

Initialize capabilities to false and handle loading failures conservatively. Refresh them when the relevant object or session changes, but assume they can become stale. If a backend denies an action, show a controlled message and reconcile the UI rather than repeatedly retrying.

Avoid downloading sensitive data merely to hide its field. Visibility restrictions do not remove values from network responses, model state, exports, or developer tools. Use server-side projections for field-level protection, as explained in [data protection](data-protection.md).

## Close alternate access paths

Inventory collection reads, entity reads, navigation properties, `$expand`, `$count`, bound actions, unbound operations, attachments, exports, background jobs, and batch requests. Each can cross a different authorization boundary.

A `$batch` request is not one indivisible authorization decision. Evaluate each embedded operation and preserve transaction semantics for change sets. A permitted outer endpoint must not authorize every enclosed document identifier.

Server-generated exports need the same row and field restrictions as interactive reads, plus any separate export capability. When an export runs asynchronously, bind the job and resulting artifact to the authenticated requester and tenant. Recheck access at download time and define what happens if permission is revoked while the job is running.

Search indexes, analytical aggregates, and caches must preserve authorization context. Never cache a user-filtered response under a key based solely on URL. Include the necessary principal or entitlement context, or avoid shared caching of protected results.

## Test denial as a first-class behavior

Create synthetic users with no roles, read-only roles, narrowly scoped approver roles, and controlled administrative roles. Pair them with documents in different organizations and tenants. Test allowed and denied cases for the same operation so a broken feature is not mistaken for successful enforcement.

Send direct API requests using approved test infrastructure. Change object identifiers, organizational parameters, navigation paths, and requested fields. Submit an unauthorized property even if the UI makes it read-only. Confirm both the response and the absence of database changes, emitted events, generated files, and downstream calls.

For authorization changes, test session refresh and revocation latency. Include a draft created before revocation, a queued export, and an in-flight approval where the platform supports those scenarios. Validate concurrent changes to document ownership or approval state.

Record the policy case, fixture role, operation, expected result, observed result, and sanitized correlation identifier. Do not attach real employee role dumps or production document data to a general development issue.

## Govern assignments and exceptions

Assign owners for role definitions, organizational values, approval thresholds, and emergency access. Periodically review effective assignments, not just role design. Remove obsolete grants when users change responsibilities, leave the organization, or stop needing a temporary project role.

Separation of duties is not merely an application checkbox. Coordinate role provisioning and transactional checks so creating a supplier, changing bank details, and approving payment cannot accidentally converge in one uncontrolled identity. Document permitted compensating controls and their monitoring.

Emergency access needs a bounded duration, named approver, purpose, and retrospective review. Do not embed an emergency bypass parameter in a UI5 route or hide one in minified JavaScript. Administrative paths must be explicit, authenticated, authorized, and audited.

Review policy drift after upgrades, new projections, service regeneration, and role transports. A new endpoint or association can expand accessible data without changing existing role names.

## Implementation checklist

- [ ] Document capabilities, organizational dimensions, and separation-of-duties conditions.
- [ ] Map each request path to a trusted principal and concrete enforcement point.
- [ ] Select real business authorization objects and fields from the target SAP system.
- [ ] Review PFCG or cloud IAM assignments using the deployment's supported administration.
- [ ] Enforce instance, field, action, tenant, and state restrictions on the backend.
- [ ] Verify CDS and RAP behavior rather than assuming annotations are sufficient.
- [ ] Treat UI visibility and capability hints as usability only.
- [ ] Protect batch operations, attachments, exports, counts, and background work.
- [ ] Test negative cases and prove absence of unauthorized side effects.
- [ ] Define revocation latency, periodic review, and emergency access ownership.

Use [security patterns](security-patterns.md) for transaction-bound designs and the [code review checklist](code-review-checklist.md) for release evidence.

## Sources

- **S1 — SAP ABAP Authorization Checks:** https://github.com/SAP-samples/abap-cheat-sheets/blob/main/25_Authorization_Checks.md
- **S2 — SAPUI5 Browser Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/browser-security-91f3b66.md
- **S3 — OWASP Authorization Cheat Sheet:** https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Authorization_Cheat_Sheet.md
- **S4 — SAP BTP Security Services Integration Libraries:** https://github.com/SAP/cloud-security-services-integration-library/blob/main/README.md
- **S5 — SAPUI5 Server Security:** https://github.com/SAP-docs/sapui5/blob/main/docs/05_Developing_Apps/server-security-91f3cac.md

S4 provides the trusted middleware foundation for the BTP examples. S5 explains CORS at the server boundary: browser cross-origin policy is not a replacement for any business authorization described here. Source examples must be adapted to the installed SAP release and reviewed authorization design.
