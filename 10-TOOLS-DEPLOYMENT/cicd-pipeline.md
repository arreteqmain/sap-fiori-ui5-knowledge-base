# CI/CD for UI5: Trust Boundaries, Gates, and Artifact Promotion

## Scope and pipeline contract

This guide gives actual GitHub Actions YAML and Jenkins Declarative Pipeline examples for a trusted UI5 release branch. Both build a Cloud Foundry MTA and promote the same archive through quality assurance and production. ABAP systems require a different publication and transport sequence, explained below and in [deployment targets](deployment-targets.md).

Examples remain inside this document; this repository does not install executable workflow files. They assume an existing application, reviewed package files, committed lockfiles, an `mta.yaml`, and working `lint`, `test:ci`, and `build` package scripts. Those scripts are application contracts, not built-in UI5 CLI commands. Define them before adopting the example, and make each return nonzero on failure.

Use [UI5 tooling](ui5-tooling.md) for extension behavior and [build-process.md](build-process.md) for artifact inspection. The [README](../README.md) provides the broader navigation.

The verified compatibility reference is **UI5 CLI 4.0.0**, whose official tagged metadata declares Node **`^20.11.0 || >=22.0.0`** and npm **`>=8`**. The examples require an approved Node 22 image and an exact patch recorded by infrastructure; they do not download whichever Node release happens to be newest. That historical CLI package received a direct advisory check during preparation with no advisory returned. Assess the complete current lockfile independently. [C1]

## Separate validation from privileged publication

A pull-request validation job executes attacker-controlled code: package scripts, test helpers, custom tasks, and dependency installation hooks can all run commands. Never expose production credentials to that job, even if the workflow itself looks harmless. Use isolated, disposable workers without deployment network access or reusable workspace state.

The release workflow below is intentionally **push-to-main only**. It assumes protected `main`, required review, required status checks, restricted workflow modification, and organization-controlled runner access. A separate pull-request workflow should run ordinary checks without deployment secrets. Do not convert this example to a privileged pull-request event, and do not use `pull_request_target` to build an untrusted head revision.

Approval protects the transition into a deployment environment, not every earlier line of code. A compromised build can still create a malicious artifact. Therefore branch controls, dependency review, artifact evidence, and environment approval work together; none replaces the others.

Use two identities per target class: an artifact publisher with append-only object permissions and a deployer limited to a specific platform space or ABAP development client. Production must not reuse the QA identity merely because both jobs run the same command.

## GitHub Actions prerequisites

The example uses only `run` steps, deliberately avoiding third-party `uses` actions and their version-selection concerns. GitHub does **not** automatically check out repository content for a run step. The first step performs an explicit Git fetch of the event commit. This version assumes a publicly readable repository; a private repository needs an organization-approved, short-lived Git credential mechanism that does not persist credentials in the remote URL.

Provision these items before running:

1. An ephemeral runner labeled `ui5-build-trusted` with Git, approved Node 22, npm, GNU Make, and the approved Cloud MTA Build Tool.
2. Ephemeral runners labeled `ui5-deploy-trusted` with curl, SHA-256 utilities, Cloud Foundry CLI v8, and the approved MultiApps plugin. The plugin officially requires CF CLI v8. Configure the image's `CF_PLUGIN_HOME` independently from the per-job `CF_HOME`, so isolated session directories do not hide the installed plugin. [C5][C9]
3. An HTTPS artifact service whose documented interface supports authenticated `PUT` and `GET` at the shown object paths. Configure write-once objects; ordinary filesystem hosting does not automatically satisfy this contract.
4. Repository variable `ARTIFACT_BASE_URL`, without trailing slash, and `APPROVED_NODE_VERSION`, including the leading `v` returned by Node.
5. Secret `ARTIFACT_WRITE_TOKEN`, scoped only to publishing the application prefix.
6. Protected `qa` and `production` environments, each with its own `ARTIFACT_READ_TOKEN`, `CF_CONFIG_JSON`, and variables `CF_ORG`, `CF_SPACE`.

`CF_CONFIG_JSON` is a short-lived, preauthenticated CF CLI configuration issued by an approved credential broker for that environment. It is sensitive token material, not a service key copied into source control. The broker must issue tokens with enough lifetime for deployment and revoke or expire them afterwards. If your organization instead authenticates with a dedicated automation account or supported federation, replace this injection mechanism with that documented login procedure.

