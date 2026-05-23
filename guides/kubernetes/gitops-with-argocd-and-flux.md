# GitOps with Argo CD and Flux

**TL;DR:** GitOps is "the cluster's desired state lives in git, and a controller
makes the cluster match git". Argo CD has the better UX and ApplicationSet model;
Flux is more modular and CLI-driven. Both are production-grade. Pick by team
ergonomics. Both share the same gotchas around bootstrap, secrets, and drift
handling — and both require discipline to avoid the "app-of-apps spaghetti"
that the second year of GitOps loves to deliver.

## What GitOps actually requires

Not a buzzword, a discipline:

1. **The desired state of the cluster is in git** — manifests (raw, Helm,
   Kustomize, jsonnet — anything that renders to YAML).
2. **A controller in the cluster reconciles** — pulls from git, applies, and
   continually checks for drift.
3. **No one applies manifests by hand.** kubectl apply is a smell. The
   controller is the only writer.
4. **Changes happen via git.** Every cluster change is a commit with an
   audit trail.

Without all four, you have "Kubernetes deployments stored in git", not GitOps.

## Argo CD vs Flux — the comparison

| Dimension | Argo CD | Flux |
|---|---|---|
| **UX** | First-class web UI, app-tree visualization | CLI-first, no native UI (third-party) |
| **Model** | Applications (top-level CRD) | Modular controllers (Source, Kustomize, Helm) |
| **Multi-cluster** | ApplicationSets, generators | Bootstrap to remote clusters, easier IMO |
| **Helm support** | Native | Native, via HelmRelease |
| **Sync behavior** | Pull-based, polled or webhook | Pull-based, polled or webhook |
| **Drift handling** | Configurable per-app | Configurable per-Kustomization |
| **Ecosystem** | Argo Workflows, Rollouts, Image Updater | Flagger, Flux subsystems |
| **Maturity** | CNCF Graduated | CNCF Graduated |

Both are production-grade. The choice is largely team preference:

- **Argo CD** wins on UX and is easier for teams that want a UI for ops staff.
  The Application/ApplicationSet model is intuitive once you grasp it.
- **Flux** wins on CLI ergonomics and composability. It feels more
  "Kubernetes-native" — small controllers with single responsibilities.

Don't run both. Pick one and commit.

## Repository layout

The repo structure that scales:

```
gitops/
  apps/                    # one folder per app
    api/
      base/               # raw manifests or Helm values
      overlays/
        dev/
        staging/
        prod/
    payments/
      base/
      overlays/
        ...
  clusters/                # cluster-level definitions
    dev-cluster/
      apps.yaml           # which apps are deployed; pointers to apps/*
      infrastructure/
        cert-manager.yaml
        ingress-nginx.yaml
    prod-cluster/
      apps.yaml
      infrastructure/
        ...
  bootstrap/               # initial manifests that bring up Argo/Flux itself
```

**Rules:**

- **One repo or many?** One repo for most orgs; split when team boundaries
  demand it (compliance, ownership). The "app of apps" pattern works either way.
- **`apps/` is the source.** Each cluster references which apps it deploys.
- **`bootstrap/` is the egg.** Manually-applied manifests that install the
  GitOps controller. After that, everything is GitOps.

## Argo CD — the Application model

The atom of Argo CD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/gitops
    targetRevision: main
    path: apps/api/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: api
  syncPolicy:
    automated:
      prune: true            # remove resources no longer in git
      selfHeal: true         # revert manual changes back to git state
    syncOptions:
      - CreateNamespace=true
```

**ApplicationSets** generate many Applications from a template + generator
(list, git directory, cluster generator). The pattern for "deploy this app to
every cluster" or "every dev branch gets a preview env":

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: per-cluster-monitoring
spec:
  generators:
    - clusters: {}            # one Application per registered cluster
  template:
    metadata:
      name: '{{name}}-monitoring'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/gitops
        path: apps/monitoring
        targetRevision: main
      destination:
        server: '{{server}}'
        namespace: monitoring
```

This is where Argo CD really earns its keep — fan-out across clusters with
one declaration.

## Flux — the modular model

Flux splits concerns across CRDs:

```yaml
# 1) Where to pull from
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops
  namespace: flux-system
spec:
  url: https://github.com/example/gitops
  ref:
    branch: main
  interval: 1m
---
# 2) What to apply (and from where in the repo)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: api-prod
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/api/overlays/prod
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops
  targetNamespace: api
```

For Helm, add a `HelmRelease`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nginx
spec:
  interval: 10m
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
  values:
    controller:
      replicaCount: 3
```

Flux's composability shows here: GitRepository + Kustomization + HelmRelease
+ HelmRepository are separate small things that combine.

## Sync waves and dependencies

Both tools support "apply this before that".

**Argo CD** — sync waves via annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # apply first
```

Lower numbers apply earlier. CRDs at wave -10, the operator at -5, then
app resources at 0.

**Flux** — `dependsOn`:

```yaml
spec:
  dependsOn:
    - name: cert-manager
      namespace: flux-system
```

Both have their use; both can be over-used. Keep dependency chains short.

## Drift handling

What happens when someone `kubectl edit`s a resource:

- **`selfHeal: true` (Argo) / `prune: true` (Flux)** — the controller reverts
  the change to match git. Aggressive but correct.
