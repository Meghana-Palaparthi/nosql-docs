---
title: Migrate to Azure DocumentDB
titleSuffix: Azure Cosmos DB for MongoDB
description: Learn how to migrate from Azure Cosmos DB for MongoDB to Azure DocumentDB using built-in migration tools. Step-by-step guide with prerequisites and job monitoring.
author: sandeepsnairms
ms.author: sandnair
ms.service: azure-cosmos-db
ms.subservice: mongodb
ms.topic: how-to
ms.date: 05/18/2026
appliesto:
  - MongoDB
---

# Migrate from Azure Cosmos DB for MongoDB to Azure DocumentDB

[!INCLUDE[Note - Recommended services](includes/note-recommended-services.md)]

In this guide, you take an existing collection and migrate it from Azure Cosmos DB for MongoDB to Azure DocumentDB using the tooling built-in to the service and Azure portal.

## Prerequisites

- An Azure subscription

    - If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn) before you begin.

- A target Azure DocumentDB cluster

    - If you don't already have a cluster, [create a new Azure DocumentDB cluster](../../documentdb/quickstart-portal.md).
    
    - Ensure that you have the **native authentication credentials** required to connect to your cluster. 

- An Azure Key Vault account (not required if your Azure DocumentDB supports Microsoft Entra ID authentication)

    - Ensure that the Key Vault account is enabled for role-based access control. For more information, see [role-based access control in Azure Key Vault](/azure/key-vault/general/rbac-guide).

## Configure managed identity for your source account 

