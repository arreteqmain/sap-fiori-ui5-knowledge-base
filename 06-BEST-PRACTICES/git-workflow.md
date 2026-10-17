# Git Workflow for SAP Fiori and UI5

## Scope and prerequisites

This guide defines a practical branch and release workflow for a UI5 application maintained by a team. It covers source control, review, integration, release promotion, and backport discipline. It does not prescribe a hosting vendor, transport system, deployment action, or new dependency.

Assume an existing Git repository, a remote named `origin`, reviewed CI scripts, a known UI5 runtime baseline, and a documented deployment target. Examples use ordinary Git commands and symbolic branch names. Replace illustrative identifiers with real values only after inspecting repository state. Never run a history-changing command automatically against another contributor's branch.

Before adopting the workflow, identify the release cadence, required approval roles, supported maintenance versions, deployment mechanism, and rollback constraints. A source branch is not an environment. The same release artifact should move through test and production wherever the platform permits; rebuilding the same commit under different unrecorded dependencies weakens traceability.

## Assign branch responsibilities

Use `main` for the reviewed production release line, `develop` for the next integrated release, and short-lived `feature/*` branches for individual changes. Protect both long-lived branches with required checks and reviews. Restrict direct pushes and define emergency exceptions explicitly.

The essential invariant is that `develop` remains deployable to an integration environment. It may contain unreleased features, but it must not become a dumping ground for known broken work. Integrate incomplete features behind approved, server-enforced capability boundaries or keep them out until a safe increment exists.

Use `release/*` only when stabilization must proceed while new work continues on `develop`. Limit release branches to fixes, release metadata, and necessary documentation. A release branch that accepts new features indefinitely becomes another develop branch and multiplies merge risk.

Use `hotfix/*` from the production tag or current `main` when a production repair cannot wait. Define how the repair reaches every supported forward branch. A hotfix that exists only in production can disappear at the next release.

## Start a bounded feature

Describe the user outcome, affected services, runtime constraints, and verification plan before branching. Identify whether the change touches generated Fiori elements content, supported extension points, or application-owned modules. Do not treat generated output as the authoritative source.

The following commands illustrate creating a new branch from the latest fetched integration line. They assume a clean working tree and an existing `origin/develop`. Inspect local changes and remote configuration first.

```sh
git status --short
git fetch origin
git switch --create feature/order-validation origin/develop
```

Keep the branch focused on one reviewable outcome. Separate a formatter migration, framework upgrade, and business-rule change unless their coupling is unavoidable and explained. Smaller coherent diffs reduce review time and make selective backports safer.

Avoid keeping a feature branch open across many release cycles. Split large work into compatible increments: introduce an unused service boundary, add a guarded screen, then enable the completed journey. Each increment must preserve existing behavior and remain testable.

## Write useful commits

A commit should represent a coherent change that a reviewer can explain and, ideally, revert independently. Use an imperative subject and explain motivation when the diff cannot. A message such as “Prevent duplicate order submission while pending” is more useful than “Changes” or a ticket number alone.

Include the relevant issue reference according to repository convention. Record compatibility constraints, migration implications, and non-obvious tradeoffs in the body. Do not include credentials, customer payloads, or confidential service URLs in commit messages; rewriting Git history later does not reliably remove copies from remotes and caches.

Inspect staged changes before committing. Stage specific files or hunks rather than assuming every modified file belongs to the task. Generated preload bundles, local service destinations, browser profiles, and debug traces are common accidental additions in UI5 work.

If the project uses a structured commit convention for release automation, follow the existing convention without inventing new categories. Commit syntax is not a substitute for meaningful release notes. A business-visible compatibility change must still be described for operators and consumers.

## Keep source and artifacts distinct

Commit authored views, controllers, models, descriptors, approved metadata fixtures, and lockfiles according to project policy. Build output normally belongs in a release artifact store rather than in ordinary feature commits. If an ABAP deployment workflow requires packaged resources in a particular repository, document that exception and its regeneration process.

Never edit `Component-preload.js` to repair a release. UI5 Builder derives preloads and minified resources from source; task behavior depends on project type and configuration [1]. A manual bundle edit can disappear at the next build and cannot be reviewed reliably against authored modules.

Record the toolchain and framework versions used to produce an artifact. The UI5 runtime version, UI5 CLI version, Node version, and deployment adapter version are different pieces of evidence. A source tag alone does not prove what binaries or framework resources were shipped.

## Synchronize without surprising collaborators

Fetch and inspect upstream changes regularly. Decide at repository level whether personal feature branches are rebased or merged from `develop`. Rebasing creates a cleaner linear story but rewrites commit identities; merging preserves published history but introduces integration commits.

