# Elasticsearch Diagnostics — API Reference

This file documents all REST API calls used during a cluster diagnostic, organized by category, along with analysis thresholds derived from Elastic consulting best practices.

---

## Diagnostic API Calls

Run all calls with a timeout. Skip any that return errors — some endpoints require specific licenses or versions.

```bash
ES="https://your-cluster-url"
AUTH="ApiKey your-base64-key"
```

### Batch A — Cluster overview

| Endpoint | Purpose |
|----------|---------|
| `GET /_cluster/health` | Overall cluster health (status, shard counts, pending tasks) |
| `GET /_cluster/stats?human` | Aggregate stats across all nodes and indices |
| `GET /_cluster/settings?flat_settings&include_defaults` | All persistent, transient, and default settings |
| `GET /_cluster/pending_tasks?human` | Tasks queued on the master node |
| `GET /_health_report` | Structured health report (8.7+) |
| `GET /_cluster/allocation/explain` | Explains why the first unassigned shard is unassigned |
| `GET /_cluster/allocation/explain?include_disk_info=true` | Same, with disk-level detail |

```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/health?pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/stats?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/settings?flat_settings&include_defaults&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/pending_tasks?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_health_report?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/allocation/explain?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cluster/allocation/explain?include_disk_info=true&pretty" 2>/dev/null || true
```

### Batch B — Nodes (detailed)

| Endpoint | Purpose |
|----------|---------|
| `GET /_cat/nodes?v` | Compact node summary with heap, RAM, CPU, disk |
| `GET /_nodes/stats?human` | Full node-level stats (JVM, thread pools, indexing pressure, breakers) |
| `GET /_nodes?human` | Node metadata: versions, roles, attributes |
| `GET /_nodes/hot_threads` | Stack traces for busy threads |
| `GET /_cat/nodeattrs?v` | Node allocation attributes |
| `GET /_cat/thread_pool?v` | Thread pool queue/rejected counts per node |

```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/nodes?v&h=name,role,heap.percent,ram.percent,cpu,disk.used_percent,load_1m,master&s=name"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes/stats?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes/hot_threads?threads=10000" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/nodeattrs?v&h=node,attr,value"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/thread_pool?v" 2>/dev/null || true
```

### Batch C — Indices & shards

| Endpoint | Purpose |
|----------|---------|
| `GET /_cat/indices?v` | Index health, doc count, store size |
| `GET /_cat/shards?v` | Per-shard state, unassigned reasons |
| `GET /_cat/allocation?v` | Disk usage and shard count per node |
| `GET /_shard_stores` | Store status for shards (corrupted, available copies) |
| `GET /_cat/recovery?v` | Active and recent shard recoveries |
| `GET /_stats?level=shards` | Full index/shard-level stats |

```bash
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/indices?v&s=store.size:desc&h=health,status,index,pri,rep,docs.count,store.size,pri.store.size&expand_wildcards=all"
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/shards?v&s=state,index&h=index,shard,prirep,state,unassigned.reason,unassigned.details,docs,store,node&expand_wildcards=all"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/allocation?v"
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_shard_stores?pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/recovery?v&expand_wildcards=all" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_stats?level=shards&human&expand_wildcards=all&ignore_unavailable=true" 2>/dev/null || true
```

### Batch D — Index lifecycle management

| Endpoint | Purpose |
|----------|---------|
| `GET /_ilm/status` | ILM operation mode (RUNNING / STOPPING / STOPPED) |
| `GET /_ilm/policy` | All ILM policies with configuration |
| `GET /*/_ilm/explain?only_errors=true` | Indices stuck in ILM error state |
| `GET /_data_stream` | Data stream definitions and backing indices |
| `GET /_index_template` | All composable index templates |

```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ilm/status?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ilm/policy?human&pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/*/_ilm/explain?only_errors=true&human&expand_wildcards=all&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_data_stream?expand_wildcards=all&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_index_template?pretty" 2>/dev/null || true
```

### Batch E — Snapshots & tasks

| Endpoint | Purpose |
|----------|---------|
| `GET /_slm/policy` | Snapshot lifecycle policies and last run status |
| `GET /_slm/status` | SLM operation mode |
| `GET /_snapshot` | Registered snapshot repositories |
| `GET /_snapshot/*/*?verbose=false` | All snapshots across all repositories |
| `GET /_tasks?detailed=true` | All running tasks with elapsed time |
| `GET /_migration/deprecations` | Deprecations blocking the next major upgrade |

```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_slm/policy?human&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_slm/status?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_snapshot?pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_snapshot/*/*?verbose=false&pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_tasks?human&detailed=true&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_migration/deprecations?pretty" 2>/dev/null || true
```

### Batch F — ML, transforms, features

| Endpoint | Purpose |
|----------|---------|
| `GET /_ml/anomaly_detectors/_stats` | ML anomaly detection job states |
| `GET /_ml/datafeeds/_stats` | Datafeed states for ML jobs |
| `GET /_transform/_stats` | Transform health and failure counts |
| `GET /_enrich/policy` | Enrich policies |
| `GET /_watcher/stats/_all` | Watcher execution stats |
| `GET /_license` | License type and expiry |
| `GET /_xpack/usage` | Feature usage summary |

