# Secrets management for cloud workloads

**TL;DR:** Secrets in env vars are last decade. Modern workloads use a secret store
(Key Vault / Secrets Manager / GCP Secret Manager / HashiCorp Vault) plus workload
identity — the app authenticates as itself, asks for the secret it needs, and the
store gates the access. Rotation becomes routine instead of risky. The biggest
gotcha: every "we just hardcoded it for now" survives to production.

## The hierarchy of bad-to-good secret handling

| Approach | Problem |
|---|---|
| Hardcoded in source code | Forever in git history. Don't. |
| Plaintext in `.env` files | Better, but easy to commit accidentally. Easy to leak via logs. |
| Plaintext env vars at runtime | Visible in process listings, container envs, error pages. |
| Encrypted at rest in config (SOPS, sealed-secrets) | Better; decryption key is the new secret. |
| **External secret store + workload identity** | Modern default. Secret lives in the store; workload pulls at runtime. |
| **External store + ephemeral / dynamic secrets** | Vault-style: per-request DB credentials, valid for an hour. |

Pick the highest rung your platform supports. For most cloud-native workloads,
the right rung is **external store + workload identity**.

## The big four secret stores

| Store | Best for | Notable feature |
|---|---|---|
| **Azure Key Vault** | Azure-native shops | Tight Managed Identity integration |
| **AWS Secrets Manager** | AWS-native shops | Native rotation for RDS, Redshift, etc. |
| **GCP Secret Manager** | GCP-native shops | Tight Workload Identity integration |
| **HashiCorp Vault** | Multi-cloud / on-prem; advanced needs | Dynamic secrets, transit engine, policy as code |

For 90% of cloud workloads, the cloud-native store is sufficient. Add Vault
when:

- You need dynamic database credentials (Vault issues a per-session DB user
  and revokes it later).
- You need cross-cloud secret access from a single store.
- You need the transit secrets engine (encryption-as-a-service for app data).

## Workload identity — the gateway

The pattern: the workload (pod, function, VM) has an identity issued by the
cloud. It uses that identity to authenticate to the secret store. No
credentials stored on disk.

- **AWS:** IAM Roles for Service Accounts (IRSA), EC2 instance profiles,
  Lambda execution roles, ECS task roles.
- **Azure:** Managed Identity (system-assigned or user-assigned), Workload
  Identity for AKS.
- **GCP:** Workload Identity for GKE, Compute Engine default/custom service
  accounts.

The flow:

1. Pod starts. It has a Kubernetes ServiceAccount associated with a cloud
   identity.
2. App code uses the cloud SDK. The SDK detects the workload identity and
   acquires a token.
3. App calls `KeyVaultClient.GetSecret("db-password")`.
4. Key Vault validates the token, checks IAM, returns the secret.
5. App uses the secret in memory only.

Nothing stored on disk. Nothing in environment variables. The compromise
of any single component is bounded.

## The CSI driver pattern (Kubernetes)

For Kubernetes, the **Secrets Store CSI driver** mounts secrets from external
stores as files in the pod.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
spec:
  provider: azure
  parameters:
    keyvaultName: "prod-keyvault"
    tenantId: "<tenant-id>"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
