---
title: MCP Toolkit
description: Use the Azure DocumentDB MCP Toolkit to give AI agents and MCP-aware applications a curated, audited tool surface against Azure DocumentDB clusters.
author: khelanmodi
ms.topic: how-to
ms.date: 05/06/2026
ms.author: khelanmodi
ms.collection:
  - ce-skilling-ai-copilot
---

# Azure DocumentDB MCP Toolkit

The **Azure DocumentDB MCP Toolkit** is an open-source [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server that lets AI agents and MCP-aware applications operate against Azure DocumentDB through a curated, audited tool surface. It's written in TypeScript on Node.js 20+ and distributed from the [`microsoft/documentdb-mcp`](https://github.com/microsoft/documentdb-mcp) repository.

The toolkit is a **tools-only** server: database connection details (endpoints, credentials, auth modes) are administrator-controlled and never accepted as MCP arguments — agents only ever name a profile.

> [!NOTE]
> The MCP Toolkit is in public preview. Interfaces, configuration, and tool behavior may change.

## What is Model Context Protocol (MCP)?

The Model Context Protocol is an open JSON-RPC protocol that standardizes how a large language model (LLM) client discovers and invokes:

- **Tools** – functions the model can call.
- **Resources** – addressable read-only data the host can attach to context.
- **Prompts** – reusable templates.

A typical deployment has one **MCP host** (the application running the LLM, such as GitHub Copilot CLI, Claude Code, or VS Code) connected to one or more **MCP servers** that expose tools, resources, and prompts. Communication runs over **stdio**, **streamable HTTP**, or **SSE**.

## Use cases

| Use case | Example |
| --- | --- |
| Conversational data exploration | "What's the schema of the `orders` collection?" |
| AI-assisted DBA tasks | "Create a unique index on `users.email`." |
| Schema discovery for agent grounding | Sample-based shape inference for prompt construction |
| Read-only analytics through agents | Aggregations, filtered counts, sample projections |
| Controlled write operations | Inserts, updates, deletes guarded by RBAC, capability flags, and confirmation |
| Data-aware copilots | Local stdio MCP integration with Copilot CLI, Claude Code, or VS Code |

## Key features

### Enterprise-grade security

- **Microsoft Entra ID authentication** — Token-based access with role-based authorization on every tool call.
- **Managed identity support** — No connection strings or shared secrets in production; the server exchanges its workload identity for a DocumentDB access token.
- **Role-based access** — Each tool is gated on a `read`, `write`, or `management` role; capability flags let operators disable entire tool classes.
- **Secure communication** — TLS-only transport to the DocumentDB cluster; HTTPS endpoints with bearer-token auth between MCP clients and the server.
- **Safety guardrails** — Retype-to-confirm on destructive tools (`drop_database`, `drop_collection`, `drop_index`, `rename_collection`) and a write-stage guard that blocks `$out` / `$merge` in `aggregate` unless explicitly allowed.

### Flexible deployment

- **Multiple transports** — Run over `stdio` for local agents (Copilot CLI, Claude Code, VS Code) or `streamable-http` / `sse` for shared and production deployments.
- **Run anywhere** — Self-host on Azure Container Apps, AKS, a VM, or any container runtime; the server is a standard Node.js 20+ container with no Azure-specific runtime dependencies.
- **Auditable operations** — Structured logs and an `[MCP-AUDIT]` JSON stream record every allow or deny decision for ingestion into Log Analytics, Application Insights, or any log aggregator.

## Architecture

:::image type="content" source="media/mcp-toolkit/architecture.png" alt-text="Architecture diagram showing MCP clients (Copilot CLI, Claude Code, VS Code) communicating over JSON-RPC with the DocumentDB MCP server, which applies flexible transports, a pre-auth security gate, and the dbGuard security chokepoint before connecting to an Azure DocumentDB cluster over the MongoDB wire protocol with TLS." lightbox="media/mcp-toolkit/architecture.png" border="false":::

## Tool catalog

The server registers 18 tools. Every tool requires a `connection_profile` argument; tool inputs without a valid profile are rejected.

| Tool | Category | Required role | Notes |
| --- | --- | --- | --- |
| `list_databases` | Database | `read` | Lists databases on the cluster. |
| `drop_database` | Database | `management` | Requires `confirm_db_name` retype. |
| `sample_documents` | Collection | `read` | Random sample for schema inference. |
| `get_statistics` | Collection | `read` | `collStats` data. |
| `current_ops` | Collection | `management` | Server-side `currentOp`. |
| `rename_collection` | Collection | `management` | Requires confirmation retype. |
| `drop_collection` | Collection | `management` | Requires `confirm_collection_name` retype. |
| `find_documents` | Document | `read` | Filter, projection, sort, limit, skip. |
| `count_documents` | Document | `read` | |
| `aggregate` | Document | `read` (or `write` if pipeline contains `$out` or `$merge`) | Write stages blocked unless `ALLOW_AGGREGATE_WRITE_STAGES=true`. |
| `explain_operation` | Document | `read` | Query planner output. |
| `insert_documents` | Document | `write` | |
| `update_documents` | Document | `write` | |
| `delete_documents` | Document | `write` | |
| `find_and_modify` | Document | `write` | |
| `list_indexes` | Index | `read` | |
| `create_index` | Index | `write` | |
| `drop_index` | Index | `management` | Requires `confirm_index_name` retype. |

## Prerequisites

Before you deploy the Azure DocumentDB MCP Toolkit:

- **Azure subscription** with access to your Azure DocumentDB cluster. [Create a free account](https://azure.microsoft.com/free/).
- **Azure CLI** installed and signed in. [Install Azure CLI](/cli/azure/install-azure-cli).
- **Existing Azure DocumentDB cluster** with data — the toolkit connects to your existing cluster and doesn't provision one.
- **Microsoft Entra ID permissions** to register an application and assign roles on the DocumentDB cluster (required for `streamable-http` / `sse` deployments).
- *Optional:* **Azure OpenAI service** if your agent generates embeddings for vector queries that it runs through the `aggregate` tool. Embedding generation happens on the agent side; the toolkit doesn't call Azure OpenAI directly.
- *Optional:* **Docker** for local container development. [Install Docker Desktop](https://www.docker.com/products/docker-desktop/).
- *Optional:* **Node.js 20+** for running the server locally outside containers. [Install Node.js](https://nodejs.org/).


## Quickstart configurations

### GitHub Copilot CLI

In `~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "DocumentDB": {
      "command": "npx",
      "args": ["-y", "github:microsoft/documentdb-mcp"],
      "env": {
        "TRANSPORT": "stdio",
        "ALLOW_UNAUTHENTICATED_STDIO": "true",
        "CONNECTION_PROFILES": "{\"local\":{\"authMode\":\"connectionString\",\"uri\":\"mongodb://localhost:27017\"}}"
      }
    }
  }
}
```

### Claude Code

Either add the server to `.mcp.json` at your project root using the same JSON shape as the GitHub Copilot CLI configuration, or register it from the command line:

```bash
claude mcp add DocumentDB \
  -e TRANSPORT=stdio \
  -e ALLOW_UNAUTHENTICATED_STDIO=true \
  -e 'CONNECTION_PROFILES={"local":{"authMode":"connectionString","uri":"mongodb://localhost:27017"}}' \
  -- npx -y github:microsoft/documentdb-mcp
```

### Visual Studio Code

For local stdio use with GitHub Copilot in Visual Studio Code, add the server to `settings.json`:

```json
{
  "mcp.servers": {
    "documentdb": {
      "command": "npx",
      "args": ["-y", "github:microsoft/documentdb-mcp"],
      "env": {
        "TRANSPORT": "stdio",
        "ALLOW_UNAUTHENTICATED_STDIO": "true",
        "CONNECTION_PROFILES": "..."
      }
    }
  }
}
```

To connect to a remote server (`streamable-http` transport with Microsoft Entra), reference its URL and supply a Microsoft Entra bearer token instead:

```json
{
  "mcp.servers": {
    "documentdb": {
      "url": "https://<your-mcp-host>/mcp",
      "headers": {
        "Authorization": "Bearer <entra-jwt>"
      }
    }
  }
}
```

### Production (HTTP, Microsoft Entra, managed identity)

```env
TRANSPORT=streamable-http
HOST=0.0.0.0
PORT=8070

AUTH_REQUIRED=true
ENTRA_TENANT_ID=<tenant>
ENTRA_AUDIENCE=<app-id-uri-or-client-id>

ENABLE_READ_TOOLS=true
ENABLE_WRITE_TOOLS=false
ENABLE_MANAGEMENT_TOOLS=false

CONNECTION_PROFILES={"prod":{"authMode":"entra","endpoint":"...","tokenScope":"...","allowedHosts":["*.documents.azure.com"]}}
```

## Configuration reference

All configuration is environment-driven. The repository's `.env.example` documents every variable, and `dotenv` loads `.env` automatically.

### Transport and binding

| Variable | Default | Purpose |
| --- | --- | --- |
| `TRANSPORT` | `streamable-http` | `stdio`, `sse`, or `streamable-http`. |
| `HOST` | `localhost` | HTTP/SSE bind address. |
| `PORT` | `8070` | HTTP/SSE port. |

### MCP authentication

| Variable | Default | Purpose |
| --- | --- | --- |
| `AUTH_REQUIRED` | `true` | Require JWT on HTTP and SSE. |
| `ENTRA_TENANT_ID` | — | Tenant for token issuer validation. |
| `ENTRA_AUDIENCE` | — | Required audience claim (Application ID URI or client ID). |
| `ALLOW_UNAUTHENTICATED_STDIO` | `false` | Permit unauthenticated stdio (development only). |

### Rate limiting

| Variable | Default | Purpose |
| --- | --- | --- |
| `RATE_LIMIT_ENABLED` | `true` | Apply pre-auth rate limit. |
| `RATE_LIMIT_WINDOW_MS` | `60000` | Window size in milliseconds. |
| `RATE_LIMIT_MAX_REQUESTS` | `120` | Per-IP cap per window. |

### Connection profiles

Define profiles either inline in `CONNECTION_PROFILES` or in a JSON file referenced by `CONNECTION_PROFILES_FILE`.

```json
{
  "prod": {
    "authMode": "entra",
    "endpoint": "<cluster>.documents.azure.com",
    "tokenScope": "<oauth-resource-uri>",
    "allowedHosts": ["*.documents.azure.com"],
    "tls": true,
    "retryWrites": true,
    "appName": "documentdb-mcp"
  },
  "local": {
    "authMode": "connectionString",
    "uriEnv": "DOCUMENTDB_LOCAL_URI"
  }
}
```

## Deployment topologies

| Topology | Transport | MCP auth | Backend auth | Use case |
| --- | --- | --- | --- | --- |
| Local development with Copilot CLI, Claude Code, or VS Code | `stdio` | Unauthenticated (trusted machine) | `connectionString` to local DocumentDB | Single developer. |
| Self-hosted shared agent | `streamable-http` | Microsoft Entra | Microsoft Entra (managed identity) | Team-shared MCP endpoint behind a reverse proxy. |
| Azure Container Apps or AKS | `streamable-http` | Microsoft Entra | Microsoft Entra (workload identity) | Production multi-tenant deployment. |

In Azure, prefer the `entra` profile mode with a managed identity granted backend access through the cluster's RBAC. This avoids storing database credentials at rest.

## Authentication

### MCP-side authentication (HTTP and SSE)

- Microsoft Entra ID JWT bearer tokens.
- The server validates the JWT issuer, audience, and signature (via JWKS) against the tenant and audience that the operator configures.
- Authentication is required by default. On `stdio`, authentication can be skipped only behind an explicit opt-in, intended for trusted local development.
- Authentication failures return JSON-RPC error code `-32001` and HTTP 401.

For the exact environment variables, see [Configuration reference](#configuration-reference).

### Backend authentication

Each connection profile uses one of two modes:

- **`entra`** – the server uses `DefaultAzureCredential` (Azure CLI, managed identity, workload identity, Visual Studio, and so on) to acquire an OAuth 2.0 token for the configured token scope and presents it to the cluster. **No database password on disk.**
- **`connectionString`** – the server reads the URI from configuration. Suitable for local development.

Each profile's `allowedHosts` allowlist enforces that the resolved endpoint matches an expected host pattern, mitigating misconfigured profiles. Profile structure and configuration variables are in [Configuration reference](#configuration-reference).

## Authorization

### Role hierarchy

A `management` caller can invoke `write` and `read` tools; a `write` caller can invoke `read` tools.

Roles are derived from JWT claims (`roles`, `groups`, or `scp`) and mapped through these variables:

| Variable | Maps claim values to | Recommended default |
| --- | --- | --- |
| `MCP_READ_ROLE_VALUES` | `read` | `DocumentDB.MCP.Read` |
| `MCP_WRITE_ROLE_VALUES` | `write` | `DocumentDB.MCP.Write` |
| `MCP_MANAGEMENT_ROLE_VALUES` | `management` | `DocumentDB.MCP.Management` |

### Capability flags

Capability flags are independent of the caller's role. If a flag is off, the corresponding tools are denied even for callers with the matching role. They act as a deployment-wide kill switch.

| Flag | Default | Effect |
| --- | --- | --- |
| `ENABLE_READ_TOOLS` | `true` | Permit read-tool invocation. |
| `ENABLE_WRITE_TOOLS` | `false` | Permit write-tool invocation. |
| `ENABLE_MANAGEMENT_TOOLS` | `false` | Permit management-tool invocation. |
| `ALLOW_AGGREGATE_WRITE_STAGES` | `false` | Permit `$out` or `$merge` stages in `aggregate`. |

## Safety guardrails

For destructive and high-impact tools, the server stacks four independent layers, each of which can deny the call:

| Layer | What it does |
| --- | --- |
| Capability flag | The relevant `ENABLE_*` flag must be `true`. |
| Role | The caller must have the required role. |
| Profile | A valid administrator-defined profile must be supplied. |
| Retype-to-confirm | The caller must echo the target name in a confirmation field. |

### Retype-to-confirm

| Tool | Confirmation field | Must equal |
| --- | --- | --- |
| `drop_database` | `confirm_db_name` | `db_name` |
| `drop_collection` | `confirm_collection_name` | `collection_name` |
| `drop_index` | `confirm_index_name` | `index_name` |

This pattern hardens against prompt injection: an attacker would have to coerce the model into both naming the target and producing the matching confirmation in the same call.

### Aggregate write-stage guard

The `aggregate` tool rejects `$out` and `$merge` stages unless `ALLOW_AGGREGATE_WRITE_STAGES=true`.

### Other invariants

- Tools never accept connection strings as MCP arguments. Connection strings are server-side process configuration only.
- No tool exposes a raw shell or `eval`-style operation.
- Profile names are not enumerable before authentication.
- HTTP and SSE always require an explicit `connection_profile`. Implicit single-profile selection is gated to `stdio` only.

## Audit log

Every allow or deny decision is written to **stderr** as a single JSON line prefixed with `[MCP-AUDIT]`:

```json
{
  "timestamp": "2026-05-06T17:00:00.123Z",
  "toolName": "find_documents",
  "requiredRole": "read",
  "decision": "allow",
  "reason": null,
  "connectionProfile": "prod",
  "transport": "streamable-http",
  "sessionId": "...",
  "requestId": "...",
  "principal": { "oid": "...", "sub": "...", "tid": "...", "upn": "...", "name": "..." }
}
```

## Limitations

### Functional

- Tools-only. MCP resources and prompts are not implemented.
- MongoDB-compatible API surface only.
- No transaction tool. Multi-document transactions are not surfaced.
- No bulk import tool. Use `mongoimport` and equivalents.
- No first-class vector-search tool. Vector queries can be issued through `aggregate` if the backend supports them.
- Schema discovery is sample-based and best-effort.

### Security and data handling

- No data masking. Tool results are returned verbatim, and documents flow into the LLM context (and any provider logging or memory). Use database-side masked views for sensitive workloads.
- No field- or row-level access control inside the server. The profile's identity is the data plane.
- No outbound proxy or egress isolation. The server connects directly to allowed hosts.

### Operational

- No persistent state. Connection pools are per process, and there's no hot reload of configuration; restart to apply changes.
- Per-IP rate limit only.
- The audit log is stderr-only.
- No first-party metrics endpoint; instrument at the reverse-proxy or sidecar layer if needed.
- Public preview. The API surface and configuration may evolve.

### Compatibility

- Requires Node.js 20+.
- Validated against Azure DocumentDB. Other MongoDB-compatible engines might work where they implement the same wire-protocol commands but aren't validated in CI.
- No published npm package yet. Install from source or with `npx -y github:microsoft/documentdb-mcp`.

## Related content

- [Azure DocumentDB integrations for AI applications](ai-frameworks.md)
- [DocumentDB MCP server (GitHub)](https://github.com/microsoft/documentdb-mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
