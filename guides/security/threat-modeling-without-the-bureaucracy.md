# Threat modeling without the bureaucracy

**TL;DR:** Threat modeling has a bad reputation because corporate versions of it
involve 40-page documents nobody reads. The useful version is a 30-minute team
exercise asking four questions, capturing actionable findings, and feeding them
into the backlog. Do it lightly, do it often, and treat the output as input to
engineering — not as an audit artifact.

## The four-question version

Adam Shostack's four-question framework, in plain English:

1. **What are we working on?** (a diagram or description)
2. **What can go wrong?** (the threats)
3. **What are we going to do about it?** (the mitigations)
4. **Did we do a good enough job?** (the validation)

That's the whole framework. Everything else — STRIDE, attack trees, kill
chains — is *technique* you use *within* this loop. The framework itself is
the loop.

## When to do it

- **New service / new feature with security implications** (auth, data
  storage, external integration). Before code is written.
- **Major architectural changes** to existing services.
- **After incidents** — "did our threat model see this coming?"
- **On a cadence** for critical services — every 6 months minimum.
- **Before launching to a new customer segment** (enterprise, government,
  regulated industry).

If you're doing it for the first time, pick the most important service. Don't
try to model everything in one go.

## STRIDE — the analytical lens

STRIDE is a checklist of threat categories. Walk through your diagram and
ask which apply.

| Letter | Threat | Question to ask |
|---|---|---|
| **S** | Spoofing | Can an attacker pretend to be someone else? |
| **T** | Tampering | Can data be modified in transit or at rest? |
| **R** | Repudiation | Can an actor deny they did something? |
| **I** | Information disclosure | Can data leak to unintended parties? |
| **D** | Denial of service | Can the service be made unavailable? |
| **E** | Elevation of privilege | Can a low-privilege actor gain higher privileges? |

Use STRIDE as a prompt, not as an exhaustive taxonomy. The point is to *not
forget* a category, not to write a paragraph per cell.

## The 30-minute team exercise

What it actually looks like:

1. **Pre-work (5 min before):** owner pastes the C4 container diagram or a
   simple sketch in a Miro/whiteboard.
2. **Round 1: What are we working on?** (5 min) Owner walks the diagram.
   Everyone asks clarifying questions. No threats yet.
3. **Round 2: What can go wrong?** (15 min) Each person silently brainstorms
   threats per STRIDE category, then shares. Capture each as a sticky note
   with: threat, asset affected, attacker capability assumed.
4. **Round 3: What are we going to do about it?** (10 min) Cluster
   duplicates. Quickly tag each: *mitigate* / *accept* / *transfer* /
   *avoid*. For *mitigate*, name a specific control.
5. **Closeout (5 min):** convert the *mitigate* items into backlog tickets.
   Decide owner and rough priority.

**Output:** a list of tickets and a short summary doc with the diagram.

Not output: a 40-page threat model document with risk-rating matrices.

## Documenting the result

The minimum viable threat model document:

```markdown
# Threat model: Payments service v2 launch

**Date:** 2025-04-15
**Participants:** payments team + 1 security engineer
**Diagram:** [link to C4 container diagram]

## Assets
- Card numbers (PCI scope)
- Customer email + name
- Auth tokens

## Threats and decisions

| ID | Threat | Category | Decision | Action |
|---|---|---|---|---|
| T1 | Card numbers in app logs | Information disclosure | Mitigate | Add log scrubber (ticket #1234) |
| T2 | Token replay on idle sessions | Spoofing | Mitigate | Add token expiry + rotation (ticket #1235) |
| T3 | Volumetric DDoS on /checkout | DoS | Accept | Behind AWS Shield Standard; cost cap on auto-scaling |
| T4 | Internal staff querying customer data | Info disclosure | Mitigate | Add data-access audit log (ticket #1236) |

## Not addressed (out of scope this iteration)
- Long-term key management for tokenization vendor
- Side-channel timing attacks on token validation

## Next review
2025-10-15
```

That's it. Half a page. Living in the repo, version-controlled.

## Common decisions (mitigate / accept / transfer / avoid)

- **Mitigate** — add a control. The default for most actionable findings.
- **Accept** — risk is below threshold or cost of mitigation is dispro-
  portionate. Document *why*.
