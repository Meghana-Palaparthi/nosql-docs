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

A typical deployment has one **MCP host** (the application running the LLM, such as GitHub Copilot CLI, Claude Desktop, VS Code, or a Microsoft Foundry agent) connected to one or more **MCP servers** that expose tools, resources, and prompts. Communication runs over **stdio**, **streamable HTTP**, or **SSE**.

## Use cases

| Use case | Example |
| --- | --- |
| Conversational data exploration | "What's the schema of the `orders` collection?" |
| AI-assisted DBA tasks | "Create a unique index on `users.email`." |
| Schema discovery for agent grounding | Sample-based shape inference for prompt construction |
| Read-only analytics through agents | Aggregations, filtered counts, sample projections |
| Controlled write operations | Inserts, updates, deletes guarded by RBAC, capability flags, and confirmation |
| Data-aware copilots | Local stdio MCP integration with Copilot CLI, Claude Desktop, or VS Code |
| Microsoft Foundry agent connectors | Streamable HTTP with Microsoft Entra–managed identity |

## Tool catalog

The server registers 18 tools across four categories. Every tool requires a `connection_profile` argument; tool inputs without a valid profile are rejected.

### Database tools

| Tool | Required role | Notes |
| --- | --- | --- |
| `list_databases` | `read` | Lists databases on the cluster. |
| `drop_database` | `management` | Requires `confirm_db_name` retype. |

### Collection tools

| Tool | Required role | Notes |
| --- | --- | --- |
| `sample_documents` | `read` | Random sample for schema inference. |
| `get_statistics` | `read` | `collStats` data. |
| `current_ops` | `management` | Server-side `currentOp`. |
| `rename_collection` | `management` | Requires confirmation retype. |
| `drop_collection` | `management` | Requires `confirm_collection_name` retype. |

### Document tools

| Tool | Required role | Notes |
| --- | --- | --- |
| `find_documents` | `read` | Filter, projection, sort, limit, skip. |
| `count_documents` | `read` | |
| `aggregate` | `read` (or `write` if pipeline contains `$out` or `$merge`) | Write stages blocked unless `ALLOW_AGGREGATE_WRITE_STAGES=true`. |
| `explain_operation` | `read` | Query planner output. |
| `insert_documents` | `write` | |
| `update_documents` | `write` | |
| `delete_documents` | `write` | |
| `find_and_modify` | `write` | |

### Index tools

| Tool | Required role | Notes |
| --- | --- | --- |
| `list_indexes` | `read` | |
| `create_index` | `write` | |
| `drop_index` | `management` | Requires `confirm_index_name` retype. |

## Architecture

:::image type="content" source="media/mcp-toolkit/architecture.png" alt-text="Architecture diagram showing MCP clients (Copilot CLI, Claude Desktop, VS Code) communicating over JSON-RPC with the DocumentDB MCP server, which applies flexible transports, a pre-auth security gate, and the dbGuard security chokepoint before connecting to an Azure DocumentDB cluster over the MongoDB wire protocol with TLS." lightbox="media/mcp-toolkit/architecture.png" border="false":::

### Request lifecycle

1. The client opens an MCP session over stdio, streamable HTTP, or SSE.
1. (HTTP/SSE only) A per-IP rate limit is applied **before** authentication.
1. (HTTP/SSE only) The `Authorization: Bearer <jwt>` header is validated, and the principal is attached to the request context.
1. The client sends `tools/call` for one of the 18 tools.
1. The `dbGuard` chokepoint runs in this order, failing fast on the first deny:
   1. Capability flag check.
   1. Role check.
   1. Confirmation token check (destructive operations only).
   1. Profile resolution.
1. The tool handler executes the validated operation through a `MongoClient` for the resolved profile.
1. An audit event is written for the allow.
1. The result is returned to the client.

## Authentication

### MCP-side authentication (HTTP and SSE)

- Microsoft Entra ID JWT bearer tokens.
- Issuer is fixed to `https://login.microsoftonline.com/<ENTRA_TENANT_ID>/v2.0`.
- The audience must match `ENTRA_AUDIENCE` (Application ID URI or client ID).
- Signatures are validated via JWKS.
- Authentication is required by default (`AUTH_REQUIRED=true`). On `stdio`, authentication is skipped only when `ALLOW_UNAUTHENTICATED_STDIO=true`, intended for trusted local development.
- Authentication failures return JSON-RPC error code `-32001` and HTTP 401.

### Backend authentication

Each connection profile uses one of two modes:

- **`entra`** – the server uses `DefaultAzureCredential` (Azure CLI, managed identity, workload identity, Visual Studio, and so on) to acquire an OAuth 2.0 token for the configured `tokenScope` and presents it to the cluster. **No database password on disk.**
- **`connectionString`** – the server reads the URI inline (`uri`) or from a named environment variable (`uriEnv`). Suitable for local development.