1. Sign in to the Azure portal (<https://portal.azure.com>).

1. Navigate to your source Azure Cosmos DB for MongoDB account.

1. In your source account, navigate to **Settings > Identity**.

1. Turn on the **system-assigned** managed identity for your source account by setting the **Status** option to **On**. Note down the value of the **Object (principal) ID** to use later in this guide.
    
    :::image source="media/how-to-migrate-documentdb/configure-identity.png" alt-text="Screenshot of the Azure portal Identity settings page showing system-assigned managed identity configuration options.":::

    > [!TIP]
    > If you're using a user-assigned managed identity instead, ensure that at least one user-assigned managed identity is assigned to the source account.

1. Run the command to update your source account to use the preferred identity mechanism as the default identity.
    
    ```azurecli      
    az cosmosdb update \
        --resource-group "<resource-group-name>" \
        --name "<source-account-name>" \
        --default-identity "SystemAssignedIdentity"
    ```

    > [!TIP]
    > If you're using a user-assigned managed identity instead, run this command:
    >
    > ```azurecli-interactive
    > az cosmosdb update \
    >     --resource-group "<resource-group-name>" \
    >     --name "<source-account-name>" \
    >     --default-identity "UserAssignedIdentity=<fully-qualified-resource-id-of-user-assigned-managed-identity>"
    > ```

## Set up Azure Key Vault (Optional)

> [!TIP]
> The migration job can use the target DocumentDB's native authentication credentials or use Microsoft Entra ID to connect. You don't need to set up Azure Key Vault if you're planning to use Microsoft Entra ID authentication. 

When using  DocumentDB's native authentication, the migration job uses an Azure Key Vault to retrieve the target  authentication credentials.

To store the credentials in your existing key vault.

1. Navigate to your existing key vault.

1. If the Key Vault uses the **Role-Based Access Control (RBAC)** permission model, select the **Access Control (IAM)** option in the resource menu and assign the **Key Vault Secret User** role to the principal ID (object ID) of the managed identity used for your source account. Otherwise, use the **Access policies** option in the resource menu to create an access policy with **Get** and **List Secret** permissions, then assign it to the principal ID (object ID).
 
1. Navigate to **Objects > Secrets**, select **Generate/Import** to create a new secret. Use these values for your secret:

    | | Description |
    | --- | --- |
    | **Name** | Secret names are used to identify the secret and can only contain alphanumeric characters and dashes. This value is eventually used in the migration job's **Secret Name** field. |
    | **Secret value** | Paste the native authentication credentials for your Azure Cosmos DB for MongoDB target cluster here. |

    > [!NOTE]
    > Ensure you use the native DocumentDB connection string as the **Secret value**.

1. In your newly created secret, gather the value of **Vault URI**. This value is eventually used in the migration job's **Vault URI** field.

## Create migration job

First, create a migration job with the configuration required to start migrating data to the target cluster.

1. Sign in to the Azure portal (<https://portal.azure.com>).

1. Navigate to your Azure Cosmos DB for MongoDB account again.

1. On the account page, select **Migrate to DocumentDB** from the resource menu.

    :::image source="media/how-to-migrate-documentdb/home.png" alt-text="Screenshot of home page in the migration to DocumentDB workflow.":::

1. Select **Start a new migration job**.

## Select migration mode

The **Select Migration Mode** section is used to provide the migration mode that's most appropriate for your migration needs.

1. Select the appropriate mode from these options:

    | | Description |
    | --- | --- |
    | **Offline** | Offline migration captures a snapshot of the collection at the beginning, offering a simpler and predictable approach. It works well when using a static copy of the collection is acceptable, and real-time updates aren't essential. Use this option for nonproduction migrations. |
    | **Online** | Online migration copies collection data, ensuring updates are also replicated during the process. This method is advantageous with minimal downtime, allowing continuous operations for business continuity. Use this option when ongoing operations are crucial, and reducing downtime is a priority. |

    :::image source="media/how-to-migrate-documentdb/mode-selection.png" alt-text="Screenshot of the mode selection options for a migration job.":::

    > [!IMPORTANT]
    > Continuous Backup is a prerequisite for online migrations. For more information, see [continuous backup](../continuous-backup-restore-introduction.md).

1. Select **Next**.

## Configure target migration credentials

The **Select Target Account** section is used to provide the connection details to the target Azure DocumentDB cluster. The migration job can use the target DocumentDB's native authentication credentials or use Microsoft Entra ID to connect.

As a security best practice, Microsoft Entra ID is the preferred authentication mechanism.

### Configure using DocumentDB's native authentication

1. Set the **Vault URI** and **Secret Name** fields to the values you recorded earlier in this guide.

    :::image source="media/how-to-migrate-documentdb/target-key-vault.png" alt-text="Screenshot of target selection section.":::

1. Select **Next**.

### Configure using Microsoft Entra ID authentication

1. Make sure the managed identity configured on the source account has been [assigned read-write privileges on the target DocumentDB cluster](../../documentdb/how-to-connect-role-based-access-control.md#enable-microsoft-entra-id-authentication).

1. Set the **Azure DocumentDB (with MongoDB Compatibility) account name** field value.  

## Configure Network Security

The **Configure Network Security** section ensures that the target Azure DocumentDB cluster's network configuration allows the migration job to connect. Select one of the following options based on your target cluster's network configuration.

### Public network access is enabled on target — Use firewall rules

Azure DocumentDB clusters are created locked down by default. To enable communication from the Azure Cosmos DB for MongoDB account, add the IP address shown in this step to the Azure DocumentDB firewall.

1. Observe the **IP Address** in this step.

1. Navigate to your target Azure DocumentDB cluster using another browser window or tab.

1. Select **Networking** in the **Settings** section of the resource menu.

1. Add a rule to allow access to the migration job's IP address. For more information, see [manage cluster-level firewall rules](../../documentdb/how-to-configure-firewall.md#grant-access-from-azure-services).

    > [!NOTE]
    > If network security is enabled on your Azure Key Vault, ensure the same [IP is added to the Azure Key Vault Firewall](/azure/key-vault/general/network-security#key-vault-firewall-enabled-ipv4-addresses-and-ranges---static-ips) as well.

1. Navigate back to the browser window or tab with the migration job configuration steps.

1. Select **Next**.

### Public network access is disabled on target — Use Entra ID and network bypass

Network bypass mode provides a secure alternative to adding firewall rules when public network access is disabled on your target Azure DocumentDB cluster. Instead of opening firewall rules, bypass mode allows trusted Azure Cosmos DB services to access the cluster while blocking native authentication and all other public access. This creates a secure channel specifically for migration. For more information, see [network bypass mode](#network-bypass-mode).

> [!IMPORTANT]
> Make sure the managed identity [configured on the source account](#configure-managed-identity-for-your-source-account) has been assigned read-write privileges on the target DocumentDB cluster.

1. Follow the steps in the [enable network bypass mode](#enable-network-bypass-mode) section to configure your target cluster.

1. Navigate back to the browser window or tab with the migration job configuration steps.

1. Select **Next**.


## Configure and start job

Use the **Select Collections** and **Confirm & Submit** sections to finalize your job's configuration.

1. Select the collections you plan to migrate in the **Select Collections** section.

    :::image source="media/how-to-migrate-documentdb/select-collections.png" alt-text="Screenshot of the section to select collections to migrate.":::

1. Select **Next**.

1. Review the job configuration and provide a unique job name.

    > [!IMPORTANT]
    > 1. The migration job doesn't transfer the indexes to the target collections. Before proceeding, use this sample [migration script](https://aka.ms/mongoruschemamigrationscript) to create the indexes on the target collections. Once the indexes are ready, select the checkbox.
    > 2. The migration job doesn't support changing the shard key. If you need a different shard key, migrate the data as an unsharded collection. Once the migration is complete, shard the collection on target using the desired shard key.

1. Select **Submit** to create and start the job.

## Monitor migration jobs

Once a job is submitted, you can monitor the status of the newly created job along with other pending or completed jobs.

1. Navigate to your source Azure Cosmos DB for MongoDB account.

1. On the account page, select **Migrate to DocumentDB** from the resource menu.

1. Select **Monitor existing migration job(s)**.

    :::image source="media/how-to-migrate-documentdb/monitor-jobs.png" alt-text="Screenshot of the page where existing migration jobs can be monitored or modified.":::

1. All migration jobs created for the current source account are listed.

1. Optionally, to change the status of a job, select the context menu (**..**) corresponding to the specific job. Options include:

    | Option | Description |
    | --- | --- |
    | **Pause** | Temporarily pause a currently running job |
    | **Resume** | Resume a paused job |
    | **Cancel** | Permanently cancel a currently running job |
    | **Cutover** | Finalize the migration when the source and target are synced |

    > [!NOTE]
    > The **cutover** option is only applicable to online migrations. Once cutover is completed, the sync between the source account and target cluster is terminated. After performing a cutover, you should update the credentials in your client application to target the new Azure DocumentDB cluster.

## Network bypass mode

Network bypass mode provides a secure alternative to adding firewall rules when public network access is disabled on your target Azure DocumentDB cluster. Instead of opening firewall rules, bypass mode allows trusted Azure Cosmos DB services to access the cluster through the `AzureCosmosDB` service tag while blocking native authentication and all other public access. This creates a secure channel specifically for migration.

> [!IMPORTANT]
> Network bypass mode requires API version `2026-02-01-preview` or later. The ARM property is `properties.networkBypassMode`.

| Value | Description |
|---|---|
| `None` (default) | No bypass. Standard network rules apply. |
| `AzureCosmosDB` | Azure Cosmos DB service-tag traffic is allowed through the NSG. |

### Prerequisites for network bypass mode

All three of the following conditions must be met on the target cluster before you can enable network bypass mode:

| # | Requirement | Detail |
|---|---|---|
| 1 | No firewall rules exist | Delete all existing firewall rules. |
| 2 | Public network access is disabled | `properties.publicNetworkAccess` must be `Disabled`. |
| 3 | Microsoft Entra ID is the only authentication mode | `properties.authConfig.allowedModes` must be `["MicrosoftEntraID"]` (native authentication disabled). |

### Enable network bypass mode

Complete the following steps in order. Wait for the cluster to reach **Succeeded** provisioning state between each step.

#### Delete all firewall rules

List existing firewall rules:

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}/firewallRules?api-version=2026-02-01-preview"
```

For each rule returned, delete it:

```bash
az rest --method delete \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}/firewallRules/{ruleName}?api-version=2026-02-01-preview"
```

> [!NOTE]
> If the cluster has no firewall rules, skip this step. Each firewall rule deletion completes in under a minute.

#### Disable public network access

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"publicNetworkAccess": "Disabled"}}'
```

This operation typically completes in 2-5 minutes.

#### Switch to Microsoft Entra ID-only authentication

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"authConfig": {"allowedModes": ["MicrosoftEntraID"]}}}'
```

> [!NOTE]
> If the cluster already uses Microsoft Entra ID-only authentication, skip this step.

This operation typically completes in 5-10 minutes.

#### Set network bypass mode

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"networkBypassMode": "AzureCosmosDB"}}'
```

> [!IMPORTANT]
> This is an asynchronous operation. Poll the operation URL from the response headers until the cluster returns to **Succeeded** state. This operation typically completes in 5-10 minutes. You can confirm the configuration using the [verify the configuration](#verify-the-configuration) step.

#### Verify the configuration

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview"
```

Confirm the response includes:

```json
{
  "properties": {
    "networkBypassMode": "AzureCosmosDB",
    "publicNetworkAccess": "Disabled"
  }
}
```

After verification, return to the migration wizard and select **Next** to continue.

### Side effects while bypass mode is active

While `networkBypassMode` is set to `AzureCosmosDB`:

- **Firewall rules**: Creating, updating, or deleting firewall rules returns an error. Listing rules returns an empty list.
- **Public network access**: Re-enabling public network access is blocked until bypass mode is disabled.

### Troubleshoot network bypass mode

| Error message | Cause | Resolution |
|---|---|---|
| *"networkBypassMode is only supported when publicNetworkAccess has been disabled…"* | Public network access is still enabled. | Set `publicNetworkAccess` to `Disabled` first. |
| *"networkBypassMode cannot be enabled while firewall rules exist…"* | One or more firewall rules are present. | Delete all firewall rules first. |
| *"networkBypassMode cannot be enabled unless Microsoft Entra ID authentication mode is enabled and Native authentication mode is disabled…"* | Authentication configuration doesn't meet requirements. | Set `authConfig.allowedModes` to `["MicrosoftEntraID"]`. |
| *"Cannot enable public network access while network bypass mode is active…"* | Attempting to enable public access while bypass is on. | Set `networkBypassMode` to `None` first. |
| *"Firewall rules cannot be added while networkBypassMode is 'AzureCosmosDB'…"* | Attempting to add a firewall rule while bypass is on. | Set `networkBypassMode` to `None` and then enable public access. |

## Post migration actions

After the migration completes successfully, complete the following steps to revert any temporary configuration changes.

### Disable network bypass mode

If you enabled [network bypass mode](#network-bypass-mode) during the migration, disable it to restore standard network rules.

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"networkBypassMode": "None"}}'
```

This operation typically completes in 5-10 minutes. Disabling bypass mode has no prerequisites and can be done at any time. The `AzureCosmosDB` service tag is removed from the NSG and normal network rules resume.

### Re-enable public network access (optional)

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"publicNetworkAccess": "Enabled"}}'
```

### Re-enable native authentication (optional)

> [!NOTE]
> Native admin credentials must be provided when re-enabling native authentication. Replace `{adminUser}` and `{adminPassword}` with the cluster's admin username and a new password.

```bash
az rest --method patch \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DocumentDB/mongoClusters/{clusterName}?api-version=2026-02-01-preview" \
  --body '{"properties": {"authConfig": {"allowedModes": ["NativeAuth"]}, "administrator": {"userName": "{adminUser}", "password": "{adminPassword}"}}}'
```
