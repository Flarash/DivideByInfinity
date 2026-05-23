# Service mesh — when and why (mostly, when not)

**TL;DR:** Most clusters don't need a service mesh. The features it gives you (mTLS,
traffic shaping, retries, observability) are valuable, but they come with an
operational tax that's easy to underestimate. Adopt a mesh when at least *two* of
its features are load-bearing for your roadmap, not because it's table stakes.
Istio is the most powerful and the most painful. Linkerd is the easiest and the
most opinionated. Cilium service mesh is the newest and the most cloud-native.

## What a service mesh actually does

A mesh sits between your services as a layer of sidecars (or eBPF, in newer
designs) and gives you:

1. **mTLS between services** — automatic certificate management.
2. **Traffic shaping** — weighted routing, header-based routing, canary
   support without changing app code.
3. **Resilience primitives** — retries, timeouts, circuit breakers, outlier
   detection — declared in YAML, not coded in each service.
4. **Observability** — automatic per-request metrics, distributed tracing
   headers propagated, traffic graphs.
5. **Policy** — who can call whom (AuthorizationPolicy in Istio,
   ServerAuthorization in Linkerd).

Each one is great. The question is: do you need *more than one*?

## When a mesh is justified

- **Compliance requires mTLS by default** between all pods. You're not
  retrofitting it into every service.
- **You operate dozens of services** owned by multiple teams. The traffic
  graph isn't tractable without one.
- **You're doing serious progressive delivery** (header-based canary, per-cohort
  rollout) and the in-cluster service-to-service traffic needs the same
  control as ingress.
- **You need zero-trust networking** inside the cluster.
- **You want observability that doesn't require app changes** — automatic
  RED metrics (rate, errors, duration) per service.

If two or more of these are true and worth ongoing operational investment,
adopt a mesh.

## When a mesh is wrong

- **You have fewer than ~10 services.** The mesh management overhead exceeds
  the value.
- **You don't have Kubernetes maturity.** The mesh adds Kubernetes complexity
  on top of itself.
- **mTLS is your only goal** — you can probably get there with a network policy
  + cert-manager + a cluster-internal CA, without a mesh.
- **Your traffic patterns are simple.** Ingress + app-level retries + standard
  observability covers most cases.
- **You're already drowning in operational toil.** Adding a mesh adds
  toil — it doesn't remove it.

## The big three

| Mesh | Architecture | Strengths | Weaknesses |
|---|---|---|---|
| **Istio** | Envoy sidecar (or ambient mode) + control plane | Most features, biggest ecosystem | Operational complexity, upgrade pain |
| **Linkerd** | linkerd2-proxy sidecar (Rust) + control plane | Lightweight, opinionated, easier ops | Fewer features; smaller ecosystem |
| **Cilium Service Mesh** | eBPF, no sidecars | No sidecar tax; very modern | New; some features still maturing |
| **Consul Connect** | Envoy sidecar | Strong for hybrid Kubernetes + VMs | Less Kubernetes-native than Istio/Linkerd |
| **AWS App Mesh** | Envoy sidecar | AWS-native integration | Deprecated; AWS is steering users to Istio/Cilium |

The realistic shortlist for new adoption in 2026: **Linkerd** (start here if
you can live with its opinions), **Istio ambient mode** (start here if you
need Istio's feature surface), **Cilium service mesh** (if you're already on
Cilium for CNI).

## Sidecar vs sidecarless (ambient / eBPF)

The newer architectures avoid the per-pod sidecar:

- **Istio ambient mode** — a per-node "ztunnel" plus per-namespace
  "waypoint" proxies replace the per-pod sidecar. Less overhead, easier
  pod-restart semantics, fewer "did the sidecar inject" debugging sessions.
- **Cilium service mesh** — uses eBPF in the kernel for L4 mesh, and Envoy at
  node level (not per pod) for L7. No sidecar at all in the L4 case.

Sidecar mode is mature, well-documented, and well-understood. Sidecarless is
the future but still maturing. For greenfield in late 2026, sidecarless is
the better default *if* your team is ready to operate it.

## mTLS — usually the gateway feature

The most common single reason to install a mesh. Worth knowing what you're
signing up for:

- **Automatic certificate issuance and rotation** — yes, this is genuine value.
- **mTLS between *all* pods, by default** — also yes, but you have to handle
  exceptions (legacy apps, external traffic, sidecar-less pods).
- **Cert rotation breaks pods that hold connections** — long-lived gRPC
  connections may need explicit re-establishment.
- **Debugging is harder** — packet captures show TLS, not plaintext. You need
  the mesh's CLI tools or app-level logging.

If you only need mTLS at the perimeter (ingress) and don't need
service-to-service mTLS, **cert-manager + your ingress controller suffices.**
You don't need a mesh.

## Traffic shaping — when it pays off

Mesh-level traffic shaping makes sense when:

