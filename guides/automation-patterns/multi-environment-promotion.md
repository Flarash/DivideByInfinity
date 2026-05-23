# Multi-environment promotion

**TL;DR:** Build artifacts once, promote the *same artifact* through dev → staging →
prod. Inject environment config at deploy time. Stop rebuilding the same code three
times. Anything else creates drift between environments and "works in staging,
broken in prod" mysteries that take days to debug.

## The build-once-promote-many rule

The single rule that prevents most multi-env pain:

> **An artifact built for one environment is the *exact same artifact* deployed
> to every other environment.**

Concretely:

- The Docker image with SHA `abc123` built from commit `def456` is what runs
  in dev, staging, *and* prod.
- Environment-specific behavior comes from **config injected at deploy time**
  (env vars, secret references, ConfigMaps, parameter store).
- The build pipeline produces one artifact. The deploy pipeline takes that
  artifact and applies it to N environments.

When this rule breaks ("we rebuilt for prod with the prod env vars baked in"),
you no longer have evidence that the thing in staging is the thing in prod.
Every "it worked in staging" debugging session is reminded that the artifacts
aren't actually the same.

## The promotion path

```
PR merge → build artifact ABC123 → deploy to dev (auto)
                                 → deploy to staging (auto or manual gate)
                                 → deploy to prod (manual gate, approvals)
```

Three rules of healthy promotion:

1. **Same artifact, all environments.** No rebuilds.
2. **Gates get stricter at each tier.** Dev: automatic. Staging: smoke tests
   green. Prod: human approval + change window + canary.
3. **Forward-only by default.** Rollbacks are a *forward* deploy of the
   previous artifact, not a "downgrade" of state.

## Config injection patterns

Where environment-specific values live, in order of preference:

| Pattern | Use for | Example |
|---|---|---|
| **Env vars** | Non-secret config | `DATABASE_HOST=prod-db.internal` |
| **Secret references** | Anything sensitive | `DATABASE_PASSWORD=keyvault://prod-kv/db-pass` |
| **ConfigMaps** (k8s) | Larger config blobs | Feature toggles, allow-lists |
| **Mounted files** | Certs, large config | `/etc/app/config.yaml` from a Volume |
| **Runtime config service** | Frequently-changing values | LaunchDarkly, AWS AppConfig |

The artifact reads config at startup. It doesn't *contain* config.

What this looks like:

```yaml
# Helm values for prod
image:
  repository: ghcr.io/example/api
  tag: "abc123"     # the artifact SHA promoted from staging
env:
  DATABASE_HOST: "prod-db.internal"
  LOG_LEVEL: "info"
  FEATURE_FLAGS_BACKEND: "https://flags.prod.internal"
secretRefs:
  DATABASE_PASSWORD: "keyvault://prod-kv/db-pass"
```

```yaml
# Helm values for staging
image:
  repository: ghcr.io/example/api
  tag: "abc123"     # SAME tag as prod (this is the same artifact)
env:
  DATABASE_HOST: "staging-db.internal"
  LOG_LEVEL: "debug"
  FEATURE_FLAGS_BACKEND: "https://flags.staging.internal"
secretRefs:
  DATABASE_PASSWORD: "keyvault://staging-kv/db-pass"
```

Same image, different config. That's it.

## Promotion gates — what changes between tiers

| Tier | Trigger | Tests required | Approvals |
|---|---|---|---|
| **Dev** | PR merge or commit to main | Unit tests pass | None |
| **Staging** | After dev deploy + N minutes burn-in | Integration tests + smoke tests | None (or 1 reviewer for risky changes) |
| **Pre-prod / UAT** (optional) | Manual trigger | Manual QA cycle | Business stakeholder |
| **Prod canary** | Auto after staging green | Canary metrics (error rate, p99 latency) | None if metrics pass |
| **Prod full** | Auto after canary | Full prod metrics | Manual abort capability |

The trick is making each gate *meaningful*. Gates that everyone clicks through
mechanically are theatre. If every prod deploy gets approved in 30 seconds
because the approver doesn't read it, replace the gate with stronger automated
checks.

## GitOps promotion (Argo CD / Flux)

In GitOps, the desired state of each environment is a file in git. Promotion
becomes "update the file."

```
environments/
  dev/
    api/values.yaml       # image: abc123
  staging/
    api/values.yaml       # image: abc123
  prod/
    api/values.yaml       # image: xyz789 (last approved)
```

Promotion to prod is a PR that bumps `prod/api/values.yaml` from `xyz789` to
`abc123`. The PR carries the changelog, gets review, and merging triggers Argo
CD to reconcile.

**Pros:**
- Git is the single source of truth.
- Rollback = revert the PR.
- Audit trail is git history.

**Cons:**
- Promotion is a git operation. Tooling around "bump and PR" matters
  (e.g., Renovate, custom GitHub Actions).
- Drift between cluster and git state needs reconciliation policies.
- Multi-cluster syncs need ApplicationSets (Argo CD) or Kustomization stacks
  (Flux).

This is the dominant pattern for serious Kubernetes shops.

## Pipeline-as-code promotion (Azure DevOps / GitLab / Jenkins)

The build/deploy pipeline knows about environments. Promotion is a stage
transition.