Environment protection rules and secret availability depend on GitHub plan and repository visibility. Configure reviewers, branch restrictions, and prevention of self-review where available; naming an environment in YAML alone does not create a meaningful approval gate. GitHub documents when environment secrets become available to jobs. [C2][C3]

## GitHub Actions release example

```yaml
name: UI5 trusted release
on:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ui5-production-release
  cancel-in-progress: false

jobs:
  build:
    runs-on: [self-hosted, ui5-build-trusted]
    timeout-minutes: 30
    outputs:
      digest: ${{ steps.digest.outputs.value }}
    steps:
      - name: Fetch exact public source revision
        shell: bash
        env:
          SOURCE_URL: ${{ github.server_url }}/${{ github.repository }}.git
        run: |
          set -euo pipefail
          test -z "$(ls -A)"
          git init .
          git remote add origin "$SOURCE_URL"
          git fetch --depth=1 origin "$GITHUB_SHA"
          git checkout --detach FETCH_HEAD
          test "$(git rev-parse HEAD)" = "$GITHUB_SHA"

      - name: Restore, validate, and package
        shell: bash
        env:
          APPROVED_NODE_VERSION: ${{ vars.APPROVED_NODE_VERSION }}
        run: |
          set -euo pipefail
          test "$(node --version)" = "$APPROVED_NODE_VERSION"
          test "$(node -p 'process.versions.node.split(".")[0]')" = 22
          npm ci
          npm run lint
          npm run test:ci
          mbt build -p cf --mtar app.mtar
          test -s mta_archives/app.mtar
          ./node_modules/.bin/ui5 --version
          node --version
          npm --version
          mbt -v

      - name: Calculate artifact identity
        id: digest
        shell: bash
        run: |
          set -euo pipefail
          digest="$(sha256sum mta_archives/app.mtar | cut -d' ' -f1)"
          echo "value=$digest" >> "$GITHUB_OUTPUT"
          printf 'Commit: %s\nSHA256: %s\n' \
            "$GITHUB_SHA" "$digest" >> "$GITHUB_STEP_SUMMARY"

      - name: Publish write-once candidate
        shell: bash
        env:
          ARTIFACT_BASE_URL: ${{ vars.ARTIFACT_BASE_URL }}
          ARTIFACT_WRITE_TOKEN: ${{ secrets.ARTIFACT_WRITE_TOKEN }}
        run: |
          set -euo pipefail
          set +x
          curl --fail --silent --show-error \
            --oauth2-bearer "$ARTIFACT_WRITE_TOKEN" \
            --upload-file mta_archives/app.mtar \
            "$ARTIFACT_BASE_URL/$GITHUB_SHA/app.mtar"

  qa:
    needs: build
    environment: qa
    runs-on: [self-hosted, ui5-deploy-trusted]
    timeout-minutes: 30
    steps:
      - name: Verify candidate and deploy to QA
        shell: bash
        env:
          EXPECTED_SHA256: ${{ needs.build.outputs.digest }}
          ARTIFACT_BASE_URL: ${{ vars.ARTIFACT_BASE_URL }}
          ARTIFACT_READ_TOKEN: ${{ secrets.ARTIFACT_READ_TOKEN }}
          CF_CONFIG_JSON: ${{ secrets.CF_CONFIG_JSON }}
          CF_ORG: ${{ vars.CF_ORG }}
          CF_SPACE: ${{ vars.CF_SPACE }}
        run: |
          set -euo pipefail
          set +x
          umask 077
          export CF_HOME="$PWD/.cf-session"
          mkdir -p "$CF_HOME/.cf"
          trap 'rm -rf "$CF_HOME"' EXIT
          printf '%s' "$CF_CONFIG_JSON" > "$CF_HOME/.cf/config.json"
          curl --fail --silent --show-error \
            --oauth2-bearer "$ARTIFACT_READ_TOKEN" \
            "$ARTIFACT_BASE_URL/$GITHUB_SHA/app.mtar" -o app.mtar
          printf '%s  app.mtar\n' "$EXPECTED_SHA256" | sha256sum -c -
          cf target -o "$CF_ORG" -s "$CF_SPACE"
          cf deploy app.mtar
          cf mtas

  production:
    needs: [build, qa]
    environment: production
    runs-on: [self-hosted, ui5-deploy-trusted]
    timeout-minutes: 30
    steps:
      - name: Promote the verified archive
        shell: bash
        env:
          EXPECTED_SHA256: ${{ needs.build.outputs.digest }}
          ARTIFACT_BASE_URL: ${{ vars.ARTIFACT_BASE_URL }}
          ARTIFACT_READ_TOKEN: ${{ secrets.ARTIFACT_READ_TOKEN }}
          CF_CONFIG_JSON: ${{ secrets.CF_CONFIG_JSON }}
          CF_ORG: ${{ vars.CF_ORG }}
          CF_SPACE: ${{ vars.CF_SPACE }}
        run: |
          set -euo pipefail
          set +x
          umask 077
          export CF_HOME="$PWD/.cf-session"
          mkdir -p "$CF_HOME/.cf"
          trap 'rm -rf "$CF_HOME"' EXIT
          printf '%s' "$CF_CONFIG_JSON" > "$CF_HOME/.cf/config.json"
          curl --fail --silent --show-error \
            --oauth2-bearer "$ARTIFACT_READ_TOKEN" \
            "$ARTIFACT_BASE_URL/$GITHUB_SHA/app.mtar" -o app.mtar
          printf '%s  app.mtar\n' "$EXPECTED_SHA256" | sha256sum -c -
          cf target -o "$CF_ORG" -s "$CF_SPACE"
          cf deploy app.mtar
          cf mtas
```