- **`selfHeal: false`** — the controller flags drift but doesn't act.
  Manual intervention required.

The trade-off:

- `selfHeal: true` is the GitOps purist's answer. Discipline-forcing.
  Painful when an on-call hotfix is reverted at 3am.
- `selfHeal: false` requires you to actually pay attention to drift alerts.
  Most teams don't, then the cluster diverges.

A pragmatic middle: `selfHeal: true` for production, `selfHeal: false` for
dev (where engineers want to fiddle).

## Multi-cluster topologies

Two patterns:

**Hub-and-spoke (Argo CD):** one Argo CD instance registers many clusters and
deploys to them. Centralized control, single UI.

**Per-cluster (Flux's traditional model):** each cluster runs its own Flux,
pulling from its own subset of the repo. More resilient (no central
dependency); each cluster is independent.

Both work. Hub-and-spoke is easier to operate small-medium scale. Per-cluster
scales better and survives the loss of any one cluster.

## Secret management

Secrets in git are a problem. Solutions:

| Approach | How it works | Trade-off |
|---|---|---|
| **Sealed Secrets** (Bitnami) | Encrypt secrets to a cluster-specific key. Sealed secret is safe to commit. | Each cluster has its own key. Hard to share across clusters. |
| **External Secrets Operator** | Pull secrets from Vault / Key Vault / Secrets Manager at runtime | Best practice. Requires the secret store to be set up. |
| **SOPS + age/PGP** | Encrypt secrets in git with age or PGP keys | Tooling overhead; key management is on you. |
| **Argo CD Vault Plugin** | Templates pull from Vault during render | Argo-specific. |

**Default recommendation:** External Secrets Operator + Azure Key Vault / AWS
Secrets Manager / Vault. The secret lives in the secret store; the cluster
reads it via ESO; the manifest in git only references it.

## The bootstrap chicken-and-egg

GitOps controllers themselves have to be installed. They can't manage their
own install via GitOps (until they exist).

**Argo CD bootstrap:**
1. `kubectl apply` Argo CD's install manifest.
2. Create an Argo CD `Application` that points at the gitops repo's
   `bootstrap/` folder.
3. That bootstrap Application installs everything else (cert-manager,
   ingress, etc.).
4. The bootstrap Application can also manage Argo CD's own upgrades after
   that.

**Flux bootstrap:**
- `flux bootstrap github` automates this: creates the GitOps repo, installs
  Flux, configures it to sync from the repo.

Either way, the first apply is manual. After that, GitOps takes over —
including upgrades of the GitOps controller itself.

## The app-of-apps anti-pattern

Argo CD's "app-of-apps" pattern (an Application that creates more
Applications) is powerful and can become a maze.

Symptoms:

- 3+ levels of Application-creates-Application.
- Changes in the leaf Application require updates to the root, the
  generator, the template, and the leaf.
- Nobody knows which Application is the source of truth for a given resource.

Mitigations:

- **Prefer ApplicationSets over app-of-apps** for fan-out. ApplicationSets are
  generators; app-of-apps is layered. The semantics are clearer.
- **Limit depth to 2** — root → leaf, no deeper.
- **Naming convention** — leaf Applications get cluster-name prefixes so it's
  obvious what they manage.

## Gotchas

- **Argo CD's auto-prune deletes resources removed from git.** Including
  ones you didn't mean to remove. Test in dev first.
- **Helm hooks + Argo CD** — Helm hooks (pre-install, post-upgrade) don't map
  perfectly to Argo's sync waves. Read the Argo Helm docs before adopting
  charts that use hooks heavily.
- **Flux's `prune` is opt-in per Kustomization.** Forgetting it leaves orphans.
- **GitOps polling intervals add latency** — a 5-min poll means changes take
  up to 5 min. Webhooks bring it to seconds; configure them.
- **Don't use the same git branch for multiple environments.** One branch =
  one set of desired-state. Use folders or branches, not commits.
- **Kustomize patches in nested overlays** — Flux and Argo both build
  Kustomize the same way; double-check the merged output if something looks
  wrong.
- **Resource ownership conflicts** — if two Applications/Kustomizations
  manage the same resource, they fight. Ownership must be unique.

## When NOT to GitOps

- **You have one Kubernetes cluster, one dev, one team, one environment.**
  Helm + manual `kubectl apply` is fine.
- **You're not at "we want to ban kubectl apply" yet.** GitOps only works if
  the discipline holds. If kubectl edit is still routine, fix that first.
- **Your manifests are generated by a tool you don't control** (some
  managed-Kubernetes setups). GitOps and the tool's reconciler fight.

## Related

- [`helm-vs-kustomize-vs-raw-yaml.md`](./helm-vs-kustomize-vs-raw-yaml.md) —
  what GitOps actually deploys.
- [`service-mesh-when-and-why.md`](./service-mesh-when-and-why.md) — mesh
  resources are also GitOps-managed.
- [`pod-autoscaling-deep-dive.md`](./pod-autoscaling-deep-dive.md) —
  autoscaling resources in git, but actual scaling state in the cluster.
  Handle drift carefully.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS-specific bootstrap and addon considerations.
- [`../automation-patterns/multi-environment-promotion.md`](../automation-patterns/multi-environment-promotion.md)
  — promotion via git PR is the GitOps promotion pattern.
