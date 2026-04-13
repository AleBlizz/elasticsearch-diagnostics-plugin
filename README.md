# Elasticsearch Diagnostics Plugin for Claude Code

A Claude Code plugin that runs comprehensive Elasticsearch cluster diagnostics — similar to Elastic's official [support-diagnostics tool](https://github.com/elastic/support-diagnostics) — directly from your terminal.

## What it does

Connects to any Elasticsearch cluster via REST API and produces a structured diagnostic report covering:

- **Cluster health** (green/yellow/red, shard counts, pending tasks)
- **Node health** (heap %, RAM %, CPU, disk usage with watermark thresholds)
- **Shard problems** (unassigned shards with allocation explain, shard sizing)
- **ILM issues** (stuck indices, missing policies, policies without rollover)
- **Disk pressure** (flags at 85% / 90% / 95% watermarks)
- **Memory pressure** (JVM old gen ≥ 85%, circuit breaker trips, thread pool rejections)
- **Snapshot status** (SLM policy health, last success/failure)
- **Transforms & ML** (stopped transforms, unhealthy jobs)
- **Upgrade blockers** (indices incompatible with next major version)
- **Deprecations** (frozen indices, rollup jobs, deprecated settings)

## Installation

```
/plugin marketplace add your-org/elasticsearch-diagnostics-plugin
```

Then install the plugin:
```
/plugin install elasticsearch-diagnostics
```

## Usage

Just describe your cluster issue naturally — Claude will trigger the diagnostic automatically:

> "My Elasticsearch cluster seems slow, can you check it?"
> "Why are there unassigned shards?"
> "Run a full diagnostic on my cluster"

Or invoke it explicitly:
```
/elasticsearch-diagnostics
```

You'll be asked for your cluster URL and API key (base64-encoded, from Kibana → Stack Management → API Keys).

## Thresholds used

Based on Elastic consulting best practices:

| Metric | Warning | Critical |
|--------|---------|----------|
| Disk usage | ≥ 85% (low watermark) | ≥ 90% (high) / ≥ 95% (flood) |
| JVM old gen heap | — | ≥ 85% |
| Long-running writes | — | > 60s |
| Long-running searches | — | > 300s |
| Long-running snapshots | — | > 1h |

## Updating

```
/plugin update elasticsearch-diagnostics
```

## License

Apache 2.0
