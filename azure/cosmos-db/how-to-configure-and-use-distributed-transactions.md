---
title: Use distributed transactions in Azure Cosmos DB for NoSQL
description: Step-by-step guide to enable distributed transactions on an Azure Cosmos DB for NoSQL account and use them from the .NET SDK to perform atomic, multi-partition writes.
author: sushantrane
ms.author: srane
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: how-to
ms.date: 06/02/2026
appliesto:
  - ✅ NoSQL
---

# Use distributed transactions in Azure Cosmos DB for NoSQL


> [!IMPORTANT]
> Distributed transactions in Azure Cosmos DB for NoSQL are currently in **public preview**. This preview is provided without a service-level agreement (SLA). Behavior, limits, and supported scenarios may change before general availability. Public preview is **gated** — accounts must be explicitly onboarded by the Azure Cosmos DB engineering team.

This article shows you how to enable distributed transactions on an Azure Cosmos DB for NoSQL account and use them from the .NET SDK to commit atomic write operations that span multiple logical partitions, containers, and databases within the same account and region.

If you're new to the feature, start with [Distributed transactions in Azure Cosmos DB for NoSQL](distributed-transactions.md) for a conceptual overview.

## Prerequisites

Before you begin, make sure you have:

- An active **Azure subscription**. If you don't have one, [create a free account](https://azure.microsoft.com/free/).
- An **Azure Cosmos DB for NoSQL account**. The account must:
  - Use the **NoSQL (Core SQL) API**. MongoDB, Cassandra, Table, and Gremlin APIs are not supported in preview.
  - Be a **provisioned throughput** (manual or autoscale) account. Serverless accounts are not supported in preview.
  - Run on a **public Azure cloud region**. Sovereign, air-gapped, and government clouds are not supported in preview.
  - Be a **single-write-region** account. Multi-region write (multi-master) accounts aren't supported in preview.
  - Not have any of the following features enabled: **Customer-Managed Keys (CMK)**, **Per-Partition Automatic Failover (PPAF)**, **Partition Reuse**, **Continuous backup / Point-in-Time Restore (PITR)**, **Long-Term Retention (LTR)**, **Merge**, **Hierarchical Partition Keys (HPK)**, **16 MB document support**, or **Fabric Native databases**.
- The **Azure Cosmos DB .NET v3 SDK** that includes the `CreateDistributedWriteTransaction` API. Use the latest preview package from NuGet (search for `Microsoft.Azure.Cosmos` preview versions tagged for distributed transactions).
- A **database named `dts-log-db` must not already exist** on the account. Enabling distributed transactions creates a system database with this name. Enrollment fails if the name is already in use.

## Step 1: Request enrollment for your account

Distributed transactions are a **public preview** feature. Self-service enrollment through the Azure portal, Azure CLI, or PowerShell is **not** available in preview.

