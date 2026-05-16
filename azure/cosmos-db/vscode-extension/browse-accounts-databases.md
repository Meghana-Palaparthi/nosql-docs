---
title: Browse Azure Cosmos DB accounts and databases in Visual Studio Code
description: Learn how to browse Azure Cosmos DB accounts, databases, and containers in the Visual Studio Code extension.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Browse Azure Cosmos DB accounts and databases in Visual Studio Code

## What this feature does

Use the extension's resource explorer to:

- View subscriptions and Azure Cosmos DB accounts.
- Expand databases and containers.
- Open context actions from account and container nodes.

## Prerequisites

- Visual Studio Code with the Azure Databases extension installed.
- An Azure account with permission to view Azure Cosmos DB resources.
- At least one Azure Cosmos DB account in your selected subscription.

## Sign in and load resources

1. Open Visual Studio Code.
1. Select the Azure icon in the activity bar.
1. If prompted, sign in to Azure.
1. Expand your subscription.
1. Expand Azure Cosmos DB to list available accounts.

## Browse account, database, and container resources

1. Expand an Azure Cosmos DB account.
1. Expand a database to view containers.
1. Select a container node to view actions such as opening a query editor or item view.

:::image type="content" source="media/browse-accounts-databases/browse-accounts-databases.png" alt-text="Screenshot of Azure Cosmos DB resource tree in Visual Studio Code showing an expanded account with databases and containers." lightbox="media/browse-accounts-databases/browse-accounts-databases.png":::

## Useful navigation tips

- Refresh a node when you create resources outside Visual Studio Code.
- Right-click account, database, and container nodes to discover available actions.
- Use container-level actions to jump directly to query and document workflows.

## Troubleshooting

- If no accounts appear, verify that the correct subscription is selected.
- If you get authorization errors, verify your Azure role assignments.
- If the tree view is stale, reload the window and refresh the Azure node.

## Related articles

- [Visual Studio Code extension for Azure Cosmos DB](overview.md)