- **Transfer** — risk shifts to a third party (insurance, vendor SLA).
- **Avoid** — change the design to remove the risk (don't store the data
  at all, don't expose the endpoint).

Most threats are *mitigate*. The interesting ones are *accept* (which require
explicit documentation of the trade-off) and *avoid* (which usually save
work in the long run).

## Where threat modeling fails

- **Done by security alone, presented to engineering.** Engineers don't own
  the findings, don't push back on assumptions, don't internalize the
  decisions. Output: shelfware.
- **Done at the wrong time.** After launch is firefighting, not modeling.
- **No follow-through.** Threats identified, tickets never filed.
- **Treated as a compliance artifact.** "We did a threat model" → tick the
  box → file it → forget it.
- **Tool-centric.** Heavy threat-modeling tools (Microsoft TMT, IriusRisk)
  can help large orgs; for small teams they're overhead. A whiteboard works.
- **Risk-rating matrices.** "Likelihood × Impact = priority" is a number
  that sounds objective but isn't. Use it as a discussion prompt, not a
  decision oracle.

## What "did we do a good enough job?" actually looks like

Two checks:

1. **Did engineering actually implement the mitigations?** Look at the
   tickets six months later. If "add log scrubber" is still open, the
   threat model didn't ship.
2. **Did real incidents match what we predicted?** If yes — model was
   useful. If no — model missed something; learn and update.

Threat models that aren't validated against reality become folklore.

## Specific patterns that often surface

The threats you'll find in 80% of threat models:

- **Hardcoded credentials** somewhere (Git history, container env vars,
  config files).
- **Over-permissioned service identities** (see
  [`iam-least-privilege-in-practice.md`](./iam-least-privilege-in-practice.md)).
- **Logging that captures PII or secrets**.
- **No rate limiting on expensive endpoints.**
- **Insufficient input validation** (especially on file uploads and search
  fields).
- **Missing CSRF tokens** on state-changing endpoints.
- **CORS misconfigurations** (allowing arbitrary origins).
- **Open S3 buckets / blob containers**.
- **Public IPs on workloads** that don't need them.
- **Trust boundaries unclear** ("we trust the internal network" with no
  east-west controls).
- **No audit log** for admin actions.
- **Cleanup of personal data** missing (GDPR deletion requests).

If your threat model doesn't surface at least one of these, you're not
asking hard enough questions.

## Integrating with the SDLC

- **At design** — threat model new features before implementation.
- **At PR** — security-relevant changes get a "threat model touched?"
  checkbox in the PR template. Quick re-review if yes.
- **At launch** — final walk-through of the threat model + mitigations
  before shipping.
- **At incident** — incident postmortem checks the threat model. If the
  incident matched a known threat that was *accepted*, was acceptance
  still right?
- **On cadence** — every 6 months for critical services.

## A small note on attacker capability

Threats are scoped by *who the attacker is*. Useful tiers:

- **Random internet attacker** — scripted, opportunistic. Defended by
  basic hygiene.
- **Targeted attacker** — knows your stack, may research employees. Needs
  authentication discipline + monitoring.
- **Insider, accidental** — well-meaning employee makes mistakes. Need
  guardrails + audit trail.
- **Insider, malicious** — rare but catastrophic. Need separation of duties
  + audit trail + careful permissioning.
- **State-level / advanced** — out of scope for most orgs; if in scope,
  this is a different discipline.

Most threat models reasonably assume tier 1-3 and explicitly skip tier 5.
Be explicit about which tiers you're modeling against.

## Gotchas

- **Diagram drift.** The system changes; the diagram doesn't. Stale models
  identify stale threats.
- **Compliance pressure** can warp the exercise toward "what does the
  auditor want to see" rather than "what's actually risky."
- **One-time effort.** A threat model done once and never updated rots
  faster than docs do.
- **Confusing threats with vulnerabilities.** A threat is "what could
  happen." A vulnerability is "what we found." Threat modeling is upstream
  of vulnerability discovery (pentests, SAST, DAST).
- **The "we already do that" reflex.** Engineers may dismiss threats they
  think are handled. Push for evidence — what control, where, last tested
  when?

## When threat modeling is overkill

- **Internal-only tools with no sensitive data.** Quick chat is enough.
- **Truly tiny services** (a single CRUD endpoint). The default controls
  (auth, TLS, input validation) cover most threats.
- **Throwaway prototypes** that won't ship.

But: anything that handles money, personal data, or has admin functions
warrants a threat model, no matter how small.

## Related

- [`iam-least-privilege-in-practice.md`](./iam-least-privilege-in-practice.md)
  — IAM mistakes are the most common threat-model finding.
- [`secrets-management-for-cloud-workloads.md`](./secrets-management-for-cloud-workloads.md)
  — secret handling is the second-most-common.
- [`network-security-layers.md`](./network-security-layers.md) — network
  controls show up as mitigations for spoofing/tampering/info-disclosure.
- [`../cloud-architecture/well-architected-decisions.md`](../cloud-architecture/well-architected-decisions.md)
  — security is one of the pillars; threat models are how you exercise it.
- [`../cloud-architecture/disaster-recovery-playbook.md`](../cloud-architecture/disaster-recovery-playbook.md)
  — DoS threats often map to DR planning.
