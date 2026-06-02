---
title: Per-partition automatic failover
description: Learn how per-partition automatic failover (PPAF) delivers sub-3-minute RTO and partition-scoped recovery for single-write-region Azure Cosmos DB accounts.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: concept-article
ms.date: 05/19/2026
appliesto:
  - ✅ NoSQL
ai-usage: ai-generated
---

# Per-partition automatic failover in Azure Cosmos DB

Per-partition automatic failover (PPAF) is an Azure Cosmos DB capability that automatically recovers write availability **at the partition level** during partial or full regional outages. Instead of failing over an entire database account, Azure Cosmos DB redirects writes only for the affected partitions to the next preferred region, while unaffected partitions continue writing to the original region.

PPAF is designed for single-write-region accounts on the API for NoSQL that want the low recovery time of a multi-write configuration without the complexity of conflict resolution.

## Why per-partition automatic failover

In the traditional model, when a write region experiences an outage Azure Cosmos DB has to fail over **every partition** in the account to the next region. That orchestration is heavy, and recovery can take significant time—even when only a small portion of the region is affected.

PPAF changes that in three ways:

| Aspect | Account-level failover | Per-partition automatic failover |
|---|---|---|
| Granularity | All partitions in the account move regions together | Only impacted partitions move; healthy partitions stay in place |
| Trigger | Manually initiated, or service-managed at the account level | Detected and triggered automatically for each affected partition |
| Operator action | Often requires manual initiation | Fully automatic—detection and failover |
| Typical RTO | 15–30 minutes or more, depending on how the outage progresses | **Less than 3 minutes at P99** |
| Failback | Manual; requires a full region sync | Automatic detection, automatic reconciliation |
| Blast radius | Entire account | Scoped to the affected partition set |

The net effect: smaller blast radius, faster recovery, and no waiting for manual intervention.

## How it works

Every partition in Azure Cosmos DB is replicated across the regions configured on your account. With PPAF, each partition independently detects that its current write region is unhealthy and promotes a new write region on its own—without affecting any other partition in the account.

### 1. Continuous health monitoring

The write replica for each partition is continuously monitored. If it stops responding—whether due to a node fault, a network problem, or a full regional outage—the issue is detected within tens of seconds.

### 2. Failover decision

When a write region is determined to be unhealthy, the partition automatically selects a new write region using two inputs:

- **Failover priority order:** The ordered list of regions configured on your account is the primary signal for which region should host writes next.
- **Replica freshness:** Within the eligible regions, PPAF picks the replica with the most recent committed state to minimize the chance of losing acknowledged writes.

The decision is **scoped to that one partition**. Other partitions in the same account are unaffected and continue serving writes from their existing write region.

### 3. Transparent client redirect

Once a new write region is chosen, the partition's routing information is updated. The Azure Cosmos DB SDK caches routing at the partition-key-range level and, on the next write to the affected partition, transparently sends the request to the new write region—no application code change, no restart, no reconnection required. Healthy partitions keep routing to the original region.

### 4. Automatic failback

