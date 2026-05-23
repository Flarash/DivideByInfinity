# Right-sizing and savings plans

**TL;DR:** Most cloud bills are 30-50% over-spent on over-provisioned compute that
nobody right-sized after launch. Right-sizing is a continuous process, not a
quarterly project. Layer on reserved instances / savings plans / committed use
discounts for *predictable* workloads, and spot/preemptible for *tolerant* ones.
The single biggest mistake: right-sizing prod aggressively, breaking it, and
swearing off the practice forever.

## What right-sizing actually is

Matching the resources you provision to the resources your workload uses,
with a safety margin you can defend. The math has two parts:

- **Sizing the resource** — instance type, VM SKU, container CPU/memory
  requests.
- **Sizing the count** — how many instances / pods / replicas.

Both can be wrong in either direction:

- **Over-provisioned** — paying for idle capacity. Common.
- **Under-provisioned** — performance suffers, autoscaler thrashes, on-call
  pages. Less common but more visible.

Right-sizing is finding the line.

## The right-sizing process

1. **Collect data.** At least 2 weeks, ideally 30 days, of:
   - CPU utilization (max and p95)
   - Memory utilization (max and p95)
   - Network throughput
   - Disk IOPS / throughput

2. **Pick a target.** Common defaults:
   - CPU p95 around 50-70%.
   - Memory p95 around 60-80%.
   - Leave headroom for spikes (the difference between max and p95).

3. **Calculate the new size.** If you're at 20% CPU on a 4-vCPU instance,
   a 2-vCPU instance at 40% saves 50% with the same headroom.

4. **Apply in non-prod first.** Watch for the unexpected. Roll forward
   gradually.

5. **Apply in prod with a rollback plan.** Pin the old size somewhere
   accessible.

6. **Monitor for two weeks.** Page on degradation; revert if needed.

## Data sources

| Tool | Strength | Cost |
|---|---|---|
| **Cloud-native** (AWS Compute Optimizer, Azure Advisor, GCP Recommender) | Free, built-in, vendor-aware | Limited customization |
| **Kubernetes VPA in recommend-only mode** | Per-pod request recommendations | Free, requires VPA installed |
| **Goldilocks** | UI on top of VPA recommendations | Free |
| **Datadog / Dynatrace / Grafana** | Custom dashboards, multi-cloud | Subscription cost |
| **Kubecost** | Kubernetes-specific cost + recommendations | Free tier + paid |

Start with cloud-native + Goldilocks. Add a paid tool when you've outgrown
their views.

## Reserved instances, savings plans, committed use

The discount mechanics across the big three:

| Cloud | Commitment options | Discount range | Flexibility |
|---|---|---|---|
| **AWS** | Reserved Instances (RI), Savings Plans (Compute / EC2 / SageMaker) | 30-75% | Savings Plans most flexible |
| **Azure** | Reserved Instances, Savings Plan for Compute | 30-72% | RIs are SKU-specific; Savings Plan is flexible |
| **GCP** | Committed Use Discounts (resource-based, spend-based) | 25-70% | Spend-based CUDs are flexible |

**The rule:** commit only to the *baseline* workload — the part that runs
24/7 regardless. Use on-demand for the peaks.

If your baseline is 40 vCPUs that always run, commit those. If your peak
adds another 100 vCPUs occasionally, leave the peak as on-demand.

**Commitment terms:**
- **1-year:** safer, ~30-40% discount.
- **3-year:** higher discount (~60%) but longer lock-in.

**Payment options:**
- **No upfront** — same monthly bill, no lump sum.
- **Partial upfront** — better discount.
- **All upfront** — best discount, but ties up cash.

For most teams: **1-year, no-upfront, Savings Plan / flexible commit**.
The discount delta vs 3-year is small; the flexibility is worth it.

## Spot / preemptible instances

The cloud has spare capacity. They'll rent it to you cheap (50-90% off) but
can take it back with short notice (2 minutes typically).

**Good fits:**
- Batch jobs (resumable).
- CI runners.
- Stateless web tier behind a load balancer with healthy on-demand baseline.
- Big-data / ML training (with checkpointing).
- Dev/test environments.

**Bad fits:**
- Stateful single-instance services.
- Anything with long startup time (the interruption replaces too often).
- Workloads where the SLA can't tolerate occasional unavailability.

**Per-cloud:**
- **AWS Spot** with Spot Fleet for diversification.
- **Azure Spot VMs** + Spot Scale Sets.
- **GCP Preemptible** + Spot VMs (the newer, more flexible variant).

**Karpenter** (AWS, expanding to Azure) handles Spot beautifully — provisions
Spot when available, falls back to on-demand on shortage, all transparently.

## The "right-sized prod and broke it" trap

The story that scares teams off right-sizing:

1. Read recommendation: prod could be 50% smaller.
2. Resize the instances.
3. Sunday afternoon: a normal traffic burst hits a smaller fleet.
4. Saturated CPU → cascading retries → outage.
5. "Right-sizing is too risky."

What actually went wrong:

- **No headroom for bursts.** Resized to p95, ignored max.
- **No gradual rollout.** Full resize at once.
- **No monitoring of saturation metrics during/after the change.**
- **No rollback plan.**

The discipline fix:

