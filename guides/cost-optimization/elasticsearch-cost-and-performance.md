# Elasticsearch cost and performance

**TL;DR:** Most Elasticsearch (and OpenSearch) clusters are 2-5x over-provisioned
because shard sizing, index lifecycle, and query patterns were never tuned
after the initial setup. The big wins: index lifecycle management (hot/warm/
cold/frozen), shard sizing math (50GB rule of thumb, not 1000 shards per node),
query optimization using filter context, and rollups for old data. The Elastic
Cloud vs OpenSearch decision matters more than most teams realize.

## The cost levers, in priority order

1. **Index lifecycle management (ILM).** Move old data to cheaper tiers
   instead of keeping it all on hot SSDs.
2. **Shard sizing.** Right-sizing shards reduces overhead per cluster.
3. **Query optimization.** Slow queries cost CPU; CPU drives cluster sizing.
4. **Rollups and downsampling.** Aggregate old metrics; keep only summaries.
5. **Index template hygiene.** Field mapping discipline reduces storage.
6. **Right-sizing nodes.** Match the cluster to the workload.

Most teams underuse 1 and 4, mis-size 2 and 6, and have at least a few of 3
that quietly burn CPU.

## ILM — the most impactful single change

Data isn't queried uniformly. Last hour's logs are queried constantly; last
year's are queried once a quarter. ILM matches storage tier to query
frequency:

| Tier | Storage | Use |
|---|---|---|
| **Hot** | Fast SSD (typically NVMe) | Active indexing + frequent queries |
| **Warm** | Slower SSD or HDD | Read-only, queried weekly |
| **Cold** | Object storage (S3, Azure Blob) with searchable snapshots | Queried rarely |
| **Frozen** | Object storage with partial mount | Queried very rarely; long retention |

Example lifecycle:

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_size": "50GB", "max_age": "7d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "s3-repo" }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

**Real-world impact:** moving 90% of data off hot tier reduces hot-tier
storage cost by 90%. The warm/cold/frozen tiers are far cheaper.

## Shard sizing — the math

Two competing concerns:

- **Each shard has overhead.** Cluster state grows with shard count; query
  fan-out hits every shard.
- **Each shard has a size limit.** Indexing throughput per shard caps;
  search latency on huge shards degrades.

**Rules of thumb:**

- **Target shard size: 20-50 GB.** Smaller shards = too many shards = high
  overhead. Larger shards = slow recovery and segment management.
- **One primary shard per write throughput unit.** If your index sustains
  20MB/s writes and one shard handles 5MB/s, you need ~4 primaries.
- **Heap-to-shard ratio: ~20 shards per GB of heap.** A node with 32GB heap
  shouldn't host >600 shards. Past that, shard overhead dominates.
- **Total shards in cluster:** keep under 10,000 for most use cases. Past
  that, master node performance degrades.

The "1000 shards per node" myth came from old defaults that no longer apply.
Modern guidance is the heap-based ratio.

### Time-series indices: use rollover, not pre-sized

For logs / metrics / time series, **don't pre-size shards**. Use ILM rollover
to a new index when:

- Size exceeds 50GB (typical).
- Age exceeds 7-30 days (typical).
- Doc count exceeds 200M (rare trigger).

Each new index can have a different shard count tuned to current ingest.
Old indices shrink to fewer shards via the ILM `shrink` action.

## Query optimization

CPU time on query execution drives node sizing. Slow queries = bigger
cluster = more cost.

### Filter context vs query context

