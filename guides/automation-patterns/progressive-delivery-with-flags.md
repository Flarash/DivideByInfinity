# Progressive delivery with feature flags

**TL;DR:** Feature flags are powerful and dangerous. The power: deploy and release
become independent, you can canary by cohort, you can kill features without
redeploying. The danger: every flag is permanent debt unless you actively kill it.
Most flag pain comes from teams that treat all flags the same. Categorize them up
front, set expiry dates on release/experiment flags, and treat kill switches as
first-class permanent infrastructure.

## What "progressive delivery" actually means

Three independently-toggleable things:

- **Who sees a feature** (cohort: internal, beta users, by region, random %).
- **Whether a feature is on at all** (kill switch).
- **How a feature behaves** (configuration variant: A, B, C).

A static deploy can't do any of these without code changes. Feature flags can do
all three at runtime.

That's it. "Progressive delivery" is just "deploy and release decoupled" + "we
can change behavior without redeploying".

## Flag categories — the half-rule nobody writes down

All flags look the same to the SDK. Their *lifespans* are wildly different.
If you don't categorize, you'll end up with a graveyard of dead release flags
that nobody dares delete.

| Category | Purpose | Expected lifespan | Default state |
|---|---|---|---|
| **Release** | Ship code dark; flip on for launch | Days-weeks (DELETE after launch) | Off, then on |
| **Experiment** | A/B/n test | Weeks (DELETE after experiment closes) | Variant per arm |
| **Permission** | Per-user/per-tier feature access | Permanent | Per-user logic |
| **Operational / kill switch** | Emergency disable | Permanent | **On**, can be turned off |
| **Configuration** | Tune behavior (rate limit, batch size) | Permanent | Default value |

The discipline:

- Release and experiment flags ship with an **expiry date** in metadata.
- A scheduled job lints flags past expiry → opens issues / PRs.
- Permission, operational, and configuration flags are **named differently**
  and exempted from the expiry lint.

This single practice prevents the flag graveyard.

## Provider landscape

| Provider | Type | Strength |
|---|---|---|
| **LaunchDarkly** | SaaS | Most mature, expensive at scale |
| **Statsig** | SaaS | Strong experimentation + analytics |
| **Split** | SaaS | Experimentation-first |
| **ConfigCat** | SaaS | Cheap, simple, sufficient for most |
| **Unleash** | Self-hosted (or SaaS) | Strong OSS option |
| **Flipt** | Self-hosted | Lightweight, single binary |
| **Flagsmith** | Self-hosted (or SaaS) | Good UX, integrates with most stacks |
| **OpenFeature** | Standard (not a provider) | Vendor-neutral SDK; pick any backend |

**OpenFeature** is the boring right answer for the SDK layer. It's a vendor-neutral
spec; you swap backends without changing app code. If you're picking now,
start with OpenFeature + ConfigCat or Unleash (cheap/free), and reserve the
SaaS budget for when experimentation needs it.

## SDK patterns

The minimal flag:

```typescript
import { OpenFeature } from '@openfeature/server-sdk';

const client = OpenFeature.getClient();

async function checkout(userId: string) {
  const newFlow = await client.getBooleanValue('new-checkout-flow', false, {
    targetingKey: userId,
  });

  return newFlow ? newCheckoutFlow() : oldCheckoutFlow();
}
```

Three things to notice:

- **Default value is `false`.** Always pass an explicit default. If the SDK
  can't reach the backend, this is what users get.
- **Targeting key is the user ID.** Not the request ID — same user should see
  consistent behavior across requests.
- **`await` is real.** Flag evaluation may hit the network. Initialize the SDK
  early and cache.

### Kill switches default to ON

A kill switch is "the feature is currently on; emergency, turn it off."

```typescript
const featureEnabled = await client.getBooleanValue(
  'payments.stripe-fallback',
  true,    // default: feature is ON
  { targetingKey: userId }
);
```

If the flag provider is down, payments still work. Kill switches that fail
*closed* take down features when nothing changed.

### Release flags default to OFF

A release flag is "the new feature is hidden until we say so."

```typescript
const newFlow = await client.getBooleanValue(
  'release.new-checkout-v2',
  false,   // default: hidden until explicitly on
  { targetingKey: userId }
);
```

If the provider is down during a deploy, users see the old feature, which is
the conservative outcome.

The default direction encodes intent. Get it wrong and your kill switch fails
unsafe.

## Progressive rollout patterns

### Percentage rollout

