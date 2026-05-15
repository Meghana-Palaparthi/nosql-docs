---
title: Execute Azure Cosmos DB SQL queries with syntax highlighting in Visual Studio Code
description: Learn how to run SQL queries in the Azure Cosmos DB Visual Studio Code extension using syntax highlighting and result views.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Execute Azure Cosmos DB SQL queries with syntax highlighting in Visual Studio Code

## What this feature does

The query editor helps you author and run Azure Cosmos DB SQL queries with:

- Syntax highlighting.
- Multiple results views (table, JSON, tree).
- Query execution from the editor.

## Open the query editor

1. In the Azure view, expand your Azure Cosmos DB account.
1. Expand a database and then a container.
1. Right-click the container and select the query editor action.

:::image type="content" source="media/sql-query-editor/sql-query-editor.png" alt-text="Azure Cosmos DB SQL query editor in Visual Studio Code with query input and result panes." lightbox="media/sql-query-editor/sql-query-editor.png":::

## Write and run a query

1. Enter a query in the editor.
1. Run the query using the query action in the editor toolbar.
1. Start with a validation query if needed:

```sql
SELECT TOP 10 * FROM c
```

4. Iterate by adding filters and ordering:

```sql
SELECT c.id, c.status, c.updatedAt
FROM c
WHERE c.status = "active"
ORDER BY c.updatedAt DESC
```

## Review results and stats

- Use table view to quickly scan fields.
- Use JSON view for full payload validation.
- Use tree view for nested documents.
- Check query statistics when tuning performance-sensitive queries.

## Save and rerun patterns

- Keep reusable query snippets in your workspace.
- Re-run baseline and tuned variants to compare behavior.

## Troubleshooting

- If query execution fails, validate SQL syntax and field names.
- If results are empty, run `SELECT TOP 10 * FROM c` to inspect sample documents.
- If access fails, verify account and data permissions in Azure.

## Related articles

- [CRUD operations on documents and collections](manage-documents-collections.md)
- [IntelliSense and auto-completion (preview)](intellisense-auto-completion-preview.md)