Each profile's `allowedHosts` allowlist enforces that the resolved endpoint matches an expected host pattern, mitigating misconfigured profiles.

## Authorization

### Role hierarchy

```
management ⊃ write ⊃ read
```

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

```jsonc
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
| Local development with Copilot CLI, Claude, or VS Code | `stdio` | Unauthenticated (trusted machine) | `connectionString` to local DocumentDB | Single developer. |
| Self-hosted shared agent | `streamable-http` | Microsoft Entra | Microsoft Entra (managed identity) | Team-shared MCP endpoint behind a reverse proxy. |
| Microsoft Foundry agent connector | `streamable-http` | Microsoft Entra | Microsoft Entra (workload identity) | Foundry agents calling the MCP server. |
| Azure Container Apps or AKS | `streamable-http` | Microsoft Entra | Microsoft Entra (workload identity) | Production multi-tenant deployment. |

In Azure, prefer the `entra` profile mode with a managed identity granted backend access through the cluster's RBAC. This avoids storing database credentials at rest.

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

### Claude Desktop

Use the same JSON shape under `mcpServers` in `%APPDATA%\Claude\claude_desktop_config.json` (Windows) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS).

### Visual Studio Code

In `settings.json`:

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

## Example test scenarios

After the toolkit is configured, use these scenarios to confirm that each tool category responds end to end. Each row lists the inputs to provide and the tool to invoke from your MCP client.

| Scenario | Inputs | Tool |
| --- | --- | --- |
| List databases | None | Select `list_databases`, then select **Invoke Tool** |
| Explore containers | Database name | Select `list_collections` |
| Recent documents | Database name and container name | Select `get_recent_documents` |
| Search content | Search parameters (query, fields, limit) | Select `text_search` |
| Vector search | Search text and vector property | Select `vector_search` |

## Observability

### Audit log

Every allow or deny decision is written to **stderr** as a single JSON line prefixed with `[MCP-AUDIT]`:

```jsonc
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

### Operational logs

Diagnostic logs (startup, transport, errors) also go to stderr. There's no built-in sink integration; ship stderr to your central log platform such as Azure Monitor, Splunk, ELK, or Loki.

### Metrics

There's no first-party metrics endpoint. Instrument at the reverse-proxy or sidecar layer if needed.

## Security properties

| Property | Status |
| --- | --- |
| Connection strings excluded from MCP arguments | Yes — architectural invariant. |
| Default-deny for write and management capabilities | Yes — capability flags off by default. |
| Pre-auth rate limiting on HTTP and SSE | Yes — 120 requests per IP per 60 seconds by default. |
| JWT issuer, audience, and signature validation | Yes — via JWKS. |
| Hierarchical role model (read ⊂ write ⊂ management) | Yes. |
| Retype-to-confirm on irreversible destructive operations | Yes — `drop_database`, `drop_collection`, `drop_index`. |
| `$out` and `$merge` blocked by default in `aggregate` | Yes. |
| Allowed-host allowlist per profile | Yes. |
| Stdio unauthenticated only behind explicit flag | Yes. |
| Audit log of every allow or deny | Yes — to stderr. |
| Backend Microsoft Entra (no database password on disk) | Yes — recommended path. |
| Data masking | Not supported. |
| Field-level RBAC inside the server | Not supported. |
| Per-identity quotas | Not supported (per-IP only). |

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
- Public preview. The API surface and configuration may evolve.

### Compatibility

- Requires Node.js 20+.
- Validated against Azure DocumentDB. Other MongoDB-compatible engines might work where they implement the same wire-protocol commands but aren't validated in CI.
- No published npm package yet. Install from source or with `npx -y github:microsoft/documentdb-mcp`.

## Operational runbook

| Scenario | Procedure |
| --- | --- |
| Disable a tool family | Set the relevant `ENABLE_*` flag to `false` and restart. |
| Rotate connection profiles | Update `CONNECTION_PROFILES` (or the profiles file) and restart. |
| Tighten rate limit | Lower `RATE_LIMIT_MAX_REQUESTS` and restart. |
| Investigate a denied call | Filter stderr for `[MCP-AUDIT]` events with `decision:"deny"`; the `reason` field gives the cause. |
| Identify the caller | Inspect `principal.oid`, `upn`, or `name` on the audit record. |
| Enable destructive operations for a one-time migration | Set `ENABLE_MANAGEMENT_TOOLS=true`, perform the call (with retype-to-confirm), then revert. |
| Detect token tampering or brute force | Watch for sudden spikes in deny events with `Missing bearer token` or `JWT signature` reasons, plus rate-limit 429 responses. |

## Related content

- [Azure DocumentDB integrations for AI applications](ai-frameworks.md)
- [DocumentDB MCP server (GitHub)](https://github.com/microsoft/documentdb-mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
