---
title: Configure per-partition automatic failover
description: Configure per-partition automatic failover (PPAF) for an Azure Cosmos DB account to improve availability with partition-level failover.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: how-to
ms.date: 05/18/2026
ai-usage: ai-assisted
ms.custom:
  - build-2025
appliesto:
  - ✅ NoSQL
---

# Configure per-partition automatic failover for Azure Cosmos DB

This article explains how to configure per-partition automatic failover (PPAF) on your Azure Cosmos DB account.

**Per-partition automatic failover (PPAF)** is an Azure Cosmos DB feature that improves availability for single-write region accounts. Instead of failing over an entire database account during a regional outage, Azure Cosmos DB can **automatically fail over at the partition level**, which minimizes downtime and accelerates recovery.


## Prerequisites

Before enabling PPAF, ensure your environment meets the following **prerequisites**:

- **Multi-region account:** Single-write region account with **at least one** other **read region** configured.
- **Consistency model:** **Strong**, **Session**, **Consistent prefix**, or **Eventual** consistency are currently supported. **Bounded staleness** will be supported in a future release.
- **API type:** The account must use the **Core (SQL) API** (NoSQL API).
- **Azure region:** The account must be in a **global Azure region**
- **SDK version:** Your application must use a supported Azure Cosmos DB SDK that implements PPAF logic. The preview currently supports:
  - **.NET SDK v3** : v3.59.0 or later
  - **Java SDK**: v4.79.0 or later
  - **Python SDK**: v4.16.0 or later
  - **Node.js SDK**: v4.7.0 or later


## How to enable PPAF on your Azure Cosmos DB account

You can enable PPAF by using the Azure portal, Azure CLI, or Azure PowerShell.

> [!IMPORTANT]
> Before you enable per-partition automatic failover, confirm that your account meets every requirement in the [Prerequisites](#prerequisites) section and that **all** application instances are upgraded to a supported SDK version. Enabling PPAF with an unsupported SDK or a misconfigured account can cause availability issues, including failed writes during a partition-level failover.

#### [Azure portal](#tab/azure-portal)

1. Sign in to the [Azure portal](https://portal.azure.com/).
1. Navigate to your Azure Cosmos DB account.
1. In the left menu, select **Features** under the **Settings** section.
1. Select **Per-partition automatic failover**.
1. Review the information and prerequisites, and then switch to **Enable** PPAF.

   :::image type="content" source="media/how-to-configure-per-partition-automatic-failover/enable-per-partition-automatic-failover-portal.png" alt-text="Screenshot of the per-partition automatic failover feature in the Azure portal with the Enable toggle highlighted.":::

#### [Azure CLI](#tab/azure-cli)

1. Retrieve the existing capabilities on your account so that you don't accidentally remove any when you update it. The `az cosmosdb update` command replaces the full capability list, so you must include every existing capability along with `EnablePerPartitionAutomaticFailover`.

    ```azurecli-interactive
    az cosmosdb show \
      --resource-group "<resource-group-name>" \
      --name "<account-name>" \
      --query "capabilities"
    ```

1. Update the account by passing every existing capability returned in the previous step plus `EnablePerPartitionAutomaticFailover`.

    ```azurecli-interactive
    az cosmosdb update \
      --resource-group "<resource-group-name>" \
      --name "<account-name>" \
      --capabilities <existing-capability-1> <existing-capability-2> EnablePerPartitionAutomaticFailover
    ```

#### [Azure PowerShell](#tab/azure-powershell)

1. Retrieve the existing capabilities on your account. The `Update-AzCosmosDBAccount` cmdlet replaces the full capability list, so you must include every existing capability along with `EnablePerPartitionAutomaticFailover`.

    ```azurepowershell-interactive
    $account = Get-AzCosmosDBAccount -ResourceGroupName "<resource-group-name>" -Name "<account-name>"
    $account.Capabilities.Name
    ```

1. Update the account by passing every existing capability returned in the previous step plus `EnablePerPartitionAutomaticFailover`.

    ```azurepowershell-interactive
    Update-AzCosmosDBAccount `
      -ResourceGroupName "<resource-group-name>" `
      -Name "<account-name>" `
      -Capabilities "<existing-capability-1>", "<existing-capability-2>", "EnablePerPartitionAutomaticFailover"
    ```

---

## PPAF pricing

PPAF is part of the Business Critical service tier and is charged accordingly. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

## Configure the application for PPAF

Configuring your application's Azure Cosmos DB SDK is **critical** so that it knows to handle partition-level failovers.

- **Upgrade SDK:** Make sure your app is running the **latest SDK version** that supports PPAF (as identified in [Prerequisites](#prerequisites)).
- **Configure secondary region:** Make sure your Azure Cosmos DB account has at least one secondary region.

## Test the PPAF setup (simulate a fault)

With the account and client configured, validate that everything works as expected before a real outage occurs. Azure Cosmos DB provides a way to simulate partition failures in the preview for PPAF-enabled accounts:

- **Chaos simulation (preview):** A preview of the fault-management feature for PPAF is available via REST API. For ease of use, a PowerShell script is provided to manage the fault.
  - Download the script [`EnableDisableChaosFault.ps1` at azurecosmosdb/ppaf-samples](https://github.com/AzureCosmosDB/ppaf-samples/blob/main/ppaf-fault-script/EnableDisableChaosFault.ps1).
  - Start PowerShell and sign in to your subscription by running `az login`.
  - Navigate to the folder that contains the PowerShell script and invoke it with the required parameters to inject the fault:
    - It might take up to 15 minutes for the fault to take effect.
    - The fault is applied to 10% of the partitions in the specified collection, with a maximum of 10 partitions and a minimum of 1 partition.

    ```powershell
    .\EnableDisableChaosFault.ps1 -FaultType "PerPartitionAutomaticFailover" -ResourceGroup "{ResourceGroupName}" -AccountName "{DatabaseAccountName}" -DatabaseName "{DatabaseName}" -ContainerName "{CollectionName}"  -SubscriptionId "{SubscriptionId}" -Region "{PreferredWriteRegion}" -Enable
    ```

- **Application testing:** Test critical transactions of your application during the failover.
- **Metrics:**
  - Verify the traffic in the Azure portal **Metrics** blade for your account. Look at metrics like **Total Requests** broken down by region. You should see write operations occurring in a secondary region during the simulation, confirming the failover worked.
  - A new metric named **PartitionWriteGlobalStatus** reports the count of write partitions for a region at any given time. Use this metric to track how many partitions failed over due to the fault.

- **Disable the fault:** Invoke the same script with the `-Disable` switch to remove the fault. It might take up to 15 minutes for the fault to be disabled.

    ```powershell
    .\EnableDisableChaosFault.ps1 -FaultType "PerPartitionAutomaticFailover" -ResourceGroup "{ResourceGroupName}" -AccountName "{DatabaseAccountName}" -DatabaseName "{DatabaseName}" -ContainerName "{CollectionName}"  -SubscriptionId "{SubscriptionId}" -Region "{PreferredWriteRegion}" -Disable
    ```

## Related content

- [High availability in Azure Cosmos DB](high-availability.md)
- [Consistency levels in Azure Cosmos DB](consistency-levels.md)
- [Distribute your data globally with Azure Cosmos DB](distribute-data-globally.md)