# Helm vs Kustomize vs raw YAML

**TL;DR:** Use Helm for *packaging* and Kustomize for *layering*. Most production
workloads benefit from both: Helm for the app's chart, Kustomize on top for
environment overlays. Raw YAML is fine for tiny clusters and is the failure mode
you should be willing to fall back to when the templating gets out of hand. The
right answer is rarely just one of the three.

## What each one actually does

| Tool | Model | Strength | Pain point |
|---|---|---|---|
| **Raw YAML** | Files | No magic, no templating. WYSIWYG. | Repetition across envs |
| **Helm** | Go-template + values + lifecycle | Packaging, distribution, releases | Templating-in-strings rabbit hole |
| **Kustomize** | Base + overlays via patches | Layered config without templating | Patches get clever; merge semantics surprise people |
| **Jsonnet** (Tanka, kapitan) | A real language | Most powerful | Smallest community, steep learning curve |
| **CDK8s** | TypeScript/Python → YAML | Real language, types | Yet another layer |

The two-question decision matrix:

1. **Do you need to *package and distribute*** the app for other teams or
   third parties to install? → Helm.
2. **Do you need to *layer* environment-specific differences** on a shared
   base? → Kustomize.

Most teams answer "yes" to both. The healthy pattern is **Helm chart + Kustomize
overlay**, not "choose one".

## Helm — for packaging

Helm packages a set of manifests, parameterized by values, with release
semantics (install / upgrade / rollback). It's the *kubectl-friendly* way to
distribute an app.

When Helm shines:

- **Third-party apps** — every serious project ships a Helm chart. Use it.
- **Your own app, consumed by multiple teams** — your chart, their values.
- **Release lifecycle** — `helm rollback <release> 3` is a real, working
  operation.
- **Chart dependencies** — when your app needs Postgres + Redis + Prometheus,
  charts can declare those as dependencies.

When Helm hurts:

- **Templating logic** — `{{- if .Values.x }} {{- range .Values.y }}` nested
  three deep, branching on string presence. You're writing Go templates in
  YAML strings. The IDE can't help. Whitespace-sensitivity bites.
- **Conditional resources** — including or excluding entire manifests via
  templating is awkward.
- **Per-env divergence beyond simple value overrides** — when "the staging
  Deployment doesn't need this sidecar" requires conditional Helm logic,
  you're past Helm's sweet spot.

### Helm chart shape that works

```
chart/
  Chart.yaml
  values.yaml                # production-grade defaults
  templates/
    _helpers.tpl             # named templates (labels, names)
    deployment.yaml
    service.yaml
    configmap.yaml
    serviceaccount.yaml
    rbac.yaml
    pdb.yaml                 # PodDisruptionBudget — yes, you need one
  templates/tests/
    test-connection.yaml     # `helm test` post-install smoke
```

Rules:

- **One responsibility per template.** Don't bundle `deployment.yaml +
  service.yaml` into one file.
- **`_helpers.tpl` for shared logic** — labels, fullname, image references.
- **Don't put environment data in `values.yaml`.** That belongs in
  `values-prod.yaml` (or a Kustomize overlay if you're layering).
- **Version the chart** — bump on every release. Pin chart versions in
  consumers.

## Kustomize — for layering

Kustomize takes a base set of manifests and applies overlays (patches) for
each environment. No templating. Pure YAML transformations.

```
manifests/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/
    dev/
      kustomization.yaml
      replicas-patch.yaml      # smaller replicas
    staging/
      kustomization.yaml
      replicas-patch.yaml
      sidecar-patch.yaml       # add a sidecar
    prod/
      kustomization.yaml
      hpa.yaml                 # add an HPA
      pdb.yaml                 # add a PDB
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
  - hpa.yaml
  - pdb.yaml
patches:
  - path: ../staging/replicas-patch.yaml
    target:
      kind: Deployment
      name: api
images:
  - name: api
    newTag: "abc123"
```

When Kustomize shines:

- **Environment overlays** — small, structured, predictable patches.
- **No templating** — easier to read, easier to debug. `kustomize build` shows
  exactly what'll be applied.
- **Native to kubectl** (`kubectl apply -k ./overlays/prod`).
- **GitOps-friendly** — Argo CD and Flux both handle Kustomize natively.

When Kustomize hurts:

- **Distributing an app** — Kustomize isn't a packaging format. You can't
  `kustomize install community-app/...` like you can `helm install`.
- **Complex per-environment branching** — beyond ~5 overlays, the patches get
  hard to navigate.
- **Strategic merge patch semantics** — list merging (especially environment
  variables) has gotchas. Reading the docs is required.

## Helm + Kustomize together

The pattern that scales: **Helm renders the app's chart; Kustomize overlays
environment differences on top.**

```bash
# Render Helm template, then layer with Kustomize
helm template api ./chart -f values.yaml --namespace prod > /tmp/rendered.yaml
kubectl apply -k ./overlays/prod
```

Or use Kustomize's `helmCharts` directive (Kustomize 4.1+):

```yaml
# overlays/prod/kustomization.yaml
helmCharts:
  - name: api
    repo: oci://ghcr.io/example/charts
    version: 2.3.1
    releaseName: api-prod
    valuesFile: values-prod.yaml
patches:
  - path: extra-env.yaml
    target: { kind: Deployment, name: api-prod }
