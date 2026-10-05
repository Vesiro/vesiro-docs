# VesiroSearch

**A drop-in native search engine for Elasticsearch.**

VesiroSearch is a plugin that intercepts the query phase and executes it in a purpose-built C++ search engine instead of the standard Java/Lucene pipeline. We also offer a new index data structure that can increase performance even further. Your indices, mappings, queries, and clients stay exactly the same. Only the hot path changes.

- **Drop-in.** No reindexing, no query rewrites, no client changes. Install the plugin and restart.
- **Transparent fallback.** Anything VesiroSearch can't execute natively is silently handed back to the standard query phase, so a request never fails because of the plugin.
- **Same semantics.** Scoring, ordering, and aggregation results are designed to match the engine you're already running.

---

## Table of contents

- [Compatibility](#compatibility)
- [Getting started](#getting-started)
  - [Installation](#installation)
  - [Verifying the install](#verifying-the-install)
- [Advanced settings](#advanced-settings)
- [Transparent fallback](#transparent-fallback)
- [Troubleshooting](#troubleshooting)
- [Support](#support)

---

## Compatibility

| | Supported |
| --- | --- |
| **Elasticsearch** | 8.13 – 8.17 |
| **Operating system** | Linux |
| **Architecture** | x86_64 |

---

## Getting started

### Installation

#### 1. Create an account and download the plugin

Create a Vesiro account at <DOWNLOAD_URL>. Once signed in, download the plugin zip that matches your exact Elasticsearch version.

#### 2. Install the plugin on a node

VesiroSearch ships as a standard plugin zip. It doesn't have to be on every node in the cluster, but only nodes with the plugin installed run searches through VesiroSearch.

Copy the zip to the node, then run this from the Elasticsearch home directory:

```bash
bin/elasticsearch-plugin install file:///path/to/vesiro-<version>.zip
```

The installer asks you to confirm the extra permissions the plugin needs. Answer `y`, or pass `--batch` to skip the prompt.

#### 3. Restart the node

```bash
systemctl restart elasticsearch   # or however you manage the service
```

Repeat steps 2 and 3 on each node where you want the speedup, one node at a time. Nodes with and without the plugin can run side by side in the same cluster; nodes without it serve searches through the standard query phase.

### Verifying the install

VesiroSearch exposes a single informational endpoint:

```bash
curl -s localhost:9200/_vesiro
```

```json
{
  "vsl_version": "1a2b3c4d5",
  "vsl": {
    "branch": "main",
    "commit": "1a2b3c4d5",
    "dirty": false,
    "features": []
  }
}
```

`vsl_version` is the commit the plugin was built from. A successful response means the native library loaded, the license is valid, and the engine is answering calls. If the endpoint 404s, the plugin isn't installed or the node didn't restart.

---

## Advanced settings

Settings, tuning and logging are documented in [advanced-settings.md](advanced-settings.md):

- [Configuration](advanced-settings.md#configuration)
- [Per-request control](advanced-settings.md#per-request-control)
- [Performance tuning](advanced-settings.md#performance-tuning)
- [Elasticsearch and system configuration](advanced-settings.md#elasticsearch-and-system-configuration)
- [Logging](advanced-settings.md#logging)

---

## Transparent fallback

VesiroSearch is designed so that **a request never fails because of the plugin**. If the plugin cannot handle a request, whether an unsupported query construct, a feature not yet ported, or an unavailable native engine, it lets the standard query phase run instead.

When `vesiro.warn` is enabled (the default), those requests also carry a response header:

```
Warning: 299 Elasticsearch-8.17.5 "Request not supported in VesiroSearch. Fallback triggered: ..."
```

The same message is written to the log, at `WARN` for unexpected errors and at `DEBUG` for unsupported requests. To see the `DEBUG` ones, enable [debug logging](advanced-settings.md#logging), run the query again, and look for:

```
Request not supported in VesiroSearch. Fallback triggered:
```

This makes fallbacks observable rather than silent. Watch for them during rollout: a query that always falls back gets no benefit from the plugin, and the message tells you why.

Fallback also applies to unexpected native errors: the error is logged and the request is retried through the standard path.

---

## Troubleshooting

### No performance improvement

If queries are no faster than before, Vesiro may be falling back to Elasticsearch instead of running them natively. See [Transparent fallback](#transparent-fallback) for how to spot a fallback.

Each fallback means that query used something Vesiro does not run natively, so Elasticsearch handled it. Results are still correct, but those queries get no speedup.

---

## Support

Found a bug? Open an issue in this repository:
<https://github.com/Vesiro/vesiro-docs/issues>

For anything else, including licensing and commercial enquiries, contact
<info@vesiro.com>.
