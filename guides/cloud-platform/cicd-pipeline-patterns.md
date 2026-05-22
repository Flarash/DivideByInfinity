# CI/CD pipeline patterns — Azure DevOps, GitLab CI, Jenkins

> The shape of a production CI/CD pipeline is the same across the big three.
> What differs is how each platform models secrets, runners, environments,
> and approvals. This guide is the shared pattern, the platform-specific
> notes, and the gotchas per platform.

---

## TL;DR

- **The pipeline shape is universal:** lint → build → test → scan → publish
  artifact → deploy (per env, gated).
- **Build once, deploy many.** Promote artifacts, don't rebuild per env.
- **Secrets come from a vault** (Key Vault / GitLab vault / Jenkins
  credentials), never inline in YAML.
- **Per-PR ephemeral environments** are the single biggest reviewer experience
  upgrade you can ship.
- **Manual approval gates live on the deploy job to prod**, not on merge.
- Each platform has its own footgun. Read the section for yours.

---

## The shared pipeline shape

```
                  ┌────────────────┐
on PR ──►         │ lint + format  │
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ unit tests     │
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ build          │ ──► artifact (container image,
                  └───────┬────────┘     zip, jar, package)
                          ▼
                  ┌────────────────┐
                  │ security scan  │ ──► SBOM, vuln report,
                  │ (SAST + deps)  │     IaC scan
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ integration    │ ──► against ephemeral env
                  │ tests          │
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
on merge ──►      │ publish        │ ──► immutable, tagged
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ deploy dev     │ ──► auto
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ deploy staging │ ──► auto, smoke tests after
                  └───────┬────────┘
                          ▼
                  ┌────────────────┐
                  │ deploy prod    │ ──► manual approval gate
                  └────────────────┘
```

**Build once.** The artifact published after merge is the same artifact that
goes to dev, staging, and prod. Configuration is injected per environment;
the binary/image isn't rebuilt.

If you rebuild per env, "it works in staging, fails in prod" is allowed to
exist. With promote-the-artifact, it isn't.

---

## Azure DevOps Pipelines

### Strengths

- Cleanest integration with Azure Key Vault and Azure resources
- **Environments** are first-class: approvals, checks, deployment history per
  environment
- YAML pipelines + reusable templates

### Pattern

```yaml
# azure-pipelines.yml
trigger:
  branches: { include: [main] }

variables:
  - group: shared-build-vars       # variable group
  - group: secrets-from-keyvault   # Key Vault-linked group

stages:
  - stage: build
    jobs:
      - job: build_and_publish
        steps:
          - task: AzureKeyVault@2
            inputs:
              azureSubscription: 'sc-shared'
              KeyVaultName: 'kv-cicd'
              SecretsFilter: 'registry-password'
          - script: |
              docker build -t $(image):$(Build.BuildId) .
              docker push $(image):$(Build.BuildId)

  - stage: deploy_prod
    dependsOn: build
    jobs:
      - deployment: prod
        environment: prod         # ◄── approvals attach here
        strategy:
          runOnce:
            deploy:
              steps:
                - script: ./deploy.sh prod
```

### Gotchas

- **Variable groups vs Key Vault-linked variable groups.** Plain groups have
  values in the UI (and anyone with edit can read them). Key Vault-linked
  groups fetch at runtime — use these for secrets.
- **`isOutput: true` is required** to pass variables across jobs/stages. Easy
  to forget; fails silently with empty strings downstream.
- **Self-hosted agents need `Allow scripts to access the OAuth token`** set
  on the job to call back into Azure DevOps APIs. The error is opaque.
- **Service connections need granular scope.** A wildcard subscription-level
  connection is a foot-cannon — scope per resource group when you can.
- **Environment approvals don't apply to retries** in some configurations.
  Test it. The first time you hit an approval-bypass you'll wish you had.
- **Variables vs parameters vs runtime parameters.** Three things that look
  similar; only runtime parameters are typed and validated. Prefer them for
  user input.

---

## GitLab CI/CD

### Strengths

- Pipeline + repo + registry + secrets all in one product
- **Protected variables** + **protected environments** model is genuinely good
- `rules:` give you fine control over when jobs run
- Native ephemeral environments (`environment.action: stop`)

### Pattern

```yaml
# .gitlab-ci.yml
stages: [lint, build, test, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

lint:
  stage: lint
  script: [npm ci, npm run lint]

build:
  stage: build
  image: docker:stable
  services: [docker:dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

review:
  stage: deploy
  script: ./deploy-review.sh
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://$CI_MERGE_REQUEST_IID.review.example.com
    on_stop: stop_review
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop_review:
  stage: deploy
  script: ./destroy-review.sh
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual

deploy_prod:
  stage: deploy
  script: ./deploy.sh prod
  environment:
    name: prod
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual          # approval = manual job
```

### Gotchas

- **`protected: true` on variables matters.** Without it, a feature branch
  can read prod secrets by adding a job that echoes them. Always pair
  protected variables with **protected branches**.
- **`when: manual` is the closest thing to an approval gate in GitLab CE.**
  GitLab Premium has proper protected environments with approver lists; CE
  has "anyone with run permission can press the button". Lock down by branch
  protection.
- **DinD vs Kaniko vs BuildKit.** DinD with privileged mode works everywhere
  but is privileged. Kaniko is rootless but slow for big builds. Pick one and
  stick to it across the org.