- Resize one AZ first, monitor for 48h.
- Use p99 (not p95) and account for bursts.
- Keep autoscaler limits wide enough to absorb the spike during transition.
- Have a one-command rollback to the old size.

Right-sizing isn't risky; *unmonitored, all-at-once* right-sizing is risky.

## Kubernetes-specific right-sizing

Two dimensions:

- **Requests** — what the pod *asks for*. Drives scheduling and HPA math.
- **Limits** — what the pod is *capped at*. Drives OOM-kill behavior.

The common shape:

- **Requests = p95 of usage** (or VPA recommendation).
- **Limits = some multiple of requests** (1.5-3x for memory; CPU limits are
  more debatable — some teams skip CPU limits entirely to avoid throttling).

**The "no CPU limits" debate:**
- Pro: avoids CFS throttling that hurts latency.
- Con: a runaway pod can consume the whole node.

Reasonable position: **request the average, no CPU limit, memory limit at
2x request**. Mileage varies by workload.

## Storage right-sizing

The forgotten cost line.

- **Block storage** (EBS, Azure managed disks, GCP persistent disks):
  provisioned IOPS is expensive. Most workloads don't need gp3/Premium SSD;
  gp2/Standard SSD suffices.
- **Object storage**: lifecycle policies are free money. Move 90-day-old
  data to infrequent-access; 365-day-old to archive.
- **Snapshots and backups**: accumulate forever if no retention policy.
  Audit quarterly.
- **Orphaned volumes** (detached EBS, unattached managed disks): cost
  nothing in compute but $ per month per GB. List them; delete or attach.

## Auto-scaling for cost

Autoscaling is right-sizing in real time:

- **HPA at the pod level** — replicas scale to match load.
- **Cluster Autoscaler / Karpenter** — nodes scale to match pod demand.
- **Scale to zero** for batch / queue workers via KEDA.

Aggressive auto-scaling = better cost match to actual load. The trade-off
is cold-start latency on scale-up. For web tiers, keep minReplicas high
enough to absorb sudden bursts.

See [`../kubernetes/pod-autoscaling-deep-dive.md`](../kubernetes/pod-autoscaling-deep-dive.md).

## The cost-cap mindset

Beyond right-sizing, set budgets that *do something*:

- **Cloud-native budgets** (AWS Budgets, Azure Cost Management Budgets,
  GCP Budgets) — alerts at 50% / 80% / 100% of monthly target.
- **Per-team / per-project budgets** — visibility before pain.
- **Hard caps** for non-prod: dev/staging that exceeds budget triggers
  shutdown or scale-down.

The alert that nobody reads is theatre. Wire alerts to a Slack channel
the team actually monitors.

## What to optimize first (the 80/20)

In rough cost-impact order for most teams:

1. **Idle / over-provisioned compute** (the biggest line). Right-size.
2. **Storage with no lifecycle policy.** Add tiering.
3. **Snapshots / backups without retention.** Add retention policies.
4. **Egress traffic** (especially cross-region). Architect to minimize.
5. **Idle databases** (test environments running 24/7). Schedule off-hours.
6. **Reserved capacity not used.** Right-size first; then commit to the
   right-sized baseline.
7. **Premium tier for non-prod** (HA databases, premium storage in dev). Use
   cheaper tiers in non-prod.
8. **Orphaned resources** (unused load balancers, NAT gateways, public IPs).

Don't try to optimize 1+2+...+8 simultaneously. Pick the biggest line, fix
it, re-measure.

## Gotchas

- **Reserved capacity doesn't apply to autoscaling spikes.** Spikes hit
  on-demand pricing. Plan accordingly.
- **Cross-region commitments** — most RIs/SPs are region-scoped. Buying for
  one region doesn't cover the other.
- **Currency exposure** — buying 3-year all-upfront in a foreign currency
  hedges currency risk too. Or doesn't, depending on your finance setup.
- **Spot interruption replaces too frequently** for cold-start-heavy apps.
  Don't make Spot the *only* tier for latency-sensitive workloads.
- **Autoscaler thrashing** can cost more than over-provisioning. Tune
  stabilization windows.
- **CPU credits** (AWS t-family, Azure B-series) — bursty instances run
  out of credits and silently throttle. Use only for genuinely bursty
  workloads with low baseline.
- **Right-sizing the *operator overhead*** — your monitoring stack, your
  CI runners, your bastion. Often forgotten, often substantial.

## When NOT to right-size

- **Pre-product traffic** — you don't know the shape yet.
- **Workloads about to be replaced.** Don't optimize what you're killing.
- **The savings are < 5% of the bill.** Diminishing returns; spend the
  effort elsewhere.
- **Compliance-required redundancy.** Some over-provisioning is mandatory.

## Related

- [`finops-tagging-and-showback.md`](./finops-tagging-and-showback.md) — you
  can't optimize what you can't allocate.
- [`elasticsearch-cost-and-performance.md`](./elasticsearch-cost-and-performance.md)
  — domain-specific right-sizing for search clusters.
- [`../kubernetes/pod-autoscaling-deep-dive.md`](../kubernetes/pod-autoscaling-deep-dive.md)
  — autoscaling is right-sizing in real time.
- [`../iac/policy-as-code-and-drift-detection.md`](../iac/policy-as-code-and-drift-detection.md)
  — policies to enforce "no premium tier in non-prod", "require cost tags",
  etc.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS-specific instance type guidance.
