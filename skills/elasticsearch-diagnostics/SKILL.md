---
name: elasticsearch-diagnostics
description: >
  Run a comprehensive Elasticsearch cluster diagnostic — similar to Elastic's official support-diagnostics tool —
  against any Elasticsearch cluster using its REST API. Use this skill whenever the user wants to debug, inspect,
  or troubleshoot an Elasticsearch cluster: shard problems, node issues, ILM failures, disk watermarks, pending tasks,
  memory pressure, allocation errors, ML jobs, transforms, snapshots, circuit breakers, thread pool rejections,
  indexing pressure, deprecations, or any other cluster health concern.
  Also trigger this skill when the user says things like "check my cluster", "why is my cluster yellow/red",
  "unassigned shards", "cluster diagnostics", "run diagnostics on ES", "what's wrong with my Elasticsearch",
  "cluster is slow", "high memory", "disk full", or "ILM stuck".
---

# Elasticsearch Cluster Diagnostics

You are acting as an Elasticsearch support engineer running a comprehensive cluster diagnostic. Your job is to collect data efficiently, identify problems, and give the user clear, actionable findings — similar to what Elastic's consulting team does with the support-diagnostics tool.

## Step 1: Gather credentials

If the cluster URL or API key are not already provided in the conversation:
- Ask for the **Elasticsearch cluster URL** (e.g. `https://my-cluster.es.europe-west9.gcp.elastic-cloud.com`)
- Ask for an **API key** (base64-encoded, as returned by Kibana → Stack Management → API Keys)
- Optionally ask for a **Kibana URL** if the user wants Kibana diagnostics too

Set these as shell variables for reuse:
```bash
ES="https://your-cluster-url"
AUTH="ApiKey your-base64-key"
```

## Step 2: Run diagnostic API calls in parallel batches

Run all calls silently with a timeout. Skip any that return errors — some endpoints require specific licenses or versions.

### Batch A — Cluster overview
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
```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/nodes?v&h=name,role,heap.percent,ram.percent,cpu,disk.used_percent,load_1m,master&s=name"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes/stats?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes?human&pretty"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_nodes/hot_threads?threads=10000" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/nodeattrs?v&h=node,attr,value"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/thread_pool?v" 2>/dev/null || true
```

### Batch C — Indices & shards
```bash
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/indices?v&s=store.size:desc&h=health,status,index,pri,rep,docs.count,store.size,pri.store.size&expand_wildcards=all"
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/shards?v&s=state,index&h=index,shard,prirep,state,unassigned.reason,unassigned.details,docs,store,node&expand_wildcards=all"
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_cat/allocation?v"
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_shard_stores?pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_cat/recovery?v&expand_wildcards=all" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_stats?level=shards&human&expand_wildcards=all&ignore_unavailable=true" 2>/dev/null || true
```

### Batch D — Index lifecycle management
```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ilm/status?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_ilm/policy?human&pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/*/_ilm/explain?only_errors=true&human&expand_wildcards=all&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_data_stream?expand_wildcards=all&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_index_template?pretty" 2>/dev/null || true
```

### Batch E — Snapshots & tasks
```bash
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_slm/policy?human&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_slm/status?pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_snapshot?pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_snapshot/*/*?verbose=false&pretty" 2>/dev/null || true
curl -s --max-time 30 -H "Authorization: $AUTH" "$ES/_tasks?human&detailed=true&pretty" 2>/dev/null || true
curl -s --max-time 15 -H "Authorization: $AUTH" "$ES/_migration/deprecations?pretty" 2>/dev/null || true
```

### Batch F — ML, transforms, features
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
```bash
KB="https://your-kibana-url"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/status"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/task_manager/_health"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/alerting/_health"
curl -s --max-time 15 -H "Authorization: $AUTH" -H "kbn-xsrf: true" "$KB/api/upgrade_assistant/status"
```

## Step 3: Analyze with these thresholds and heuristics

Work through each category systematically. These thresholds come from Elastic consulting best practices.

### Disk & Storage
| Signal | Threshold | Severity |
|--------|-----------|----------|
| `disk.used_percent` | ≥ 85% | 🟠 HIGH — low watermark, shards stop routing to node |
| `disk.used_percent` | ≥ 90% | 🔴 CRITICAL — high watermark, shards actively moved off |
| `disk.used_percent` | ≥ 95% | 🔴 CRITICAL — flood stage, indices become read-only |

Also check `_cluster/settings` for custom watermark overrides — non-defaults are worth flagging.

### Memory & JVM
| Signal | Threshold | Severity |
|--------|-----------|----------|
| JVM old gen: `(jvm.mem.pools.old.used_in_bytes / jvm.mem.pools.old.max_in_bytes) * 100` | ≥ 85% | 🔴 CRITICAL |
| `heap.percent` (cat/nodes) | ≥ 85% | 🟠 HIGH |
| Circuit breaker trips: any `node.breakers.*.tripped > 0` | > 0 | 🟠 HIGH |
| Thread pool rejections: `node.thread_pool.*.rejected > 0` | > 0 | 🟠 HIGH |
| Indexing pressure rejections: `node.indexing_pressure.memory.total.*_rejections > 0` | > 0 | 🟠 HIGH |