---
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa     # bound to a Managed Identity
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secrets
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "my-app-secrets"
```

The app reads `/mnt/secrets/db-password` like any file. The CSI driver fetched
it via the pod's workload identity. The secret is never an env var, never a
Kubernetes Secret (unless you opt into syncing one).

**Cross-cloud equivalents:**
- **AWS**: secrets-store-csi-driver-provider-aws.
- **GCP**: secrets-store-csi-driver-provider-gcp.
- **Vault**: vault-csi-provider.

This is the cleanest pattern for Kubernetes secrets in 2026.

## Rotation patterns

Static secrets that never rotate are the second-worst-case scenario (after
plaintext-in-code).

### Automatic rotation
- **AWS Secrets Manager** — native rotation for RDS, Redshift, DocumentDB.
  Lambda runs the rotation; secret version bumps.
- **Azure Key Vault** — rotation policy + automation runbook (or Azure
  Function) to rotate.
- **Vault dynamic secrets** — per-request credentials. Rotation is
  intrinsic — every issuance is a "rotation".

### Manual rotation
- Document the procedure.
- Schedule it (every 90 days at minimum; sensitive secrets more frequently).
- Coordinate with app teams — apps must re-read the secret after rotation.

### App behavior during rotation
The hard problem: the app has secret version N in memory. The store rotates
to version N+1. Now what?

Three patterns:

1. **Restart-on-rotation** — secret rotates → app pods restart → pick up new
   secret. Simple, brief downtime.
2. **Cached-with-TTL** — app re-reads secret every N minutes. New secret
   picked up within TTL. No restart, but cache window of mismatch.
3. **Long-polling / event-driven** — app subscribes to "secret changed"
   events; refreshes on demand. Most complex; instant.

Most teams should choose pattern 1 (restart-on-rotation) for simplicity.
Pattern 2 if downtime is costly. Pattern 3 if you're already using event-based
infra.

## Dynamic secrets — Vault's killer feature

Vault can issue *per-request* credentials with TTLs.

Example: a service needs to query the database. Instead of a long-lived
password, the service asks Vault for credentials. Vault:

1. Creates a new database user with limited permissions.
2. Returns username + password to the service.
3. Sets a TTL (e.g., 1 hour).
4. After TTL, automatically deletes the user.

If the service is compromised, the attacker has 1-hour credentials, then
nothing. If the credentials leak in a log, they're useless after the TTL.

This is the gold-standard pattern, but requires Vault and the database
plugins to be set up. Often overkill for small workloads; worth it for
high-value databases.

## The "secret in env var" problem

Why env vars are bad even with rotation:

- **Visible in process listings** to anyone on the host (`ps eauxw`).
- **Logged when the app crashes** if env is dumped in the stack trace.
- **Visible to sidecars** sharing the pod (`/proc/$PID/environ`).
- **Stuck for the process lifetime** — can't rotate without restart.
- **Stuck in container snapshots** if you forget and snapshot the env.

The CSI-mounted-file pattern avoids most of this. The app reads the file
when it needs the value and can re-read for rotation.

## What about Kubernetes Secrets?

Native `kind: Secret` resources are *base64-encoded*, not encrypted, by
default. With encryption-at-rest enabled (etcd encryption), they're
encrypted on disk but plaintext in API responses.

The realistic stance:

- **Don't put long-lived secrets in native K8s Secrets.** Use a secret store +
  CSI driver.
- **It's OK for** ephemeral, in-cluster-only values (TLS certs from
  cert-manager, registry pull credentials provisioned automatically).
- **External Secrets Operator** is the middle ground — manages Secrets in
  the cluster but sources them from an external store, can refresh on a
  schedule. Better than nothing if CSI isn't an option.

## The boundary with workload identity

A common confusion: "we have workload identity, do we still need a secret
store?"

- **Workload identity** authenticates the workload.
- **The secret store** is what the workload accesses.

Workload identity doesn't make secrets unnecessary — it makes secret
*retrieval* secure. You still need:

- Database passwords (the DB doesn't know about Azure AD).
- API tokens for third-party services (Stripe, SendGrid, GitHub).
- Encryption keys for app-level encryption.
- TLS certificates if not managed by cert-manager.

The workload identity is the credential to get the secret, not a replacement
for it.

## Gotchas

- **Caching without invalidation** — apps cache secrets for an hour, you
  rotate, ten minutes later the cached secret is wrong. Configure cache TTL <
  rotation window.
- **Secret stores have rate limits.** Hot-path apps that fetch a secret per
  request will hit them. Cache aggressively; refresh on a schedule.
- **TLS to the secret store.** Mismatched CA bundles fail mysteriously.
  Verify your container image trusts the right CAs.
- **Don't print secrets in logs, ever.** Log "DB connection failed" not
  "DB connection failed: postgres://user:pass@host/db".
- **Secret rotation breaks long-lived connections.** Database connection
  pools, persistent message-bus connections — they hold the old password.
  Reconnection after rotation must work.
- **Backup of the secret store.** Treat it as the production data it is.
- **Cross-region replication of secret stores** for DR — without it, your
  DR region can't read prod secrets when you fail over.
- **Local-dev secrets** — for local development, devs need *something*. Use
  `.env.example` checked in, `.env.local` in `.gitignore`, and a doc pointing
  to dev-grade secrets in a shared password manager. Never share prod secrets
  for local dev.

## When the cloud-native store isn't enough

- **Multi-cloud secret sharing** — Vault is genuinely better here.
- **Dynamic / per-session secrets** — Vault, or build it yourself (don't).
- **Air-gapped or on-prem environments** — Vault.
- **Compliance requirement for HSM-backed key material** — most cloud KV
  services have HSM tiers; check yours.

For everything else, the cloud-native store is the right answer.

## Related

- [`iam-least-privilege-in-practice.md`](./iam-least-privilege-in-practice.md)
  — workload identity needs IAM grants on the secret store.
- [`network-security-layers.md`](./network-security-layers.md) — secrets in
  transit are a network problem; private endpoints to the secret store matter.
- [`threat-modeling-without-the-bureaucracy.md`](./threat-modeling-without-the-bureaucracy.md)
  — secret-handling failures are common findings.
- [`../agent-workspaces/secrets-management-for-ai-agents.md`](../agent-workspaces/secrets-management-for-ai-agents.md)
  — the desktop-agent variant of the same problem.
- [`../iac/policy-as-code-and-drift-detection.md`](../iac/policy-as-code-and-drift-detection.md)
  — policies to enforce "no secrets in code" and "Key Vaults must have private
  endpoints".
