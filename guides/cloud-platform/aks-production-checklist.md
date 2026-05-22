# Kubernetes (AKS) production checklist

> The things you'll wish you'd set up *before* the first paging incident.
> Opinionated, AKS-flavoured, but most of it ports to EKS/GKE.

This is the "production" checklist, not the "demo a cluster on your laptop"
checklist. If you're putting traffic on this thing, work through every section.

---

## TL;DR

- Multi-zone, ≥ 3 nodes per pool, separate **system** and **user** node pools.
- **Azure CNI Overlay** unless you have a hard reason to use Azure CNI flat —
  the subnet IP math will eat you alive otherwise.
- **Ingress = NGINX + cert-manager** for most teams. AGIC only if you need WAF
  integration with App Gateway you already own.
- **Secrets via Key Vault CSI driver**, not raw k8s `Secret` objects.
- **AAD-integrated RBAC**, namespace-scoped role bindings, no `cluster-admin`
  for humans.
- **PodDisruptionBudgets on every Deployment.** Resource requests/limits set.
  `LimitRange` per namespace as a backstop.
- **Treat the cluster as cattle.** Velero for stateful backups, GitOps
  (Flux/ArgoCD) to recreate the cluster from scratch in < 1 hour.
- **Observability before scale**, not after. See
  [the observability stack guide](./observability-stack.md).

---

## Cluster topology

### Node pools

Two pools minimum:

| Pool | Purpose | Taints | VM family |
|---|---|---|---|
| `system` | CoreDNS, metrics-server, ingress controller, CSI drivers | `CriticalAddonsOnly=true:NoSchedule` | `Standard_D4ds_v5` (3+ nodes) |
| `user-general` | App workloads | none | `Standard_D8s_v5` or similar |

Add more user pools for specialized hardware (GPU, memory-optimized) when you
actually have a workload that needs them. Not before.

### Why the system/user split

If a runaway pod fills the cluster and CoreDNS gets evicted, **everything
breaks**, including the ability to fix things. The system pool with taints +
`CriticalAddonsOnly` toleration on the addons ring-fences DNS and ingress.

### Zones

Provision the cluster across `["1", "2", "3"]`:

```hcl
default_node_pool {
  zones = ["1", "2", "3"]
  # ...
}
```

This costs basically nothing extra and removes the entire "what if a zone
fails" question. Confirm your workloads use `topologySpreadConstraints` so
pods actually land in different zones.

---

## Networking

### Azure CNI Overlay vs Azure CNI vs kubenet

| Mode | Pod IPs come from | Use when |
|---|---|---|
| **Azure CNI Overlay** ✅ | Cluster-private CIDR (e.g. `10.244.0.0/16`) | Default choice. Subnet only needs IPs for nodes, not pods. |
| Azure CNI (flat) | The node subnet | You absolutely need pod IPs reachable from outside the cluster (rare). |
| kubenet | NAT'd from the node | Legacy. Don't pick this for new clusters. |

### The subnet-math trap

With **Azure CNI flat**, every pod takes an IP from the node subnet. A
30-node cluster with `maxPods = 110` needs **3,300+ IPs in the subnet**, plus
upgrade surge headroom. A `/24` gives you 251 usable. Teams who pick "the
sensible CNI choice" without doing this math hit IP exhaustion in week three.

Overlay sidesteps the whole problem.

### Egress

By default, AKS node SNATs outbound through a managed load balancer with
**1024 ports per node**. Above moderate concurrency you hit SNAT exhaustion —
the symptom is "intermittent outbound timeouts that lsof says aren't there".

Fixes, in increasing order of effort:

1. **NAT Gateway** attached to the node subnet (64,000 SNAT ports, much
   higher per-node limit). Single biggest reliability win for chatty
   clusters.
2. Per-pod outbound IPs via `azure-cni` + dedicated IP. Hairy.
3. Connection pooling in the app.

If you call external APIs from k8s pods at any meaningful rate, **set up NAT
Gateway from day one.**

### Private cluster?

Private clusters (API server on a private IP) sound great and add a real
operational tax — kubectl needs a VPN/jump box, CI runners need network
access to the API. Default to **public API server + authorized IP ranges**
unless your compliance regime forces private. If it does, plan the runner
network access at the same time you plan the cluster.

---

## Ingress and TLS

### Pick one ingress controller

The two reasonable options on AKS:

| Option | When |
|---|---|
| **NGINX ingress** (community or `ingress-nginx`) | You want the standard k8s thing. Works everywhere, biggest community, fewest surprises. |
| **Application Gateway Ingress Controller (AGIC)** | You already have an App Gateway in front for WAF, and want a single layer. Pricier and operationally heavier. |

Don't run both. Pick one and standardize.

### cert-manager + Let's Encrypt

For NGINX:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Then add `cert-manager.io/cluster-issuer: letsencrypt-prod` to your Ingress.
You get auto-renewing TLS for free.

**Production tip:** start with `letsencrypt-staging` to avoid rate-limiting
yourself out of LE during cluster setup, then flip to prod.

---

## Secrets

### Don't use raw k8s `Secret` objects in prod

They're base64, not encrypted at rest by default (unless you enable KMS). And
they live in etcd, which means anyone with cluster admin can read them.