Rebase only branches you own when collaborators are not depending on their current history. Never force-push a shared long-lived branch. If the repository permits rewriting a personal feature branch, use the documented safe push procedure and coordinate with reviewers so obsolete review comments are not mistaken for current findings.

Resolve conflicts semantically. A descriptor merge can be syntactically valid while pointing two routes to the wrong target. An XML merge can retain both sides but duplicate local IDs. A lockfile conflict should be resolved through the project's package-manager process rather than arbitrary line selection.

After synchronization, rerun the smallest relevant validation. Integration can change model initialization, library dependencies, or fixture assumptions even when Git reports no textual conflict. Clean merges are not evidence of behavioral compatibility.

## Open a reviewable pull request

Target feature pull requests at `develop` under this workflow. State the problem, user-visible behavior, implementation boundaries, risk, and verification evidence. Include exact runtime assumptions and any service contract dependencies.

List what was tested and what was not. “Tests pass” is insufficient if the change requires a deployed shell or a backend operation absent from mocks. Distinguish local QUnit evidence, OPA5 evidence, manual keyboard checks, and controlled real-service validation.

For visual changes, provide approved screenshots only after checking them for confidential data. Never expose customer orders or authenticated session details in public artifacts. Explain responsive and accessibility behavior in text because a screenshot cannot prove focus order or screen-reader output.

Keep draft status while major design or verification questions remain. Request reviewers with relevant ownership: service contracts, accessibility, release operations, or application architecture. Avoid treating reviewer count as a substitute for the correct expertise.

## Apply meaningful review gates

Require existing static checks, targeted unit tests, integration journeys, and build validation appropriate to the change. UI5's QUnit and OPA5 documentation provides the framework testing foundation [2][3]; project scripts determine the actual automation.

Reviewers should inspect async cleanup, stale navigation results, stable IDs, localization, public APIs, error recovery, and logging. Confirm that tests assert behavior rather than only private call sequences. Check that newly introduced caches do not mix users or retain sensitive responses.

Security-sensitive jobs must not expose secrets to untrusted contributions. Separate ordinary validation from deployment and privileged integration tests. Review changes to scripts and CI definitions as executable code, even when the feature appears limited to XML or documentation.

Require resolution of substantive feedback before merging. A maintainer override should identify the unresolved risk, owner, and rationale. Never normalize bypassing a broken required check without determining whether it reflects infrastructure failure or a genuine defect.

## Choose a merge policy deliberately

Squash merging works well for feature branches with exploratory commits: it gives `develop` a concise, reversible unit per feature. Preserve important reasoning in the final commit message and pull request because individual branch commits may not remain in the main history.

Merge commits preserve the branch structure and can be useful for complex coordinated work. Rebase-and-merge provides linear history but changes commit identities. Pick a default and document exceptions so automated release tooling and backport procedures remain predictable.

Do not squash the relationship between long-lived release lines casually. Repeatedly squash-merging `develop` into `main` can make future ancestry comparisons confusing. Prefer a release integration strategy that preserves the intended ancestry or explicitly reset the branch model through a controlled, documented process.

Delete merged feature branches when repository policy permits. Retain durable evidence in commits, pull requests, tags, and artifact records, not in abandoned branch names.

## Stabilize a release

Create `release/<version>` from a known tested `develop` commit when a stabilization window is required. Record the candidate commit, included work, exclusions, backend prerequisites, and supported runtime. Stop feature intake on that branch.

Run the release-level validation against the built artifact, not only against the development server. Asynchronous loading and preload generation can expose dependency mistakes that are hidden when individual source files happen to load in a forgiving order. Official module guidance emphasizes explicit dependencies and case-sensitive names [4].

Verify deployment-specific resource paths, cache invalidation, translations, startup routes, and shell integration. Reconcile descriptor application version, release notes, and artifact metadata according to the organization's established versioning policy. Do not assume every UI5 application's marketing version must equal every package or component version.

When approved, merge the release into `main` through the protected process and create the release tag at the actual release commit. Merge release fixes back into `develop`; otherwise the next release may regress a defect already repaired during stabilization.

## Promote immutable evidence

Associate each release with a commit, tag, artifact digest, framework baseline, toolchain versions, validation results, and deployment record. Use signed tags or equivalent provenance where the repository already supports them. Treat release tags as immutable.

Promote the same artifact between environments when practical. Supply environment-specific endpoints and credentials through controlled deployment configuration rather than rebuilding source with embedded secrets. If the target platform requires a rebuild, record the full inputs and explain how equivalence is verified.

Keep artifact access aligned with source confidentiality. Source maps, debug variants, and test reports can reveal application internals or business data. UI5 Builder creates debug variants and source maps during minification [1]; decide explicitly which artifacts are deployed and which remain restricted.

Check runtime resource cache behavior during promotion. A successful upload does not prove clients receive a consistent release. Verify a fresh session and an already-open session according to supported upgrade behavior, especially where application cache-buster metadata or shell caches are involved [5].