The single biggest query optimization:

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } }      // scored (slow)
      ],
      "filter": [                                  // not scored (cached)
        { "term": { "service": "payments" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

- **Query context** (under `must`) scores documents — slow, not cached.
- **Filter context** (under `filter`) doesn't score — fast, cached.

Move every clause that doesn't need scoring into filter context. Sometimes
this alone halves query CPU.

### Other query wins

- **`_source` filtering** — don't return fields you don't need. Reduces
  network and deserialization.
- **`doc_values: true`** on aggregation fields (default for keyword,
  numeric). Without it, aggregations rebuild from scratch.
- **Limit `aggs` cardinality** — `terms` agg with `size: 1000` on a
  high-cardinality field is expensive. Use `composite` agg for pagination.
- **Profile slow queries** with `/_search?profile=true` — see what's
  expensive.
- **Avoid wildcard prefix searches** (`*foo`). They can't use the index
  efficiently. If common, add a reverse-indexed field.

### The Kibana / Grafana query trap

Dashboards run dozens of queries on refresh. A 30-second auto-refresh on a
heavy dashboard with 20 panels = ~40 queries per second per viewer.
Multiply by viewers. This is often the biggest CPU consumer.

Fixes:
- Reduce auto-refresh interval.
- Cache dashboard data (Grafana + Elasticsearch query cache).
- Pre-aggregate hot dashboard metrics (rollups or pre-computed indices).

## Rollups and downsampling

Old metrics rarely need per-second resolution. Aggregate to per-hour or
per-day:

```json
{
  "rollup_job": {
    "id": "metrics-daily",
    "index_pattern": "metrics-*",
    "rollup_index": "metrics-rollup-daily",
    "cron": "0 0 * * * ?",
    "page_size": 1000,
    "groups": {
      "date_histogram": { "field": "@timestamp", "interval": "1d" },
      "terms": { "fields": ["service", "host"] }
    },
    "metrics": [
      { "field": "request_count", "metrics": ["sum"] },
      { "field": "latency_ms", "metrics": ["avg", "p95", "max"] }
    ]
  }
}
```

The rolled-up index is tiny compared to the raw data. Queries against old
ranges hit the rollup instead.

**Elastic's newer "downsampling" feature** (8.x+) is the modern equivalent
and easier to configure.

## Index template hygiene

The biggest mapping mistakes:

- **`text` fields that should be `keyword`.** Text is analyzed and tokenized;
  keyword isn't. For exact-match fields (status codes, hostnames), use
  keyword.
- **`text` fields with no need for full-text search.** Use keyword instead;
  saves significant index size.
- **Dynamic mapping on JSON logs with arbitrary keys.** Each unique key
  becomes a field in the mapping. Some logs explode the mapping into
  thousands of fields, killing performance.
- **`_source` enabled when not needed.** If you only need fields for
  aggregation, `_source: false` saves substantial disk.
- **Long string fields with `keyword`.** Keyword fields cap at 32KB by
  default; longer values are silently truncated. Use a different mapping
  for long strings.

**Index templates** standardize mappings. Every index pattern should have a
template; no `dynamic: true` without thinking about field explosion.

## Right-sizing cluster nodes

| Role | What it does | Sizing dimension |
|---|---|---|
| **Master** | Cluster state | Small (4-8 vCPU, 8-16 GB RAM); dedicated |
| **Data (hot)** | Active indexing + queries | CPU + RAM + fast disk |
| **Data (warm/cold)** | Read-only | Less CPU + cheaper disk |
| **Ingest** | Pipeline processing | CPU-bound |
| **ML / Transform** | Specialized | Workload-dependent |

**Rules:**

- **Dedicated master nodes** for clusters > 5 data nodes. Mixed master/data
  nodes are fine for small clusters but problematic at scale.
- **Heap size: 50% of RAM, max 31 GB.** Past 31 GB, Java compressed-OOPs
  benefit disappears; smaller heaps are more efficient.
- **OS page cache matters.** Leave 50% of RAM for the OS to cache files.
- **NVMe for hot tier.** Substantial query latency win vs SATA SSD.
- **Warm tier on slower SSD or even HDD.** Costs much less.

## Elastic Cloud vs OpenSearch vs self-hosted

| Option | Pros | Cons |
|---|---|---|
| **Elastic Cloud (managed Elasticsearch)** | Latest features, vendor support | Most expensive |
| **AWS OpenSearch** | Cheaper, AWS-native | Fork; features lag Elastic |
| **Azure / GCP managed OpenSearch** | Cloud-native, managed | Fewer features than Elastic Cloud |
| **Self-hosted on Kubernetes** (ECK / Bitnami chart) | Full control, lowest cost | Operational overhead |
| **Self-hosted on VMs** | Old-school control | Manual scaling, manual upgrades |

The 2026 reality:

- **Elastic vs OpenSearch fork** — OpenSearch has caught up on most features.
  Elastic still leads on ML / vector search / new analytics features.
- **Vector search / AI features** — both have them now; Elastic's are more
  mature.
- **Pricing** — OpenSearch ~30-50% cheaper for equivalent capacity. The gap
  is shrinking as Elastic Cloud adds tiers.

**Decision shape:**
- Need vector search + ML + latest ES features → Elastic Cloud.
- Need search at AWS scale, OK with OpenSearch features → AWS OpenSearch.
- Have ops team, want max control → self-hosted with ECK.

## The "we're spending $X on ES, can we move?" question

Moving search clusters is genuinely expensive. The pragmatic order:

1. **Optimize the current cluster first** (ILM, query tuning, shard sizing).
   Often saves 30-50% with no migration.
2. **Reduce data retention** if business allows.
3. **Move only old data** to cheaper search-on-snapshot if available.
4. **Migrate cluster** only as a last resort.

Migration involves reindexing, query compatibility checks (especially if
moving Elasticsearch → OpenSearch), and dashboard rewrites. Plan months.

## Gotchas

- **Bulk indexing during cold-tier moves** — ILM transitions are I/O heavy.
  Schedule for low-traffic hours.
- **Searchable snapshots have query latency** that's much higher than hot
  tier. Don't put dashboards against cold data unless they're tolerant.
- **Shard recovery after node restart** can take hours for large indices.
  Plan rolling restarts carefully.
- **Aggregations on high-cardinality fields** — `service.name` with 50,000
  distinct values blows up memory.
- **Cluster yellow vs green** — yellow means primaries are healthy but
  some replicas aren't. Still functional, but no HA for affected shards.
- **Filebeat / Logstash drops** — when ingest is overwhelmed, data is
  silently dropped at the collector. Monitor backpressure.
- **The "let's just add more memory" reflex** — masks underlying issues
  (bad queries, too many shards) that get worse over time.

## When Elasticsearch isn't the right tool

- **Pure analytics on structured data** — ClickHouse, BigQuery, or
  Snowflake is usually cheaper and faster.
- **Pure logs** — Loki is much cheaper if you don't need full-text search
  across all fields.
- **Real-time metrics** — Prometheus + remote storage (Mimir, Thanos) is
  the right shape.
- **Tiny scale** (< 10GB data, < 100 QPS) — managed search is overkill;
  Postgres full-text search or even SQLite FTS5 works.

Elasticsearch is best when you genuinely need full-text search + analytics
+ aggregations across moderate-to-large data. For other workloads, cheaper
tools exist.

## Related

- [`right-sizing-and-savings-plans.md`](./right-sizing-and-savings-plans.md)
  — general right-sizing applies to ES data nodes.
- [`finops-tagging-and-showback.md`](./finops-tagging-and-showback.md) —
  ES costs surface in per-team dashboards.
- [`../cloud-platform/observability-stack.md`](../cloud-platform/observability-stack.md)
  — ELK vs Loki trade-off; observability spend overlap.
- [`../iac/terraform-module-design.md`](../iac/terraform-module-design.md)
  — modules for ES clusters with sane defaults.