When the original region recovers, PPAF automatically returns the affected partitions to the preferred write region. Failback uses **incremental catch-up** rather than rebuilding the partition from scratch, so it completes quickly and without manual intervention. If any writes diverged during the outage, PPAF reconciles them automatically (see [Failback and reconciliation](#failback-and-reconciliation)).


## Considerations and limitations

PPAF is a resilience feature, not a consistency, or data-model change. The following remain in your control and **are not modified** by enabling PPAF:

- **Your consistency level.** PPAF honors the consistency level configured on your account. The failover algorithm only completes a region promotion when the consistency guarantees can be preserved.
- **Your account topology.** PPAF doesn't add or remove regions. Your existing failover priority order is what PPAF uses to choose the next write region.
- **Your data model and partitioning.** Containers, partition keys, indexing policies, and stored procedures are unaffected.
- **Your endpoint and connection string.** Applications continue to use the account endpoint. The SDK handles regional routing internally.
- **Your RPO for Global Strong.** Strong-consistency accounts continue to guarantee **RPO = 0** through PPAF failovers.

What you **cannot do** while PPAF is enabled (these are deliberate guardrails so failover remains correct):

- Change account consistency between Strong and non-Strong while PPAF is enabled.
- Use **Bounded Staleness** consistency *(support is on the roadmap).*
- Run on a **serverless** account—provisioned throughput (manual or autoscale) is required.
- Use **Synapse Link**.
- Use **in-account restore**.
- Use Azure regions other than global Azure regions
- Execute **Partition Merge** or **Region offline** on a PPAF-enabled account. Disable PPAF first, perform the operation, then re-enable.

## Consistency support

PPAF supports the following consistency levels at GA:

- Strong
- Session
- Consistent Prefix
- Eventual

Bounded Staleness support is on the roadmap.

For **Strong consistency** accounts with two regions, PPAF can temporarily downshift the write quorum to a single region if one region has an issue or stops responding. Two-region quorum is restored automatically when the secondary region recovers. See [Consistency levels](consistency-levels.md) for background.

## Failback and reconciliation

Failback—returning a partition to its preferred write region after recovery—is fully automated. Two design choices make it fast and safe.

### Partition reuse with incremental catch-up

When the original write region comes back online, PPAF does **not** discard the existing replicas or rebuild them from scratch. Instead, it brings the recovered replicas current using **incremental catch-up—only the writes that occurred during the failover window are replayed. This is dramatically faster than a full partition rebuild (often seconds to minutes instead of hours) and means failback adds little load to the recovered region.

### Reconciling divergent writes

**Strong consistency accounts don't require reconciliation.** Because every acknowledged write is committed by a quorum before it's confirmed to the client, no divergent writes can exist when the original region rejoins. Failback is a straightforward incremental catch-up.

For **Session**, **Consistent Prefix**, and **Eventual** consistency, divergence is possible. During an outage, the original write region might have accepted and acknowledged a few writes that didn't replicate to other regions before the failure. When the original region rejoins, those writes can conflict with newer writes that were accepted in the new write region during the outage.

PPAF reconciles these automatically using a **last-writer-wins** policy based on the system timestamp on each write. Reconciliation runs in the background; reconciled data becomes visible to readers progressively as the work completes. No client involvement is required.

Autoreconciliation is **enabled by default**. If your application needs custom reconciliation semantics—for example, application-level conflict resolution on counters or sets—you can opt out via a support request and reconcile divergent writes yourself by reading them from the conflict feed and applying your own resolution logic. For details, see [Read from conflict feed](how-to-manage-conflicts.md#read-from-conflict-feed).

### Brief pause during failback

Failback completes a graceful handoff to restore the preferred write region. During the handoff there's a short window—typically a few seconds—when writes to the affected partition could experience elevated latency or transient retries. The Azure Cosmos DB software development kits (SDKs) retry these automatically; applications don't need to handle them explicitly.

## Application changes

For most applications, the only requirement is to upgrade to a supported SDK version. Once PPAF is enabled on the account, the SDK:

- Automatically detects PPAF and redirects writes to the new write region for any failed-over partition.
- Caches partition-to-region routing so failover is transparent on subsequent requests.
- Uses the account's failover priority order automatically. Setting `ApplicationPreferredRegions` or `ApplicationRegion` is no longer mandatory but remains a best practice.
- Enables **per-partition circuit breaker** by default to protect read availability for an affected partition.
- Enables **read hedging** by default so reads to a slow region are transparently retried against another region. You can override or disable this with a custom availability strategy.

No application code changes are required beyond the SDK upgrade.

## Benefits summary

- **Recovery time objective (RTO)   < 3 minutes at P99** for partition-level failover, compared with 15–30 minutes for account-level failover.
- **Recovery point objective (RPO) = 0** for Global Strong consistency through failover.
- **Reduced blast radius—only the impacted partition set moves regions; everything else stays in place.
- **Active-active behavior with a single writer.** You get a level of resiliency previously reserved for multi-write accounts, without the cost and complexity of conflict resolution.
- **No application changes** beyond an SDK upgrade.
- **Transparent failback** with optional automatic reconciliation.

## Observability

PPAF introduces a new server-side metric, **`PartitionWriteGlobalStatus`**, which reports the count of write partitions per region at any moment. Use it to confirm that a failover happened, see how many partitions moved, and watch failback progress. The metric is available through Azure Monitor alongside your existing Azure Cosmos DB metrics.

The Azure Cosmos DB SDKs also emit per-request region information through their built-in diagnostics, so you can see which region served each request from the client side.

## Operational guidance

PPAF is designed to be hands-off. The most common operational pattern is to **let PPAF drive** and use monitoring to confirm behavior rather than to intervene.

### What to do during a failover

- **Manual change write region is still available.** PPAF and the account-level *change write region* operation work together. Use it when you want to consolidate writes in one region for an extended period—for example, when `PartitionWriteGlobalStatus` shows that a large portion of your partitions has already failed over to the secondary, or during a prolonged regional outage where you want to align with a capacity decision or free the original region for maintenance. For typical outages of minutes to hours, PPAF's automatic failback is the correct path.
- **Watch `PartitionWriteGlobalStatus`** in Azure Monitor to see partitions move and to confirm failback once the original region recovers.
- **Let the SDK retry.** Application code should already handle transient errors per the [Azure Cosmos DB SDK guidance](conceptual-resilient-sdk-applications.md). During the failover window, the SDK retries automatically against the new write region.

### Frequently asked questions

#### Do all my partitions fail over together?

No. Each partition decides independently. A partial regional outage typically moves only the subset of partitions affected; healthy partitions stay in the original region.

#### Will my application see errors during failover?

Writes to affected partitions might see transient errors until the new region is elected and the SDK refreshes its routing—usually under three minutes. The SDK retries automatically. Reads to other regions and writes to unaffected partitions continue normally.

#### Can I lose data?

For **Strong** consistency, no—RPO is 0 through PPAF failovers. For other consistency levels, PPAF picks the replica with the most recent committed state to minimize loss, and reconciles any divergent writes on failback using last-writer-wins.

#### Do I need to do anything on failback?

No. Failback is automatic, uses incremental catch-up rather than a full rebuild, and reconciles divergent writes in the background.

#### Does PPAF replace multi-write?

PPAF is for single-write-region accounts that want fast, automatic recovery without conflict-resolution complexity. Multi-write (multi-region writes) remains the right choice for workloads that need active-active write capability across regions always.

## Pricing

PPAF is part of the **Business Critical** service tier for Azure Cosmos DB. For more information and current rates, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

## Prerequisites at a glance

| Requirement | Value |
| --- | --- |
| API | NoSQL (Core SQL API) |
| Account topology | Single write region with at least one read region |
| Throughput | Provisioned (manual or autoscale) |
| Consistency | Strong, Session, Consistent Prefix, or Eventual |
| Cloud | Azure public cloud regions |
| Connection mode | Direct |
| SDK | .NET v3 ≥ 3.60.0 - Java v4 ≥ 4.79.0 - Python ≥ 4.16.0 - Node.js ≥ 4.7.0 |

## Related content

- [Configure and use per-partition automatic failover](how-to-configure-per-partition-automatic-failover.md)
- [Consistency levels in Azure Cosmos DB](consistency-levels.md)
- [High availability in Azure Cosmos DB](high-availability.md)
- [Sample app and chaos script (AzureCosmosDB/ppaf-samples)](https://github.com/AzureCosmosDB/ppaf-samples)
- [Implementing decentralized per-partition automatic failover in Azure Cosmos DB](https://arxiv.org/pdf/2505.14900)