To request enrollment, submit your onboarding request at [https://aka.ms/cosmosdb/dtx-onboard](https://aka.ms/cosmosdb/dtx-onboard). Requests are typically fulfilled within 1-2 business days.

Enrollment runs a backend workflow that:

1. Validates that none of the blocked features listed in the prerequisites are enabled.
2. Creates a system database `dts-log-db` and a system container `dts-log-coll` on the account. These store the coordinator transaction log and are readable and writable only by the system.
3. Sets an account-level capability that allows new distributed transactions to be initiated.

You'll receive a confirmation once your account is ready.

> [!NOTE]
> Do not create a database named `dts-log-db` on your account. This name is reserved for the distributed transactions system database.

## Step 2: Install the required .NET SDK

Add the preview Azure Cosmos DB .NET SDK to your project:

```dotnetcli
dotnet add package Microsoft.Azure.Cosmos --version <latest-version>
```

Make sure the installed version supports distributed transactions. The `CreateDistributedWriteTransaction()` method on `CosmosClient` is the indicator that the feature is available in your SDK build.

## Step 3: Initialize the client

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

## Step 4: Commit a multi-partition transaction

The .NET v3 preview SDK adds a fluent `CreateDistributedWriteTransaction()` API on `CosmosClient`. Chain one operation per item, then call `CommitTransactionAsync` to submit the entire batch as a single atomic unit.

The following example atomically creates three items: two in one container (with different partition key values) and one in a container in a different database.

```csharp
var doc1 = new { id = "order-1001", pk = "customerA", total = 250.00 };
var doc2 = new { id = "order-1002", pk = "customerB", total = 175.00 };
var doc3 = new { id = "audit-1001", pk = "audit-2026-06", action = "order-create" };

DistributedTransactionResponse response = await client
    .CreateDistributedWriteTransaction()
    .CreateItem(databaseId: "orders-db",   containerId: "orders",  new PartitionKey(doc1.pk), doc1)
    .CreateItem(databaseId: "orders-db",   containerId: "orders",  new PartitionKey(doc2.pk), doc2)
    .CreateItem(databaseId: "audit-db",    containerId: "audit",   new PartitionKey(doc3.pk), doc3)
    .CommitTransactionAsync(CancellationToken.None);

if (response.IsSuccessStatusCode)
{
    Console.WriteLine($"Transaction committed.");
}
```

All three items are either committed together or none of them are. There's no partial state.

### Mix operation types in a single transaction

You can mix `CreateItem`, `UpsertItem`, `ReplaceItem`, `PatchItem`, and `DeleteItem` within the same transaction.

```csharp
await client
    .CreateDistributedWriteTransaction()
    .UpsertItem("inventory-db", "inventory", new PartitionKey("sku-A100"), updatedSkuA)
    .UpsertItem("inventory-db", "inventory", new PartitionKey("sku-B200"), updatedSkuB)
    .CreateItem ("ledger-db",   "ledger",    new PartitionKey("2026-06"),  ledgerEntry)
    .CommitTransactionAsync(CancellationToken.None);
```

## Step 5: Use conditional checks for read-modify-write

Distributed transactions use a single-request batch model. All operations must be known up front — there is no `BEGIN`/`COMMIT` session that lets you read and then conditionally write within one transaction.

To safely implement a read-modify-write pattern (for example, debit one account and credit another), use the two-step optimistic pattern:

1. **Read** the items you intend to modify, capturing each item's `ETag`.
2. **Submit** a write transaction that includes a `DocumentConditionCheck` against the captured `ETag` for every item you read. If any item has been modified by another process in between, the transaction is aborted with HTTP `412 Precondition Failed`, and your application should retry from step 1.

```csharp
// Step 1: Read both account documents
ItemResponse<Account> bobRead    = await accountsContainer.ReadItemAsync<Account>("bob",   new PartitionKey("bob"));
ItemResponse<Account> aliceRead  = await accountsContainer.ReadItemAsync<Account>("alice", new PartitionKey("alice"));

Account bob   = bobRead.Resource;
Account alice = aliceRead.Resource;

if (bob.Balance < 100) throw new InvalidOperationException("Insufficient funds");

bob.Balance   -= 100;
alice.Balance += 100;

// Step 2: Conditional write transaction
DistributedTransactionResponse response = await client
    .CreateDistributedWriteTransaction()
    .CheckCondition("bank-db", "accounts", new PartitionKey("bob"),   "bob",   bobRead.ETag)
    .CheckCondition("bank-db", "accounts", new PartitionKey("alice"), "alice", aliceRead.ETag)
    .ReplaceItem  ("bank-db", "accounts", new PartitionKey("bob"),   bob)
    .ReplaceItem  ("bank-db", "accounts", new PartitionKey("alice"), alice)
    .CommitTransactionAsync(CancellationToken.None);
```

If a `412 Precondition Failed` is returned, re-read the items and retry the transaction with a **new** idempotency token.

## Step 6: Retry safely using the idempotency token

Each transaction is identified by a client-side idempotency token (a UUID v4 the SDK generates by default). If the SDK times out or the network drops before you receive a response, retrying with the **same** token is safe — the system de-duplicates and returns the original outcome:

- If the original transaction is still running, you receive an *in-progress, retry later* status.
- If it has already committed or aborted, you receive the original committed or aborted result.
- If it failed non-retriably, you receive the original error.

Use the same `idempotencyToken` on retry:

```csharp
string token = Guid.NewGuid().ToString();

try
{
    var response = await client
        .CreateDistributedWriteTransaction(idempotencyToken: token)
        .UpsertItem(/* ... */)
        .UpsertItem(/* ... */)
        .CommitTransactionAsync();
}
catch (CosmosException ex) when (ex.IsRetriable())
{
    // Retry with the SAME token
    var response = await client
        .CreateDistributedWriteTransaction(idempotencyToken: token)
        .UpsertItem(/* ... */)
        .UpsertItem(/* ... */)
        .CommitTransactionAsync();
}
```

Only generate a **new** token when you intend to submit a new logical attempt (for example, after re-reading items to resolve a `412` conflict).

## Step 7: Handle errors

The `DistributedTransactionResponse` exposes per-operation results. If the transaction is aborted, the response indicates which sub-operation triggered the abort.

Common error codes:

| HTTP status | Meaning | Recommended action |
|---|---|---|
| 200 / 201 | Transaction committed successfully. | — |
| 412 | Precondition failed (ETag mismatch). | Re-read items; retry with a new idempotency token. |
| 429 | Throughput exceeded on one or more participant partitions. | Back off and retry. Consider increasing RU/s or enabling autoscale. |
| 409 | Write conflict (for example, document already exists, or two transactions targeted the same item). | Application-level: handle conflict or retry. |
| 404 | Item not found. | Verify the item exists; check for concurrent deletes. |
| 403 | Authorization failure. | Verify the identity has data-plane write permission on **every** container in the transaction. |
| 408 / SDK timeout | Ambiguous outcome. | Safe to retry with the **same** idempotency token. |

For diagnostics, capture the full `CosmosException.Diagnostics` string — it contains the per-operation timeline, the transaction ID, and activity IDs you can share with support.

## Step 8: Multi-region considerations

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
| Maximum payload size per transaction | 4 MB |
| Maximum transaction lifetime | 60 seconds |
| Maximum document size | 2 MB (standard Cosmos DB limit; 16 MB documents not supported) |
| Maximum scope | One Cosmos DB account, one region (write region) |

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
| .NET (C#) v3 preview | Available |
| Java | Coming soon |
| Python | Coming soon |
| JavaScript / TypeScript | Coming soon |

## Verify the feature is enabled

After enrollment, you can verify the feature is enabled by listing the system database on your account:

```csharp
try
{
    DatabaseResponse db = await client
        .GetDatabase("dts-log-db")
        .ReadAsync();
    Console.WriteLine("Distributed transactions are enabled on this account.");
}
catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
{
    Console.WriteLine("Distributed transactions are NOT enabled. Contact the engineering team to enroll.");
}
```

Do not delete or modify `dts-log-db` or `dts-log-coll`. They are managed by the system.

## Send feedback

Distributed transactions are in public preview, and your feedback shapes the path to general availability. To share feedback, please reach out to the team at [azcosmosdbdtxpreview@microsoft.com](mailto:azcosmosdbdtxpreview@microsoft.com).



## Next steps

- [Distributed transactions in Azure Cosmos DB for NoSQL — concepts](distributed-transactions.md)
- [Consistency levels in Azure Cosmos DB](consistency-levels.md)
- [Optimistic concurrency control with ETags](database-transactions-optimistic-concurrency.md)
- [Azure Cosmos DB .NET SDK v3 reference](/dotnet/api/microsoft.azure.cosmos)
