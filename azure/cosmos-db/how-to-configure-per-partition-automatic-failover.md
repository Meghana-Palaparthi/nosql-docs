---
title: Configure and Use Per Partition Automatic Failover
description: Learn how to enable and use Per Partition Automatic Failover for Azure Cosmos DB
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/14/2025
ms.custom:
  - build-2025
appliesto:
  - ✅ NoSQL
---

# How to onboard and adopt Per-Partition Automatic Failover (PPAF) for Azure Cosmos DB

This article explains how to configure Per Partition Automatic Failover on your Azure Cosmos DB account.

**Per-Partition Automatic Failover (PPAF)**  is a new Azure Cosmos DB feature that improves availability for single-write region accounts. Instead of failing over an entire database account during a regional outage, Cosmos DB can **automatically fail over at the *partition level***, thus minimizing downtime and faster recovery. 


## Prerequisites

Before enabling PPAF, ensure your environment meets the following **prerequisites**:

- **Multi-region account:** Single-write region account with **at least one** other **read region** configured.
- **Consistency Model:** **Strong**, **Session**, **Consistent Prefix**, or **Eventual** consistencies are currently supported. **Bounded Staleness** will be supported in a future release.
- **API Type:** The account must use the **Core (SQL) API** (NoSQL API).
- **Azure Region:** The account should be in **Azure public cloud regions** (Global Azure). Accounts in sovereign clouds are not supported. .
- **SDK Version:** Your application must use a **latest supported Azure Cosmos DB SDK** that implements PPAF logic. Currently, the preview supports:
  - **.NET SDK v3** : v 3.59.0 or later
  - **Java SDK**: v 4.75.0 or later
  - **Python SDK**: v 4.15.0 or later
  - **Node.js SDK**: v 4.7.0 or later


## How to enable PPAF on your Azure Cosmos DB account

You can enable PPAF by using the Azure portal, Azure CLI, or Azure PowerShell.

> [!IMPORTANT]
> Before you enable Per Partition Automatic Failover, confirm that your account meets every requirement in the [Prerequisites](#prerequisites) section and that **all** application instances are upgraded to a supported SDK version. Enabling PPAF with an unsupported SDK or a misconfigured account can cause availability issues, including failed writes during a partition-level failover.

#### [Azure portal](#tab/azure-portal)

1. Sign in to the [Azure portal](https://portal.azure.com/).
1. Navigate to your Azure Cosmos DB account.
1. In the left menu, select **Features** under the **Settings** section.
1. Select **Per Partition Automatic Failover**.
1. Review the information and prerequisites, and then switch to **Enable** PPAF.

   :::image type="content" source="media/how-to-configure-per-partition-automatic-failover/enable-per-partition-automatic-failover-portal.png" alt-text="Screenshot of the Per Partition Automatic Failover feature in the Azure portal with the Enable toggle highlighted.":::

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

## PPAF Pricing
PPAF is part of Business Critical Service Tier and is charged accordingly. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

## Configure the application for PPAF

Configuring your application’s Cosmos DB SDK is **critical** so that it knows to handle partition-level failovers. 

- **Upgrade SDK:** Ensure your app is running the **latest SDK version** that supports PPAF (as identified in prerequisites).
- **Configure secondary region:** Ensure your Azure Cosmos DB account has at least 1 secondary region.

## Test the PPAF Setup (Simulate Fault)

With the account and client configured, it’s prudent to **test** that everything works as expected before a real outage occurs. Azure Cosmos DB provides a way to simulate partition failures in the preview for PPAF enabled accounts:

- **Chaos Simulation (Preview):** We're releasing a preview version of the fault management feature for PPAF via REST API. For ease of use, we're providing a PowerShell script for managing the fault.
  - Download the script [`EnableDisableChaosFault.ps1` at azurecosmosdb/ppaf-samples](https://github.com/AzureCosmosDB/ppaf-samples/blob/main/ppaf-fault-script/EnableDisableChaosFault.ps1).
  - Start PowerShell and login to your subscription using "az login."
  - Navigate to the folder with the PowerShell script and invoke the script with the required parameters to invoke the fault: 
    - It might take up to 15 minutes for the fault to become effective.
    - The fault gets effective on 10% of total partition for the specified collection with a maximum of 10 partition and minimum 1 Partition.
    ``` powershell
    .\EnableDisableChaosFault.ps1 -FaultType "PerPartitionAutomaticFailover" -ResourceGroup "{ResourceGroupName}" -AccountName "{DatabaseAccountName}" -DatabaseName "{DatabaseName}" -ContainerName "{CollectionName}"  -SubscriptionId "{SubscriptionId}" -Region "{PreferredWriteRegion}" -Enable
    ```

- **Application Testing:** Test critical transactions of your application during the failover.
- **Metrics:** 
  - You can verify the traffic in the Azure portal Metrics for your account. Look at metrics like **Total Requests** broken down by region. You should see write operations occurring in a secondary region during the simulation, confirming the failover worked.
  - We have introduced a new metric known as **PartitionWriteGlobalStatus** that shows the count of write partitions for a region at any given time. You can also use this metric to track how many partitions are failed over due to fault. 

- **Disable the fault:**
  - Navigate to the folder with the PowerShell script and invoke the script with the required parameters to invoke the fault: 
    - It might take up to 15 minutes for the fault to be disabled.
    ```powershell 
    .\EnableDisableChaosFault.ps1 -FaultType "PerPartitionAutomaticFailover" -ResourceGroup "{ResourceGroupName}" -AccountName "{DatabaseAccountName}" -DatabaseName "{DatabaseName}" -ContainerName "{CollectionName}"  -SubscriptionId "{SubscriptionId}" -Region "{PreferredWriteRegion}" -Disable
    ```
