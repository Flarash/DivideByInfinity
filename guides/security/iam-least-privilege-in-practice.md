# IAM least-privilege in practice

**TL;DR:** Least-privilege is easy to say and hard to operate. The wildcard policy
("Action: *, Resource: *") is the sin everyone commits and the one that costs the
most when something gets compromised. The discipline: start with deny-all, grant
incrementally based on observed need, audit quarterly, and use workload identity
(not service-principal secrets) wherever the cloud supports it.

## What least-privilege actually means

A principal (user, role, service identity) can perform *exactly* the actions
they need on *exactly* the resources they need — no more.

Three things this rules out:

- **Wildcard actions** (`Action: *` or `"storage.objects.*"`).
- **Wildcard resources** (`Resource: *` or `arn:aws:s3:::*`).
- **Overly-broad roles** (Owner, Contributor, Editor) for non-admin work.

What it requires:

- A clear answer to "what does this identity need to do?" for every principal.
- The discipline to write narrower policies even when broader ones would "just work".
- A review cadence to catch drift toward over-permissioning.

## The wildcard policy is the bug

Almost every breach story includes "the role had wider permissions than it
needed." The dynamic:

1. New service ships. Developer needs it working today.
2. Strict policy fails with `AccessDenied`. Stack trace is opaque.
3. Developer broadens the policy until it works.
4. Nobody re-narrows it. Policy stays wide forever.
5. Years later, the service identity is compromised. The attacker inherits
   the wide policy.

Counter-tactics:

- **CI lint for wildcards.** Reject PRs that add `"*"` actions or resources
  without a comment explaining why.
- **IAM Access Analyzer / Azure Policy / GCP Recommender** to flag unused
  permissions. These tools tell you what was actually used vs granted; the
  delta is over-permission.
- **Time-boxed escalation.** Need broad access for debugging? Grant for 24
  hours via a process, not by editing the role.

## RBAC vs ABAC

| Model | What it grants on | Where it shines |
|---|---|---|
| **RBAC** (Role-Based) | Role assignments. "User X has Role Y on Scope Z." | Stable team structures; well-understood. |
| **ABAC** (Attribute-Based) | Conditions on attributes. "Role Y on resources tagged `team:payments`." | Highly multi-tenant; dynamic environments. |

Most clouds support both. The practical answer:

- **RBAC** for the org-chart use cases (team membership, on-call rotation).
- **ABAC conditions** layered on top for resource scoping
  (`condition: resource.tag.team == principal.tag.team`).

Pure ABAC is hard to audit. Pure RBAC at scale produces role explosion.
Hybrid is the working sweet spot.

## Modeling the three clouds

### AWS IAM
- **Users, Groups, Roles, Policies** — composable. Best practice: humans
  assume roles via SSO; services use IAM Roles attached to EC2/Lambda/EKS.
- **Resource-based policies** — buckets, KMS keys, SQS queues attach
  policies to themselves. The intersection of identity-policy and
  resource-policy must allow.
- **Permission boundaries** — a ceiling on what a role can ever grant
  itself, even if it has IAM-create permissions. Use for delegated admin.
- **SCP (Service Control Policies)** — org-level guardrails across accounts.

### Azure RBAC
- **Role assignments at scope**: management group, subscription, resource
  group, resource. Inherited downward.
- **Custom roles** for narrowly-scoped action sets.
- **Azure ABAC** (in preview / limited) for tag-based conditions.
- **Azure Policy** runs alongside RBAC and enforces broader rules (no public
  IPs in this RG, enforce encryption, etc.).

### GCP IAM
- **Roles bound to principals at a resource scope** — project / folder /
  organization. Inherited downward.
- **Custom roles** for the same reason as Azure custom roles.
- **VPC Service Controls** — perimeter around APIs, prevents data exfil.
- **IAM Conditions** for tag-based or time-based access.

The mental model translates across clouds: principals + roles + scope. The
syntax differs; the principles don't.

## Service identities — the right primitives

Hard-coded service principals with passwords/secrets are the past. Use:

| Cloud | Primitive | What you do instead of secrets |
|---|---|---|
| **AWS** | IAM Roles for Service Accounts (IRSA), EC2 instance profiles, ECS task roles | The compute assumes a role; no long-lived keys |
| **Azure** | Managed Identity (system-assigned or user-assigned), Workload Identity | The compute has an identity; the platform issues tokens |
| **GCP** | Workload Identity for GKE, Compute Engine default service accounts | The compute has a service account; the platform issues tokens |

For **CI/CD pipelines**:
- **GitHub Actions** → OIDC federation to AWS/Azure/GCP. No long-lived secrets
  in GitHub.
- **Azure DevOps** → Workload Identity Federation to AWS/Azure/GCP. Same idea.
- **GitLab CI** → ID token-based federation.

Every secret you don't store is one you don't rotate and one that can't leak.
Federation is more setup, less ongoing operational debt.

## Role assumption chains

Production access is rarely a direct grant. The chain:

1. User logs into SSO (Entra ID, Okta, Google Workspace).
2. SSO federates to the cloud → user is now an "SSO identity" in the cloud.
3. SSO identity is allowed to assume specific roles (`AssumeRole`,
   PIM-elevated role activation).
