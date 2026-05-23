# Deployment strategies

**TL;DR:** Pick the strategy that matches your failure cost, not the one that
sounds most sophisticated. Most teams need rolling or blue-green; canary and
flags add value only when you have the observability to drive them. The
strategy is half the work — the *rollback path* is the other half, and most
teams skip it.

## The strategies at a glance

| Strategy | What changes | Rollback speed | Complexity | When |
|---|---|---|---|---|
| **Recreate** | Stop old → start new | Slow (full redeploy) | Low | Dev only. Never prod. |
| **Rolling** | Replace pods/instances one batch at a time | Medium (next deploy) | Low | Default for stateless services |
| **Blue-green** | Two full environments; switch traffic | Instant (flip back) | Medium | When rollback latency matters |
| **Canary** | New version to 1% → 10% → 50% → 100% | Medium (drop new tier) | High (needs metrics + automation) | High-traffic services, expensive bugs |
| **Feature flags** | Code ships dark; toggle at runtime | Instant (flip flag) | High (flag lifecycle) | Product experimentation, kill switches |

These aren't mutually exclusive. A mature pipeline often combines rolling
deployment + feature flags. Or blue-green + canary on top.

## Rolling — the boring default

Replace N instances at a time with the new version. Healthy ones keep serving.
When all are replaced, deployment is done.

**Kubernetes:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%   # how many old pods can be down
    maxSurge: 25%         # how many extra pods can spin up
