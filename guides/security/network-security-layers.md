# Network security layers

**TL;DR:** Network security is defense in depth: VPC/VNet design, subnet
segmentation, security groups / NSGs, private endpoints, WAF, egress filtering.
None alone is enough. Most teams over-invest in perimeter security and
under-invest in east-west (service-to-service) controls. Get the layers right
and a single misconfiguration doesn't become a breach.

## The layers, briefly

| Layer | What it controls | Common tool |
|---|---|---|
| **Network topology** | What can route to what | VPC / VNet design, peering, transit gateway |
| **Subnet segmentation** | Workload isolation | Public / private / data subnets |
| **Stateful firewalls** | Per-instance / per-resource ACLs | Security Groups (AWS), NSGs (Azure), firewall rules (GCP) |
| **Stateless ACLs** | Subnet-level allow/deny | NACLs (AWS), route tables, GCP firewall (also stateful) |
| **Private endpoints / PrivateLink** | Don't traverse the public internet | Private Link / Endpoints across clouds |
| **WAF** | Application-layer attack filtering | Azure Front Door, AWS WAF, GCP Cloud Armor |
| **DDoS protection** | Volumetric absorption | Native protection at the global tier |
| **Egress filtering** | Outbound control | NAT gateways + URL filtering / Azure Firewall / cloud NGFW |
| **East-west controls** | Service-to-service | Service mesh, network policies, micro-segmentation |

Each layer compensates for failures in another.

## Topology — get this right first

The single decision that affects everything downstream: VPC/VNet design.

**Common patterns:**

- **Single-VPC-per-environment** (default for small orgs). One VPC for dev,
  one for staging, one for prod. Simple, isolated.
- **Hub-and-spoke** (common for medium-large orgs). Shared services (DNS,
  monitoring, jumpbox) in a hub VPC. Per-app/per-env spokes peer to it.
- **Multi-account/multi-subscription** (large orgs). One account per env or
  per app. AWS Organizations, Azure Landing Zones, GCP folder hierarchy.
- **Transit gateway / Virtual WAN** for large networks with on-prem
  connectivity.

**Rules:**

- **Non-overlapping CIDRs across all VPCs/VNets you might ever peer.** Once
  you peer, you can't re-IP without pain.
- **Use private IP ranges generously** — IPv4 is precious in cloud; allocate
  /16 per VPC where you can.
- **Plan for multi-region from the start** if there's any chance. CIDR
  planning is a one-time decision with long consequences.

## Subnet segmentation

The conventional split per VPC:

- **Public subnets** — only for load balancers and NAT gateways. No
  workloads here.
- **Private subnets** — application workloads. Internet egress through NAT.
- **Data subnets** — databases, caches. Tighter ingress rules; no
  public-internet egress.
- **Management subnets** — jumpboxes, agent runners. Locked-down.

**Multi-AZ:** every tier spans at least 2 (preferably 3) availability zones.
A subnet in one AZ + a database in another doesn't HA.

## Stateful firewalls — Security Groups / NSGs

The principal control. Per-resource (or per-subnet) allow/deny on traffic.

**The discipline:**

- **Default deny.** Allow specific ports from specific sources.
- **Reference other Security Groups, not CIDRs.** "Allow 5432 from
  `sg-app-tier`" is better than "allow 5432 from 10.0.0.0/16". When IPs
  change, references don't break.
- **Document the why.** A rule's purpose lives in its description.
- **Audit unused rules.** Quarterly, remove rules that haven't been hit.

**Common mistakes:**

- "Allow 22 from 0.0.0.0/0" — SSH from anywhere. Don't. Use bastion / SSM
  Session Manager / Azure Bastion / IAP.
- "Allow all traffic within the VPC" — the east-west barn door. Specifically
  allow what you intend; deny the rest.
- "Allow all egress to 0.0.0.0/0" — fine for dev, dangerous for prod. Egress
  filtering is its own section.

## Private endpoints — the killer feature

Cloud-native services (Storage, databases, message queues) traditionally
have public endpoints. Traffic from your VPC to them goes over the *internet*
by default — even if your VPC is in the same region.

**Private endpoints / PrivateLink** put a private IP for the service inside
your VPC. Traffic stays on the cloud's backbone.

**Why it matters:**

- Removes the "stolen credentials + public endpoint = exfil" path. With
  private endpoints + service firewall rules, only your VPC can reach the
  storage account.
- Simplifies network rules — the service is "inside" your VPC.
- Often required for compliance.

**Cost:** small per-endpoint hourly fee + processing fee. Multiply by N
services × N regions; budget for it.

**Cloud equivalents:**
- **AWS:** VPC Endpoints (gateway for S3/DynamoDB, interface/PrivateLink for
  most others).
- **Azure:** Private Endpoints + Private DNS Zone.
- **GCP:** Private Service Connect.

Set these up early. Retrofitting them across hundreds of services is painful.

## Egress filtering — the forgotten layer

Most teams focus on ingress and forget that compromised workloads exfil
data outbound.

**The shape:**

- All private-subnet egress routes through a NAT gateway or firewall.
- The firewall enforces an allow-list of destination FQDNs / IPs.
- Anything not on the list is blocked.

**Per-cloud:**

- **AWS Network Firewall / VPC traffic mirroring.**
- **Azure Firewall (with FQDN filtering).**
- **GCP Cloud NGFW Enterprise.**
- **Third-party NGFW** (Palo Alto, Check Point, Fortinet) in any of the above.

**The hard part:** building the allow-list. Apps make outbound calls to
many things — package registries, monitoring endpoints, third-party APIs.
You will discover all of them when you turn on the filter and watch the
denied logs.