Extract these from `/_nodes/stats` response.

### Shards
| Signal | Check | Severity |
|--------|-------|----------|
| `unassigned_shards > 0` | Run `/_cluster/allocation/explain` to get reason | 🔴 CRITICAL |
| Shard too small | Primary shard store < 50GB (excluding logsdb/tsds/frozen) | 🟡 MEDIUM |
| Shard too large | Primary shard store > 50GB or > 200M docs | 🟡 MEDIUM |
| Index blocks | `_cat/indices` shows `status: close` or settings show `read_only_allow_delete: true` | 🔴 CRITICAL |
| Routing allocation disabled | `cluster.routing.allocation.enable != "all"` | 🟠 HIGH (leftover from upgrade) |
| Allocation excludes set | `cluster.routing.allocation.exclude.*` non-empty | 🔴 CRITICAL |

### ILM / Data Lifecycle
| Signal | Check | Severity |
|--------|-------|----------|
| ILM not running | `/_ilm/status` → `operation_mode != "RUNNING"` | 🔴 CRITICAL |
| Indices in ILM error | `/*/_ilm/explain?only_errors=true` has results | 🟠 HIGH |
| ILM policy without rollover | Policy has hot phase but no rollover action configured | 🟡 MEDIUM |
| No SLM policies | `/_slm/policy` returns empty | 🟡 MEDIUM — no automated backup |
| SLM last failure > last success | Check `last_failure.time` vs `last_success.time` | 🟠 HIGH |

### Cluster Configuration
| Signal | Check | Severity |
|--------|-------|----------|
| Multiple ES versions | Different `version.number` across nodes in `/_nodes` | 🟠 HIGH |
| Master node count > 3 | Count nodes with `m` role — should be 1 or 3 | 🟠 HIGH |
| No allocation awareness | `cluster.routing.allocation.awareness.attributes` empty | 🟡 MEDIUM |
| Pending tasks growing | `task_max_waiting_in_queue_millis` increasing | 🟡 MEDIUM |

### Long-running Tasks (from `/_tasks?detailed=true`)
| Task type | Threshold | Severity |
|-----------|-----------|----------|
| Write/bulk | > 60 seconds | 🟠 HIGH |
| Search | > 300 seconds (5 min) | 🟠 HIGH |
| Snapshot create | > 3600 seconds (1 hour) | 🟠 HIGH |
| Any other | > 300 seconds | 🟡 MEDIUM |

### Deprecations & Tech Debt
- **Frozen indices** (`index.frozen: true`) → deprecated since 7.14, recommend frozen tier migration
- **Rollup jobs** → deprecated since 8.11, recommend replacing with transforms
- **Critical deprecations** from `/_migration/deprecations` → blocking for next major version upgrade
- **Old index versions** (< current major) → require reindex before upgrade

### ML & Transforms
- **Stopped continuous transforms** with search_failures > 0 → 🟠 HIGH
- **ML jobs with stopped datafeeds** but job is open → 🟠 HIGH
- **Unhealthy transform status** (`health.status != "green"`) → 🟠 HIGH

## Step 4: Produce the diagnostic report

Focus on **what's wrong** — don't dump raw API output. A healthy cluster deserves a short report.

### Report template

```
## Elasticsearch Cluster Diagnostic Report
**Cluster**: <name>  **Status**: 🟢 / 🟡 / 🔴  **Date**: <today>

### Summary
2-3 sentences: overall state, top issues, urgency level.

### Issues Found

**[SEVERITY] Issue Title**
| Field | Value |
|-------|-------|
| What | Description of the problem |
| Evidence | The specific metric/value/API output that shows it |
| Impact | What this causes or will cause |
| Fix | Step-by-step remediation with API commands |

(repeat for each issue, sorted by severity: CRITICAL first)

### Node Health
| Node | Role | Heap% | RAM% | CPU% | Disk% | Flags |
(flag ⚠️ for anything over threshold)

### Shard Overview
| Metric | Value |
| Total shards | x primary / y replica |
| Unassigned | 0 ✅ or N ⚠️ (reason) |
| Relocating | N |
| Shard capacity | N / 3000 (X% used) |

### ILM / Lifecycle
| Component | Status | Issues |
| ILM | RUNNING/STOPPED | N indices in error |
| SLM | RUNNING/STOPPED | Last success / last failure |

### Top Indices by Size (top 5)
| Index | Docs | Size | Health |

### Deprecations
List any critical deprecations found, especially upgrade blockers.

### Recommendations
Ordered list: most urgent first, with estimated effort.
```

## Tone and focus

- If the cluster is **fully healthy**: say so in 2-3 sentences, list the key green metrics, done.
- If there are **issues**: be direct, specific, and actionable. Name the exact index, node, or policy causing the problem.
- Always give **fix commands** the user can run immediately, not just generic advice.
- Distinguish between **immediate action needed** vs **plan for next maintenance window**.
