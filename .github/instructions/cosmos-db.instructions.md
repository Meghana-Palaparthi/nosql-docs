---
description: Code review and content authoring guidelines for Azure Cosmos DB documentation
applyTo: "azure/cosmos-db/**/*.md,cosmos-db/**/*.md"
---

# Azure Cosmos DB documentation guidelines

## Agent skills

The `cosmosdb-best-practices` skill from the [Azure Cosmos DB Agent Kit](https://github.com/AzureCosmosDB/cosmosdb-agent-kit) is pre-installed in the Copilot cloud agent environment via `copilot-setup-steps.yml`. This skill contains multiple rules across various categories for validating technical accuracy of Azure Cosmos DB content.

**For VS Code users**: install locally with `npx skills add AzureCosmosDB/cosmosdb-agent-kit`.

When reviewing or authoring content in this docset, apply the `cosmosdb-best-practices` skill to validate technical accuracy. The skill covers data modeling, partition key design, query optimization, SDK usage, indexing, throughput, global distribution, monitoring, vector search, and full-text search.

Use the skill to verify that:

- Code samples follow current SDK best practices
- Documented patterns match the recommended approaches
- Prerequisites and caveats are complete
- Configuration examples are accurate

## Terminology

Product naming and branding rules for prose text in documentation.

### Product naming

| Preferred | Avoid in new content |
| --- | --- |
| Azure Cosmos DB | Cosmos DB (missing "Azure" prefix) |
| Azure Cosmos DB | Azure Cosmos DB for NoSQL |
| Azure Cosmos DB | Cosmos DB for NoSQL |

Use "Azure Cosmos DB" as the product name in new and substantially revised content. The previous convention of "Azure Cosmos DB for NoSQL" is acceptable in existing articles and doesn't need to be updated retroactively, but new content should use "Azure Cosmos DB" without the API suffix. This aligns with the URI consolidation from `/azure/cosmos-db/nosql` to `/azure/cosmos-db`.

Do not shorten to "Cosmos DB" — always include the "Azure" prefix.

### Exceptions

The following metadata fields are acceptable and should not be changed:

- `ms.service: azure-cosmos-db` - Microsoft Learn taxonomy value
- `ms.subservice: nosql` - Microsoft Learn taxonomy value
- `appliesto: ✅ NoSQL` - Standard documentation badge

## Content formatting

Code blocks, queries, and documentation structure guidelines.

### Content completeness

- Ensure account creation steps reference the correct Azure portal experience

### Code blocks

Use the appropriate fenced code block with the corresponding language identifier for SDK samples. Supported languages:

- `csharp` - .NET SDK
- `javascript` - Node.js SDK (JavaScript)
- `typescript` - Node.js SDK (TypeScript)
- `python` - Python SDK
- `java` - Java SDK
- `go` - Go SDK
- `rust` - Rust SDK

Example pattern:

````markdown
```csharp
using Microsoft.Azure.Cosmos;
CosmosClient client = new(endpoint, credential);
```
````

### Multi-language SDK samples

Use consecutive fenced code blocks with language identifiers for SDK samples.

Example pattern:

````markdown
```csharp
using Microsoft.Azure.Cosmos;
CosmosClient client = new(endpoint, credential);
```

```python
from azure.cosmos import CosmosClient
client = CosmosClient(endpoint, credential)
```

```javascript
const { CosmosClient } = require('@azure/cosmos');
const client = new CosmosClient({ endpoint, aadCredentials: credential });
```
````

### NoSQL queries

- Use `nosql` for Azure Cosmos DB query language
- Use `json` for query results or document examples

````markdown
```nosql
SELECT c.id, c.name FROM c WHERE c.status = "active"
```

```json
[
  { "id": "1", "name": "Example" }
]
```
````

### Azure CLI and infrastructure

- Use `azurecli` for Azure CLI commands
- Use `json` for ARM/Bicep output
- Use zone pivots (`zone_pivot_groups`) for portal/CLI/PowerShell/Bicep variations

## Technical review

When reviewing content for technical accuracy, apply the `cosmosdb-best-practices` skill. The skill provides detailed rules with code examples for each category below. Focus your review on verifying that the documentation accurately describes these topics — the skill handles the technical validation.

### Connection and authentication

- Verify connection strings use the correct Azure Cosmos DB format
- Ensure authentication examples show both connection string and Microsoft Entra ID options where applicable
- Prioritize Microsoft Entra ID authentication in examples, as connection string auth is not preferred

### Security

- Disable public network access and use private endpoints exclusively for production deployments
- Enable Network Security Perimeter (NSP) for additional network isolation boundaries
- Use managed identities for Azure service-to-service authentication - avoid embedding credentials in code
- Separate Azure identities for control plane (account management) and data plane (data operations)
- Use Azure RBAC for control plane operations and native RBAC for data plane access
- Enforce TLS 1.3 for transport security - use minimum TLS version configuration