**Pragmatic rollout:**

1. Turn on egress logging (no filtering).
2. Collect a few weeks of data.
3. Build the allow-list.
4. Turn on filtering in alert-only mode.
5. After two weeks of tuning, enforce.

## WAF — application layer

A WAF inspects HTTP traffic and blocks attacks the network firewall can't see:

- SQL injection.
- XSS.
- Path traversal.
- Bot traffic.
- Geographic blocking.

**Reality check:**

- **Pre-built rule sets** (OWASP Core Rule Set, AWS Managed Rules, etc.) are
  the starting point.
- **False positives** are inevitable. Every WAF has a "log mode" — start
  there, tune for weeks, then enforce.
- **Custom rules** for app-specific patterns. Rate-limit per-IP, block
  suspicious referrers, etc.
- **Don't WAF-protect what you don't need to.** Internal-only services
  behind a private endpoint don't need a WAF.

**Tools:**
- **Azure Front Door / Application Gateway WAF.**
- **AWS WAF** (on CloudFront / ALB / API Gateway).
- **GCP Cloud Armor.**
- **Cloudflare** (if you're CDN-fronted).

## DDoS protection

The cloud-native protection (AWS Shield Standard, Azure DDoS Protection
Basic, GCP basic DDoS) covers volumetric attacks for free. Most workloads
don't need more.

The paid tiers (AWS Shield Advanced, Azure DDoS Protection Standard) add:

- Cost protection for scaling during attacks.
- 24/7 response team.
- Application-layer attack visibility.

Worth it for revenue-critical, public-facing services. Overkill for
internal tools.

## East-west controls

The most under-invested layer. Once an attacker is inside the perimeter,
what stops them from moving laterally?

**Kubernetes:**

- **NetworkPolicy** (with a CNI that enforces them — Calico, Cilium,
  Azure CNI with network policy). Default-deny at the namespace level;
  explicit allows for cross-namespace traffic.
- **Service mesh** with mTLS + AuthorizationPolicy (Istio) or
  ServerAuthorization (Linkerd). Identity-based, not IP-based.

**VMs / non-Kubernetes:**

- **Security Group references** between tiers.
- **Subnet segmentation.**
- **Micro-segmentation tools** (Illumio, GuardiCore) for fine-grained policy.

The pragmatic minimum: **deny by default east-west, explicitly allow what
the architecture diagram shows.**

## Bastion / jumpbox patterns — what to use instead

The old way: bastion VM, SSH keys, hope nobody steals the key.

The new way:

- **AWS SSM Session Manager** — no SSH key, no bastion VM, audit log of
  every command.
- **Azure Bastion** — no SSH/RDP keys on user machines, audit log, MFA at
  the bastion.
- **GCP Identity-Aware Proxy (IAP)** — same idea, identity-based.
- **Teleport / strongDM** — third-party, multi-cloud, advanced features.

If you still have a VM called "bastion-prod" with SSH on port 22, modernize.

## TLS everywhere

Application-layer TLS is the assumption now:

- **TLS to load balancers** — yes, always.
- **TLS from load balancer to backend** — yes for prod; some teams skip
  inside the VPC. Risky.
- **TLS service-to-service** — service mesh mTLS handles this for K8s.
  For non-K8s, app-level TLS.
- **TLS to databases** — yes; cloud-managed DBs make this trivial.

Self-signed certs in prod are a smell. cert-manager + Let's Encrypt + DNS-01
challenge is the standard pattern.

## Gotchas

- **Public IPs accidentally** — Auto-created public IPs on EC2 or VMs in
  public subnets. Lock down with VPC settings: "no public IPs by default".
- **NACLs are stateless** — return traffic needs its own rule. Many teams
  break things here.
- **Service Tags / FQDN rules in cloud firewalls drift.** If a cloud
  service changes IP ranges, your "allow Azure Monitor" rule may stop
  working until tags catch up.
- **Private DNS zones not linked** — private endpoint exists, app can't
  resolve. Link the zone to the VPC.
- **Peering ≠ transitive.** A peers B, B peers C — A and C can't talk
  through B unless you set up a transit gateway / virtual WAN.
- **Default routes** — every subnet has a default route. Make sure private
  subnets' default is NAT, not internet gateway.
- **Long-lived TLS certs** — manage rotation with cert-manager or equivalent.
  90-day Let's Encrypt with auto-rotation > 1-year manual.
- **Public S3 buckets / blobs / objects** — still happens. Bucket-level
  block-public-access settings are mandatory.

## When some of these layers are optional

- **Internal-only services** — no WAF, no public endpoint, maybe no DDoS
  protection beyond cloud baseline.
- **Pre-prod / dev** — relaxed egress filtering is acceptable; tighter in
  prod. Don't drop ingress or east-west controls.
- **Edge-only / CDN-fronted services** — the CDN provides DDoS + WAF; you
  don't double up.

But: never skip subnet segmentation, never skip default-deny security
groups, never let private workloads have public IPs.

## Related

- [`iam-least-privilege-in-practice.md`](./iam-least-privilege-in-practice.md)
  — IAM controls who can change network policies.
- [`secrets-management-for-cloud-workloads.md`](./secrets-management-for-cloud-workloads.md)
  — private endpoints to the secret store close the exfil path.
- [`threat-modeling-without-the-bureaucracy.md`](./threat-modeling-without-the-bureaucracy.md)
  — network failures are common in threat models.
- [`../kubernetes/service-mesh-when-and-why.md`](../kubernetes/service-mesh-when-and-why.md)
  — mTLS via mesh is the east-west control for Kubernetes.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — AKS-specific network policy and CNI choices.