```yaml
# in the flag provider
new-checkout-flow:
  rollout:
    - percentage: 1   # 1% of users
    - percentage: 10  # then 10%
    - percentage: 50
    - percentage: 100
```

Manual stepping. Monitor metrics between steps. Auto-rollout exists but
requires confidence in your metric thresholds — see "canary" in
[`deployment-strategies.md`](./deployment-strategies.md).

### Cohort rollout

```yaml
new-checkout-flow:
  targeting:
    - rule: "userTier = 'internal'"
      serve: true
    - rule: "country IN ['US', 'CA']"
      serve: true
    - else: false
```

Common pattern: dogfood (internal) → beta cohort → 1% prod → full.

### Sticky bucketing

Once a user lands on variant A, they keep variant A across sessions. Without
this, A/B tests are useless because users see inconsistent behavior.

Most providers handle this via the `targetingKey` (user ID). Don't roll your
own bucketing.

## The flag debt problem

Every flag is permanent debt unless actively retired. Mature teams measure:

- **Flag count over time** — flat or declining is healthy. Always increasing
  means you're accumulating debt.
- **Average flag age** — release flags older than 90 days are smells. Older
  than 1 year are bugs.
- **Code branch coverage** — both sides of the flag should be exercised by
  tests. Else one side rots silently.

The retirement process:

1. Flag is "permanently on" in all environments for N weeks.
2. Open a PR removing the flag check and the dead code path.
3. Delete the flag in the provider.

Skipping step 3 leaves a zombie. Skipping step 2 leaves dead code. Do all three.

## Audit trail — the bit auditors will ask about

Every flag change is a production change. Who flipped what, when, and why?

What you need:

- **Per-change audit log** — provider stores who/what/when. Most do.
- **Reason field** — require a reason on every flip. Free text is fine; it's
  for forensics, not analytics.
- **Approval workflow** for prod flags — at least a 4-eyes requirement, ideally
  integrated with PR review.
- **Notifications** — Slack/Teams webhook on every prod flag change.

If your provider doesn't surface this, you'll regret it during incident
review and during SOC 2 audits.

## Flag-aware testing

Tests have to be flag-aware:

```typescript
describe('checkout', () => {
  beforeEach(() => mockFlag('new-checkout-flow', false));

  it('serves old flow when flag off', async () => {
    expect(await checkout('u1')).toContain('legacy-form');
  });

  it('serves new flow when flag on', async () => {
    mockFlag('new-checkout-flow', true);
    expect(await checkout('u1')).toContain('react-form');
  });
});
```

Most provider SDKs ship a mock/test client. Use it.

Without flag-aware tests, you have two branches of code: one tested, one
"works on my machine when the flag is off in dev". Production finds out.

## When progressive delivery actively hurts

- **Tiny services with low traffic.** A 1% rollout of 10 RPS = 0.1 RPS = noise.
  Just deploy.
- **Workflows that span more than one service** — flag state in service A doesn't
  imply flag state in service B. Cross-service progressive rollout is a
  distributed-systems problem.
- **High-cardinality variants.** 12 simultaneous A/B/C tests interact. The
  combinatorics defeat the analysis.
- **Stateful flows.** A user mid-checkout when you flip the flag = inconsistent
  experience. Bucket on session start, not per request.
- **Backend changes the user doesn't see.** Probably doesn't need a flag.
  Deploy it.

## Gotchas

- **SDK initialization is async.** Code that checks flags before the SDK is
  ready returns defaults. Initialize on app startup and gate-block on
  readiness.
- **Network calls per flag check** are how flag providers ruin your tail
  latency. Use SDKs that bulk-fetch and cache.
- **Stale evaluations after a flip** — caches have TTLs. "Instant rollback"
  is "instant up to N seconds depending on cache settings." Configure
  accordingly.
- **Flag explosion in big-bang refactors.** "Let's gate the new architecture
  behind a flag" = 50 flags, all coupled. Either commit to the rewrite or
  don't.
- **GDPR / privacy with `targetingKey`.** User IDs in flag evaluations are
  *processing of personal data*. Document it.
- **Cross-team flag namespace collisions.** Use prefixes (`payments.`, `growth.`)
  or topic ownership becomes a fight.

## Related

- [`deployment-strategies.md`](./deployment-strategies.md) — flags are the
  release strategy that decouples from deploy.
- [`multi-environment-promotion.md`](./multi-environment-promotion.md) — flags
  intersect with environment promotion (per-env flag values).
- [`pr-as-publish-gate.md`](./pr-as-publish-gate.md) — same idea (gated
  release) applied at the PR level instead of the runtime level.
