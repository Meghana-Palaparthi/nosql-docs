---
title: Distributed transactions in Azure Cosmos DB for NoSQL
description: Learn how distributed transactions provide atomic, all-or-nothing commit across multiple logical partitions, containers, and databases in an Azure Cosmos DB for NoSQL account.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: concept-article
ms.date: 06/02/2026
appliesto:
  - ✅ NoSQL
---

# Distributed transactions in Azure Cosmos DB for NoSQL


> [!IMPORTANT]
> Distributed transactions in Azure Cosmos DB for NoSQL are currently in **public preview**. This preview is provided without a service-level agreement (SLA). Behavior, limits, and supported scenarios may change before general availability.

**Distributed transactions** in Azure Cosmos DB for NoSQL let an application commit a set of write operations as a single atomic unit, even when those operations span **multiple logical partitions, containers, or databases** within the same account and region. Either all operations succeed and are visible together, or none of them apply. There's no partial state.

This article explains what distributed transactions are, the guarantees they offer, how they're coordinated under the hood, and when you should — and shouldn't — use them.

## Why distributed transactions

Before this feature, Azure Cosmos DB for NoSQL offered atomic multi-document writes through three mechanisms:

- JavaScript stored procedures
- The atomic **Batch** API
- The MongoDB 4.0 transaction API (MongoDB API)

All three were limited to operations on documents that shared the **same partition key value** within a **single container**. Anything that crossed partitions or containers had to be implemented at the application level with custom compensation logic, sagas, or best-effort retry — patterns that are hard to get right and expensive to operate.

Distributed transactions remove that constraint. They give you a single API call that commits across partitions and containers, with the same correctness guarantees you would expect from a traditional database — without giving up the scale, global distribution, and elasticity of Azure Cosmos DB.

## What you can do

A single distributed transaction can:

- Write to **multiple items with different partition key values** within a container.
- Write to **multiple containers** in the same or different databases of the same account.
- Include **conditional checks** (ETag-based preconditions) so writes only commit if the items haven't changed since you read them.
- Be **retried safely** using a client-generated idempotency token — duplicate retries don't produce duplicate side effects.

## Guarantees

Distributed transactions in Azure Cosmos DB for NoSQL provide the following guarantees within their preview scope:

| Guarantee | What it means |
|---|---|
| **Atomic commit** | Either all operations in the transaction commit, or none do. Readers never observe a partial state. |
| **Serializable isolation** | Concurrent transactions produce the same result as if they ran in some serial order chosen by the system — the strongest standard isolation level. |
| **Durable commit** | Once a transaction is acknowledged as committed, its writes are persisted according to the account's configured replication. The Gateway maintains a durable coordinator log so in-flight transactions can be recovered after a coordinator failure. |
| **Idempotent retry** | Each transaction carries a client-generated UUID v4 idempotency token; safe to retry on transient failure without risk of double-commit. |
| **Constraint enforcement** | The transaction respects ETag preconditions and unique-index constraints at commit time. |

Atomicity in public preview is scoped to a single write region of a single account. Cross-region atomicity is on the roadmap to general availability, not in scope for the preview.

Causal ordering across transactions is preserved using a **Hybrid Logical Clock (HLC)** assigned to each transaction, so application logs and downstream readers can reason about the order in which transactions took effect across partitions.

## The transaction model

Distributed transactions use a **single-request batch model**: all operations the transaction will perform are submitted in one request, and either succeed or fail together. There's no interactive session — no `BEGIN`, no streamed statements, no `COMMIT`.

This model is intentional. It keeps transactions short, prevents long-held locks, and avoids the multi-tenant pitfalls of arbitrary long-running interactive transactions. The same design choice was made by Amazon DynamoDB for the same reasons.

**What this means in practice:**

