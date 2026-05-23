# Pod autoscaling deep dive (HPA, VPA, KEDA, Cluster Autoscaler, Karpenter)

**TL;DR:** Kubernetes has five scaling controllers and most teams misunderstand
how they interact. HPA scales pod replicas on metrics. VPA scales pod resources.
KEDA scales pods on external events. Cluster Autoscaler adds and removes
*nodes* (and Karpenter is its more modern AWS-native cousin). Use HPA + KEDA on
pods, Cluster Autoscaler or Karpenter on nodes, and *almost never* VPA on the
same workload as HPA.

## The five scalers, what they actually scale

| Scaler | Scales | Trigger |
|---|---|---|
| **HPA** (HorizontalPodAutoscaler) | Pod replica count | CPU / memory / custom metrics |
| **VPA** (VerticalPodAutoscaler) | Pod resource requests | Historical usage |
| **KEDA** (Kubernetes Event-Driven Autoscaler) | Pod replica count (via HPA under the hood) | External events (queue depth, Kafka lag, cron, HTTP) |
| **Cluster Autoscaler** | Node count in node pools | Unschedulable pods (pending pods triggering scale-up) |
| **Karpenter** | Nodes (across instance types) | Unschedulable pods + cost/spec optimization |

**HPA is for replica counts on CPU/memory.**
**KEDA is HPA for event-driven workloads (queues, message buses, custom).**
**VPA is for right-sizing requests, generally offline.**
**Cluster Autoscaler / Karpenter run at the node layer.**

Most production setups use **HPA + Cluster Autoscaler (or Karpenter on AWS)**,
add **KEDA** for queue-based or cron workloads, and use **VPA in
recommendation-only mode** as a sizing input.

## HPA — what it actually does

The HPA controller polls metrics, compares to a target, and adjusts replica
count:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 4
          periodSeconds: 30
        - select: Max
```

**Gotchas:**

- **CPU target should reflect typical headroom.** 70% is the common default.
  Higher (80-90%) saves cost but leaves little spike room. Lower (50-60%)
  reacts faster.
- **`stabilizationWindowSeconds`** matters a lot. 0 on scale-up = react
  instantly. Default 300 on scale-down = wait 5 min before scaling in
  (good — avoids thrashing).
- **`behavior.scaleUp.policies`** lets you cap aggressive scaling.
  `100% per 30s` doubles the count; `4 pods per 30s` is a flat cap. Pick the
  `Max` (scale by whichever is bigger).
- **HPA needs `resources.requests` set** on the target deployment. Without
  CPU/memory requests, HPA has no baseline for utilization math.

## The metrics-server gotcha

HPA reads metrics from the metrics API. By default this is `metrics-server`.

- **metrics-server must be installed** — most managed clusters ship it; some
  don't. Without it, `kubectl top pods` fails and HPA shows "unknown" metrics.
- **It scrapes the kubelet** — if kubelet stops reporting, HPA stops working.
- **It's *current* metrics, not historical** — HPA reacts to "right now",
  not trends.

For **custom metrics** (e.g., Prometheus queue depth), you need
**prometheus-adapter** (or KEDA, which abstracts this). The Adapter exposes
Prometheus queries as Kubernetes metric APIs that HPA can target.

## KEDA — when HPA isn't enough

HPA's CPU/memory model doesn't fit queue consumers, scheduled batch jobs,
or "scale based on HTTP request rate to a different service". KEDA fixes
this by providing scalers for ~70 external sources (Azure Service Bus, Kafka,
RabbitMQ, AWS SQS, GCP Pub/Sub, Cron, HTTP, Postgres, etc.).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0      # KEDA can scale to zero (HPA can't)
  maxReplicaCount: 30
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: jobs
        messageCount: "10"     # scale up when queue > 10 per pod
      authenticationRef:
        name: sb-auth
```

KEDA creates an HPA under the hood; you don't manage one separately.

**KEDA's killer features:**

- **Scale to zero.** When the queue is empty, scale to 0 pods. HPA can't
  do this. Saves money on bursty workloads.
- **Cron scaler.** Scale up before scheduled load (every Monday 9am).
- **External-source scalers** for ~every queue/event source you'd want.

**Gotcha:** scale-from-zero has cold-start cost (image pull, app startup).
For latency-sensitive workloads, set `minReplicaCount: 1`.

## VPA — usually not what people want

VPA observes pod CPU/memory and adjusts the *requests* on the deployment to
match. Three modes:

- **Off** — recommendation-only; doesn't modify pods.
- **Initial** — sets requests when a pod is created.
- **Auto** — evicts pods to apply new requests (causes restarts).

**The HPA conflict:** if HPA scales on CPU and VPA changes CPU requests, they
fight. HPA reads "CPU at 60% of requests"; VPA changes the requests; the
ratio changes; HPA reacts; VPA reacts. Don't do this.

**Pragmatic use:**

- **VPA in `Off` mode** as a recommendation engine. Look at the
  recommendations weekly; update requests in your manifests manually (or via
  a tool like Goldilocks).
- **VPA in `Auto`** only for stable workloads where HPA isn't relevant (a
  daemon, a singleton service).
- **Never VPA + HPA on the same workload's CPU dimension.**

## Cluster Autoscaler

When pods can't be scheduled (insufficient cluster capacity), Cluster
Autoscaler adds nodes. When nodes are underutilized, it removes them.

