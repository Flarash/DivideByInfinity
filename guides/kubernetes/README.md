# Kubernetes

Deep dives on Kubernetes-specific operational patterns. Complements
[`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
with topic-focused content.

## Guides

- [**Helm vs Kustomize vs raw YAML**](./helm-vs-kustomize-vs-raw-yaml.md) —
  packaging vs layering, the two-question decision matrix, Helm chart shape
  that works, Kustomize patch flavors, the values.yaml sprawl problem, and
  when raw YAML is correct.
- [**Service mesh — when and why (mostly, when not)**](./service-mesh-when-and-why.md)
  — what a mesh actually does, when adoption is justified, Istio vs Linkerd vs
  Cilium vs Consul comparison, sidecar vs sidecarless (ambient/eBPF), the
  operational tax, and the pragmatic ramp-up plan.
- [**GitOps with Argo CD and Flux**](./gitops-with-argocd-and-flux.md) — the
  four GitOps requirements, Argo's Application/ApplicationSet model, Flux's
  modular controllers, repo layout, sync waves, drift handling, multi-cluster
  topologies, secret management, bootstrap, and the app-of-apps anti-pattern.
- [**Pod autoscaling deep dive**](./pod-autoscaling-deep-dive.md) — HPA, VPA,
  KEDA, Cluster Autoscaler, and Karpenter — what each scales, how they
  interact, the metrics-server gotcha, custom metrics via prometheus-adapter,
  and the "scaled to zero and nothing wakes up" failure mode.
