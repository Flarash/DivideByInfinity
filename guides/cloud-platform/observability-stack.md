# Observability stack: Prometheus + Grafana + ELK

> Metrics, logs, traces — what each is for, what stack handles each well, and
> how to wire them together without paying a SaaS bill the size of your AWS
> bill.

This is the practical "we just need observability that works" version, not
the SRE-book version. Opinionated about defaults, honest about trade-offs.

---

## TL;DR

- **Three pillars: metrics, logs, traces.** Don't try to make one tool do all
  three well — they have different shapes.
- **Metrics: Prometheus.** Pull-based, cheap, tag everything carefully —
  **label cardinality is the killer**.
- **Logs: ELK *or* Loki + Grafana.** ELK if you need full-text search and have
  the budget; Loki if you mostly grep on labels and want it 5–10× cheaper.
- **Dashboards: Grafana.** Provision them from JSON in git, not click-ops.
- **Alerts: one rule = one human action.** If you can't write the runbook
  link, delete the alert.
- **Splunk fits the same shape** as ELK — same pillar (logs), proprietary,
  more $, fewer surprises in heavily-regulated environments.
- **The observability stack itself needs DR**, and is the thing teams forget.

---

## The three pillars (in 60 seconds)

| Pillar | Question it answers | Shape | Tools |
|---|---|---|---|
| **Metrics** | "Is the system healthy *right now*, and was it before?" | Numbers over time, tagged | Prometheus, Azure Monitor, Datadog metrics |
| **Logs** | "What exactly happened during *that* event?" | Append-only text records | ELK, Loki, Splunk, CloudWatch Logs |
| **Traces** | "How did this one request flow through 12 services?" | Span trees | Jaeger, Tempo, Honeycomb, Datadog APM |

Metrics are cheap to keep forever, expensive to query for unique events. Logs
are expensive to keep forever, cheap to query for specific events. Traces are
mid-cost and answer "where in the architecture is the latency?".

You need all three. Don't bolt three things on at once — start with metrics,
add logs when grep stops working, add traces when "which service is slow" is
not obvious.

---

## Metrics: Prometheus

### Why Prometheus

- Pull-based, so a misbehaving app can't DoS your metrics backend by spamming
- Service discovery built in (k8s, Consul, file_sd)
- PromQL is genuinely good once you learn it
- The de-facto open-source standard — every tool integrates

### The Prometheus mental model

```
   ┌──────────┐   ┌──────────┐
   │  app A   │   │  app B   │   each app exposes /metrics
   │ /metrics │   │ /metrics │   in Prometheus text format
   └─────┬────┘   └────┬─────┘
         │             │
         ▼             ▼
       ┌─────────────────┐
       │  Prometheus     │   pulls every <scrape_interval>
       │  (TSDB)         │   stores locally, evaluates rules
       └────────┬────────┘
                │
                ├──► Grafana   (dashboards + alerts UI)
                ├──► AlertManager  (routes alerts)
                └──► remote_write → long-term storage
                         (Mimir/Thanos/Cortex/Azure Monitor)
```

### Scrape config

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

Apps opt in via annotations:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "9090"
```

### Label cardinality — the killer

A Prometheus time series is uniquely identified by `metric_name{label1=v1,label2=v2,...}`.
Every unique combination is its own series, and series cost RAM and disk.

**Do:**

```
http_requests_total{method="GET",status="200",route="/orders/:id"}
```

**Don't:**

```
http_requests_total{method="GET",status="200",order_id="12345",user_id="abc-de-f"}
```

That second one creates a new series for every order and every user. On a
busy service, you'll OOM Prometheus inside a week.

Rule: **labels are for things with bounded, low cardinality** (status codes,
method, region, env, service). Anything with unbounded cardinality (user ID,
order ID, request ID, URL with IDs in the path) belongs in **logs or traces**,
not in label values.

If you need to know per-user behavior, use logs/traces. If you need to know
"how many users are doing X", use a metric with the dimension that matters
(`plan="free|paid"`), not the user ID.

### Recording rules

If a PromQL query is slow because it aggregates a lot, precompute it:

```yaml
groups:
  - name: orders.recording
    interval: 30s
    rules:
      - record: orders:requests:rate5m
        expr: sum(rate(http_requests_total{service="orders"}[5m])) by (status)