```yaml
# typical Cluster Autoscaler config (simplified)
scale-down-enabled: true
scale-down-utilization-threshold: 0.5
scale-down-unneeded-time: 10m
scale-down-delay-after-add: 10m
expander: least-waste     # or random, most-pods, price, priority
```

**Per-cloud:**

- **AKS:** built-in; enable on each node pool.
- **EKS:** install Cluster Autoscaler manually (or use Karpenter).
- **GKE:** built-in; per-node-pool autoscaling.

**Gotchas:**

- **Pod disruption budgets (PDBs)** can block scale-down. A pod with no PDB
  is freely evictable; a pod with a `minAvailable: 100%` PDB is unevictable.
  Tune carefully.
- **DaemonSets count.** A node with only DaemonSet pods is "empty" from a
  workload perspective and will be removed.
- **Local storage pods** — `local-volume` PVCs anchor a pod to a node; CA
  won't evict.
- **Long scale-up delay.** Provisioning a new VM takes 2-5 min. Plan capacity
  buffers for fast-traffic services.

## Karpenter — the next-gen alternative

Karpenter (AWS-native, expanding to Azure) replaces Cluster Autoscaler with
a smarter, more flexible model:

- **Provisions nodes based on actual pending pods**, choosing instance type +
  size based on what those pods need.
- **Bin-packs aggressively** to reduce cost.
- **Uses Spot instances by policy.**
- **Consolidates** — when nodes are underutilized, schedules pods elsewhere
  and removes the node.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [c, m, r]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["3"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h    # 30 days; recycles to pick up newer AMIs
```

**Verdict:** if you're on AWS, Karpenter is the better choice for new
clusters. Cluster Autoscaler still works fine but Karpenter is more
operationally friendly.

## The "scaled to zero and now nothing wakes up" failure mode

A KEDA-managed deployment scaled to 0 stops getting traffic because there's
nothing to serve it. The trigger that should scale it up requires the
*deployment* to be receiving traffic.

The escape hatches:

- **Don't scale HTTP services to zero** without a wake-up mechanism. Use
  KEDA's HTTP scaler (which has a wake-up proxy) or keep `minReplicaCount: 1`.
- **Queue-driven workloads are safe** — KEDA's queue scaler polls the queue
  directly, not via the pods.
- **Knative** is the more sophisticated answer for HTTP scale-to-zero.

## Custom metrics for HPA

When CPU isn't the right signal — autoscale on requests-per-second, queue
depth, p99 latency, or business metrics.

**Setup:**

1. Have Prometheus scraping the metric.
2. Install prometheus-adapter, configure rules to expose the metric as a
   Kubernetes API.
3. Reference it in HPA:

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

**Gotcha:** the metric must be a *rate*, not a counter. Prometheus-adapter
configuration handles this via the `metricsQuery` field. Get it wrong and HPA
sees "10000 cumulative requests" and scales infinitely.

KEDA bypasses all of this — its Prometheus scaler takes a PromQL query
directly.

## PodDisruptionBudgets and autoscaling

A PDB caps how many pods of a deployment can be disrupted simultaneously.
Critical for node-level operations (Cluster Autoscaler, node upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

**Rules:**

- **Every production deployment needs a PDB.** Without it, voluntary
  disruptions (drains, autoscaler scale-down) can take all pods down at once.
- **`minAvailable` works for stable replica counts.** Use `maxUnavailable`
  if the deployment scales up and down.
- **`minAvailable: 100%`** is "never evict any pod" — blocks voluntary
  disruptions entirely. Useful for singletons; dangerous for the cluster.

## Gotchas

- **CPU vs Memory autoscaling.** Memory-based HPA is risky — JVM/Node heap
  doesn't shrink, so memory usage looks flat at high water mark. CPU is the
  better signal for most apps.
- **Sidecar CPU/memory counts.** Service mesh sidecars add to the pod's
  resource usage. HPA sees the combined number. Tune accordingly.
- **Headless services confuse HPA** — make sure the metric provider
  recognizes the pods.
- **GitOps + autoscaling drift.** HPA modifies the Deployment's `replicas`
  field. GitOps controllers may revert this. Use `replicas:` as ignored in
  Argo CD (`ignoreDifferences`) or Flux (annotations).
- **Scale-down stabilization too short** = flapping. Default 5 min is sane;
  going lower causes oscillation under bursty load.
- **Cluster Autoscaler can't scale a pool to zero** by default — you have to
  enable it explicitly. Useful for batch workloads.

## When NOT to autoscale

- **Singletons** — only one pod ever. HPA doesn't apply.
- **Predictable, flat load** — manual sizing + headroom is simpler than HPA.
- **Cost-capped clusters where reaching the cap is worse than slow response**
  — better to reject requests than to scale into infinity.
- **Stateful workloads with leader election** — scaling these is more
  complicated than HPA models.

## Related

- [`helm-vs-kustomize-vs-raw-yaml.md`](./helm-vs-kustomize-vs-raw-yaml.md) —
  HPA/PDB manifests need per-env overlays (replicas differ by environment).
- [`service-mesh-when-and-why.md`](./service-mesh-when-and-why.md) — sidecars
  affect autoscaling metrics.
- [`gitops-with-argocd-and-flux.md`](./gitops-with-argocd-and-flux.md) —
  ignoring `replicas` in GitOps reconciliation.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS-specific autoscaling, including KEDA add-on.
- The forthcoming `cost-optimization/` topic folder — autoscaling is one
  lever in the cost story; right-sizing requests is the other half.