- All operations must be known up front. You cannot read inside a transaction and then conditionally write based on what you just read in the same transaction.
- Read-modify-write patterns are still supported, using a **two-step optimistic pattern**: read items first (capturing each item's ETag), then submit a write transaction that includes ETag-based conditional checks. If anything changed in between, the transaction aborts with `412 Precondition Failed`, and the application retries.
- Long-running queries cannot run inside a transaction.

If you need true interactive multi-statement transactions across partitions (the `BEGIN`/`SELECT`/`UPDATE`/`COMMIT` pattern), consider whether **Azure Cosmos DB for MongoDB (vCore)** running on PostgreSQL is a better fit for that workload.

## How it works

Under the hood, a distributed transaction is coordinated using a **two-phase commit (2PC)** protocol orchestrated by a **transaction coordinator** that runs on the Cosmos DB Compute Gateway.

![Conceptual diagram: SDK sends a batch to the Compute Gateway coordinator, which runs prepare and commit phases against participant partitions.](media/distributed-transactions/coordinator.png)

A simplified end-to-end flow:

1. **Client submits the batch.** The .NET SDK builds the transaction (a set of operations across one or more partitions and containers) and sends it in a single request to the Compute Gateway. The SDK attaches a client-generated **idempotency token** (UUID v4) that uniquely identifies the attempt.
2. **Coordinator assigns a transaction ID and HLC timestamp.** Any Compute Gateway node can act as the coordinator. The coordinator writes the transaction request, the token, and the HLC timestamp to a system collection — the **coordinator transaction log** (`dts-log-coll` in the system database `dts-log-db`).
3. **Prepare phase.** The coordinator routes each operation to the participant partition that owns the target item. Each participant validates preconditions (ETag checks, no concurrent in-flight transaction on the same item, no clock-ordering violation) and persists a *prepared* record in its local transaction log. It then votes **Yes** or **No** to the coordinator.
4. **Decision.** If every participant voted Yes, the coordinator records a **Commit** decision in the coordinator log. If any participant voted No (or didn't respond), it records an **Abort** decision. The decision is durable before any participant is notified.
5. **Commit/abort phase.** The coordinator instructs every participant to commit or roll back. Participants apply (or discard) the writes, update each affected document's HLC timestamp, and clear their local transaction log entry.
6. **Response to client.** The SDK receives a `DistributedTransactionResponse` with per-operation results, an updated session token for each operation, and the total request units (RUs) consumed.

A background **recovery task** running on each account's Master partition periodically scans the coordinator log for transactions that are stuck (for example, because a coordinator crashed between prepare and commit) and drives them to completion using the persisted decision. This is safe even if multiple coordinators try to drive the same transaction, because every phase's decision is durable before it's communicated — a duplicate prepare or commit returns the previously persisted outcome rather than re-deciding.

### Why HLC?

Each transaction is stamped with a **Hybrid Logical Clock** value: a 64-bit integer that combines the node's physical wall-clock time (upper 48 bits, millisecond precision) with a monotonic logical counter (lower 16 bits). HLC gives two important properties:

- **Causal ordering across partitions** — if transaction *T*₁ committed before transaction *T*₂ started, *T*₂'s HLC is strictly greater than *T*₁'s. Readers and downstream systems can reason about the order in which writes took effect even though no global wall clock exists.
- **Tolerance for clock skew** — HLC does **not** require microsecond-synchronized hardware clocks (unlike Spanner's TrueTime). Cosmos DB nodes are loosely synchronized within a region to millisecond granularity by Azure host NTP, and HLC's logical counter handles any residual drift.

### Idempotency

Every transaction carries a client-generated UUID v4 idempotency token. If the client retries a transaction with the same token, the coordinator de-duplicates:

- If the original is still in flight, the coordinator returns *in-progress, retry later*.
- If it already committed or aborted, the coordinator returns the original outcome.
- If it failed non-retriably, the coordinator returns the original error.

This makes safe-retry the default, even across network failures, SDK timeouts, and coordinator crashes. The token is account-scoped, pre-authenticated by the Compute Gateway before reaching the coordinator, and has 122 bits of randomness — so it's safe to expose at the client without enabling brute-force replay.

## Reads inside transactions

Transactional **reads** (a consistent snapshot across multiple items on different partitions) are supported through the same coordinator using a similar protocol. A read transaction returns the values of all requested items as they would appear at a single logical point in time, even if the items live on different physical partitions.

The classic use case: a bank balance summary that reads accounts A and B. A non-transactional pair of reads could observe an inconsistent total mid-transfer (A debited but B not yet credited). A transactional read returns either the *before* state or the *after* state of the transfer, never the in-between.

In public preview, transactional reads are routed to the account's write region, like transactional writes.

## Consistency levels

Distributed transactions are compatible with all of the Cosmos DB consistency levels:

| Consistency level | Behavior with distributed transactions |
|---|---|
| **Strong** | The commit response includes the global LSN per participant partition. The client polls until a quorum of regions has seen the commit before treating the transaction as durable. |
| **Bounded staleness** | Backend mechanism; works as today. |
| **Session** | The commit response returns updated session tokens per operation. Subsequent reads use those tokens to guarantee **read-your-own-writes** within the session. |
| **Consistent prefix** | Works as today. |
| **Eventual** | Works as today. |

## Scope and limits in public preview

Distributed transactions in public preview are intentionally scoped narrowly so we can stabilize the feature with real customer workloads before broadening it.

### Supported

- **NoSQL (Core SQL) API** only
- **Single-region atomicity** — atomic within the write/hub region
- Cross-partition writes within a container
- Cross-container and cross-database writes within a single account
- Create, upsert, replace, patch, delete on documents
- Conditional checks via ETag preconditions
- Idempotent commits via client-generated UUID v4 tokens
- **.NET v3 preview SDK** (Java, Python, and JavaScript SDKs coming soon)
- All Cosmos DB consistency levels (Strong, Bounded staleness, Session, Consistent prefix, Eventual)
- Provisioned throughput (manual and autoscale)
- Account-level authentication: account keys and Microsoft Entra ID (AAD)
- Splits and partition migration (transparent to the application)

### Not supported in preview

- **Other APIs**: MongoDB, Cassandra, Table, Gremlin — explicitly blocked
- **Cross-account** transactions — a transaction cannot span multiple Cosmos DB accounts
- **Cross-region atomicity** — geo-replication remains asynchronous per-partition; readers in secondary regions can briefly observe partial updates
- **Interactive multi-step** transactions (`BEGIN`/`COMMIT` with arbitrary reads and writes between)
- **Long-running queries** inside a transaction
- **Serverless** accounts
- **Customer-Managed Keys (CMK)**
- **Per-Partition Automatic Failover (PPAF)**
- **Partition Reuse**
- **Hierarchical Partition Keys (HPK)**
- **16 MB documents**
- **Continuous backup / Point-in-Time Restore (PITR)** — including in-account restore of deleted containers
- **Long-Term Retention (LTR)**
- **Merge**
- **Fabric Native** databases
- **Sovereign, air-gapped, and government clouds**
- **Self-service enrollment** via Azure portal, CLI, or PowerShell — enrollment is gated through engineering

### Hard limits in preview

| Limit | Value |
|---|---|
| Operations per transaction | 100 |
| Payload size per transaction | 16 MB |
| Transaction lifetime | 60 seconds |
| Scope | One account, one write region |

These limits may change before general availability.

## Multi-region and failover behavior

In a multi-region account:

- **All transactional reads and writes are routed to the write/hub region.** The SDK enforces this. If a request hits a non-write region, the gateway returns `403.3` and the SDK retries against the correct region.
- **Replication to secondary regions is asynchronous and per-partition.** It does not honor transaction boundaries. A reader in a read region may briefly see partial updates from a recently committed distributed transaction until all participant partitions have replicated.
- **Failover behavior** is consistent with strong-consistency expectations — committed transactions remain committed in the new write region after failover, and stuck in-progress transactions are driven to completion by the recovery path.

If your workload requires global read-after-write of transactional data, route reads to the write region or use the **session token** returned by the commit response.

## Backup, restore, and PITR

Point-in-Time Restore is **blocked** for accounts that enable distributed transactions in public preview. This is the same posture as Amazon DynamoDB: PITR doesn't honor distributed transaction boundaries because each partition's backup is flushed independently and asynchronously. We're working on transactional-boundary-aware restore for general availability.

## Performance considerations

Distributed transactions trade latency for cross-partition atomicity. A two-phase commit performs multiple rounds of consensus writes — the coordinator log write, prepare on every participant, the commit decision in the coordinator log, and commit on every participant. Expect latency to be **meaningfully higher than single-partition Batch operations**.

Guidance:

- **Use distributed transactions only where cross-partition or cross-container atomicity is genuinely required.** For single-partition atomicity, continue to use the existing Batch API — it's faster.
- **Keep transactions small.** Stay well below the 100-operation and 4-MB limits where you can.
- **Use autoscale or generous provisioned RU/s** on participant containers. Throttling on any participant aborts the transaction with `429`.
- **Expect higher latency** in multi-region accounts because participant strong writes traverse more regions.

## When to use distributed transactions

Distributed transactions are designed for workloads where **correctness across partitions matters more than the lowest possible latency**. Typical patterns include:

- **Financial movements** — transfers between accounts that live on different partitions.
- **Inventory and ledger updates** — decrement stock in one container and write an audit row in another, as a single unit.
- **Cross-tenant or cross-entity state changes** — a single operation that updates two related entities with different partition keys.
- **Idempotent business workflows** — workflows where retries must be safe and partial state is unacceptable.

If your workload only requires atomicity across documents that share a partition key, prefer the existing **Batch** API.

## Related content

- [Use distributed transactions in Azure Cosmos DB for NoSQL — how-to](how-to-distributed-transactions.md)
- [Consistency levels in Azure Cosmos DB](consistency-levels.md)
- [Optimistic concurrency control with ETags](database-transactions-optimistic-concurrency.md)
- [Transactional batch operations in Azure Cosmos DB for NoSQL](transactional-batch.md)