```

Then `orders:requests:rate5m` is fast everywhere — dashboards, alerts,
ad-hoc.

### Long-term storage

Prometheus's local TSDB is great for 15 days. Past that, push to something
durable via `remote_write`:

- **Mimir** (Grafana Labs) — most active OSS option
- **Thanos** — older, sidecar-based, store gateway pattern
- **Cortex** — the original; less active now
- **Azure Monitor managed Prometheus** — pay-as-you-go, no ops

For most teams, managed wins until cost matters. The metrics-bill spike
*always* surprises someone.

---

## Logs: ELK or Loki

### ELK (Elasticsearch / Logstash / Kibana)

- Full-text search on log content
- Powerful aggregations
- **Expensive** at scale — Elasticsearch is RAM-hungry, every field is
  indexed by default
- Mature, has a UI everyone knows (Kibana)

### Loki

- Pull from Grafana Labs, designed to look like Prometheus for logs
- **Only labels are indexed**, log content is stored as compressed chunks
- Query language (LogQL) is similar to PromQL — same labels, same selectors
- 5–10× cheaper than ELK at comparable volume
- **Weaker** if you need full-text search across years of logs

### How to pick

- "I need to search arbitrary text in years-old logs for compliance/forensics":
  ELK (or Splunk if you have the budget).
- "I mostly look at logs by service/env/level and grep within": Loki.
- "I'm already paying for Splunk Enterprise": Splunk. Don't fight it.
- "I just want logs in Azure Monitor / CloudWatch and it's good enough":
  it's good enough until it isn't. Plan the exit before you hit the bill.

### Log hygiene

Whatever backend you pick:

- **Structured logs (JSON).** Plain-text logs are a tax on your future self.
- **One log line = one event.** No multi-line stack traces glued by `\n`
  literals; use a structured `error.stack_trace` field.
- **Standard top-level fields:** `timestamp`, `level`, `service`, `env`,
  `trace_id`, `span_id`, `message`, plus app-specific keys.
- **`trace_id` everywhere.** It's the joining key between logs and traces.
- **Levels mean something.** `error` = "human needs to look at this".
  `warn` = "unusual, but handled". `info` = "noteworthy state change".
  `debug` = "off in prod". If everything is `error`, nothing is.

---

## Dashboards: Grafana

### Dashboards-as-code

Click-ops dashboards become unmaintainable. Two patterns that work:

**Provisioning** — store dashboards as JSON in git, mount them into Grafana:

```yaml
# grafana provisioning config
apiVersion: 1
providers:
  - name: 'dashboards'
    folder: 'Provisioned'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

The folder mounts from a ConfigMap or a git-sync sidecar.

**Grafana as code with Grizzly or grafonnet** — build dashboards from
templates, generate JSON in CI. Higher upfront cost, big win at scale.

Either way: dashboards in git → reviewed in PRs → no "who changed this?".

### Dashboard hierarchy

Three levels for a service:

1. **Service overview** — RED metrics (Rate, Errors, Duration), one panel per.
   Read this first when paged.
2. **Service drill-down** — broken down by route/endpoint/dependency. Read
   when overview points somewhere.
3. **Component-specific** — DB connections, queue depth, cache hit rate. Read
   when drill-down points at the component.

Don't put a 60-panel dashboard at the top.

### Alerting (Grafana unified alerting)

Three alert quality rules:

1. **Every alert links to a runbook.** No exceptions. If you can't write the
   runbook, the alert isn't ready.
2. **`for:` is non-zero.** `for: 5m` means "the condition must hold for 5
   minutes". Eliminates 90% of flapping alerts.
3. **Alert on symptoms, not causes.** "Error rate > 1%" is a symptom. "CPU
   > 80%" is a cause that might or might not be a problem. Page on the
   first; graph the second.

```yaml
- alert: OrdersAPIHighErrorRate
  expr: |
    (
      sum(rate(http_requests_total{service="orders",status=~"5.."}[5m]))
      /
      sum(rate(http_requests_total{service="orders"}[5m]))
    ) > 0.02
  for: 10m
  labels:
    severity: page
    team: orders
  annotations:
    summary: "Orders API error rate above 2% for 10m"
    runbook_url: "https://runbooks.example.com/orders/error-rate"
```

