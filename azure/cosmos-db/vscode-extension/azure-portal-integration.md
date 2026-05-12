---
title: Use Azure portal integration from the Azure Cosmos DB Visual Studio Code extension
description: Learn how to open and manage Azure Cosmos DB resources in the Azure portal directly from the Visual Studio Code extension.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Use Azure portal integration from the Azure Cosmos DB Visual Studio Code extension

## What this feature does

Use context actions in Visual Studio Code to open Azure Cosmos DB resources in the Azure portal for advanced configuration tasks.

## Open an account in the Azure portal

1. In Visual Studio Code, open the Azure view.
1. Expand your Azure Cosmos DB account.
1. Right-click the account node and select the action to open in Azure portal.

## Open database and container experiences

Use Azure portal when you need operations that are outside your current editor flow, such as:

- Account-level configuration.
- Networking, security, and keys.
- Monitoring and diagnostics views.

## Work between editor and portal

1. Make account-level changes in Azure portal.
1. Return to Visual Studio Code.
1. Refresh the Azure resource tree to pick up metadata changes.
1. Validate the change by running a query or opening container data.

## Troubleshooting

- If the wrong resource opens, verify tenant and subscription context in Visual Studio Code.
- If portal actions fail, sign in again and retry.
- If changes do not appear in Visual Studio Code, reload the window and refresh the node.

## Related articles

- [Browse Azure Cosmos DB accounts and databases in Visual Studio Code](browse-accounts-databases.md)
