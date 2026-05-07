---
title: MCP Toolkit for AI agents
description: Use the Azure DocumentDB MCP Toolkit to give AI coding agents like Claude Code, Cursor, Codex, Gemini CLI, and GitHub Copilot first-class access to Azure DocumentDB.
author: khelanmodi
ms.topic: how-to
ms.date: 05/06/2026
ms.author: khelanmodi
ms.collection:
  - ce-skilling-ai-copilot
---

# Azure DocumentDB MCP Toolkit

The [Azure DocumentDB MCP Toolkit](https://github.com/Azure/documentdb-agent-kit) (also called the *DocumentDB agent kit*) bundles a [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server with a curated set of [Agent Skills](https://agentskills.io/) so that AI coding agents can design schemas, write queries, tune indexes, and operate Azure DocumentDB clusters with the same expertise as the product team.

The toolkit is built on top of the official [DocumentDB MCP server](https://github.com/microsoft/documentdb-mcp) (`documentdb-mcp-server` on npm) and ships plugin manifests for every major coding agent.

## What's in the toolkit

The toolkit combines two layers:

- **MCP server** – Exposes Azure DocumentDB tools (query, index, schema, diagnostics) to any MCP-compatible agent. The server is administrator-controlled: connection profiles are defined server-side so tools never accept runtime connection strings.
- **Agent skills** – A catalog of best-practice rule files and end-to-end workflows that agents load on demand.

### Skill catalog

| Skill | When to use |
| --- | --- |
| `documentdb-data-modeling` | Designing schemas, embed vs reference, the 16 MB document limit, denormalization, schema versioning. |
| `documentdb-cluster-sharding` | Picking an M-tier, scaling decisions, shard-key design at TB scale. |
| `documentdb-query-optimization` | Writing queries that use indexes; reading `explain("executionStats")`. |
| `documentdb-indexing` | Choosing the right index type (single, compound, multikey, wildcard, hashed, 2dsphere, TTL); ESR ordering; safe index lifecycle. |
| `documentdb-driver` | Singleton `MongoClient`, connection reuse fundamentals. |
| `documentdb-vector-search` | `cosmosSearch` with DiskANN, HNSW, or IVF; product quantization; half-precision; cosine normalization. |
| `documentdb-full-text-search` | `$search` with `createSearchIndexes`, custom analyzers, fuzzy/phrase/prefix queries, and BM25 + vector hybrid. |
| `documentdb-high-availability` | High availability, cross-region replicas, and 99.99% / 99.995% SLAs. |
| `documentdb-security` | TLS, Private Endpoint, Microsoft Entra RBAC, customer-managed keys (CMK). |
| `documentdb-monitoring` | Diagnostic settings, slow-query logs, metrics, and alerts. |
| `documentdb-local-deployment` | Docker image choice, Compose, TLS, env-driven config, dev/prod parity. |
| `documentdb-mcp-setup` | Configuring `DOCUMENTDB_CONNECTION_PROFILES`, transport, and shell profile for the MCP server. |
| `documentdb-azure-deployment` | Provisioning a cluster (`Microsoft.DocumentDB/mongoClusters`) via Bicep, Azure CLI, Terraform, or the portal. |
| `documentdb-natural-language-querying` | Translating "how do I query / filter / group / aggregate…" or SQL to MQL (read-only). |
| `documentdb-query-optimizer` | "Why is this slow?" reviews, `explain()`-driven tuning. |
| `documentdb-connection` | Pool size, timeout, and retry tuning for serverless, OLTP, OLAP, or bursty workloads. |

For the full catalog, see [`docs/SKILLS.md`](https://github.com/Azure/documentdb-agent-kit/blob/main/docs/SKILLS.md) in the toolkit repository.

## Install

The toolkit publishes a plugin or extension manifest for each agent. Pick the matching command for the agent you use.

### Claude Code

```text
/plugin marketplace add Azure/documentdb-agent-kit
/plugin install documentdb@azure-documentdb
```

### Cursor

```text
/add-plugin azure/documentdb-agent-kit
```

### Codex

```bash
codex plugin marketplace add azure/documentdb-agent-kit
codex plugin install documentdb
```

### Gemini CLI

```bash
gemini extensions install https://github.com/Azure/documentdb-agent-kit
```

### GitHub Copilot CLI

```bash
/plugin install https://github.com/Azure/documentdb-agent-kit.git
```

Restart Copilot CLI after installing to activate the MCP server.

### Skills only (any agent)

To install just the skill catalog—without the MCP server—use the [skills.sh](https://skills.sh/) CLI:

```bash
npx skills add Azure/documentdb-agent-kit
```

## Configure the MCP server

The MCP server is administrator-controlled: tools never accept runtime connection strings. Define a connection profile in the `DOCUMENTDB_CONNECTION_PROFILES` environment variable before launching the agent.

### Microsoft Entra (recommended)

```bash
export DOCUMENTDB_CONNECTION_PROFILES='{"sandbox":{"authMode":"entra","endpoint":"<cluster>.mongocluster.cosmos.azure.com","tokenScope":"https://ossrdbms-aad.database.windows.net/.default","allowedHosts":["*.mongocluster.cosmos.azure.com"]}}'

az login --tenant <tenant-id>
```

In Azure-hosted environments, use a managed identity or workload identity and grant that identity access to the backend database. The server uses `DefaultAzureCredential`, so the same profile shape works for local Azure CLI sign-in and managed deployments.

### Local or sandbox SCRAM

```bash
export DOCUMENTDB_CONNECTION_PROFILES='{"local":{"uriEnv":"DOCUMENTDB_LOCAL_URI"}}'
export DOCUMENTDB_LOCAL_URI='mongodb://localhost:27017'
```

### Tool capability gates

Only **read** tools are enabled by default. Opt in to higher-impact tools by exporting these variables before starting the agent:

```bash
export ENABLE_WRITE_TOOLS=true        # insert / update / delete / find_and_modify
export ENABLE_MANAGEMENT_TOOLS=true   # drop_database, drop_collection, create_index, ...
```

## Related content

- [Azure DocumentDB integrations for AI applications](ai-frameworks.md)
- [Azure DocumentDB agent kit (GitHub)](https://github.com/Azure/documentdb-agent-kit)
- [DocumentDB MCP server (GitHub)](https://github.com/microsoft/documentdb-mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
