---
title: Per-partition automatic failover in Azure Cosmos DB
description: Learn how per-partition automatic failover (PPAF) delivers sub-2-minute RTO and partition-scoped recovery for single-write-region Azure Cosmos DB for NoSQL accounts.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: conceptual
ms.date: 05/15/2026
appliesto:
  - ✅ NoSQL
---

# Per-partition automatic failover in Azure Cosmos DB

Per-partition automatic failover (PPAF) is an Azure Cosmos DB capability that automatically recovers write availability **at the partition level** during partial or full regional outages. Instead of failing over an entire database account, Azure Cosmos DB redirects writes only for the affected partitions to the next preferred region, while unaffected partitions continue writing to the original region.

PPAF is designed for single-write-region accounts on the API for NoSQL that want the low recovery time of a multi-write configuration without the complexity of conflict resolution.

## Why per-partition automatic failover

In the traditional model, when a write region experiences an outage Azure Cosmos DB must fail over **every partition** in the account to the next region. This is coarse-grained and adds coordination time, even when only a small portion of the region is affected.

PPAF changes that in three ways:

| Aspect | Account-level failover | Per-partition automatic failover |
|---|---|---|
| Granularity | All partitions in the account move regions together | Only impacted partitions move; healthy partitions stay in place |
| Trigger | Detected and driven by the control plane via customer | Detected and driven by each partition in the data plane |
| Operator action | DRI invocation often required | Fully automatic — detection and failover |
| Typical RTO | Customer dependent and is usually 15–30 minutes. Based on issue progression - service managed failover can take an 1+ hour to trigger | **Less than 3 minutes at P99** |
| Failback | Manual region online and full region sync | Automatic detection, automatic reconciliation |
| Blast radius | Entire account | Scoped to the affected partition set |

The net effect: smaller blast radius, faster recovery, and no waiting for an on-call engineer.

## How it works

Every partition in Azure Cosmos DB is a replica set spread across the regions configured on your account — referred to as a **partition set**. PPAF lets each partition set independently detect that its current write region is unhealthy and promote a new write region without involving any other partition. This decentralized design is what makes both detection and failover complete in under three minutes at P99.

### 1. Continuous health monitoring

Each partition's current write replica continuously emits heartbeats to a highly available, fault-tolerant coordination layer that is itself replicated across regions. Read replicas in other regions watch those heartbeats. The cadence and quorum requirements are tuned so that a missing heartbeat is detected within tens of seconds regardless of whether the cause is a node failure, network partition, hardware fault, or full regional outage.

### 2. Failover decision

When the heartbeats from a write replica are missing for long enough to constitute an outage, the partition set begins selecting a new write region. The selection is deterministic and uses two inputs:

- **Failover priority order:** The ordered list of regions configured on your account is the primary signal for which region should host writes next.
- **Replica progress:** Within the eligible regions, PPAF picks the replica that has the most recent committed state. This minimizes the chance of losing writes that were already acknowledged.

The election is **scoped to that one partition**. Other partitions in the same account are not affected and continue serving writes from their existing write region.

### 3. Topology update and client redirect

Once a new write region is elected, the partition's topology record is updated and replicated. The Azure Cosmos DB SDK caches routing information at the partition-key-range level. On the next write to the affected partition, the SDK transparently sends the request to the new write region — no application code change, no restart, no reconnection required. Healthy partitions continue routing to the original region.

### 4. Automatic failback

