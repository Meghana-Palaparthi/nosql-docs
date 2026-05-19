---
title: Connect Visual Studio Code to the local Azure Cosmos DB emulator
description: Learn how to connect the Azure Cosmos DB Visual Studio Code extension to a local Azure Cosmos DB emulator.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
---

# Connect Visual Studio Code to the local Azure Cosmos DB emulator

## What this feature does

The extension can connect to a local emulator for offline or preproduction development.

## Prerequisites

- Azure Cosmos DB Emulator installed and running.
- Visual Studio Code with the Azure Databases extension.

## Connect to the emulator

1. Start the Azure Cosmos DB Emulator on your machine.
1. In Visual Studio Code, open the Azure view.
1. Use the extension action to connect to a local emulator endpoint.
1. Provide the emulator endpoint and key when prompted.

:::image type="content" source="media/connect-local-emulator/connect-emulator.png" alt-text="Screenshot of connect to local emulator dialog in Visual Studio Code for Azure Cosmos DB with endpoint and key fields." lightbox="media/connect-local-emulator/connect-emulator.png":::

Common local endpoint:

```text
https://localhost:8081
```

## Validate the connection

1. Expand the emulator connection in the resource tree.
1. Open a database and container.
1. Run a quick query:

```sql
SELECT TOP 10 * FROM c
```

## Troubleshooting

- If TLS warnings appear, verify emulator certificates are trusted.
- If connection fails, confirm the emulator is running before connecting.
- If results are empty, verify that data exists in the local container.

## Related articles

- [Import and export data](import-export-data.md)