### Use the Key Vault CSI driver

Mounts Key Vault secrets as files into pods at runtime. The secret never
exists as a k8s `Secret` object.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: orders-secrets
spec:
  provider: azure
  parameters:
    keyvaultName: kv-orders-prod
    tenantId: <tenant>
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
```

```yaml
# Pod spec
volumeMounts:
  - name: secrets-store
    mountPath: /mnt/secrets
    readOnly: true
volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: orders-secrets
```

Use **Workload Identity** (federated) for the pod's access to Key Vault. No
SP secrets stored in the cluster.

---

## RBAC

### AAD-integrated, not local accounts

```hcl
azure_active_directory_role_based_access_control {
  managed                = true
  azure_rbac_enabled     = true
  admin_group_object_ids = [data.azuread_group.k8s_admins.object_id]
}
```

`azure_rbac_enabled = true` lets you grant Kubernetes RBAC permissions via
Azure roles. You manage access in AAD; revoking access in AAD instantly
revokes cluster access.

### Avoid `cluster-admin` for humans

Define namespace-scoped roles:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: orders
  name: orders-team-edit
subjects:
  - kind: Group
    name: <object-id-of-orders-team-group>
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

Cluster-admin should belong to the small platform-team group, gated behind
PIM/just-in-time elevation.

---

## Workload hygiene

### Every workload sets requests and limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1
    memory: 512Mi
```

`requests` is what the scheduler reserves. `limits` is what the kernel
enforces. If you skip `requests`, the scheduler treats the pod as best-effort
and it's the first thing evicted.

### `LimitRange` per namespace as a backstop

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: orders
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
```

So that pods deployed without explicit values still get *something*.

### PodDisruptionBudget on every Deployment

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders-api
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: orders-api
```

Without a PDB, node upgrades and cluster autoscaler scale-downs can take all
your pods down at once. With it, the eviction respects "always keep 2
available".

---

## HA and DR

### HA inside a cluster

- Multi-zone node pools (above)
- `topologySpreadConstraints` on every Deployment so pods spread across zones
- PDBs (above)
- Cluster Autoscaler **and** node-pool autoscaler, with sane min/max
- Ingress controller replicas ≥ 2

### DR across clusters

The "treat clusters as cattle" rule: you should be able to delete the cluster
and recreate it from git + backups in ≤ 1 hour. To make that real:

1. **Cluster config is in git** (the Terraform stack from the
   [Terraform guide](./terraform-on-azure.md))
2. **Workloads are deployed via GitOps** (Flux or ArgoCD pointed at a git repo).
   No `kubectl apply` from laptops.
3. **Stateful data has off-cluster backups**:
   - Velero with an Azure Storage backend for k8s resources + PV snapshots
   - Database-native backups (Postgres point-in-time, etc.) — *not* relying
     on Velero PV snapshots for the database
4. **Run a DR drill quarterly.** Spin up a second cluster from git, restore
   one stateful service, time it. Fix what was slow.

### Two-cluster patterns

If you need cross-region failover:

- **Active-passive**: prod cluster in region A, warm spare in region B with
  the same GitOps repo. Front it with Azure Front Door or DNS-level failover.
- **Active-active**: both clusters take traffic. Requires a data layer that
  handles multi-region (Cosmos DB multi-write, Postgres logical replication,
  etc.). More expensive and more reliable.

Pick based on RTO/RPO requirements, not vibes.

---

## Cluster autoscaler + PDB interaction

Common bug: cluster autoscaler tries to scale down a node, but a Deployment
without a PDB has 1 replica on that node, so the eviction respects "at least
0 available" (default) — pod gets evicted, app has downtime, autoscaler
succeeds.

Always:

- PDB with `minAvailable >= 1` (or `maxUnavailable`)
- Deployments with `replicas >= 2`
- `topologySpreadConstraints` so the two replicas aren't on the same node

This costs you a tiny amount of compute and removes a whole class of paging
incidents.

---

## The "first hour with a new cluster" checklist

When you provision a new AKS cluster, do these before deploying any app:

- [ ] Cluster is in `["1","2","3"]` zones
- [ ] System and user node pools split
- [ ] AAD RBAC enabled, `cluster-admin` belongs only to the platform group
- [ ] Diagnostic settings → Log Analytics workspace (logs + metrics)
- [ ] Container Insights enabled
- [ ] NGINX ingress + cert-manager installed via Helm + GitOps
- [ ] Key Vault CSI driver installed
- [ ] Velero installed, pointing at a backup storage account in a different RG
- [ ] `LimitRange` in every shared namespace
- [ ] NetworkPolicy default-deny in every shared namespace
- [ ] NAT Gateway attached if any workload calls external APIs at scale
- [ ] Alerts wired for: node not-ready, ingress 5xx > threshold, certificate
      expiry, PV usage > 80%

---

## Related

- [Terraform on Azure](./terraform-on-azure.md) — provisions the cluster
- [CI/CD pipeline patterns](./cicd-pipeline-patterns.md) — deploys to it
- [Observability stack: Prometheus + Grafana + ELK](./observability-stack.md) — watches it
- [Hybrid agent-workspace pattern](../agent-workspaces/hybrid-agent-workspace-pattern.md) — for repos where agents help operate clusters