---

## A note on Splunk

If you're at a company already using Splunk, you're not migrating off — the
data gravity and re-training cost is real. What to know:

- It's a logs platform first, but does metrics and APM too. Splunk Infrastructure
  Monitoring (ex-SignalFx) is the metrics arm; Splunk Observability Cloud bundles
  metrics + logs + traces.
- SPL (Splunk's search language) is powerful and *very* different from PromQL/LogQL.
  Budget training time.
- Ingest is the cost driver. **Drop logs early** (in the forwarder/collector)
  rather than indexing everything and querying selectively.
- Splunk + Prometheus is a common shape: Prometheus for metrics, Splunk for
  logs. Grafana can query Splunk. Don't try to push high-cardinality metric
  data into Splunk — it's the wrong tool.

---

## Wiring it together on AKS

A reasonable default open-source stack on AKS:

```
┌─────────────────────────────────────────────────────────────┐
│ AKS cluster                                                  │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                │
│  │ apps     │   │ apps     │   │ apps     │  ◄── OTel SDKs │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                │
│       │              │              │                       │
│   metrics          logs           traces                    │
│       │              │              │                       │
│       ▼              ▼              ▼                       │
│  ┌──────────────────────────────────────┐                  │
│  │ OpenTelemetry Collector (DaemonSet)  │                  │
│  └────┬─────────────┬────────────┬──────┘                  │
│       │             │            │                          │
└───────┼─────────────┼────────────┼──────────────────────────┘
        ▼             ▼            ▼
   Prometheus      Loki         Tempo
   (or Mimir)      (or ELK)     (or Jaeger)
        │             │            │
        └─────────────┼────────────┘
                      ▼
                   Grafana
                      │
                      ▼
                AlertManager
                      │
                      ▼
                PagerDuty / Slack / email
```

- **OpenTelemetry SDK in apps** — single SDK exports metrics, logs, traces.
  Vendor-neutral.
- **OTel Collector as a DaemonSet** — receives everything, exports to the
  right backend, lets you swap backends without touching apps.
- **Grafana as the single pane** — three datasources, one UI.

---

## DR for the observability stack itself

The cluster is on fire and the dashboards are also down. Common because the
observability stack is often deployed *to* the cluster it watches.

Mitigations:

- **Keep at least one alert evaluator outside the cluster** — Grafana Cloud
  free tier, Azure Monitor alerts, anything. So you find out the cluster's
  down even when in-cluster Prometheus is down.
- **Replicate critical metrics off-cluster** via `remote_write` to a
  separate region or managed service.
- **Backups of dashboard JSON in git** (which they already are if you
  followed the "dashboards as code" rule).
- **Don't co-locate the alerting Slack/PagerDuty integration with the thing
  it monitors.** If your AlertManager is in the failing cluster, alerts
  never go out.

---

## The "we just spun up a new cluster, what observability do we ship?"
checklist

- [ ] Prometheus + AlertManager installed (kube-prometheus-stack Helm chart
      is the path of least resistance)
- [ ] Node exporter, kube-state-metrics scraped
- [ ] Grafana installed, dashboards provisioned from git
- [ ] Loki (or ELK) for logs, OTel Collector forwarding
- [ ] OTel SDK in app templates / starter repos
- [ ] One alert: "AlertManager not sending heartbeats" — wired to a
      different channel than everything else. (The dead-man's switch.)
- [ ] Service overview dashboards for the top 3 services on day 1
- [ ] Runbook URLs filled in (even if the runbook is "TODO")

---

## Related

- [Kubernetes (AKS) production checklist](./aks-production-checklist.md) — what you're watching
- [Terraform on Azure](./terraform-on-azure.md) — how the underlying infra is provisioned
- [CI/CD pipeline patterns](./cicd-pipeline-patterns.md) — observability for the pipelines themselves
- [PR-as-publish-gate](../automation-patterns/pr-as-publish-gate.md) — pattern for gating risky automated actions, useful for alert-driven runbooks
