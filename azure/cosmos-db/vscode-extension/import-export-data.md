---
title: Import and export Azure Cosmos DB data in Visual Studio Code
description: Learn how to import documents into Azure Cosmos DB and export documents or query results by using the Visual Studio Code extension.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Import and export Azure Cosmos DB data in Visual Studio Code

## What this feature does

The extension supports data movement operations for local workflows:

- Import documents from JSON files.
- Export documents or query results for validation and sharing.

## Import documents from JSON

1. Open the target container in Visual Studio Code.
1. Select the import action.
1. Choose a JSON file from your local machine.
1. Confirm the import and monitor completion.
1. Validate imported items by querying the container.

:::image type="content" source="media/import-export/import-documents.png" alt-text="Import documents workflow in Visual Studio Code showing JSON file selection for an Azure Cosmos DB container." lightbox="media/import-export/import-documents.png":::

## Export documents from a container

1. Open the container documents view.
1. Select the export action.
1. Choose full container export or current filtered view.
1. Save output as JSON.

:::image type="content" source="media/import-export/export-documents.png" alt-text="Export documents action in Visual Studio Code for Azure Cosmos DB showing options to export container data." lightbox="media/import-export/export-documents.png":::

## Export query results

1. Run a query in the query editor.
1. In results view, choose export.
1. Save results in the supported output format.

## Validation and troubleshooting

- Validate required fields such as `id` and partition key after import.
- If import fails, verify source JSON validity and document shape.
- If export appears incomplete, confirm whether current filters limit rows.

## Related articles

- [Perform CRUD operations on Azure Cosmos DB documents and collections in Visual Studio Code](manage-documents-collections.md)