```

This is the cleanest "use both" pattern. Helm owns packaging + versioning.
Kustomize owns environment overlays.

## When raw YAML is correct

- **Small clusters** — one team, one cluster, two environments. The cognitive
  cost of templating exceeds the duplication cost.
- **Bootstrap manifests** — the chart that installs Argo CD that then installs
  everything else. Don't recurse.
- **Throwaway demos** — write the YAML, ship it, delete it.
- **Cluster-level objects** that don't vary per env (CRDs, ClusterRoles,
  Namespaces). Templating them adds nothing.

The smell that you should *not* be using raw YAML: copy-pasting the same
Deployment across three files with one value different. That's what Helm or
Kustomize is for.

## The values.yaml sprawl problem

The most common Helm pathology. Values files grow until nobody knows what's
where.

Mitigations:

- **One file per environment**: `values.yaml` (defaults) + `values-prod.yaml`
  (overrides). Not `values-large-prod-eu.yaml + values-large-prod-us.yaml + ...`.
- **`helm show values <chart>`** to see what's settable. If you can't, you
  forked the chart and forgot.
- **Schema validation** — `values.schema.json` enforces types and allowed
  values. Catches typos like `replicaCount: "3"` (string vs int).
- **Don't expose every value as settable.** Hard-code things that have one
  correct answer.

## Chart versioning and lock files

Helm has two version concepts:

- **Chart version** — the version of the chart itself (your packaging).
- **App version** — the version of the app being deployed.

Bump both. Pin both in consumers. Use a release process (`helm package` +
push to a registry; OCI is the modern standard).

Lock files: Helm dependencies (`charts/` subcharts) should be pinned via
`Chart.lock`. Without it, a `helm dependency update` can silently pull a new
major version of Postgres.

## Patches in Kustomize — the four flavors

| Patch type | When |
|---|---|
| **Strategic merge** (`patches:`) | Default; matches by name and kind |
| **JSON 6902** (`patches:` with `target:`) | Surgical edits with explicit paths |
| **`patchesStrategicMerge`** (legacy) | Older syntax; prefer the unified `patches:` |
| **`patchesJson6902`** (legacy) | Older syntax; prefer the unified `patches:` |

The unified `patches:` API (Kustomize 4+) handles both styles. Use it.

The gotcha: strategic merge tries to be smart about list merging. Environment
variables in a container are merged by name, not by position — usually right.
Initcontainers are merged by name. Args are *replaced wholesale*, not merged.
Test what kustomize build outputs before assuming.

## Common pitfalls

### "Helm chart says one thing; cluster has another"

Helm doesn't reconcile drift. If someone `kubectl edit`s a Deployment, Helm's
next upgrade overwrites or doesn't, depending on `--force` and the resource's
3-way merge. Use Argo CD or Flux for reconciliation if you need it.

### "I upgraded the chart and lost my CRDs"

Helm doesn't manage CRD upgrades well. `crds/` directory installs once and
doesn't upgrade. Separate CRD management from app management for complex
operators.

### "Helm hook ordering is opaque"

Hooks (pre-install, post-install, etc.) run in weight order, but the semantics
of "weight" surprise people. Read the docs; don't assume ordering will hold
across Helm versions.

### "Kustomize merged my env vars wrong"

Lists in YAML have ambiguous merge semantics. Strategic merge merges by key
field (usually `name`). If your patch lacks the name, it appends instead of
replaces. `kustomize build` shows the truth.

## Gotchas

- **Helm `--reuse-values`** silently picks up the previous deploy's values.
  Defeats overrides. Use `--values` with explicit files.
- **`kubectl apply -k`** ships in kubectl but is sometimes an older Kustomize
  version. For features, install Kustomize separately and use `kustomize build
  | kubectl apply -f -`.
- **Helm releases beneath the cluster** — `helm list -A` shows them. Releases
  in deleted namespaces become orphans.
- **Subchart values** — overriding a subchart's value requires
  `subchart-name: { key: value }` in your values, not just `key: value`.
- **Don't `helm install` in CI without `--atomic`** — if it half-applies and
  hits an error, you've left the namespace in a weird state. `--atomic` rolls
  back on failure.
- **Kustomize `commonLabels` is sticky** — once applied to a selector, you
  can't easily change it without recreating the resource (selectors are
  immutable on Deployments).

## When NOT to use either

- **Operator-managed apps** — the operator owns the manifests. Don't fight
  it with Helm.
- **Cluster-bootstrapping resources** — chicken-and-egg. Use raw YAML or
  Argo CD's app-of-apps pattern.
- **Things that aren't manifests** — Helm has been (ab)used to template
  Terraform, shell scripts, even SQL. Don't.

## Related

- [`service-mesh-when-and-why.md`](./service-mesh-when-and-why.md) — mesh
  sidecar injection complicates manifest management.
- [`gitops-with-argocd-and-flux.md`](./gitops-with-argocd-and-flux.md) — GitOps
  consumes Helm and/or Kustomize as the source of truth.
- [`pod-autoscaling-deep-dive.md`](./pod-autoscaling-deep-dive.md) — HPA / VPA
  / KEDA manifests need careful per-env overlays.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS-specific manifest patterns.
- [`../automation-patterns/multi-environment-promotion.md`](../automation-patterns/multi-environment-promotion.md)
  — promotion pattern that consumes Helm + Kustomize artifacts.
