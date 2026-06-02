---
title: MCP Toolkit
description: Use the Azure DocumentDB MCP Toolkit to give AI agents and MCP-aware applications a curated, audited tool surface against Azure DocumentDB clusters.
author: khelanmodi
ms.topic: how-to
ms.date: 05/18/2026
ms.author: khelanmodi
ms.collection:
  - ce-skilling-ai-copilot
---

# Azure DocumentDB MCP Toolkit

The **Azure DocumentDB MCP Toolkit** is an open-source [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server that lets AI agents and MCP-aware applications operate against Azure DocumentDB through a curated, audited tool surface. The toolkit runs on Node.js 20+, uses TypeScript, and ships from the [`microsoft/documentdb-mcp`](https://github.com/microsoft/documentdb-mcp) repository.

The toolkit is a **tools-only** server: database connection details (endpoints, credentials, auth modes) are administrator-controlled and never accepted as MCP arguments. Agents only ever name a profile.

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

- **Microsoft Entra ID authentication**. Token-based access with role-based authorization on every tool call.
- **Managed identity support**. No connection strings or shared secrets in production; the server exchanges its workload identity for a DocumentDB access token.
- **Role-based access**. Each tool is gated on a `read`, `write`, or `management` role; capability flags let operators disable entire tool classes.
- **Secure communication**. TLS-only transport to the DocumentDB cluster; HTTPS endpoints with bearer-token auth between MCP clients and the server.
- **Safety guardrails**. Retype-to-confirm on destructive tools (`drop_database`, `drop_collection`, `drop_index`) and a write-stage guard that blocks `$out` / `$merge` in `aggregate` unless explicitly allowed.

### Flexible deployment

- **Multiple transports**. Run over `stdio` for local agents (Copilot CLI, Claude Code, VS Code) or `streamable-http` / `sse` for shared and production deployments.
- **Run anywhere**. Self-host on Azure Container Apps, AKS, a VM, or any container runtime; the server is a standard Node.js 20+ container with no Azure-specific runtime dependencies.
- **Auditable operations**. Structured logs and an `[MCP-AUDIT]` JSON stream record every allow or deny decision for ingestion into Log Analytics, Application Insights, or any log aggregator.

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
| `rename_collection` | Collection | `management` | Renames a collection. |
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
- **Existing Azure DocumentDB cluster** with data. The toolkit connects to your existing cluster and doesn't provision one.
- **Microsoft Entra ID permissions** to register an application and assign roles on the DocumentDB cluster (required for `streamable-http` / `sse` deployments).
- *Optional:* **Azure OpenAI service** if your agent generates embeddings for vector queries that it runs through the `aggregate` tool. Embedding generation happens on the agent side; the toolkit doesn't call Azure OpenAI directly.
- *Optional:* **Docker** for local container development. [Install Docker Desktop](https://www.docker.com/products/docker-desktop/).
- *Optional:* **Node.js 20+** for running the server locally outside containers. [Install Node.js](https://nodejs.org/).


## Quickstart configurations

For VS Code and Cursor, use these one-click install links. Each link prompts for a DocumentDB connection string and registers the server with the local stdio configuration.

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Server-0098FF?logo=data:image/svg%2bxml;base64,PHN2ZyBmaWxsPSIjRkZGRkZGIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciICB2aWV3Qm94PSIwIDAgNDggNDgiIHdpZHRoPSIyNHB4IiBoZWlnaHQ9IjI0cHgiPjxwYXRoIGQ9Ik00NC45OTkgMTAuODd2MjYuMjFjMCAxLjAzLS41OSAxLjk3LTEuNTEgMi40Mi0yLjY4IDEuMjktOCAzLjg1LTguMzUgNC4wMS0uMTMuMDctLjM4LjItLjY3LjMxLjM1LS42LjUzLTEuMy41My0yLjAyVjYuMmMwLS43NS0uMi0xLjQ1LS41Ni0yLjA2LjA5LjA0LjE3LjA4LjI0LjExLjIuMSA1Ljk4IDIuODYgOC44IDQuMkM0NC40MDkgOC45IDQ0Ljk5OSA5Ljg0IDQ0Ljk5OSAxMC44N3pNNy40OTkgMjYuMDNjMS42IDEuNDYgMy40MyAzLjEzIDUuMzQgNC44NmwtNC42IDMuNWMtLjc3LjU3LTEuNzguNS0yLjU2LS4wNS0uNS0uMzYtMS44OS0xLjY1LTEuODktMS42NS0xLjAxLS44MS0xLjA2LTIuMzItLjExLTMuMTlDMy42NzkgMjkuNSA1LjE3OSAyOC4xMyA3LjQ5OSAyNi4wM3pNMzEuOTk5IDYuMnYxMC4xMWwtNy42MyA1LjgtNi44NS01LjIxYzQuOTgtNC41MyAxMC4wMS05LjExIDEyLjY1LTExLjUyQzMwLjg2OSA0Ljc0IDMxLjk5OSA1LjI1IDMxLjk5OSA2LjJ6TTMyIDQxLjc5OFYzMS42OUw4LjI0IDEzLjYxYy0uNzctLjU3LTEuNzgtLjUtMi41Ni4wNS0uNS4zNi0xLjg5IDEuNjUtMS44OSAxLjY1LTEuMDEuODEtMS4wNiAyLjMyLS4xMSAzLjE5IDAgMCAyMC4xNDUgMTguMzM4IDI2LjQ4NSAyNC4xMTZDMzAuODcxIDQzLjI2IDMyIDQyLjc1MyAzMiA0MS43OTh6Ii8+PC9zdmc+)](https://insiders.vscode.dev/redirect/mcp/install?name=DocumentDB&inputs=%5B%7B%22id%22%3A%22connection_string%22%2C%22type%22%3A%22promptString%22%2C%22description%22%3A%22DocumentDB%20connection%20string%20(e.g.%20mongodb%3A%2F%2Flocalhost%3A27017)%22%7D%5D&config=%7B%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22github%3Amicrosoft%2Fdocumentdb-mcp%22%5D%2C%22env%22%3A%7B%22TRANSPORT%22%3A%22stdio%22%2C%22ALLOW_UNAUTHENTICATED_STDIO%22%3A%22true%22%2C%22CONNECTION_PROFILES%22%3A%22%7B%5C%22local%5C%22%3A%7B%5C%22authMode%5C%22%3A%5C%22connectionString%5C%22%2C%5C%22uri%5C%22%3A%5C%22%24%7Binput%3Aconnection_string%7D%5C%22%7D%7D%22%7D%7D)
[![Install in Cursor](https://img.shields.io/badge/Cursor-Install_Server-1e1e1e?logo=data:image/svg%2bxml;base64,PHN2ZyBoZWlnaHQ9IjFlbSIgc3R5bGU9ImZsZXg6bm9uZTtsaW5lLWhlaWdodDoxIiB2aWV3Qm94PSIwIDAgMjQgMjQiIHdpZHRoPSIxZW0iCiAgICB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPgogICAgPHRpdGxlPkN1cnNvcjwvdGl0bGU+CiAgICA8cGF0aCBkPSJNMTEuOTI1IDI0bDEwLjQyNS02LTEwLjQyNS02TDEuNSAxOGwxMC40MjUgNnoiCiAgICAgICAgZmlsbD0idXJsKCNsb2JlLWljb25zLWN1cnNvcnVuZGVmaW5lZC1maWxsLTApIj48L3BhdGg+CiAgICA8cGF0aCBkPSJNMjIuMzUgMThWNkwxMS45MjUgMHYxMmwxMC40MjUgNnoiIGZpbGw9InVybCgjbG9iZS1pY29ucy1jdXJzb3J1bmRlZmluZWQtZmlsbC0xKSI+PC9wYXRoPgogICAgPHBhdGggZD0iTTExLjkyNSAwTDEuNSA2djEybDEwLjQyNS02VjB6IiBmaWxsPSJ1cmwoI2xvYmUtaWNvbnMtY3Vyc29ydW5kZWZpbmVkLWZpbGwtMikiPjwvcGF0aD4KICAgIDxwYXRoIGQ9Ik0yMi4zNSA2TDExLjkyNSAyNFYxMkwyMi4zNSA2eiIgZmlsbD0iIzU1NSI+PC9wYXRoPgogICAgPHBhdGggZD0iTTIyLjM1IDZsLTEwLjQyNSA2TDEuNSA2aDIwLjg1eiIgZmlsbD0iI2ZmZiI+PC9wYXRoPgogICAgPGRlZnM+CiAgICAgICAgPGxpbmVhckdyYWRpZW50IGdyYWRpZW50VW5pdHM9InVzZXJTcGFjZU9uVXNlIiBpZD0ibG9iZS1pY29ucy1jdXJzb3J1bmRlZmluZWQtZmlsbC0wIgogICAgICAgICAgICB4MT0iMTEuOTI1IiB4Mj0iMTEuOTI1IiB5MT0iMTIiIHkyPSIyNCI+CiAgICAgICAgICAgIDxzdG9wIG9mZnNldD0iLjE2IiBzdG9wLWNvbG9yPSIjZmZmIiBzdG9wLW9wYWNpdHk9Ii4zOSI+PC9zdG9wPgogICAgICAgICAgICA8c3RvcCBvZmZzZXQ9Ii42NTgiIHN0b3AtY29sb3I9IiNmZmYiIHN0b3Atb3BhY2l0eT0iLjgiPjwvc3RvcD4KICAgICAgICA8L2xpbmVhckdyYWRpZW50PgogICAgICAgIDxsaW5lYXJHcmFkaWVudCBncmFkaWVudFVuaXRzPSJ1c2VyU3BhY2VPblVzZSIgaWQ9ImxvYmUtaWNvbnMtY3Vyc29ydW5kZWZpbmVkLWZpbGwtMSIKICAgICAgICAgICAgeDE9IjIyLjM1IiB4Mj0iMTEuOTI1IiB5MT0iNi4wMzciIHkyPSIxMi4xNSI+CiAgICAgICAgICAgIDxzdG9wIG9mZnNldD0iLjE4MiIgc3RvcC1jb2xvcj0iI2ZmZiIgc3RvcC1vcGFjaXR5PSIuMzEiPjwvc3RvcD4KICAgICAgICAgICAgPHN0b3Agb2Zmc2V0PSIuNzE1IiBzdG9wLWNvbG9yPSIjZmZmIiBzdG9wLW9wYWNpdHk9IjAiPjwvc3RvcD4KICAgICAgICA8L2xpbmVhckdyYWRpZW50PgogICAgICAgIDxsaW5lYXJHcmFkaWVudCBncmFkaWVudFVuaXRzPSJ1c2VyU3BhY2VPblVzZSIgaWQ9ImxvYmUtaWNvbnMtY3Vyc29ydW5kZWZpbmVkLWZpbGwtMiIKICAgICAgICAgICAgeDE9IjExLjkyNSIgeDI9IjEuNSIgeTE9IjAiIHkyPSIxOCI+CiAgICAgICAgICAgIDxzdG9wIHN0b3AtY29sb3I9IiNmZmYiIHN0b3Atb3BhY2l0eT0iLjYiPjwvc3RvcD4KICAgICAgICAgICAgPHN0b3Agb2Zmc2V0PSIuNjY3IiBzdG9wLWNvbG9yPSIjZmZmIiBzdG9wLW9wYWNpdHk9Ii4yMiI+PC9zdG9wPgogICAgICAgIDwvbGluZWFyR3JhZGllbnQ+CiAgICA8L2RlZnM+Cjwvc3ZnPgo=)](https://cursor.com/en-US/install-mcp?name=DocumentDB&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsImdpdGh1YjptaWNyb3NvZnQvZG9jdW1lbnRkYi1tY3AiXSwiZW52Ijp7IlRSQU5TUE9SVCI6InN0ZGlvIiwiQUxMT1dfVU5BVVRIRU5USUNBVEVEX1NURElPIjoidHJ1ZSIsIkNPTk5FQ1RJT05fUFJPRklMRVMiOiJ7XCJsb2NhbFwiOntcImF1dGhNb2RlXCI6XCJjb25uZWN0aW9uU3RyaW5nXCIsXCJ1cmlcIjpcIiR7aW5wdXQ6Y29ubmVjdGlvbl9zdHJpbmd9XCJ9fSJ9fQ%3D%3D)

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

CONNECTION_PROFILES={"prod":{"authMode":"entra","endpoint":"<cluster>.mongocluster.cosmos.azure.com","tokenScope":"https://ossrdbms-aad.database.windows.net/.default","allowedHosts":["*.mongocluster.cosmos.azure.com"]}}
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
| `ENTRA_TENANT_ID` | (none) | Tenant for token issuer validation. |
| `ENTRA_AUDIENCE` | (none) | Required audience claim (Application ID URI or client ID). |
| `ALLOW_UNAUTHENTICATED_STDIO` | `false` | Permit unauthenticated stdio (development only). |

### Rate limiting

| Variable | Default | Purpose |
| --- | --- | --- |
| `RATE_LIMIT_ENABLED` | `true` | Apply pre-auth rate limit. |
| `RATE_LIMIT_WINDOW_MS` | `60000` | Window size in milliseconds. |
| `RATE_LIMIT_MAX_REQUESTS` | `120` | Per-IP cap per window. |

### Connection profiles

Define profiles either inline in `CONNECTION_PROFILES` or in a JSON file referenced by `CONNECTION_PROFILES_FILE`. The recommended profile mode is `entra`; the `connectionString` form is a legacy SCRAM fallback for local or sandbox use.

```json
{
  "prod": {
    "authMode": "entra",
    "endpoint": "<cluster>.mongocluster.cosmos.azure.com",
    "tokenScope": "https://ossrdbms-aad.database.windows.net/.default",
    "allowedHosts": ["*.mongocluster.cosmos.azure.com"],
    "tls": true,
    "retryWrites": true,
    "appName": "documentdb-mcp"
  },
  "local": {
    "uriEnv": "DOCUMENTDB_LOCAL_URI"
  }
}
```

The `local` profile shown above is a legacy SCRAM connection-string profile. The server resolves the URI from the environment variable named in `uriEnv` (for example, `DOCUMENTDB_LOCAL_URI=mongodb://localhost:27017`). Use it only for local or sandbox development; production deployments should use the `entra` profile mode with a managed identity.

## Deployment topologies

| Topology | Transport | MCP auth | Backend auth | Use case |
| --- | --- | --- | --- | --- |
| Local development with Copilot CLI, Claude Code, or VS Code | `stdio` | Unauthenticated (trusted machine) | `connectionString` to local DocumentDB | Single developer. |
| Self-hosted shared agent | `streamable-http` | Microsoft Entra | Microsoft Entra (managed identity) | Team-shared MCP endpoint behind a reverse proxy. |
| Azure Container Apps or AKS | `streamable-http` | Microsoft Entra | Microsoft Entra (workload identity) | Production multitenant deployment. |

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

Each profile's `allowedHosts` allow list enforces that the resolved endpoint matches an expected host pattern, mitigating misconfigured profiles. Profile structure and configuration variables are in [Configuration reference](#configuration-reference).

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

### Deleting databases, collections, or indexes

Drop operations (`drop_database`, `drop_collection`, `drop_index`) are classified as **management** tools. Every drop call must clear *all* of the following. Failing any single check denies the request:

1. **`ENABLE_MANAGEMENT_TOOLS=true`** is set on the server. This flag is off by default; management tools aren't even registered until an operator opts in.
2. The caller's token carries a claim that maps to the **`management`** MCP role (for example, `DocumentDB.MCP.Management` via `MCP_MANAGEMENT_ROLE_VALUES`).
3. The tool call names a valid administrator-defined **`connection_profile`**.
4. The caller **retypes the target name** in the matching confirmation field (see the table below). The confirmation must match exactly. The server rejects mismatches before issuing the drop.

Read and write roles cannot invoke drop tools even if `ENABLE_MANAGEMENT_TOOLS=true`, and a `management` caller cannot drop anything while `ENABLE_MANAGEMENT_TOOLS=false`. Keep off the flag in any deployment that does not need destructive operations.

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
- No published npm package yet. Install from source (`git clone https://github.com/microsoft/documentdb-mcp.git && npm install && npm run build`) or run directly with `npx -y github:microsoft/documentdb-mcp`.

## Related content

- [Azure DocumentDB integrations for AI applications](ai-frameworks.md)
- [DocumentDB MCP server (GitHub)](https://github.com/microsoft/documentdb-mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