```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ml/anomaly_detectors/_stats?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ml/datafeeds/_stats?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_transform/_stats?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_enrich/policy?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_watcher/stats/_all?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_license?pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_xpack/usage?human&pretty" 2>/dev/null || true
```

### Batch G — Kibana (only if Kibana URL provided)

| Endpoint | Purpose |
|----------|---------|
| `GET /api/status` | Kibana overall status |
| `GET /api/task_manager/_health` | Task Manager health and drift |
| `GET /api/alerting/_health` | Alerting framework health |
| `GET /api/upgrade_assistant/status` | Upgrade readiness check |

```bash
KB="https://your-kibana-url"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/status"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/task_manager/_health"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/alerting/_health"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/upgrade_assistant/status"
```

---

## Analysis Thresholds and Heuristics

These thresholds come from Elastic consulting best practices.

### Disk & Storage

| Signal | Threshold | Severity |
|--------|-----------|----------|
| `disk.used_percent` | ≥ 85% | HIGH — low watermark, shards stop routing to node |
| `disk.used_percent` | ≥ 90% | CRITICAL — high watermark, shards actively moved off |
| `disk.used_percent` | ≥ 95% | CRITICAL — flood stage, indices become read-only |

Also check `_cluster/settings` for custom watermark overrides — non-defaults are worth flagging.

### Memory & JVM

| Signal | Threshold | Severity |
|--------|-----------|----------|
| JVM old gen: `(jvm.mem.pools.old.used_in_bytes / jvm.mem.pools.old.max_in_bytes) * 100` | ≥ 85% | CRITICAL |
| `heap.percent` (cat/nodes) | ≥ 85% | HIGH |
| Circuit breaker trips: any `node.breakers.*.tripped > 0` | > 0 | HIGH |
| Thread pool rejections: `node.thread_pool.*.rejected > 0` | > 0 | HIGH |
| Indexing pressure rejections: `node.indexing_pressure.memory.total.*_rejections > 0` | > 0 | HIGH |

Extract these from `/_nodes/stats` response.

### Shards

| Signal | Check | Severity |
|--------|-------|----------|
| `unassigned_shards > 0` | Run `/_cluster/allocation/explain` to get reason | CRITICAL |
| Shard too small | Primary shard store < 50GB (excluding logsdb/tsds/frozen) | MEDIUM |
| Shard too large | Primary shard store > 50GB or > 200M docs | MEDIUM |
| Index blocks | `_cat/indices` shows `status: close` or settings show `read_only_allow_delete: true` | CRITICAL |
| Routing allocation disabled | `cluster.routing.allocation.enable != "all"` | HIGH (leftover from upgrade) |
| Allocation excludes set | `cluster.routing.allocation.exclude.*` non-empty | CRITICAL |

### ILM / Data Lifecycle

| Signal | Check | Severity |
|--------|-------|----------|
| ILM not running | `/_ilm/status` → `operation_mode != "RUNNING"` | CRITICAL |
| Indices in ILM error | `/*/_ilm/explain?only_errors=true` has results | HIGH |
| ILM policy without rollover | Policy has hot phase but no rollover action configured | MEDIUM |
| No SLM policies | `/_slm/policy` returns empty | MEDIUM — no automated backup |
| SLM last failure > last success | Check `last_failure.time` vs `last_success.time` | HIGH |

### Cluster Configuration

| Signal | Check | Severity |
|--------|-------|----------|
| Multiple ES versions | Different `version.number` across nodes in `/_nodes` | HIGH |
| Master node count > 3 | Count nodes with `m` role — should be 1 or 3 | HIGH |
| No allocation awareness | `cluster.routing.allocation.awareness.attributes` empty | MEDIUM |
| Pending tasks growing | `task_max_waiting_in_queue_millis` increasing | MEDIUM |

### Long-running Tasks (from `/_tasks?detailed=true`)

| Task type | Threshold | Severity |
|-----------|-----------|----------|
| Write/bulk | > 60 seconds | HIGH |
| Search | > 300 seconds (5 min) | HIGH |
| Snapshot create | > 3600 seconds (1 hour) | HIGH |
| Any other | > 300 seconds | MEDIUM |

### Deprecations & Tech Debt

| Finding | Action |
|---------|--------|
| Frozen indices (`index.frozen: true`) | Deprecated since 7.14 — recommend frozen tier migration |
| Rollup jobs | Deprecated since 8.11 — recommend replacing with transforms |
| Critical deprecations from `/_migration/deprecations` | Blocking for next major version upgrade |
| Old index versions (< current major) | Require reindex before upgrade |

### ML & Transforms

| Finding | Severity |
|---------|----------|
| Stopped continuous transforms with `search_failures > 0` | HIGH |
| ML jobs with stopped datafeeds but job is open | HIGH |
| Unhealthy transform status (`health.status != "green"`) | HIGH |
