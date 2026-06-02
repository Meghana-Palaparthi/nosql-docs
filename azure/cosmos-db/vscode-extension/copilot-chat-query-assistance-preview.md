---
title: Use Copilot Chat query assistance in the Azure Cosmos DB Visual Studio Code extension (preview)
description: Learn how Copilot Chat can help you create and troubleshoot Azure Cosmos DB queries in Visual Studio Code.
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

# Use Copilot Chat query assistance in the Azure Cosmos DB Visual Studio Code extension (preview)

## What this feature does

This preview capability uses Copilot Chat context to help draft and refine Azure Cosmos DB queries.

## Prerequisites

- Visual Studio Code with GitHub Copilot and GitHub Copilot Chat enabled.
- Azure Databases extension with preview build installed.
- Connected Azure Cosmos DB for NoSQL account and container context.

## Use the @cosmosdb participant

In Chat, invoke the participant with natural prompts or slash commands.

:::image type="content" source="media/chat-query-assistant/chat-query-assistant.png" alt-text="Copilot Chat in Visual Studio Code using the @cosmosdb participant to generate and refine Azure Cosmos DB queries." lightbox="media/chat-query-assistant/chat-query-assistant.png":::

Example commands:

```text
@cosmosdb /help
@cosmosdb /generateQuery Find documents where status = 'pending'
@cosmosdb /explainQuery SELECT * FROM c WHERE c.status = 'pending'
@cosmosdb /editQuery Add ORDER BY c._ts DESC
@cosmosdb /question What is the difference between IS_DEFINED and IS_NULL?
```

## Suggested workflow

1. Generate a query from natural language.
1. Run it in the query editor.
1. Use explain to validate semantics.
1. Use edit to refine projection, filters, ordering, or pagination.

## Improve answer quality

- Provide exact field names when possible.
- Include sort and pagination intent explicitly.
- Keep prompts scoped to one task per request.

## Troubleshooting

- If `@cosmosdb` is not recognized, verify preview extension installation and reload the window.
- If Copilot Chat is unavailable, verify sign-in and Copilot access.
- If explain or edit fails, verify the source query is valid.
- If responses are off-target, rephrase with explicit schema and filter details.

## Related articles

- [Use natural language to query in the Azure Cosmos DB Visual Studio Code extension (preview)](natural-language-to-query-preview.md)
