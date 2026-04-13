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

Collect data across seven batches covering: cluster overview, nodes, indices & shards, ILM, snapshots & tasks, ML/transforms, and Kibana. For the full list of endpoints and their curl commands, see [references/api-reference.md](references/api-reference.md).

**Batches to run:**
- **A** — Cluster overview (`/_cluster/health`, `/_cluster/stats`, `/_cluster/settings`, `/_health_report`, `/_cluster/allocation/explain`)
- **B** — Nodes: `/_cat/nodes`, `/_nodes/stats`, `/_nodes/hot_threads`, `/_cat/thread_pool`
- **C** — Indices & shards: `/_cat/indices`, `/_cat/shards`, `/_cat/allocation`, `/_shard_stores`, `/_cat/recovery`
- **D** — ILM: `/_ilm/status`, `/_ilm/policy`, `/*/_ilm/explain?only_errors=true`, `/_data_stream`, `/_index_template`
- **E** — Snapshots & tasks: `/_slm/policy`, `/_snapshot`, `/_tasks?detailed=true`, `/_migration/deprecations`
- **F** — ML & transforms: `/_ml/anomaly_detectors/_stats`, `/_transform/_stats`, `/_license`, `/_xpack/usage`
- **G** — Kibana (optional): `/api/status`, `/api/task_manager/_health`, `/api/alerting/_health`

## Step 3: Analyze with thresholds and heuristics

Work through each category systematically using the thresholds in [references/api-reference.md](references/api-reference.md). Key areas to cover:

- **Disk & Storage** — low/high/flood-stage watermarks (85% / 90% / 95%)
- **Memory & JVM** — old-gen heap ≥ 85%, circuit breaker trips, thread pool rejections, indexing pressure
- **Shards** — unassigned shards, shard size (< 1GB or > 50GB), index blocks, allocation excludes
- **ILM / Data Lifecycle** — ILM not RUNNING, indices in error, missing SLM policies
- **Cluster Configuration** — mixed ES versions, master node count, allocation awareness
- **Long-running Tasks** — bulk > 60s, search > 300s, snapshot > 1h
- **Deprecations** — frozen indices, rollup jobs, critical migration deprecations
- **ML & Transforms** — stopped transforms with failures, stopped datafeeds on open jobs

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
