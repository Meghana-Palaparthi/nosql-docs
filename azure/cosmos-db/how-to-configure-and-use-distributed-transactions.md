---
title: Use distributed transactions
description: Guide to enable distributed transactions on an Azure Cosmos DB account and use them from the .NET SDK to perform atomic, multi-partition reads and writes.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: how-to
ms.date: 06/02/2026
ai-usage: ai-assisted
appliesto:
  - ✅ NoSQL
---

# Use distributed transactions in Azure Cosmos DB

> [!IMPORTANT]
> Distributed transactions in Azure Cosmos DB are currently in **public preview**. This preview is provided without a service-level agreement (SLA). Behavior, limits, and supported scenarios could change before general availability.

This article shows you how to enable distributed transactions on an Azure Cosmos DB for NoSQL account and use them from the .NET SDK to commit atomic read and write operations that span multiple logical partitions, containers, and databases within the same account and region.

The examples in this article use a single scenario — a `banking` database with two containers, `accounts` (partitioned by account ID) and `ledger` (partitioned by posting month) — so the same items appear across the read and write examples.

## Prerequisites

Before you begin, make sure you have:

- An active **Azure subscription**. If you don't have one, [create a free account](https://azure.microsoft.com/free/).
- An **Azure Cosmos DB for NoSQL account**. The account must:
  - Use the **NoSQL (Core SQL) API**. MongoDB, Cassandra, Table, and Gremlin APIs aren't supported in preview.
  - Be a **provisioned throughput** (manual or autoscale) account. Serverless accounts aren't supported in preview.
  - Run on a **public Azure cloud region**. Sovereign, air-gapped, and government clouds aren't supported in preview.
  - Be a **single-write-region** account. Multi-region write accounts aren't supported in preview.
  - Not be configured with any of the following features:
    -  **Customer-Managed Keys (CMK)**
    -  **Per-Partition Automatic Failover (PPAF)**
    -  **Continuous backup**
    -  **Long-Term Retention**
    -  **Partition Merge**
    -  **Hierarchical Partition Keys (HPK)**
    -  **Fabric Native databases**
- The latest version of the **Azure Cosmos DB .NET v3 SDK** (`Microsoft.Azure.Cosmos`) from NuGet.

## Request enrollment for your account

Distributed transactions are a **public preview** feature. Self-service enrollment through the Azure portal, Azure CLI, or PowerShell is currently **not** available.

