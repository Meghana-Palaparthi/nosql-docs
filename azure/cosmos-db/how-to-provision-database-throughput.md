---
title: Provision Database Throughput
description: Learn how to provision throughput at the database level in Azure Cosmos DB for NoSQL using Azure portal, CLI, PowerShell and various other SDKs.
author: markjbrown
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: how-to
ms.date: 10/15/2020
ms.author: mjbrown
ms.custom: devx-track-azurecli, devx-track-csharp, devx-track-dotnet
appliesto:
  - ✅ NoSQL
---

# Provision standard (manual) throughput on a database in Azure Cosmos DB - API for NoSQL


> [!NOTE]
> Shared database throughput is not recommended for most workloads. While it can simplify provisioning in some scenarios, sharing throughput across multiple containers can lead to unpredictable and undesirable performance and scale behaviors. Because containers in the same database share partitions, scaling database throughput to support a large or growing container may trigger repartitioning of smaller, co‑located containers, spreading them overly thin across too many partitions. We recommend configuring throughput at the container level. Customers with advanced scenarios who understand these tradeoffs can still create and manage shared database throughput programmatically using the Azure Cosmos DB SDKs.

If you are using a different API, see [API for MongoDB](mongodb/how-to-provision-throughput.md), [API for Cassandra](cassandra/how-to-provision-throughput.md), [API for Gremlin](gremlin/how-to-provision-throughput.md) articles to provision the throughput.

## Provision throughput using Azure portal

1. Sign in to the [Azure portal](https://portal.azure.com/).

1. [Create a new Azure Cosmos DB account](how-to-create-account.md), or select an existing Azure Cosmos DB account.

1. Open the **Data Explorer** pane, and select **New Database**. Provide the following details:

   * Enter a database ID.
   * Enter a throughput that you want to provision (for example, 1000 RU/s). RU/s (Request Units per second) represent the amount of reserved capacity for your database operations.
   * Enter a name for your container under **Container ID**
   * Enter a **Partition key**
   * Select **OK**.

    :::image type="content" source="media/how-to-provision-database-throughput/provision-database-throughput-portal-sql-api.png" alt-text="Screenshot of New Database dialog box":::

> [!NOTE]
> The portal previously included a **Share throughput across containers** option. For most workloads, we recommend provisioning throughput on individual containers instead. If you have an advanced scenario that requires shared throughput, you can create it programmatically using the Azure Cosmos DB SDKs.

## Provision throughput using Azure CLI or PowerShell

To create a database with shared throughput see,

* [Create a database using Azure CLI](manage-with-cli.md#create-a-database-with-shared-throughput)
* [Create a database using PowerShell](manage-with-powershell.md#create-db-ru)

## Provision throughput using .NET SDK

> [!Note]
> You can use Azure Cosmos DB SDKs for API for NoSQL to provision throughput for all APIs. You can optionally use the following example for API for Cassandra as well.

# [.NET SDK V2](#tab/dotnetv2)

```csharp
//set the throughput for the database
RequestOptions options = new RequestOptions
{
    OfferThroughput = 500
};

//create the database
await client.CreateDatabaseIfNotExistsAsync(
    new Database {Id = databaseName},  
    options);
```

# [.NET SDK V3](#tab/dotnetv3)

[!code-csharp[](~/../samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos/tests/Microsoft.Azure.Cosmos.Tests/SampleCodeForDocs/DatabaseDocsSampleCode.cs?name=DatabaseCreateWithThroughput)]

---

## Next steps

See the following articles to learn about provisioned throughput in Azure Cosmos DB:

* [Globally scale provisioned throughput](request-units.md)
* [Provision throughput on containers and databases](set-throughput.md)
* [How to provision standard (manual) throughput for a container](how-to-provision-container-throughput.md)
* [How to provision autoscale throughput for a container](how-to-provision-autoscale-throughput.md)
* [Request units and throughput in Azure Cosmos DB](request-units.md)