The MTA descriptor must make `mbt build` the authoritative application build and run the reviewed application build script in its UI module. The tests run before packaging; artifact-level optimized-output tests should be included in that reviewed module build or in an additional explicit gate before publication. MBT's `-p`, `--mtar`, output location, and Make prerequisite are documented, not invented UI5 deployment flags. [C4]

QA deployment success is only an infrastructure gate in this example. Before approving production, reviewers must inspect authenticated browser smoke results, expected application version, backend connectivity, role checks, and failed-operation logs. `cf mtas` provides inventory, not proof of user-facing health. Add an organization-owned automated smoke step with a narrow test identity when its implementation exists; do not add a fictional command and assume it passes.

The digest is obtained from the trusted build job rather than downloaded beside the artifact from the same mutable endpoint. Production downloads the same commit-addressed object and checks that original digest. This detects artifact replacement, but not a compromised trusted build. Stronger environments should require signed provenance and a separate verification authority.

## Jenkins prerequisites and trust model

The Jenkins example assumes a **dedicated main-branch Pipeline-from-SCM job**, not an unrestricted multibranch job accepting fork Jenkinsfiles. Configure its SCM branch to protected `main`, restrict job configuration permission, and use an ephemeral agent labeled `ui5-release-trusted`. The agent has the same approved toolchain as above.

Required installed Jenkins capabilities are Declarative Pipeline, SCM checkout, Credentials Binding, Pipeline Input Step, and artifact archival. These are existing administrator-managed Jenkins components; the snippet does not prescribe new plugin versions. `checkout scm`, `archiveArtifacts`, environment bindings, and single-quoted shell scripts follow the Jenkins handbook. [C6][C7]

Create credential IDs `cf-qa-session` and `cf-prod-session` as secret files containing broker-issued short-lived CF configurations. Set nonsecret job environment variables `APPROVED_NODE_VERSION`, `QA_ORG`, `QA_SPACE`, `PROD_ORG`, and `PROD_SPACE`. Restrict production credentials to the release job's folder. Never attach them to general pull-request builders.

## Jenkins Declarative Pipeline example

