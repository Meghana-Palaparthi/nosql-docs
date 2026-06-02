---
title: Perform CRUD operations on Azure Cosmos DB documents and collections in Visual Studio Code
description: Learn how to create, read, update, and delete items and manage collections in the Azure Cosmos DB Visual Studio Code extension.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Perform CRUD operations on Azure Cosmos DB documents and collections in Visual Studio Code

## What this feature does

Use the extension to:

- Create and view documents.
- Edit and delete documents.
- Perform collection-level actions from context menus.

## Open documents in a container

1. In the Azure view, expand your Azure Cosmos DB account.
1. Expand the target database and container.
1. Open the documents or items view for that container.

:::image type="content" source="media/manage-documents/manage-documents.png" alt-text="Azure Cosmos DB items view in Visual Studio Code with document list and JSON editor for create, edit, and delete operations." lightbox="media/manage-documents/manage-documents.png":::

## Create a document

1. Select the action to add a new document.
1. Enter JSON that includes required fields such as `id` and partition key.
1. Save and verify the new item appears in the container list.

## Edit a document

1. Open an existing document.
1. Modify fields in the JSON payload.
1. Save changes and verify update behavior in the list or query results.

## Delete documents

1. Select one or more documents.
1. Choose the delete action.
1. Confirm deletion and verify results.

## Best practices

- Avoid changing partition key values for existing documents.
- Use focused queries to confirm updates before and after edits.
- Make bulk changes through controlled scripts when document count is high.

## Related articles

- [Import and export data](import-export-data.md)