```

**Azure App Service:** built-in for slot-based deploys.

**AWS EC2:** CodeDeploy with a "OneAtATime" or "HalfAtATime" config.

**Pros:**
- Simple, well-understood, no extra infra.
- Resource-efficient (no double capacity).

**Cons:**
- Mixed-version state during the deploy window. Long-lived connections /
  stateful sessions can break.
- Rollback = next deploy of the previous version. Minutes, not seconds.
- Bad version can poison N% of traffic before health checks catch it.

Use rolling for: stateless services, internal tools, anything where "bad deploy
visible for a few minutes" is acceptable.

## Blue-green — instant rollback, double cost

Two full copies of the environment (blue = current, green = new). Deploy to
green, smoke-test it, then swap traffic. To roll back: swap back to blue.

**Mechanics:**
- Traffic switch via load balancer config, DNS, or service mesh.
- Database migrations need care: green and blue both connect to the same DB,
  so schema changes must be backward-compatible.
- Sessions need a story: sticky sessions break; JWTs survive.

**Per-platform:**
- **Azure:** App Service deployment slots (built-in). Two slots, swap.
- **AWS:** ALB target groups with weighted routing, or CodeDeploy with
  BlueGreen strategy on EC2/ECS.
- **GCP:** Cloud Run revision splits, or GKE with Argo Rollouts.
- **Kubernetes:** Two Deployments + a Service selector switch, or use Argo
  Rollouts for orchestration.

**Pros:**
- Instant rollback (flip traffic back).
- Smoke-test in green before any real user sees it.
- No mixed-version state in steady state.

**Cons:**
- Double resource cost during deploy windows.
- Schema migrations require forward-compatibility discipline.
- Long-running async work on blue can become orphaned mid-swap.

Use blue-green when: rollback speed matters more than infra cost, schema
migration discipline exists, traffic switching is reliable.

## Canary — sophisticated, expensive in operational tax

New version goes to a small slice of traffic. Compare metrics (error rate,
latency, saturation) vs the old version. If healthy, expand the slice. If not,
abort.

**Slice mechanisms:**
- Random N% of requests (service mesh, ingress).
- Specific user cohorts (header-based routing).
- Geographic (one region, then more).
- Internal users first ("dogfood").

**Per-platform:**
- **Service mesh:** Istio, Linkerd, App Mesh — native traffic-splitting by
  weight or header.
- **Argo Rollouts** (Kubernetes): canary strategy as a CRD, with analysis
  templates driving the abort decision.
- **Azure Front Door / AWS App Mesh / GCP Traffic Director:** weighted routing
  at the global tier.

**Pros:**
- Bad deploys hit a small fraction of users before being rolled back.
- Production-truth signal — you find issues that staging didn't catch.
- Combined with feature flags, you can canary a *subset of behavior*.

**Cons:**
- Needs solid metrics + alerting to be useful. "Canary at 5%" without analysis
  is just rolling-with-extra-steps.
- Operationally complex. The canary stage is its own incident surface.
- Statistical significance for low-volume services is hard. A canary at 1% of
  100 RPS = 1 RPS = noisy.

Use canary when: traffic is high enough for signal, the team has metric +
alerting maturity, the cost of a bad deploy at full scale is much higher than
the operational tax of canary infra.

## Feature flags — runtime, not deploy-time

The code ships dark (deployed but not active). Flags control whether it's
visible to users. Deploys and releases become independent.

```typescript
if (await flags.isEnabled('new-checkout-flow', { userId })) {
  return newCheckoutFlow();
}
return oldCheckoutFlow();
```

**Mechanism providers:**
- **SaaS:** LaunchDarkly, Statsig, Unleash, Optimizely, Split, ConfigCat.
- **Self-hosted:** Unleash, FF4J, Flipt, Flagsmith.
- **DIY:** config in env vars / a feature-flag service in your app.

**Categories of flags** (the half-rule almost no one writes down):

| Flag type | Purpose | Lifespan |
|---|---|---|
| **Release flag** | Decouple deploy from launch | Days-weeks (delete after launch) |
| **Experiment flag** | A/B test | Weeks (delete after experiment) |
| **Permission flag** | Per-user feature access | Permanent |
| **Operational flag (kill switch)** | Emergency disable of a feature | Permanent |
| **Configuration flag** | Tune behavior without deploy | Permanent |

The mistake is treating all flags the same. Release/experiment flags are
*intended to die*. Permission/operational/config flags live forever. Mix them
up and you get the flag-graveyard problem (see
[`progressive-delivery-with-flags.md`](./progressive-delivery-with-flags.md)).

**Pros:**
- Instant rollback (flip flag, no redeploy).
- Decouple "merge-to-main" from "feature-live-in-prod".
- Enables true continuous deploy: every PR ships behind a flag.
- Per-user / per-cohort control.

**Cons:**
- Code complexity grows. `if (flag) { newPath } else { oldPath }` everywhere.
- Flag debt is real. Old flags rot.
- Testing matrix explodes: now you test combinations of flag states.
- Each flag is a runtime config that can fail (provider down, network issue).

Use flags for: anything you'd want to disable instantly in an incident, any
feature where you want to decouple deploy from launch, anything you'd A/B test.

## The rollback path — the other half

Every strategy needs an explicit rollback procedure. Most teams write the
deploy doc and skip the rollback doc. Then the incident happens.

**Rolling:** rollback = deploy previous version. Speed = deploy speed.
Document the *exact commands*; don't make on-call figure out the previous SHA
at 3am.

**Blue-green:** rollback = swap traffic back. Should be a single command or
button. Test it.

**Canary:** rollback = abort canary, drop new pods. Argo Rollouts can automate
this on metric thresholds.

**Feature flags:** rollback = flip flag. Should be one command. Has the
advantage of *partial* rollback (turn off for new users, keep for ones already
in the flow).

**Database migrations:** the asymmetric problem. Forward-compatible migrations
are deploy-safe. Backward-incompatible migrations (drop column, change type)
require the rollback-aware shape: deploy → migrate forward → app reads old
*and* new → next deploy reads only new → finally drop the old column. Three
deploys for a "simple" rename. Build this in.

## Multi-platform realities

A single org often runs different strategies in different places:

- **Kubernetes microservices:** rolling + flags (default), canary for hot paths.
- **Azure App Service / AWS Elastic Beanstalk:** blue-green via slots.
- **Functions (Lambda, Functions, Cloud Run):** versioned deploys with
  aliases; weighted routing for canary.
- **Mobile apps:** staged rollout (Play Store / TestFlight). The store *is*
  your canary mechanism.
- **Database/schema:** forward-compatible migrations + multi-step rollout.

The strategy choice belongs at the service level, not the org level. Mandating
"everything must canary" produces shadow rolling deploys with extra ceremony.

## Gotchas

- **Health checks must reflect real readiness.** A 200 from `/healthz` that
  doesn't verify downstream connectivity is a lie. Bad deploys pass the lie
  and rolling-update proceeds into a hole.
- **`maxSurge: 0` is a footgun.** No extra capacity means you can't safely
  roll back during a deploy. Default to maxSurge > 0.
- **Stateful workloads need different math.** Databases don't blue-green
  trivially. Statefulsets aren't subject to the same rolling-update mechanics.
  Read the docs for the specific resource.
- **Long-lived connections (WebSocket, gRPC streams) hate deploys.**
  graceful-shutdown windows, pre-stop hooks, and connection-draining at the LB
  matter.
- **Feature flags that fail-closed default to "off"** — which can take a
  feature down even when nothing changed. Choose the default carefully per
  flag (release flags default off; kill switches default to on).
- **Canary metrics need warm-up.** Comparing first-30-seconds-of-new-pod to
  steady-state-of-old-pod fails false-positively. Apply analysis after warm-up.

## When NOT to use each

- **Recreate:** never in prod. It's only here to be complete.
- **Canary:** when you don't have the metrics to drive the decision. Without
  signal, canary is rolling with extra steps.
- **Feature flags:** for everything. Some code paths don't need flags. The
  flag-everything mindset creates flag debt.
- **Blue-green:** when database migrations are routinely backward-incompatible.
  Fix the migration discipline first.

## Related

- [`progressive-delivery-with-flags.md`](./progressive-delivery-with-flags.md)
  — feature flag patterns in depth, kill switches, and flag debt management.
- [`multi-environment-promotion.md`](./multi-environment-promotion.md) — the
  promotion path that feeds these strategies.
- [`pr-as-publish-gate.md`](./pr-as-publish-gate.md) — gating deploys behind
  merge.
- [`../cloud-platform/cicd-pipeline-patterns.md`](../cloud-platform/cicd-pipeline-patterns.md)
  — how Azure DevOps / GitLab CI / Jenkins implement these strategies.
- The forthcoming `kubernetes/` topic folder — Argo Rollouts, deployment
  strategy CRDs, and service-mesh canary mechanics.