```groovy
pipeline {
  agent { label 'ui5-release-trusted' }
  options {
    skipDefaultCheckout(true)
    disableConcurrentBuilds()
    timeout(time: 90, unit: 'MINUTES')
  }
  stages {
    stage('Checkout protected main') {
      steps {
        deleteDir()
        checkout scm
        sh 'git rev-parse HEAD'
      }
    }
    stage('Validate and package') {
      steps {
        sh '''
          set -eu
          test "$(node --version)" = "$APPROVED_NODE_VERSION"
          test "$(node -p 'process.versions.node.split(".")[0]')" = 22
          npm ci
          npm run lint
          npm run test:ci
          mbt build -p cf --mtar app.mtar
          test -s mta_archives/app.mtar
          cd mta_archives
          sha256sum app.mtar > app.mtar.sha256
        '''
        archiveArtifacts artifacts: 'mta_archives/app.mtar*',
          fingerprint: true
      }
    }
    stage('Deploy QA') {
      steps {
        withCredentials([file(
          credentialsId: 'cf-qa-session', variable: 'CF_SESSION'
        )]) {
          sh '''
            set -eu
            set +x
            umask 077
            export CF_HOME="$WORKSPACE/.cf-qa"
            mkdir -p "$CF_HOME/.cf"
            trap 'rm -rf "$CF_HOME"' EXIT
            cp "$CF_SESSION" "$CF_HOME/.cf/config.json"
            cd mta_archives
            sha256sum -c app.mtar.sha256
            cf target -o "$QA_ORG" -s "$QA_SPACE"
            cf deploy app.mtar
            cf mtas
          '''
        }
      }
    }
    stage('Production approval') {
      steps {
        timeout(time: 20, unit: 'MINUTES') {
          input message: 'Approve digest, QA evidence, and change record?',
            submitter: 'ui5-release-managers'
        }
      }
    }
    stage('Promote production') {
      steps {
        withCredentials([file(
          credentialsId: 'cf-prod-session', variable: 'CF_SESSION'
        )]) {
          sh '''
            set -eu
            set +x
            umask 077
            export CF_HOME="$WORKSPACE/.cf-production"
            mkdir -p "$CF_HOME/.cf"
            trap 'rm -rf "$CF_HOME"' EXIT
            cp "$CF_SESSION" "$CF_HOME/.cf/config.json"
            cd mta_archives
            sha256sum -c app.mtar.sha256
            cf target -o "$PROD_ORG" -s "$PROD_SPACE"
            cf deploy app.mtar
            cf mtas
          '''
        }
      }
    }
  }
  post {
    always {
      deleteDir()
    }
  }
}
```

The dedicated agent and unchanged workspace preserve the archive across stages. Production does not run npm or MBT again. For separate stage agents, replace workspace continuity with an authenticated immutable artifact repository and verify the recorded digest after retrieval. Jenkins archival is useful evidence but, as the handbook warns, is not a complete replacement for a release artifact repository. [C6]

The checksum file in this compact Jenkins example shares the trusted workspace with the artifact. It detects accidental changes, not a malicious process able to rewrite both. Archive the digest in a separate approved release record before promotion when stronger integrity is required.

Single-quoted Groovy shell strings avoid interpolating secrets in Groovy. Shell tracing is disabled around credentials. Neither practice stops malicious code from reading environment variables or files; therefore no untrusted builds may share the agent account or executor. Agent disposal remains the final cleanup boundary if a process is terminated before traps run. [C6]

## Session isolation and rerun policy

Cloud Foundry CLI resolves the session file beneath `$CF_HOME/.cf/config.json`; `CF_HOME` is not the filename or the `.cf` directory itself. The examples create that subdirectory explicitly. The runner image must separately expose its approved plugin installation through `CF_PLUGIN_HOME`. Verify this layout during image qualification with `cf plugins` before issuing real deployment credentials. Otherwise a clean credential isolation improvement can unexpectedly make `cf deploy` unavailable. [C9]

Treat reruns as release decisions. The sample artifact service uses commit-addressed, write-once objects, so a second full run for the same commit may intentionally fail publication. Resume only the failed deployment job when its original digest is still available and the server-side operation state is understood. If policy requires a complete rebuild, issue a distinct candidate identity and repeat QA approval; do not overwrite an approved object under the old URL.

Credential lifetime must cover approval delays as well as command execution. A configuration issued before a long production approval can expire before the deploy job starts. Production-grade brokers should issue the session after approval, or administrators must rotate the environment secret immediately before the permitted job. Never increase a token's scope or lifetime merely to conceal a missing credential-refresh design.

## Gates that make promotion meaningful

Organize gates by failure class. Source gates include code review, dependency review, lint, unit tests, and policy checks. Artifact gates include descriptor identity, expected preloads, exclusion of mocks and secrets, optimized-output browser tests, and size budgets. Target gates include service availability, destination correctness, authentication, authorization, and deployment operation completion.

Require evidence for negative cases. A UI with a visible launchpad tile does not prove that an unauthorized user cannot call the backend. A service returning HTTP 200 might be a login page. A static health check should verify content identity, while an authenticated functional check should exercise a representative read and a safe business operation in a controlled tenant.