To request enrollment, submit your onboarding request at [https://aka.ms/cosmosdb/dtx-onboard](https://aka.ms/cosmosdb/dtx-onboard). Requests are typically fulfilled within 1-2 business days. You'll receive a confirmation once your account is ready.

## Install the required .NET SDK

Add the latest version o Azure Cosmos DB .NET SDK to your project.

```dotnetcli
dotnet add package Microsoft.Azure.Cosmos
```

## Initialize the client

Initialize a `CosmosClient` against your enrolled account using either an account key or Microsoft Entra ID (formerly Azure AD).

### Option A: Account key

```csharp
using Microsoft.Azure.Cosmos;

string endpoint = "https://<your-account>.documents.azure.com:443/";
string key = "<your-primary-key>";

CosmosClient client = new CosmosClient(endpoint, key);
```

### Option B: Microsoft Entra ID (recommended for production)

```csharp
using Microsoft.Azure.Cosmos;
using Azure.Identity;

string endpoint = "https://<your-account>.documents.azure.com:443/";

CosmosClient client = new CosmosClient(
    endpoint,
    new DefaultAzureCredential());
```

The identity used must have data-plane write permissions (for example, the **Cosmos DB Built-in Data Contributor** role) on **every** container that participates in a transaction. Role checks occur per individual operation inside the transaction batch.

## Commit a multi-partition write transaction

The .NET v3 SDK adds `CreateDistributedWriteTransaction()` API on `CosmosClient`. Chain one operation per item, then call `CommitTransactionAsync` to submit the entire batch as a single atomic unit.

The following example atomically transfers 100 units from `account-A` to `account-B` and records the corresponding entry in the `ledger` container. The two accounts live in different logical partitions of the `accounts` container, and the ledger entry lives in a separate container.

```csharp
// Starting state: account-A holds 1000, account-B holds 1000.
// This transaction debits 100 from account-A and credits 100 to account-B,
// and writes a matching ledger entry — all atomically.
var updatedAccountA = new { id = "account-A", pk = "account-A", balance = 900.00 };
var updatedAccountB = new { id = "account-B", pk = "account-B", balance = 1100.00 };
var ledgerEntry     = new { id = "txn-1001",  pk = "2026-06",   from = "account-A", to = "account-B", amount = 100.00 };

DistributedTransactionResponse response = await client
    .CreateDistributedWriteTransaction()
    .ReplaceItem("banking", "accounts", new PartitionKey("account-A"), updatedAccountA)
    .ReplaceItem("banking", "accounts", new PartitionKey("account-B"), updatedAccountB)
    .CreateItem ("banking", "ledger",   new PartitionKey("2026-06"),   ledgerEntry)
    .CommitTransactionAsync(CancellationToken.None);

if (response.IsSuccessStatusCode)
{
    Console.WriteLine("Transaction committed.");
}
```

All three items are either committed together or none of them are. There's no partial state.

### Mix operation types in a single transaction

You can mix `CreateItem`, `UpsertItem`, `ReplaceItem`, `PatchItem`, and `DeleteItem` within the same transaction.

```csharp
await client
    .CreateDistributedWriteTransaction()
    .UpsertItem("banking", "accounts", new PartitionKey("account-A"), updatedAccountA)
    .UpsertItem("banking", "accounts", new PartitionKey("account-B"), updatedAccountB)
    .CreateItem("banking", "ledger",   new PartitionKey("2026-06"),   ledgerEntry)
    .CommitTransactionAsync(CancellationToken.None);
```

## Commit a multi-partition read transaction

Use `CreateDistributedReadTransaction()` when you need a **point-in-time consistent snapshot** of items that live in different logical partitions, containers, or databases. Unlike issuing several independent `ReadItemAsync` calls, a distributed read transaction returns all items as they existed at a single committed instant, so the reader never observes a partially applied write transaction.

Chain one `ReadItem` call per item, then call `CommitTransactionAsync` to fetch the snapshot:

```csharp
DistributedReadTransaction txn = client.CreateDistributedReadTransaction();

txn.ReadItem("banking", "accounts", new PartitionKey("account-A"), "account-A")
   .ReadItem("banking", "accounts", new PartitionKey("account-B"), "account-B");

DistributedTransactionResponse response = await txn.CommitTransactionAsync();
```

### When to use a distributed read transaction

Distributed read transactions are most useful when correctness depends on the **mutual consistency** of items spread across partitions. Common scenarios include:

- **Cross-account balance reconciliation.** Read every account balance involved in a multi-leg funds transfer to confirm that debits and credits sum to zero, without the risk of reading one leg before and the other after a concurrent transfer commits.
- **Inventory and order validation.** Read a stock item and the corresponding pending-orders record together before deciding whether to accept a new order, so the available quantity and reserved quantity always reflect the same instant.
- **Audit and compliance snapshots.** Capture a coherent view of related records (for example, an order, its line items, and the customer profile that live in different containers) for reporting, exports, or regulatory evidence.
- **Cache or read-model rebuilds.** Hydrate a denormalized view or materialized projection from several source containers without seeing torn writes from in-flight distributed write transactions.

For single-item reads, or for unrelated items where mutual consistency isn't required, continue to use `ReadItemAsync` — it has lower latency and consumes fewer request units.


## Multi-region considerations

In a multi-region account, distributed transactions are **atomic within the write region only**. Multi-region write accounts aren't supported in preview — the account must have a single write region.

- All transactional reads and writes are routed to the account's **write/hub region** by the SDK.
- Committed data replicates to secondary regions **asynchronously and per-partition**. Readers in secondary regions may temporarily observe partial updates until replication catches up.

For applications that require global read-after-write of transactional data, either:

- Route reads to the write region, or
- Use **strong** consistency at the account level and the session token returned by the commit response.

## Limits in public preview

| Limit | Value |
|---|---|
| Maximum operations per transaction | 100 |
| Maximum payload size per transaction | 2 MB |


These limits may change before general availability.

## Supported APIs and SDKs

| API | Supported in preview |
|---|---|
| NoSQL (Core SQL) | Yes |
| MongoDB | No |
| Cassandra | No |
| Table | No |
| Gremlin | No |

| SDK | Status |
|---|---|
| .NET (C#) v3 | Available |
| Java | Coming soon |
| Python | Coming soon |
| Node.js | Coming soon |

## Related content

- [Consistency levels in Azure Cosmos DB](consistency-levels.md)
- [Optimistic concurrency control with ETags](database-transactions-optimistic-concurrency.md)
- [Azure Cosmos DB .NET SDK v3 reference](/dotnet/api/microsoft.azure.cosmos)