4. User assumes the role → temporary credentials with the role's permissions.

What this gives you:

- **No long-lived credentials in user hands.** Tokens are 1-12 hours.
- **Audit trail** at every step.
- **Just-in-time elevation** (Azure PIM, AWS IAM Identity Center session
  policies).
- **Role separation:** the SSO identity is read-only by default; assumed
  roles get specific scopes.

The chain works only if you enforce it. If users have direct console
passwords *and* role assumption, attackers will find the console password.

## The audit cadence

Permissions drift. Even with good practices, you accumulate:

- Old service identities for projects long dead.
- Users with roles from previous teams.
- Wildcard policies added "for now" and forgotten.

Quarterly review:

1. **List all principals** (users + service identities) in scope.
2. **For each, list assigned roles and policies.**
3. **For each policy, check usage** — IAM Access Analyzer / Azure Privileged
   Identity Insights / GCP IAM Recommender flag unused permissions.
4. **Remove what's unused.** Send a notification to the owner first.
5. **Document exceptions.**

This is dull work. It's also the work that catches the over-permissioned
role before it becomes the incident.

## Break-glass accounts

The account you use when SSO is down and you have to log in. Rules:

- **At least two break-glass accounts.** One isn't enough — what if it's
  compromised?
- **Stored offline.** Hardware token, paper in a safe, password manager with
  break-glass-only access.
- **Heavily alerted.** Every login = page the security team immediately.
- **Used only when SSO is broken.** Never for routine work.
- **Tested quarterly.** Confirm you can actually log in and that the
  permissions still work.

Without break-glass, an SSO outage is a complete loss of cloud admin access.

## Common anti-patterns

- **One mega-role for "the platform team"** with Owner permissions across all
  subscriptions/accounts/projects. Compromise = full org control.
- **Service identities shared across multiple services.** Compromise of one
  service blast-radiuses to all of them.
- **Granting permissions at the management-group / org level when only one
  RG / account / project needs them.** Convenient now; painful later.
- **Static API keys checked into code.** Still happens. Pre-commit hooks and
  org-wide secret scanning are the prevention.
- **"Just temporary" wildcard grants** that outlive the temporary need.

## Just-in-time / PIM patterns

The pattern that makes "elevated permissions" safer:

- User has *eligible* role assignments — not active by default.
- To use elevated permissions, the user *activates* the role:
  - Provides justification.
  - Optionally requires approval.
  - Activation is time-boxed (1-8 hours).
  - Activation logged and alertable.

After the window, permissions revert. Even if the user's account is
compromised, the attacker has the user's baseline permissions, not the
elevated ones (unless they trigger activation, which alerts).

**Implementations:**
- Azure PIM (in Entra ID).
- AWS Identity Center session policies + temporary elevation flows.
- GCP doesn't have a fully equivalent first-party feature; use custom
  workflows.

For production admin access, JIT/PIM is the modern bar. Standing admin
permissions are the legacy bar.

## Gotchas

- **Policy evaluation is order-specific in some places.** AWS evaluates
  Deny → Allow → SCPs → boundaries → resource policies. The intersection of
  *all* must allow. Reading the eval rules carefully matters.
- **Implicit deny is not the same as no statement.** If a permission isn't
  granted, it's denied. But an explicit Deny always wins over Allow.
- **Service-linked roles** (AWS) and managed identities (Azure) — auto-created
  with specific permissions. Document them; don't be surprised by them.
- **Cross-account access** requires both identity-policy on the calling side
  and resource-policy (or trust policy) on the receiving side. Either alone
  is insufficient.
- **MFA for IAM users** is non-negotiable. Even better: no IAM users; SSO
  only.
- **Don't put secrets in IAM policies as principal IDs.** The policy is
  often readable; the implicit "we hardcoded our prod account ID" reveals more
  than you think.
- **AssumeRole loops** — A trusts B, B trusts A. Possible; usually a misuse.
  Audit trust relationships.

## When relaxing least-privilege is acceptable

- **Initial bootstrap** — the first deploy needs a wide role to lay things
  down. Narrow it within the first week.
- **Genuinely automated platform services** (Cluster Autoscaler needs to
  describe many things). Pre-built service-specific roles exist; use them
  instead of inventing wide custom roles.
- **Read-only auditor roles** — wide read across resources is fine if it's
  truly read-only. Confirm there's no write/admin permission slipped in.

But: even for these, prefer the narrowest formulation that works.

## Related

- [`secrets-management-for-cloud-workloads.md`](./secrets-management-for-cloud-workloads.md)
  — workload identity replaces stored secrets; IAM grants the identity.
- [`network-security-layers.md`](./network-security-layers.md) — IAM is one
  defense layer; network security is the other.
- [`threat-modeling-without-the-bureaucracy.md`](./threat-modeling-without-the-bureaucracy.md)
  — IAM mistakes are the most common finding in any honest threat model.
- [`../iac/policy-as-code-and-drift-detection.md`](../iac/policy-as-code-and-drift-detection.md)
  — IaC policies catch IAM wildcards before they ship.
- [`../iac/terraform-state-management.md`](../iac/terraform-state-management.md)
  — restricting who can write to prod state is itself an IAM problem.