- **The shared runners on GitLab.com are heterogeneous.** Same job can take
  3 minutes or 20 minutes. For anything time-sensitive, use group/project
  runners.
- **Cache keys are stringy.** `key: $CI_COMMIT_REF_NAME` works until you have
  a branch with `/` in the name on Windows runners. Use `$CI_COMMIT_REF_SLUG`.
- **`needs:` vs `dependencies:`.** `needs` is for execution order
  (DAG-style); `dependencies` is for artifact passing. Easy to mis-wire.

---

## Jenkins

### Strengths

- Universal — runs anywhere, integrates with anything
- Shared libraries let you publish your pipeline DSL across many repos
- The plugin ecosystem covers obscure stuff nobody else does

### Weaknesses

- The controller is a stateful pet by default; you'll work to keep it as cattle
- Plugin churn breaks things. CVE patching is a continuous chore.
- "Job DSL" vs "Pipeline" vs "Declarative Pipeline" vs "Scripted Pipeline" —
  pick one and ban the others, or your repo will have all four.

### Pattern (Declarative Pipeline + shared library)

```groovy
// Jenkinsfile
@Library('org-pipeline-lib@v3') _

pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  stages {
    stage('build')  { steps { ciBuild() } }
    stage('test')   { steps { ciTest() } }
    stage('scan')   { steps { ciScan() } }
    stage('publish') {
      when { branch 'main' }
      steps { ciPublish() }
    }
    stage('deploy prod') {
      when { branch 'main' }
      input { message 'Deploy to prod?'; submitter 'releases' }
      steps { ciDeploy('prod') }
    }
  }
}
```

The repo's Jenkinsfile is trivial; the logic lives in `org-pipeline-lib`.

### Gotchas

- **Configuration as Code (JCasC) or you're snowflaking your controller.**
  Click-ops in the Jenkins UI is the road to a controller that nobody dares
  upgrade. Store config in git, apply it at boot.
- **Plugins pin themselves to controller version.** A core upgrade can break
  several plugins; a plugin upgrade can demand a core upgrade. Always test
  on a staging controller with a copy of prod config.
- **Credentials scoping.** Credentials at the global level are available to
  any job — including a Jenkinsfile a developer just pushed. Scope to
  folders/jobs, and require approval for new shared library imports.
- **Ephemeral agents matter.** Run jobs on Kubernetes pods (the `kubernetes`
  plugin) or EC2/Azure VM agents that get torn down after the build. Static
  agents accumulate state, secrets in caches, and other people's leftovers.
- **`input` blocks pause the executor.** A pipeline waiting for human
  approval holds an executor slot. On a busy controller, this starves other
  jobs. Use the `input` outside of an `agent` block:

  ```groovy
  stage('approval') {
    agent none
    input { message 'Deploy?' }
  }
  ```

- **The "build of a build" antipattern.** Triggering downstream jobs via
  freeform parameters loses traceability fast. Prefer multibranch pipelines
  + shared libraries, or a real orchestrator on top.

---

## Patterns that work across all three

### Per-PR ephemeral environments

Spin up a per-PR/MR environment, deploy the changeset to it, post the URL
back to the PR. Reviewers click a link instead of imagining what the change
looks like. Tear it down on close/merge.

- ADO: deployment job + Bicep/Terraform creating a unique RG named
  `pr-<number>`, tear-down task on PR close
- GitLab: built-in via `environment.action: stop` (above)
- Jenkins: shared library function that handles create + on-close hook

Cost: cheaper than the senior eng-hours saved arguing about a PR. By a lot.

### Change-only deploys

Don't rerun every deploy job on every merge. Use **path filters** to scope:

```yaml
# Azure DevOps
trigger:
  branches: { include: [main] }
  paths:    { include: ['services/orders/**'] }
```

```yaml
# GitLab
rules:
  - changes: ['services/orders/**']
```

```groovy
// Jenkins
when { changeset 'services/orders/**' }
```

Saves time, money, and noise.

### Promote artifacts, don't rebuild

The single largest reliability win. Tag the image once, deploy that tag to
each env. Each deploy is "swap config, restart pods" — not "rebuild and
hope".

### Approval gates only on prod

Auto-deploy to dev. Auto-deploy to staging (run smoke tests, fail loudly).
Manual gate on prod. Putting approvals on dev/staging is friction nobody pays
attention to anyway.

### CI runs against the same image you ship

Your CI tests should run inside (or against) the artifact you're going to
ship — not against a freshly-installed `npm ci` in a fresh container. The
gap between them is where "works on CI, fails in prod" lives.

---

## Picking one for a new org

Default choices:

- **You're on Azure already, with Azure AD identities, and want minimum
  setup:** Azure DevOps Pipelines.
- **You're using GitLab as your git host, or you want everything in one
  product:** GitLab CI.
- **You need to build for weird hardware / on-prem / heterogeneous
  toolchains, or you already have it:** Jenkins.

The pipeline shape is the same. The cost of switching later is real but not
catastrophic if you keep the pipeline DSL thin and the actual build/deploy
logic in scripts/Makefiles in the repo (not inline in YAML).

---

## Related

- [Terraform on Azure](./terraform-on-azure.md) — what the deploy jobs probably run
- [Kubernetes (AKS) production checklist](./aks-production-checklist.md) — where they deploy to
- [Observability stack](./observability-stack.md) — what tells you the deploy worked
- [PR-as-publish-gate](../automation-patterns/pr-as-publish-gate.md) — the same "humans gate the side effect" idea for non-code automations
