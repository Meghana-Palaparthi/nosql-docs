---
title: Use natural language to query in the Azure Cosmos DB Visual Studio Code extension (preview)
description: Learn how to generate Azure Cosmos DB SQL queries from natural language prompts in the Visual Studio Code extension.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
ms.custom: preview
ms.collection:
  - ce-skilling-ai-copilot
ms.update-cycle: 180-days
---

# Use natural language to query in the Azure Cosmos DB Visual Studio Code extension (preview)

## What this feature does

This preview capability helps you convert plain-language prompts into Azure Cosmos DB SQL queries.

## Prerequisites

- Visual Studio Code 1.103 or later.
- Azure Databases extension with preview features enabled.
- GitHub Copilot and GitHub Copilot Chat available in Visual Studio Code.
- Access to an Azure Cosmos DB for NoSQL account, or a local emulator setup.

For private preview setup and validation assets, see the preview repository: [cosmosdb-vscode-ai-assistant-preview](https://github.com/AzureCosmosDB/cosmosdb-vscode-ai-assistant-preview).

## Generate a query from natural language

1. Open your Azure Cosmos DB container in the query editor.
1. Open the AI generation flow in the query editor.
1. Enter a prompt that describes the result shape, filters, and sorting.
1. Insert the generated query into the editor.
1. Run the query and validate results.

:::image type="content" source="media/natural-language-to-query/natural-language-to-query.png" alt-text="Natural language to query experience in Visual Studio Code generating Azure Cosmos DB SQL from a plain-language prompt." lightbox="media/natural-language-to-query/natural-language-to-query.png":::

Example prompts:

```text
Find active orders from the last 30 days sorted by newest first.
Return top 20 users with the highest loyaltyPoints.
Show products where inventoryCount is less than 10.
```

## Refine generated output

If the first query is close but not exact, refine with a follow-up prompt such as:

```text
Keep the filters, return only id and status, and add ORDER BY c._ts DESC.
```

Best results come from prompts that include:

- Expected fields.
- Exact filter criteria.
- Sort and pagination requirements.

## Validate correctness and performance

1. Confirm field names and value types against your container schema.
1. Verify partition key assumptions for large containers.
1. Check query results in table, JSON, or tree view.
1. Re-run with narrower filters if RU consumption is high during iteration.

## Known limitations and preview guidance

- Generated queries can require manual refinement for complex schemas.
- Schema sampling and query history can influence follow-up suggestions.
- Keep preview test datasets non-sensitive.
- Validate AI assistant usage with your organization policy before testing production-like data.

## Troubleshooting

- If generation fails, confirm container context is selected in the query editor.
- If results do not match intent, rewrite the prompt with explicit fields and filters.
- If query output is empty, test with `SELECT TOP 10 * FROM c` to inspect available data.

## Related articles

- [Execute Azure Cosmos DB SQL queries with syntax highlighting in Visual Studio Code](sql-query-editor.md)
- [Copilot Chat query assistance (preview)](copilot-chat-query-assistance-preview.md)
