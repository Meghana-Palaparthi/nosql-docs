---
title: Enable and work with cross-region and same region replication
description: Enable and disable replication and promote replica cluster for disaster recovery (DR) in Azure DocumentDB.
author: prashanthmadi
ms.author: prmadi
ms.custom:
  - build-2024
  - ignite-2024
ms.topic: how-to
ms.date: 05/14/2026
ai-usage: ai-assisted
#Customer Intent: As a database adminstrator, I want to configure cross-region replication, so that I can have disaster recovery plans in the event of a regional outage.
---

# Manage cross-region and same region replication on your Azure DocumentDB cluster

Azure DocumentDB allows continuous data streaming to a replica cluster in another or the same Azure region. That capability provides cross-region disaster recovery (DR) protection and read scalability across the regions and in the same region. This document serves as a quick guide for developers who want to learn how to manage replication for their clusters.

## Prerequisites

[!INCLUDE[Prerequisite - Azure subscription](includes/prerequisite-azure-subscription.md)]

## Enable cross-region or same region replication

You can create a replica cluster during new cluster provisioning or at any time on an existing cluster.

### Replica cluster creation during new cluster provisioning

To enable replication on a new cluster *during cluster creation*, follow these steps:

1. Follow the steps to [start cluster creation and complete the **Basics** tab for a new Azure DocumentDB cluster](./quickstart-portal.md#create-a-cluster).
1. (optionally) Select desired network access settings for the cluster on the **Networking** tab.
1. On the **Global distribution** tab, select **Enable** for the **Cluster replica**.
1. Provide a replica cluster name in the **Read replica name** field. 
1. Select a region in the **Read replica region**. The replica cluster is hosted in the selected Azure region.
1. On the **Review + create** tab, review cluster configuration details, and then select **Create**. 

> [!NOTE]
> The replica cluster is created in the same Azure subscription and resource group as its primary cluster.

### Replica cluster creation for existing cluster

To enable replication on a new cluster *at any time after cluster creation*, follow these steps:

1. Follow the steps to [create a new Azure DocumentDB cluster](./quickstart-portal.md#create-a-cluster).
1. Skip **Global distribution** tab. This tab is used to create a cluster replica during primary cluster provisioning.
1. *Once cluster is created*, on the cluster sidebar, under **Settings**, select **Global distribution**.
1. Select **Add new read replica**.
1. Provide a replica cluster name in the **Read replica name** field. 
1. Select a region in the **Read replica region**. The replica cluster is hosted in the selected Azure region.
1. (optionally) Select **Customer-manageed key** in **Data encryption** section to enable data encryption with a customer-managed key (CMK) on replica cluster. Then follow [the steps to enable CMK](./how-to-data-encryption.md#change-data-encryption-mode-on-existing-clusters).
1. Verify your selection and select the **Save** button to confirm replica creation.

To make the replica cluster accessible for read operations, adjust its networking settings by configuring firewall rules for public access or by adding private endpoints for secure, private access.

## Trigger a forced promotion

This section describes the *forced* promotion of a replica cluster. Forced promotion gives you the most control over timing, but it's an unplanned failover that might result in data loss because replication is asynchronous. For a zero-data-loss alternative, use [graceful promotion](#trigger-a-graceful-promotion). To let Azure automatically promote the replica during a regional outage, [enable service-managed failover](#enable-service-managed-failover).

To [promote a cluster replica](./cross-region-replication.md#replica-cluster-promotion) to a read-write cluster, follow these steps:

1. Select the cluster replica you would like to promote in the portal.
1. On the cluster sidebar, under **Settings**, select **Global distribution**.
1. On the **Global distribution** page, select **Forced Promote**.
1. Read the warning text and select **Confirm**.

After the cluster replica is promoted, it becomes a readable and writable cluster. If [high availability (HA)](./high-availability.md) is enabled on the primary (read-write) cluster, it needs to be re-enabled on the replica cluster after promotion.

## Trigger a graceful promotion

A *graceful promotion* is a planned switchover that completes with zero data loss. Azure DocumentDB pauses writes on the primary cluster, drains the replication queue so the replica is fully caught up, and then promotes the replica. Use a graceful promotion for scheduled maintenance or planned region migrations. For background, see [Graceful promotion](./failover-modes.md#graceful-promotion).

> [!IMPORTANT]
> Graceful promotion requires the primary cluster to be reachable so it can drain pending replication. If the primary region is already unavailable, use [forced promotion](#trigger-a-forced-promotion) or [service-managed failover](#enable-service-managed-failover) instead.

To trigger a graceful promotion:

1. Select the cluster replica you would like to promote in the portal.
1. On the cluster sidebar, under **Settings**, select **Global distribution**.
1. On the **Global distribution** page, select **Graceful Promote**.
1. Read the warning text and select **Confirm**.

During the failover, write requests return a transient error until the switch completes. The [global read-write connection string](./cross-region-replication.md#continuous-writes-read-operations-on-cluster-replicas-and-connection-strings) automatically updates to point to the new primary cluster. Applications that use the self connection string of the former primary must be updated to point to the new primary for write operations.

## Enable service-managed failover

*Service-managed failover* lets Azure DocumentDB automatically promote the replica cluster when it detects a regional outage on the primary. After it's enabled, no operator action is required during a regional outage. For background, see [Service-managed failover](./failover-modes.md#service-managed-failover).

> [!NOTE]
> Service-managed failover is an unplanned failover and might result in data loss because of replication lag. To switch regions with zero data loss for planned maintenance, use [graceful promotion](#trigger-a-graceful-promotion) instead. The two options can be used together.

Service-managed failover is configured per cluster from the primary cluster's **Global distribution** page in the Azure portal. After it's enabled, Azure DocumentDB monitors the primary region and automatically promotes the replica cluster if it determines that the primary region is unavailable. You can disable service-managed failover at any time from the same page; while it's disabled, a regional outage requires you to [trigger a forced promotion](#trigger-a-forced-promotion).

## Check cluster replication role and replication region

To check replication role of a cluster, follow these steps:
1. Select an existing Azure DocumentDB cluster.
1. Select **Overview** page.
1. Check **Read region** (on the primary cluster) or **Write region** (on the replica cluster) value.

If **Read region** value is *Not enabled*, this cluster has replication disabled.

## Disable cross-region or same region replication

To disable replication, follow these steps:

1. Select the Azure DocumentDB *replica* cluster.
1. Select **Overview**.
1. [Confirm that it's a replica cluster](#check-cluster-replication-role-and-replication-region).
1. In the Azure portal, on the **Overview** page for the replica cluster, select **Delete**.
1. On the **Delete \<replica name>** screen, read the warning text, and enter cluster's name in the **Confirm the account name** field.
1. Select **Delete** to confirm deletion of the replica.

If you need to delete the primary and replica clusters, you would need to delete the replica cluster first.

## Use connection strings

You can connect to the cluster replica as you would to a regular read-write cluster. 
Follow these steps to [get the connection strings for different cases](./cross-region-replication.md#continuous-writes-read-operations-on-cluster-replicas-and-connection-strings):

1. Select the primary cluster or its cluster replica in the portal.
1. On the cluster sidebar, under **Settings**, select **Connection strings**.
1. Copy the self connection string for currently selected cluster to connect to that cluster.
1. (optionally, on the primary cluster only) Copy the global read-write connection string that always points to the cluster available for writes.

:::image type="content" source="media/cross-region-replication/global-read-write-connection-string-in-azure-portal.png" alt-text="Screenshot of the cluster connection strings an Azure DocumentDB cluster including global read-write connection string and self connection string.":::

Self connection strings are preserved after [the cluster replica promotion](./cross-region-replication.md#replica-cluster-promotion). You can continue to use either string or global read-write connection string for read operations. If you use self connection string for write operations, you need to update connection string in your application to point to the promoted replica cluster to continue writes to the database after promotion is completed.

## Related content

- [Compare cross-region failover modes](./failover-modes.md)
- [Learn more about cross-region and same region replication](./cross-region-replication.md)
- [See replication limits and limitations](./limitations.md#cross-region-and-same-region-replication)
- [Troubleshoot cross-region replication](./troubleshoot-replication.md)
- [Learn about reliability in Azure DocumentDB](/azure/reliability/reliability-documentdb?context=/azure/documentdb/context/context)