When the original region recovers, PPAF automatically returns the affected partitions to the preferred write region. Failback uses **incremental catch-up** rather than rebuilding the partition from scratch, so it completes quickly and without manual intervention. If any writes diverged during the outage, PPAF reconciles them automatically (see [Failback and reconciliation](#failback-and-reconciliation) below).

Because each partition makes its own independent decision, PPAF scales horizontally and handles anything from a node-level fault to a full regional outage. The mechanism is described in detail in the VLDB paper *Implementing Decentralized Per-Partition Automatic Failover in Azure Cosmos DB* (arXiv:2505.14900).

### Timeline of a regional outage

The following timeline illustrates a typical end-to-end failover when an entire write region becomes unavailable. Actual numbers vary with workload and topology; these are representative P99 values.

| Time | What happens |
|---|---|
| **T0** | Write region becomes unhealthy. In-flight writes start failing. |
| **T0 + ~60 s** | Missing heartbeats are detected. Affected partitions begin selecting a new write region. |
| **T0 + ~90 s** | New write region is elected per partition based on failover priority and replica progress. Topology is updated. |
| **T0 + < 2 min (P99)** | SDK routes writes for affected partitions to the new region. Application sees write availability restored. |
| **Later — original region recovers** | PPAF detects health, brings the original replicas current via incremental catch-up, reconciles any divergent writes, and returns partitions to the preferred write region. |

Healthy partitions are unaffected throughout — they continue serving writes from the original region the entire time.

## What stays the same — what you cannot change

PPAF is a resilience feature, not a consistency or data-model change. The following remain in your control and **are not modified** by enabling PPAF:

- **Your consistency level.** PPAF honors the consistency level configured on your account. Strong stays strong, Session stays session, and so on. The failover algorithm only completes a region promotion when the consistency guarantees can be preserved.
- **Your account topology.** PPAF does not add or remove regions. Your existing failover priority order is what PPAF uses to choose the next write region.
- **Your data model and partitioning.** Containers, partition keys, indexing policies, and stored procedures are unaffected.
- **Your endpoint and connection string.** Applications continue to use the account endpoint. The SDK handles regional routing internally.
- **Your RPO for Global Strong.** Strong-consistency accounts continue to guarantee **RPO = 0** through PPAF failovers.

What you **cannot do** while PPAF is enabled (these are deliberate guardrails so the failover algorithm remains correct):

- Change account consistency between Strong and non-Strong on the fly without coordination.
- Use **Bounded Staleness** consistency *(support is on the roadmap).*
- Run on a **serverless** account — provisioned throughput (manual or autoscale) is required.
- Use **periodic backup** — continuous backup (point-in-time restore) is required.
- Use **Synapse Link**.
- Use **in-account restore**.
- Use sovereign-cloud regions *(public Azure regions only — sovereign cloud support is on the roadmap).*
- Execute **Partition Merge**, **Add/Remove Region**, **Switch Write Region**, or **region offline** workflows directly. Coordinate with Microsoft to run these on a PPAF-enabled account.

## Consistency support

PPAF supports the following consistency levels at GA:

- Strong
- Session
- Consistent Prefix
- Eventual

Bounded Staleness support is on the roadmap.

For **Strong consistency** accounts with two regions, PPAF sets `MinimumDurability = 1` during enablement. This allows the dynamic quorum to downshift to the write region if read regions slow or stop responding, preserving write availability without violating ordering guarantees. The full Dynamic Quorum value (majority of read regions) returns automatically when read regions recover. See [Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) for background on dynamic quorum.

## Failback and reconciliation

Failback — returning a partition to its preferred write region after recovery — is fully automated. Two design choices make it fast and safe.

### Partition reuse with incremental catch-up

When the original write region comes back online, PPAF does **not** discard the existing replicas or rebuild them from scratch. Instead, it brings the recovered replicas current using **incremental catch-up** — only the writes that occurred during the failover window are replayed. This is dramatically faster than a full partition rebuild (often seconds to minutes instead of hours) and means failback adds little load to the recovered region.

### Reconciling divergent writes

During an outage, it's possible for the original write region to have accepted and acknowledged a small number of writes that did not replicate to other regions before the failure (sometimes called *false progress*). When the original region rejoins, those writes might conflict with newer writes that were accepted in the new write region during the outage.

PPAF reconciles these automatically using a **last-writer-wins** policy based on the system timestamp on each write. Reconciliation runs in the background; reconciled data becomes visible to readers progressively as the work completes. No client involvement is required.

Auto-reconciliation is **enabled by default**. Workloads that need custom merge semantics — for example, application-level conflict resolution on counters or sets — can opt out and handle reconciliation themselves.

### Brief pause during failback

Failback completes a graceful handoff to restore the preferred write region. During the handoff there is a short window — typically a few seconds — when writes to the affected partition may experience elevated latency or transient retries. The Cosmos DB SDKs retry these automatically; applications do not need to handle them explicitly.

## Application impact

For most applications, the only requirement is to upgrade to a supported SDK version. Once PPAF is enabled on the account, the SDK:

- Automatically detects PPAF on the account and adjusts retry behavior so writes are redirected to the new write region for any failed-over partition.
- Caches partition-to-region routing at the partition-key-range level so failover is transparent on subsequent requests.
- Auto-populates the preferred-region order from the account's failover priority. Setting `ApplicationPreferredRegions` or `ApplicationRegion` is no longer mandatory but remains a best practice.
- Enables **Per-Partition Circuit Breaker** by default to protect read availability for the affected partition.
- Enables **Read Hedging** by default with a 1-second threshold and 500-ms step. You can override with a custom availability strategy or disable it via `DisabledAvailabilityStrategy`.

No application code changes are required beyond the SDK upgrade.

## Benefits summary

- **RTO < 2 minutes at P99** for partition-level failover, compared with 15–30 minutes for account-level failover.
- **RPO = 0** for Global Strong consistency through failover.
- **Reduced blast radius** — only the impacted partition set moves regions; everything else stays in place.
- **Active-active behavior with a single writer.** You get a level of resiliency previously reserved for multi-write accounts, without the cost and complexity of conflict resolution.
- **No application changes** beyond an SDK upgrade.
- **Transparent failback** with optional automatic reconciliation.

## What's emitted: observability

PPAF introduces a new server-side metric, **`PartitionWriteGlobalStatus`**, which reports the count of write partitions per region at any moment. Use it to confirm a failover happened, see how many partitions moved, and watch failback progress. The metric is available through the standard Azure Monitor metric surface.

On the client side, the .NET v3 and Java v4 SDKs emit per-partition routing metrics that show which region each request was served from. See the SDK metrics documentation for details.

## Operational guidance

PPAF is designed to be hands-off. The most common operational pattern is to **let PPAF drive** and use monitoring to confirm behavior rather than to intervene.

### What to do during a failover

- **Don't trigger a manual region change** as a first response. PPAF will detect, decide, and redirect writes within ~2 minutes. A manual change initiated in parallel can race with the automated decision and slow recovery.
- **Watch `PartitionWriteGlobalStatus`** in Azure Monitor to see partitions move and to confirm failback once the original region recovers.
- **Let the SDK retry.** Application code should already handle transient errors per the [Cosmos DB SDK guidance](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/conceptual-resilient-sdk-applications). During the failover window the SDK retries automatically against the new write region.

### When to manually change the write region

The only situation that warrants a manual write-region change is a **prolonged regional outage** where the failed-over partitions have settled in the new region and you want to keep them there for an extended period — for example, to align with a regional capacity decision or to free the original region for maintenance. In that case, performing a controlled write-region change is the right tool. For typical outages of minutes to hours, PPAF's automatic failback is the correct path.

### Frequently asked questions

**Do all my partitions fail over together?**
No. Each partition decides independently. A partial regional outage typically moves only the subset of partitions actually affected; healthy partitions stay in the original region.

**Will my application see errors during failover?**
Writes to affected partitions may see transient errors until the new region is elected and the SDK refreshes its routing — usually under two minutes. The SDK retries automatically. Reads to other regions and writes to unaffected partitions continue normally.

**Can I lose data?**
For **Strong** consistency, no — RPO is 0 through PPAF failovers. For other consistency levels, PPAF picks the replica with the most recent committed state to minimize loss, and reconciles any divergent writes on failback using last-writer-wins.

**Do I need to do anything on failback?**
No. Failback is automatic, uses incremental catch-up rather than a full rebuild, and reconciles divergent writes in the background.

**Does PPAF replace multi-write?**
PPAF is for single-write-region accounts that want fast, automatic recovery without conflict-resolution complexity. Multi-write (multi-region writes) remains the right choice for workloads that need active-active write capability across regions at all times.

## Pricing

PPAF is part of the **Business Critical** service tier for Azure Cosmos DB. See [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/) for current rates.

## Prerequisites at a glance

| Requirement | Value |
|---|---|
| API | NoSQL (Core SQL API) |
| Account topology | Single write region with at least one read region |
| Throughput | Provisioned (manual or autoscale) |
| Backup | Continuous (point-in-time restore) |
| Consistency | Strong, Session, Consistent Prefix, or Eventual |
| Cloud | Azure public cloud regions |
| Connection mode | Direct |
| SDK | .NET v3 ≥ 3.55.0, Java v4 ≥ 4.75.0, Python ≥ 4.15.0, Node.js ≥ 4.7.0 (Go not supported at GA) |

## Next steps

- [Configure and use per-partition automatic failover](how-to-configure-per-partition-automatic-failover.md)
- [Consistency levels in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [High availability in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/high-availability)
- Sample app and chaos script: [AzureCosmosDB/ppaf-samples](https://github.com/AzureCosmosDB/ppaf-samples)

For questions and feedback: **cosmosdbppafpreview@microsoft.com**