## Repair production safely

Branch a hotfix from the exact production baseline. Reproduce the defect with synthetic or appropriately controlled evidence, add a focused regression test, and minimize unrelated changes. Avoid bundling a framework upgrade into an urgent repair unless that upgrade is the necessary fix.

Validate the repaired artifact in the relevant deployment environment. Emergency review may be accelerated, but should not eliminate independent verification of authorization, data mutations, or customer-visible failures.

After releasing, propagate the fix to `develop` and any active release branch. Choose merge or cherry-pick according to ancestry and branch policy. A cherry-pick copies a change into a new commit; record its relationship, commonly through the repository's traceability convention, so future maintainers can identify equivalent repairs.

Do not blindly cherry-pick generated files or lockfile changes between branches with different baselines. Reapply the intent using the target branch's approved toolchain, verify compatibility, and document any deviation.

## Understand trunk-based alternatives

The `main`/`develop` model supports scheduled releases, parallel stabilization, and organizations with formal transport windows. Its costs are delayed integration, duplicated merges, and drift between production and ongoing development.

Trunk-based development keeps one main integration line, uses very short-lived branches, and relies on fast reliable checks plus controlled feature exposure. It reduces long-lived divergence and often makes continuous delivery easier. It requires discipline: incomplete work must remain safe, and feature flags need owners, expiry dates, and tests in both relevant states.

Flags are not authorization. A hidden client action must still be rejected server-side when permission is absent. Do not ship confidential inactive code or secrets merely because a flag is off.

Choose based on release constraints and feedback speed rather than fashion. A small team with daily deployment may gain little from `develop`; a regulated landscape with several maintained releases may need explicit release branches. Revisit the model when merge overhead exceeds the coordination value.

## Plan rollback and compatibility

Prefer reverting a bad change through a new reviewed commit over rewriting published production history. A revert is not always a safe operational rollback: the release may depend on backend schema changes, new persisted values, or irreversible business actions.

Document forward-fix versus rollback criteria before deployment. Keep backend interfaces backward-compatible during staged rollout where possible, and sequence changes so old and new application versions can coexist during cache turnover.

For maintenance branches targeting older UI5 runtimes, validate API availability explicitly. A syntactically clean backport can call an unsupported asynchronous fragment API or depend on newer library behavior. Track compatibility evidence per maintained line, not only on the newest branch.

## Verification checklist

- [ ] Branch roles, merge policy, protections, and emergency exceptions are documented.
- [ ] Feature work is bounded, reviewable, and based on the correct integration line.
- [ ] Commits and artifacts contain no credentials or unnecessary business data.
- [ ] CI evidence distinguishes unit, mocked integration, and deployed validation.
- [ ] Release stabilization excludes unrelated features and returns fixes to develop.
- [ ] Tags identify actual release commits and artifacts have recorded provenance.
- [ ] Cache behavior, runtime compatibility, and rollback constraints are verified.
- [ ] Hotfixes reach every required forward and maintenance branch.
- [ ] The team has considered whether trunk-based development would reduce risk.

## Related paths

The [README index](../README.md) lists these exact chapter paths.

- [Code quality](./code-quality.md): `06-BEST-PRACTICES/code-quality.md`.
- [Testing strategy](./testing-strategy.md): `06-BEST-PRACTICES/testing-strategy.md`.
- [Performance optimization](./performance-optimization.md): `06-BEST-PRACTICES/performance-optimization.md`.
- [Accessibility](./accessibility.md): `06-BEST-PRACTICES/accessibility.md`.

## Official references

Research access: the official GitHub documentation below was retrieved successfully. Hosted UI5 documentation was not directly accessible; the workflow recommendations are not claims of a mandatory SDK branching policy.

1. [UI5 CLI v4: Builder](https://github.com/UI5/cli/blob/v4/docs/pages/Builder.md), consulted 2026-10-17. Supports the distinction between authored sources, generated preloads, debug variants, and source maps.
2. [OpenUI5: Unit Testing with QUnit](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/unit-testing-with-qunit-09d145c.md), consulted 2026-10-17.
3. [OpenUI5: Cookbook for OPA5](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/cookbook-for-opa5-ce4b180.md), consulted 2026-10-17.
4. [OpenUI5: Best Practices for Loading Modules](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/best-practices-for-loading-modules-00737d6.md), consulted 2026-10-17.
5. [OpenUI5: Optimizing Applications](https://github.com/SAP/openui5-docs/blob/main/docs/04_Essentials/optimizing-applications-2f492c4.md), consulted 2026-10-17. Links runtime and application cache-buster concepts relevant to release verification.

Branch policies above are team workflow recommendations, not requirements imposed by UI5.
