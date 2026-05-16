---
title: Use Azure portal integration with the Azure Cosmos DB Visual Studio Code extension
description: Learn how to move between Visual Studio Code and Azure portal when working with Azure Cosmos DB resources.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Use Azure portal integration with the Azure Cosmos DB Visual Studio Code extension

## What this feature does

Use integration actions to move between Visual Studio Code and Azure portal while working with Azure Cosmos DB resources.

## Open an account in the Azure portal from Visual Studio Code


1. In Visual Studio Code, open the Azure view.
1. Expand your Azure Cosmos DB account.
1. Right-click the account node and select the action to open in Azure portal.

:::image type="content" source="media/azure-portal/vs-code-open-portal.png" alt-text="Screenshot of opening a Cosmos DB account in the Azure portal from Visual Studio Code." lightbox="media/azure-portal/vs-code-open-portal.png":::

## Open in Visual Studio Code from Azure portal


1. In Azure portal, open your Azure Cosmos DB account.
1. Use the **Open in Visual Studio Code** action when available.
1. In Visual Studio Code, confirm the selected tenant and subscription if prompted.
1. Continue your workflow in the Azure extension by expanding the account, databases, and containers.

:::image type="content" source="media/azure-portal/portal-open-vs-code.png" alt-text="Screenshot of opening a Cosmos DB account in Visual Studio Code from the Azure portal." lightbox="media/azure-portal/portal-open-vs-code.png":::

## Open database and container experiences

Use Azure portal when you need operations that are outside your current editor flow, such as:

- Account-level configuration.
- Networking, security, and keys.
- Monitoring and diagnostics views.

Use Visual Studio Code when you want an editor-first workflow, such as:

- Query authoring and iteration.
- Item-level edits and validation.
- Local development with the emulator.

## Work between editor and portal

1. Make account-level changes in Azure portal.
1. Return to Visual Studio Code.
1. Refresh the Azure resource tree to pick up metadata changes.
1. Validate the change by running a query or opening container data.

## Troubleshooting

- If the wrong resource opens, verify tenant and subscription context in Visual Studio Code.
- If portal actions fail, sign in again and retry.
- If the **Open in Visual Studio Code** action is not available in Azure portal, open Visual Studio Code directly and browse to the same account from the Azure extension.
- If changes do not appear in Visual Studio Code, reload the window and refresh the node.

## Related articles

- [Browse Azure Cosmos DB accounts and databases in Visual Studio Code](browse-accounts-databases.md)