Keep flaky tests visible instead of appending `|| true`. If a test is quarantined, record the owner, reason, expiry, and compensating control. A pipeline that always returns green cannot enforce a release gate. Do not use missing scripts as optional checks by default; fail clearly when the contract is absent.

## Test reports, retention, and release accountability

Keep machine-readable test reports with the candidate identity, but do not let report collection weaken the test command's exit status. In Jenkins, publishing a report can mark a build unstable instead of failed depending on the reporting step and configuration. Explicitly prevent unstable candidates from entering deployment if your project adds such reporting. In GitHub Actions, a step configured to continue on error similarly needs a separate enforced policy decision.

Retention should reflect incident response needs rather than default workspace cleanup. Preserve the deployed archive, its digest, source revision, toolchain inventory, target configuration digest, approval record, and deployment-operation identifier. Retain source maps privately if they are excluded from public hosting. Test screenshots may contain business data, so apply the same access controls and expiry policy as other operational evidence.

Assign ownership of each gate. Developers own application tests; platform owners own target readiness and credential integrations; release managers own production approval criteria. If every failed gate is routinely overridden by the same administrator, document that residual risk rather than claiming independent separation of duties. A useful release record makes it possible to answer who approved which bytes for which environment, even after the original CI job and its disposable agent have disappeared.

## ABAP publication and transport promotion

For an on-premise ABAP target, replace CF deployment with the documented SAP ABAP deploy tool and a protected development-system credential. Upload to a transportable package, record the workbench request, and retain the application artifact digest with that transport. The CLI supports deployment independent of UI5 build and explicit archive inputs. [C8]

Promote the transport through the organization's CTS or approved change-management process. Do not rebuild and independently upload the same application into QA and production under three unrelated technical users. That loses transport traceability and may bypass approvals. Gateway service activation, system aliases, authorization roles, and launchpad content have their own lifecycle; the UI archive alone does not transport all of them.

## Recovery and operational checklist

Serialize deployments to a shared target. Cancelling a running workflow does not guarantee that a server-side MTA operation stops. Inspect active operations before retrying, preserve logs, and distinguish an upload interruption from a deployment failure. The MultiApps plugin exposes operation inventory and log-download commands. [C5]

Retain the previous approved archive and target configuration. A rollback is a new controlled deployment of known bytes, not necessarily a safe downgrade of backend data or authorization configuration. Blue-green deployment requires application-specific routing and compatibility analysis; merely changing a command does not solve state migration.

Before enabling production, verify protected source, isolated workers, exact toolchain inventory, locked restoration, mandatory tests, artifact identity, write-once storage, target-scoped credentials, actual environment approvals, deployment serialization, authenticated QA evidence, previous-release retention, and an incident owner. Document which checks are automated and which require human approval. This prevents the pipeline's attractive diagram from overstating its real enforcement.

## Official sources

Consulted on 2026-10-17:

* **[C1]** SAP, [CLI 4.0.0 metadata](https://github.com/SAP/ui5-cli/blob/v4.0.0/package.json).
* **[C2]** GitHub, [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax), for triggers, jobs, environments, outputs, and permissions.
* **[C3]** GitHub, [Managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments) and [Using secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets).
* **[C4]** SAP, [MBT usage source](https://github.com/SAP/cloud-mta-build-tool/blob/master/docs/docs/usage.md).
* **[C5]** Cloud Foundry MultiApps, [official plugin README](https://github.com/cloudfoundry/multiapps-cli-plugin/blob/master/README.md).
* **[C6]** Jenkins, [Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/), also read through its [official source](https://github.com/jenkins-infra/jenkins.io/blob/master/content/doc/book/pipeline/jenkinsfile.adoc).
* **[C7]** Jenkins, [Declarative syntax](https://www.jenkins.io/doc/book/pipeline/syntax/), researched through its [official source](https://github.com/jenkins-infra/jenkins.io/blob/master/content/doc/book/pipeline/syntax.adoc).
* **[C8]** SAP, [ABAP deployment tooling](https://github.com/SAP/open-ux-tools/blob/main/packages/deploy-tooling/README.md).
* **[C9]** Cloud Foundry CLI, [configuration loading source](https://github.com/cloudfoundry/cli/blob/main/util/configv3/load_config.go) and [plugin-home resolution](https://github.com/cloudfoundry/cli/blob/main/cf/configuration/confighelpers/config_helpers.go), for isolated session and plugin directory semantics.
