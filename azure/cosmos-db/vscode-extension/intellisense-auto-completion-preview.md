---
title: Use IntelliSense and auto-completion in the Azure Cosmos DB query editor (preview)
description: Learn how to use IntelliSense and auto-completion in the Azure Cosmos DB query editor in Visual Studio Code.
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

# Use IntelliSense and auto-completion in the Azure Cosmos DB query editor (preview)

## What this feature does

This preview capability helps accelerate query authoring with contextual completions.

## Prerequisites

- Azure Databases extension preview build.
- Connected Azure Cosmos DB account and container.
- Query editor opened on a target container.

## Trigger completions

1. Open the query editor.
1. Start typing a query, for example:

```sql
SELECT c.
FROM c
WHERE c.
```

3. Use completion suggestions to insert field names and query tokens.

## Example authoring flow

```sql
SELECT c.id, c.status, c.updatedAt
FROM c
WHERE c.status = "active"
ORDER BY c.updatedAt DESC
```

Use completions while building projection, filter, and sort clauses.

## Troubleshooting

- If no suggestions appear, confirm you are in a query editor for a connected container.
- If fields are missing, run a baseline query first so schema context can be refreshed.
- If behavior is inconsistent, reload Visual Studio Code and reopen the container query editor.

## Limitations

- Suggestions can be partial for highly dynamic or sparse document structures.
- Complex expressions can still require manual SQL authoring.

## Related articles

- [Execute Azure Cosmos DB SQL queries with syntax highlighting in Visual Studio Code](sql-query-editor.md)
