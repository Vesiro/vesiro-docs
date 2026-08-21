# VesiroSearch

**A drop-in native search engine for Elasticsearch.**

VesiroSearch is a plugin that intercepts the query phase and executes it in a
purpose-built C++ search engine instead of the standard Java/Lucene pipeline.
We also offer a new index data structure that can increase performance even further.
Your indices, mappings, queries, and clients stay exactly the same. Only the
hot path changes.

- **Drop-in.** No reindexing, no query rewrites, no client changes. Install the plugin and restart.
- **Transparent fallback.** Anything VesiroSearch can't execute natively is silently handed back to the standard query phase, so a request never fails because of the plugin.
- **Same semantics.** Scoring, ordering, and aggregation results are designed to match the engine you're already running.

---

## Table of contents

- [Compatibility](#compatibility)
- [Installation](#installation)
- [Verifying the install](#verifying-the-install)
- [Configuration](#configuration)
- [Per-request control](#per-request-control)
- [Performance tuning](#performance-tuning)
- [Elasticsearch and system configuration](#elasticsearch-and-system-configuration)
- [Transparent fallback](#transparent-fallback)

---

## Compatibility

| | Supported |
| --- | --- |
| **Elasticsearch** | All 8.x versions and 7.17 |
| **Operating system** | Linux, Windows, macOS |
| **Architecture** | x86_64 and ARM64 |

---

## Installation

### 1. Install the plugin

VesiroSearch ships as a standard plugin zip. Install it on **every node** in
the cluster:

```bash
# Elasticsearch
bin/elasticsearch-plugin install file:///path/to/vesiro-<version>.zip

### 2. Restart the node

```bash
systemctl restart elasticsearch   # or however you manage the service
```

Perform a normal rolling restart. Nodes without the plugin and nodes with it
can coexist during the rollout; nodes without it simply serve searches through
the standard query phase.

---

## Verifying the install

VesiroSearch exposes a single informational endpoint:

```bash
curl -s localhost:9200/_vesiro
```

```json
{
  "vsl_version": "0.1.0"
}
```

A successful response means the native library loaded and is answering calls.
If the endpoint 404s, the plugin isn't installed or the node didn't restart.

---

## Configuration

All settings live under the `vesiro.*` namespace and go in
`elasticsearch.yml`.

| Setting | Type | Default | Scope | Description |
| --- | --- | --- | --- | --- |
| `vesiro.enabled` | boolean | `true` | dynamic | Master switch. When `false`, all searches use the standard query phase. |
| `vesiro.warn` | boolean | `true` | dynamic | Add a response `Warning` header when a request falls back or the native engine is unavailable. |
| `vesiro.memory_mode` | `IO_BOUND` \| `REGULAR` | `IO_BOUND` | node-static | What the engine optimizes for: nodes whose I/O is under stress, or hot nodes with plenty of RAM. See [Performance tuning](#performance-tuning). |
| `vesiro.max_clause_count` | integer | `-1` | node-static | Override the boolean clause limit. `-1` keeps the engine default. |
| `vesiro.telemetry` | boolean | `true` | dynamic | Send native crash and fallback telemetry to Vesiro. |

---

## Per-request control

Use the `vesiro` search extension to override behavior for a single request.
This is useful for A/B comparisons and for isolating a suspect query.

Disable VesiroSearch for a single request:

```json
GET /my-index/_search
{
  "query": { "match": { "title": "search engine" } },
  "ext": { "vesiro": false }
}
```

Because the extension is per-request, the cleanest way to validate a
migration is to run the same query twice, once with `"vesiro": true` and once
with `"vesiro": false`, then diff the hits.

---

## Performance tuning

### `vesiro.memory_mode`

Selects what VesiroSearch optimizes for on the node.

| Value | When to use |
| --- | --- |
| `IO_BOUND` *(default)* | I/O-bound nodes: the index is much larger than available RAM and searches wait on storage. |
| `REGULAR` | Hot nodes with plenty of RAM, where the index or its hot portion is served from page cache. |

The default, `IO_BOUND`, optimizes for systems whose I/O is under stress.
`REGULAR` optimizes for hot systems with plenty of RAM.

### The Vesiro codec

VesiroSearch ships its own index codec, the **Vesiro codec**, which delivers
better performance than the default Elasticsearch codec.

Because a codec determines how segments are written on disk, enabling or
disabling it is a rollout decision rather than a switch you flip mid-flight.
Opting an index in is covered under *Optional codec adoption for full
performance*, and the exit paths under *Rolling back*, in:
<https://www.vesiro.com/blog/safely-installing-and-rolling-back-vesirosearch>

### Benchmarks

Benchmark results and performance write-ups are published on the Vesiro blog:
<https://www.vesiro.com/blog?category=performance>

---

## Elasticsearch and system configuration

This section documents Elasticsearch and operating system settings that have a
significant impact on VesiroSearch performance. Most of these recommendations
also apply to vanilla Elasticsearch. VesiroSearch mainly changes where the
trade-offs land, because most search execution happens in native code outside
the JVM heap.

### Java heap

The JVM heap size is one of the most important settings when running
Elasticsearch.

The heap size is fixed at startup. If it is configured too large, it consumes
memory that could otherwise be used by the operating system's page cache. For
large indices that are I/O-bound, the page cache is critical for performance,
and allocating too much memory to the JVM can significantly reduce search
throughput.

In some workloads, reducing the heap size by as little as 10 GB has resulted in
2x higher search performance, purely from the additional page cache made
available to the operating system.

VesiroSearch generally requires less JVM heap than vanilla Elasticsearch,
because most of the search execution takes place in native C++ code, with the
majority of native objects allocated outside the JVM heap. This makes the
heap-versus-page-cache trade-off more favorable than on a vanilla node: heap
you reclaim from the JVM goes almost entirely into caching index data.

Configuring the heap: modify the `jvm.options` file:

```
-Xms8g
-Xmx8g
```

### Maximum query clauses

One side effect of reducing the JVM heap is that Elasticsearch automatically
adjusts its maximum allowed number of query clauses based on the available
heap.

This only becomes a problem for extremely large queries. Most workloads never
come close to the limit, so if your queries are of an ordinary size you can
skip this section entirely. It matters when you want a smaller heap for better
page cache utilization but still need to execute Boolean queries with very large
numbers of clauses, typically queries generated programmatically, where a term
or filter list expands into thousands of clauses.

In that case, VesiroSearch provides the following setting to override Elasticsearch's
automatically calculated limit:

```yaml
vesiro.max_clause_count: 8192
```

This allows the heap size to be tuned independently of the maximum clause
count. The setting is node-static, so it takes effect on restart; the default
of `-1` keeps whatever limit the engine calculated.

### Swap

Elasticsearch recommends disabling swap, and this is also recommended when
running VesiroSearch.

When swap is enabled, the operating system may move infrequently used memory
pages to disk. If Elasticsearch later needs those pages, search latency can
increase dramatically while they are read back into memory.

Disable swap with:

```bash
sudo swapoff -a
```

---

## Transparent fallback

VesiroSearch is designed so that **a request never fails because of the
plugin**. If the plugin cannot handle a request, whether an unsupported query
construct, a feature not yet ported, or an unavailable native engine, it logs
a warning and lets the standard query phase run instead.

When `vesiro.warn` is enabled (the default), those requests also carry a
response header:

```
Warning: 299 Elasticsearch-8.17.5 "Request not supported in VesiroSearch. Fallback triggered: ..."
```

This makes fallbacks observable rather than silent. Watch for them during
rollout: a query that always falls back gets no benefit from the plugin, and
the header tells you why.

Fallback also applies to unexpected native errors: the error is logged and the
request is retried through the standard path.

---

## Support

Found a bug? Open an issue in this repository:
<https://github.com/Vesiro/vesiro-docs/issues>

For anything else, including licensing and commercial enquiries, contact
<info@vesiro.com>.