```yaml
# .gitlab-ci.yml
build:
  script: docker build -t $IMAGE:$CI_COMMIT_SHA . && docker push $IMAGE:$CI_COMMIT_SHA

deploy_dev:
  needs: [build]
  script: helm upgrade --install api ./chart -n dev --set image.tag=$CI_COMMIT_SHA
  environment: dev

deploy_staging:
  needs: [deploy_dev]
  script: helm upgrade --install api ./chart -n staging --set image.tag=$CI_COMMIT_SHA
  environment: staging
  when: manual    # or rules: based on smoke tests

deploy_prod:
  needs: [deploy_staging]
  script: helm upgrade --install api ./chart -n prod --set image.tag=$CI_COMMIT_SHA
  environment: prod
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**Pros:**
- All in one CI tool, easier for smaller teams.
- Pipeline visualisation of the promotion graph.

**Cons:**
- Pipeline is the source of truth, not git state. Drift between "what was
  deployed" and "what the manifest says" is possible.
- Rollback story is "re-run the previous pipeline run" — less elegant than
  GitOps.

## The hardest part — handling state

Application code promotes easily. **Database schema and persisted state don't.**

Two principles:

- **Schema migrations are forward-compatible by default.** A migration deploys
  before the code that uses it. The code works against old *and* new schema.
- **Backward-incompatible changes are multi-step.** To drop a column:
  1. Stop reading from the column (deploy app).
  2. Stop writing to the column (deploy app).
  3. Drop the column (deploy migration).

This means renaming a database column is three production deploys spaced over
time. Plan for it.

State that doesn't fit in the migration model (search indexes, ML models,
caches): rebuild in the target environment, don't try to promote it.

## Environment parity — the spectrum

"Dev should match prod" is an aspiration, not a binary. Pick the trade-offs:

| Aspect | Dev | Staging | Prod |
|---|---|---|---|
| Cloud region | Cheapest | Same as prod | Production region |
| HA setup | Single instance | Multi-AZ | Multi-AZ or multi-region |
| Data volume | Sampled or synthetic | Production-shaped subset | Real |
| Secrets | Test credentials | Staging credentials | Prod credentials |
| External dependencies | Mocks or sandboxes | Sandboxes | Real |
| Cost | <5% of prod | ~30% of prod | 100% |

Things that **must** match across environments:

- Cloud provider and region family (same provider, similar regions).
- IaC modules used (same versions across envs).
- Service versions and config shape.
- Topology (same components, even if scaled down).

Things that **can** differ:

- Instance sizes.
- Replica counts.
- HA tier (zonal vs zone-redundant).
- Data volume.

Aspects that are different need to be *known* differences, not surprises.

## Ephemeral environments (per-PR)

A complement to fixed environments. Each PR spins up its own short-lived
environment for testing.

**Mechanism:**
- PR opens → CI creates namespace `pr-1234` in the staging cluster.
- Deploys the PR's artifact to that namespace.
- Wires DNS / preview URL: `pr-1234.preview.example.com`.
- Posts URL as PR comment.
- PR closes → namespace deleted.

Useful for visual reviews, smoke tests, integration with other components in
development. Not a substitute for staging.

Watch the cost. 100 open PRs × full app stack = real money. Use small instance
sizes and TTLs.

## Gotchas

- **Helm `--reuse-values`** silently keeps the previous deploy's values,
  defeating the promotion. Use `--values` with explicit file inputs.
- **Stale Helm release secrets** can cause "deploy succeeded but nothing
  happened". Diff before apply.
- **Image tag `latest`** is the canonical mistake. Never use it in promotion;
  always use SHA or version tags.
- **Promotion bot account permissions.** The CI identity that deploys to prod
  needs prod-cluster access. Compromise = bad day. Use workload identity,
  scoped per environment.
- **Database migrations across promotion** — the migration is part of the
  artifact (Flyway, Liquibase, Alembic). Don't run migrations from a separate
  pipeline.
- **DNS/config changes promoted out of band** — every change to prod should
  go through the pipeline. Manual DNS edits during a deploy break the
  "promotion is the source of truth" model.
- **"Hotfix" pipelines that skip staging** — sometimes necessary, always
  dangerous. Build the hotfix path explicitly with its own safeguards
  (manual approval, mandatory follow-up PR-to-mainline).

## When NOT to do full multi-env promotion

- **Pre-product** — a single env is fine. Don't over-architect.
- **Tiny internal tools** — deploy from main, accept the risk.
- **Functions / serverless with versioned aliases** — the platform already
  has weighted-routing primitives that fit the model differently.

But: if you have paying customers, you need at least dev + prod. Most teams
benefit from adding staging as soon as deploys start being noticeable.

## Related

- [`deployment-strategies.md`](./deployment-strategies.md) — the strategy used
  *at* each environment (rolling, blue-green, canary).
- [`progressive-delivery-with-flags.md`](./progressive-delivery-with-flags.md)
  — per-env flag values are an extension of per-env config.
- [`pr-as-publish-gate.md`](./pr-as-publish-gate.md) — gating side-effecting
  workflows by PR merge, conceptually similar.
- [`../cloud-platform/cicd-pipeline-patterns.md`](../cloud-platform/cicd-pipeline-patterns.md)
  — Azure DevOps / GitLab CI / Jenkins implementations.
- [`../iac/terraform-state-management.md`](../iac/terraform-state-management.md)
  — environment-per-folder pattern mirrors the promotion shape.