- You want **header-based routing** (`x-user-cohort: beta` goes to v2).
- You want **weighted rollouts** without redeploying apps (5% → 10% → 50%).
- You want **mirror traffic** (send a copy to a new version for evaluation
  without serving its responses).
- You want **per-service circuit breakers and retries** without library
  changes.

These are real features. If your delivery model doesn't use them, the
mesh's traffic shaping is overhead you don't benefit from.

## The operational tax — what nobody warns you about

- **Upgrade cadence.** Istio releases every ~3 months and supports N-2.
  Linkerd is similar. You will upgrade the mesh frequently. Each upgrade is
  a cluster-wide change.
- **Sidecar lifecycle.** When the mesh restarts a sidecar, the app pod
  doesn't necessarily restart. Old proxy + new control plane = weird bugs.
- **Resource overhead.** Per-pod sidecar = +100-200MB RAM and some CPU per
  pod. Multiply by pod count. Sidecarless avoids this.
- **CNI compatibility.** Istio's init container modifies iptables. Some CNIs
  (Calico in specific modes) don't play nicely. Always test in your CNI.
- **Debugging tax.** "Where did this request go?" becomes a multi-tool answer
  (kubectl + istioctl + Grafana + Jaeger).
- **Two control planes** — you now operate Kubernetes *and* the mesh control
  plane. Both have failure modes.

## A pragmatic "ramp" if you decide to adopt

Don't enable a mesh cluster-wide on day one.

1. **Install** the mesh control plane.
2. **Opt in one namespace** (the most boring, well-understood service).
3. **Validate**: mTLS works, metrics flow, the team can debug. Two-week burn-in.
4. **Add a second namespace.** Different team, different shape.
5. **Add traffic policies**: mTLS strict, basic authorization, retries.
6. **Roll out cluster-wide** with sidecar auto-injection at the namespace
   level (label `istio-injection=enabled` etc.).
7. **Add advanced features** (canary, mirror, complex routing) on
   specific services that benefit.

Each step is its own incident surface. Don't compress them.

## Common mistakes

- **"We installed Istio because it's standard."** It's not standard; it's
  one option. Justify it on features, not vibes.
- **All-or-nothing rollout.** Cluster-wide enable + sidecar injection on every
  namespace = a 3am surprise. Ramp.
- **Skipping the mesh's upgrade docs** — Istio especially has version-specific
  upgrade procedures. Skipping the docs causes downtime.
- **Treating the mesh's metrics as observability done.** They're a layer.
  You still need app-level metrics and tracing.
- **Reusing the mesh's CA for unrelated cert work.** It's not a general-purpose
  PKI; it's the mesh's CA.

## When to remove the mesh you adopted

Honestly: this happens, and that's fine. Signs you should:

- The features you adopted it for are no longer load-bearing.
- The operational tax is consistently higher than the value.
- You've grown into Kubernetes maturity that makes app-level patterns
  feasible again.

Removal is non-trivial — you'll need to retrofit mTLS, observability, and
retries at the app layer. Plan it as a project.

## Gotchas

- **Sidecar injection requires pod restart.** A namespace label flip doesn't
  retrofit existing pods.
- **DaemonSets, jobs, and CronJobs may need sidecar exclusion** — jobs that
  never exit because the sidecar keeps running is a classic.
- **Egress traffic** — the mesh wants to manage egress too. This trips up
  apps that hit external HTTPS endpoints. ServiceEntry / egress gateways are
  the answer.
- **Helm chart for the mesh ≠ "production-ready out of the box"** — read
  the production-readiness guide. Defaults are for demos.
- **mTLS strict mode breaks one-off `kubectl port-forward` for debugging.**
  Have an off-ramp.

## When NOT to use a mesh

- **Single-service apps** — overkill.
- **Apps that already implement mesh-like features in libraries.** A
  well-functioning gRPC + Linkerd-lite library stack might be enough.
- **Edge-only workloads** (CDN, gateway-only) — the mesh isn't the right
  layer.
- **Severe resource constraints** — sidecar tax + control plane can be 10-20%
  of cluster resources for small clusters.

## Related

- [`helm-vs-kustomize-vs-raw-yaml.md`](./helm-vs-kustomize-vs-raw-yaml.md) —
  mesh sidecar injection complicates manifest authoring.
- [`gitops-with-argocd-and-flux.md`](./gitops-with-argocd-and-flux.md) — mesh
  resources (Gateway, VirtualService, AuthorizationPolicy) are GitOps-managed
  like any other CRD.
- [`pod-autoscaling-deep-dive.md`](./pod-autoscaling-deep-dive.md) — sidecars
  affect autoscaling metrics (CPU/memory include the proxy).
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS + Azure Service Mesh notes.
- [`../cloud-platform/observability-stack.md`](../cloud-platform/observability-stack.md)
  — mesh telemetry complements your existing stack, doesn't replace it.
