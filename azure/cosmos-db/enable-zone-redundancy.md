---
title: Enable zone redundancy on an Azure Cosmos DB account
description: Learn how to enable zone redundancy on an Azure Cosmos DB account.
author: seesharprun
ms.author: sidandrews
ms.topic: how-to
ms.service: azure-cosmos-db
ms.date: 03/20/2026
---

# Enable zone redundancy on an Azure Cosmos DB account

With zone redundancy, Azure Cosmos DB intelligently distributes the four replicas of your data across multiple availability zones. This ensures that, in the event of an outage in one zone, the account remains fully operational. In contrast, without zone redundancy, all replicas might be located in a single availability zone (we don't expose which), leading to potential downtime if that specific zone experiences an issue. Enabling zone redundancy is a great way to increase resilience of your Cosmos DB database without introducing additional application complexities, affecting performance, or even incurring additional costs, if autoscale is also used. For more information, see [Reliability in Azure Cosmos DB - Resilience to availability zone failures](/azure/reliability/reliability-cosmos-db#resilience-to-availability-zone-failures).

Enabling zone redundancy isn't an account-wide choice. A single Cosmos DB account can span an arbitrary number of Azure regions, each of which can independently be configured to enable zone redundancy. Some regions don't provide availability zone support. This is important, as some regions don't yet support availability zones, but adding them to an Azure Cosmos DB account doesn't prevent enabling zone redundancy in other regions configured for that account.

> [!NOTE]
> If you receive an error during deployment indicating the region is constrained and you can't enable zone redundancy, [open a support request](/azure/azure-portal/supportability/how-to-create-azure-support-request) to request capacity in the region's zones.

## Prerequisites

Before configuring zone redundancy, review the requirements and details listed in [Reliability in Azure Cosmos DB - Resilience to availability zone failures](/azure/reliability/reliability-cosmos-db#resilience-to-availability-zone-failures).

## Create a new zone-redundant account

When you create a new Azure Cosmos DB account, you can configure zone redundancy on one or more regions by using these instructions:

- [Azure portal](./quickstart-portal.md). When deploying, set the *Availability Zones* setting to *Enabled*.
- [Azure CLI](./how-to-create-account.md?tabs=azure-cli). When setting the `--locations` argument, set `isZoneRedundant=True` for the regions you want to make zone-redundant.
- [Bicep](./quickstart-template-bicep.md). Update the `isZoneRedundant` property to `true` for the regions you want to make zone-redundant.
- [Azure Resource Manager templates](./quickstart-template-json.md). Update the `isZoneRedundant` property to `true` for the regions you want to make zone-redundant.

## Enable zone redundancy on an existing account

You can't enable zone redundancy in a region that has already been added to your account, so you need to remove that region and add it again with zone redundancy enabled. To avoid any service disruption, you add a temporary region and fail over to it until the zone redundancy configuration is complete.

> [!IMPORTANT]
> When you follow the process described in this article, a small amount of write unavailability (a few seconds) occurs when adding and removing the secondary region, as the system deliberately stops writes in order to check consistency between regions.
>
> Additionally, you incur cost for the secondary region and for the data replication. Zone-redundant accounts are charged at a different rate. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

Follow the steps below to enable zone redundancy for your account in select regions.

# [Azure portal](#tab/portal)

1. Add a temporary region to your database account by following steps in [Add region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

1. Wait for the newly added region to be marked as *Available*.

    When you add a new region to your account, Azure replicates and commits all data into the new region before marking the region as available. The amount of time this operation takes depends upon how much stored data is in the account.

1. If your Azure Cosmos DB account is configured with multi-region writes, skip to the next step.

    Otherwise, change the account's write region to the newly added temporary region. Follow the steps in [Set failover priorities for your Azure Cosmos DB account](/azure/cosmos-db/how-to-manage-database-account#set-failover-priorities-for-your-azure-cosmos-db-account).

1. Remove the region for which you would like to enable zone redundancy by following steps in [Remove region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

1. Add back the region with zone redundancy enabled:
    1. [Add region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).
    1. Find the newly added region in the **Write region** column, and enable **Availability Zone** for that region.
    1. Select **Save**.

1. Wait for the newly added region to be marked as *Available*.

1. Change the account's write region to the newly zone-redundant region. Following the steps in [Set failover priorities for your Azure Cosmos DB account](/azure/cosmos-db/how-to-manage-database-account#set-failover-priorities-for-your-azure-cosmos-db-account).

1. Remove the temporary region by following steps in [Remove region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

# [Azure CLI](#tab/cli)

The following example shows how to add West US as a temporary region to an account that's configured to only use the East US region.

> [!IMPORTANT]
> You must include all of your account's existing regions, and any new regions, in these commands.

1. Add a temporary region to your database account.

    The following example shows how to add West US as a secondary region to an account configured with East US region only. You must include all of your account's existing regions as well as any new regions in these commands.

    ```azurecli
    az cosmosdb update \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --locations regionName=eastus failoverPriority=0 isZoneRedundant=False \
        --locations regionName=westus failoverPriority=1 isZoneRedundant=False
    ```

1. Wait for the newly added region to be marked as *Available*.

    When you add a new region to your account, Azure replicates and commits all data into the new region before marking the region as available. The amount of time this operation takes depends upon how much stored data is in the account.

1. If your Azure Cosmos DB account is configured to use multi-region writes, skip to the next step.

    Otherwise, change the account's write region to the newly added temporary region.

    The following example shows how to perform a failover from East US region (current write region) to West US region (current read-only region). You must include both regions in the command.

    ```azurecli
    az cosmosdb failover-priority-change \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --failover-policies westus=0 eastus=1
    ```

1. Remove the region for which you would like to enable zone redundancy.

    The following example shows how to remove East US region from an account configured with West US (write region) and East US (read-only) regions. You must include all regions that shouldn't be removed in the command.

    ```azurecli
    az cosmosdb update \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --locations regionName=westus failoverPriority=0 isZoneRedundant=False
    ```

1. Add back the region, and set the `isZoneRedundant=True` argument.

    The following example shows how to add East US as a zone redundancy-enabled secondary region to an account configured with West US region only. You must include any existing regions and all new ones in the command.

    ```azurecli
    az cosmosdb update \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --locations regionName=westus failoverPriority=0 isZoneRedundant=False \
        --locations regionName=eastus failoverPriority=1 isZoneRedundant=True
    ```

1. Wait for the newly added region to be marked as *Available*.

1. Change the account's write region to the newly zone redundant region.

    The following example shows how to change the write region from West US region (current write region) to East US region (current read-only region). You must include both regions in the command.

    ```azurecli
    az cosmosdb failover-priority-change \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --failover-policies eastus=0 westus=1
    ```

1. Remove the temporary region.

    The following example shows how to remove West US region from an account configured with East US (write region) and West US (read-only) regions. You must include all accounts that shouldn't be removed in the command.

    ```azurecli
    az cosmosdb update \
        --name MyCosmosDBDatabaseAccount \
        --resource-group MyResourceGroup \
        --locations regionName=eastus failoverPriority=0 isZoneRedundant=True
    ```

---

## Related content

- [Reliability in Azure Cosmos DB](/azure/reliability/reliability-cosmos-db?context=/azure/cosmos-db/context/context)
- [Relocate an Azure Cosmos DB NoSQL account to another region](/azure/azure-resource-manager/management/relocation/relocation-cosmos-db)
