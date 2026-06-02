---
title: Cross-region failover modes
description: Compare forced promotion, graceful (planned) failover, and service-managed failover for Azure DocumentDB cross-region disaster recovery.
author: prashanthmadi
ms.author: prmadi
ms.topic: concept-article
ms.date: 05/14/2026
ai-usage: ai-assisted
#Customer Intent: As a database administrator, I want to understand the available cross-region failover modes in Azure DocumentDB, so that I can choose the option that best matches my recovery time and recovery point objectives.
---

# Cross-region failover modes in Azure DocumentDB

When you enable [cross-region replication](./cross-region-replication.md) on an Azure DocumentDB cluster, the replica cluster in the secondary region can take over write operations if the primary cluster becomes unavailable or if you need to switch regions for any other reason. The operation that turns a replica cluster into the new read-write cluster is called a *failover*.

Azure DocumentDB supports three failover modes. Each mode targets a different combination of recovery time objective (RTO), recovery point objective (RPO), and operational control. This article describes each mode, when to use it, and how it differs from the others.

## Prerequisites

All three failover modes require [cross-region replication](./cross-region-replication.md) to be enabled on the cluster. The replica cluster can be in another Azure region or in the same region; cross-region failover modes apply only to replicas in another region.

## Compare the failover modes

The following table summarizes the three modes.

| Failover mode | Who initiates the failover | Triggered when | Data loss | Typical use case |
| --- | --- | --- | --- | --- |
| Forced promotion | You | Any time you choose | Possible (replication lag isn't drained) | Custom RTO and RPO control, drills, region migrations |
| Graceful promotion | You | Any time you choose | Zero (replication lag is drained first) | Planned maintenance, region migrations, scheduled switchovers |
| Service-managed failover | Azure | Detected regional outage on the primary | Possible (replication lag isn't drained) | Mission-critical workloads that need automatic recovery from regional outages |

Forced promotion and service-managed failover are both *unplanned* failovers and might result in data loss because of replication lag. Graceful promotion is a *planned* operation that guarantees zero data loss by waiting for the replica to catch up before switching write roles.

## Forced promotion

Forced promotion is the default failover mode. You initiate the promotion from the Azure portal, Azure CLI, or REST API whenever you want the replica cluster to start accepting writes.

Because Azure DocumentDB uses asynchronous replication between the primary and replica clusters, write operations completed on the primary might not yet be replicated when promotion starts. Any unreplicated writes aren't present on the promoted cluster.

Use forced promotion when:

- You need to fail over immediately, even at the risk of data loss (for example, when the primary region is unreachable but service-managed failover isn't enabled).
- You're running a disaster recovery drill and want full control over the timing.
- You're migrating to a new primary region as part of a planned move and your application can tolerate the small amount of data loss that can occur from in-flight replication.

For step-by-step instructions, see [Trigger a forced promotion](./how-to-cluster-replica.md#trigger-a-forced-promotion).

## Graceful promotion

Graceful promotion is a planned operation that switches write roles between the primary cluster and its replica with *zero data loss*. When you start a graceful promotion, Azure DocumentDB:

1. Stops accepting new write operations on the primary cluster.
1. Waits for all pending replication to drain so the replica is fully caught up.
1. Switches the replica cluster to the read-write role.
1. Sets the former primary cluster to read-only.

Because writes are paused while the replication lag drains, the application sees a short write-availability gap. The duration depends on the replication lag at the moment you start the failover. The [global read-write connection string](./cross-region-replication.md#continuous-writes-read-operations-on-cluster-replicas-and-connection-strings) automatically points to the new primary cluster after the switch completes.

Use graceful promotion when:

- You're performing scheduled maintenance and want to test the secondary region without risking data loss.
- You're permanently migrating the workload to a different primary region.
- Your application can tolerate a brief write-availability window in exchange for a zero-RPO switch.

> [!IMPORTANT]
> Graceful promotion requires the primary cluster to be healthy enough to drain the replication queue. If the primary is already unreachable because of a regional outage, use forced promotion or service-managed failover instead.

For step-by-step instructions, see [Trigger a graceful promotion](./how-to-cluster-replica.md#trigger-a-graceful-promotion).

## Service-managed failover

Service-managed failover lets Azure DocumentDB automatically promote the replica cluster when it detects a regional outage on the primary. You opt in to service-managed failover on the primary cluster. After it's enabled, no further action is required from your application during a regional outage. The global read-write connection string automatically updates to point to the promoted replica.

Because the failover is triggered by an outage, the primary cluster can't drain the replication queue first. Any writes that weren't replicated to the secondary region before the outage might be lost. The trade-off is automatic recovery without user-initiated action.

Use service-managed failover when:

- Your workload has a strict RTO and can't wait for a user-initiated promotion after detecting the outage.
- You want a recovery posture similar to [Microsoft-managed failover groups in Azure SQL Database](/azure/azure-sql/database/failover-group-sql-db#microsoft-managed) or [service-managed failover in Azure Cosmos DB](/azure/reliability/reliability-cosmos-db).

> [!NOTE]
> Service-managed failover doesn't replace [in-region high availability (HA)](./high-availability.md). Combine both to protect against shard-level failures (HA, zero data loss) and regional outages (service-managed failover, possible data loss).

For step-by-step instructions, see [Enable service-managed failover](./how-to-cluster-replica.md#enable-service-managed-failover).

## Choose the right mode

Use this guidance to pick a failover mode:

- If you need automatic recovery from regional outages, enable **service-managed failover**.
- If you're planning a region switch and need zero data loss, use **graceful promotion**.
- If you need full control over the timing or you're running a drill, use **forced promotion**.

You can combine modes. For example, enabling service-managed failover doesn't prevent you from triggering a graceful promotion for planned maintenance. The service-managed setting only changes who triggers the failover when the primary region becomes unavailable.

For broader recommendations, see [Best practices for high availability and cross-region replication](./high-availability-replication-best-practices.md).

## Related content

- [Learn about cross-region replication](./cross-region-replication.md)
- [Manage cross-region and same region replication](./how-to-cluster-replica.md)
- [Availability and disaster recovery internals](./availability-disaster-recovery-under-hood.md)
- [Best practices for high availability and cross-region replication](./high-availability-replication-best-practices.md)
- [Reliability in Azure DocumentDB](/azure/reliability/reliability-documentdb?context=/azure/documentdb/context/context)
