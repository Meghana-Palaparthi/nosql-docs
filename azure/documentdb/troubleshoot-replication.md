---
title: Troubleshoot common issues in Azure DocumentDB replication
description: This guide discusses the ways to troubleshoot common issues encountered in Azure DocumentDB replication.
ms.topic: troubleshooting
ms.date: 05/14/2026
author: prashanthmadi
ms.author: prmadi
ai-usage: ai-assisted
---

# Troubleshooting guide: Cross-region and same region replication in Azure DocumentDB

This guide is designed to help you troubleshoot common issues when using [cluster replication](./cross-region-replication.md) with Azure DocumentDB. It offers practical solutions for connectivity problems and optimization challenges to improve your experience.

## Common issues and solutions
### My data doesn't seem to replicate to replica cluster.
Ensure your cluster has a replica created. A replica cluster can be created during initial cluster provisioning or added later. You can [verify the existence of a replica on the 'Overview' page](./how-to-cluster-replica.md#check-cluster-replication-role-and-replication-region) or [create a new one on the 'Global distribution' page](./how-to-cluster-replica.md#enable-cross-region-or-same-region-replication) of your cluster in the Azure portal.

:::image type="content" source="media/troubleshoot-replication/no-replication.png" alt-text="Screenshot of the cluster replication status on the Overview page and Global distribution page in Azure portal.":::

### I can't find connection string for replica cluster.
The replica cluster is a full replica of the primary cluster with writes disabled. It's represented as a separate cluster resource in Azure and has its own connection string.

To locate the replica cluster:

- Open the properties page of the primary cluster in the Azure portal.
- Navigate to the 'Global distribution' page and click on the hyperlinked name of the replica cluster.
    :::image type="content" source="media/troubleshoot-replication/replica-cluster-link-in-azure-portal.png" alt-text="Screenshot of the cluster Global distribution page with a link to replica cluster in Azure portal.":::
- In the replica cluster’s properties, [navigate to the 'Connection strings' page](./how-to-cluster-replica.md#use-connection-strings) to copy the connection string.
    :::image type="content" source="media/troubleshoot-replication/replica-cluster-connection-strings.png" alt-text="Screenshot of the replica cluster properties with Connection strings page highlighted in Azure portal.":::

### I can't connect to replica cluster.
Ensure that [public or private access](./security.md#network-security) is enabled for the replica cluster. When a replica cluster is created in another region, its network settings aren't automatically replicated from the primary Azure DocumentDB cluster.  

To enable access:  
- To allow public access, add firewall rules.  
- Alternatively, create private endpoints to enable private access to the replica cluster.  

:::image type="content" source="media/troubleshoot-replication/replica-cluster-networking-settings.png" alt-text="Screenshot of the replica cluster Networking page in Azure portal.":::

### How can I track utilization of various resources such as memory or IOPS on my replica cluster?
Since the replica cluster is a full replica of the primary Azure DocumentDB cluster, it has its own metrics for monitoring resource consumption.  

To view metrics:  
1. Open the properties page of the replica cluster in the Azure portal.  
1. Navigate to the **Metrics** page to access and review the resource consumption details.
    :::image type="content" source="media/troubleshoot-replication/replica-cluster-metrics-page.png" alt-text="Screenshot of the replica cluster properties with Metrics page highlighted in Azure portal.":::

See more detailed guidance on using metrics with Azure DocumentDB on [this page](./monitor-metrics.md).

### How can I open replica cluster in another region for writes?
You can promote the replica cluster to become a read-write cluster at any time. When you promote the replica cluster in region B, the ex-primary cluster in region A is set to read-only mode, effectively making it the replica cluster for the newly promoted primary cluster.  

To promote the replica cluster:  
1. Open the properties page of the replica cluster in the Azure portal.  
1. Navigate to the **Global distribution** page.  
1. Select **Promote** and confirm the promotion.
    :::image type="content" source="media/troubleshoot-replication/replica-cluster-promote.png" alt-text="Screenshot of the replica cluster properties with Global distribution page open and Promote button highlighted in Azure portal.":::

During the promotion process, the [global read-write connection string](./how-to-cluster-replica.md#use-connection-strings) is updated to point to the newly promoted replica cluster once it's open for write operations.

### Replica cluster isn't needed anymore. How can I disable replication and delete replica cluster?
You can delete a replica cluster at any time without impacting the primary cluster or the data stored and written to it. However, if you need to delete both the replica and primary clusters, ensure that the replica cluster is deleted first.

To delete a replica cluster:
1. Open the properties page of the replica cluster in the Azure portal.
1. On the **Overview** page, select **Delete** in the toolbar.
1. Carefully read the warning message and confirm the deletion.
    :::image type="content" source="media/troubleshoot-replication/replica-cluster-delete.png" alt-text="Screenshot of the replica cluster properties with Overview page open and Delete button highlighted in Azure portal.":::

### Which failover mode should I use?
Azure DocumentDB supports three [cross-region failover modes](./failover-modes.md):

- **Forced promotion** — You promote the replica at any time. Possible data loss because of replication lag.
- **Graceful promotion** — You initiate a planned switch. Replication drains before the switch, so the operation completes with zero data loss.
- **Service-managed failover** — Azure automatically promotes the replica when it detects a regional outage on the primary. Possible data loss because of replication lag.

Use service-managed failover for automatic recovery from regional outages, graceful promotion for planned region switches, and forced promotion when you need direct control over an unplanned failover.

### Service-managed failover didn't trigger after a regional outage. What should I check?
First, confirm that service-managed failover is enabled on the primary cluster. Open the primary cluster's **Global distribution** page in the Azure portal and verify that service-managed failover is turned on. If it isn't, Azure DocumentDB doesn't promote the replica automatically; you can [trigger a forced promotion](./how-to-cluster-replica.md#trigger-a-forced-promotion).

If service-managed failover is enabled, Azure DocumentDB only triggers a failover when it determines that the primary region is unavailable and the cluster can't be recovered locally. Brief, transient errors don't trigger a service-managed failover. Check the Azure status page and your cluster's diagnostic logs to confirm whether the service detected a regional outage.

### My graceful promotion is taking longer than expected. Why?
[Graceful promotion](./failover-modes.md#graceful-promotion) waits for the replication queue to fully drain before switching write roles. The duration depends on the current replication lag, which grows under heavy write load. To reduce the time a graceful promotion takes:

- Schedule the failover during a window of low write activity on the primary cluster.
- Check current replication lag on the **Metrics** page of the replica cluster before initiating the failover.
- If the lag is consistently high, scale up the replica cluster so it can keep up with primary writes more quickly.

If the primary cluster becomes unreachable while a graceful promotion is in progress, the operation can't complete because the queue can't be drained. In that situation, cancel the graceful promotion and use [forced promotion](./how-to-cluster-replica.md#trigger-a-forced-promotion) instead.

### Can I use service-managed failover and graceful promotion together?
Yes. The two settings are independent. Service-managed failover controls whether Azure DocumentDB automatically promotes the replica during a regional outage. Graceful promotion is an on-demand operation you trigger for planned region switches. Enable service-managed failover as a safety net for outages and use graceful promotion whenever you can schedule the switch yourself.

## Next steps
- If you followed all the troubleshooting steps and still can't resolve your issue, you can open a [support request](https://azure.microsoft.com/support/create-ticket/) for further assistance.
- If you're troubleshooting common issues with Azure DocumentDB, see [this troubleshooting guide](./troubleshoot-common-issues.md).
