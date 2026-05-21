---
title: Use Bulk Executor Java Library in to Perform Bulk Import and Update Operations
description: Bulk import and update Azure Cosmos DB documents using bulk executor Java library
author: TheovanKraay
ms.author: thvankra
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.devlang: java
ms.topic: how-to
ms.date: 05/21/2026
ms.custom: devx-track-java, devx-track-extended-java
ai-usage: ai-assisted
appliesto:
  - ✅ NoSQL
---

# Perform bulk operations on Azure Cosmos DB data

This tutorial shows how to perform bulk operations in the [Azure Cosmos DB Java V4 SDK](sdk-java-v4.md). This version of the SDK includes the bulk executor library. If you're using an older version of the Java SDK, migrate to [the latest version](migrate-java-v4-sdk.md). The Azure Cosmos DB Java V4 SDK is the current recommended solution for Java bulk support.

Currently, the bulk executor library is supported only by Azure Cosmos DB for NoSQL and API for Gremlin accounts. To learn about using the bulk executor .NET library with API for Gremlin, see [perform bulk operations in Azure Cosmos DB for Gremlin](gremlin/bulk-executor-dotnet.md).


## Prerequisites

* If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn) before you begin. Or, you can use the [Azure Cosmos DB Emulator](emulator.md) with  the `https://localhost:8081` endpoint. The Primary Key is provided in [Authenticating requests](emulator.md).  

* [Java Development Kit (JDK) 1.8+](/java/azure/jdk/)  
  - On Ubuntu, run `apt-get install default-jdk` to install the JDK.  

  - Be sure to set the JAVA_HOME environment variable to point to the folder where the JDK is installed.

* [Download](https://maven.apache.org/download.cgi) and [install](https://maven.apache.org/install.html) a [Maven](https://maven.apache.org/) binary archive  
  
  - On Ubuntu, you can run `apt-get install maven` to install Maven.

* Create an Azure Cosmos DB for NoSQL account by using the steps described in the [create database account](quickstart-java.md) section of the Java quickstart article.

## Clone the sample application

Download a sample repository for the Java V4 SDK from GitHub. These sample applications perform CRUD operations and other common operations on Azure Cosmos DB. To clone the repository, open a command prompt, go to the directory where you want to copy the application, and run the following command:

```bash
 git clone https://github.com/Azure-Samples/azure-cosmos-java-sql-api-samples 
```

The cloned repository contains a sample `SampleBulkQuickStartAsync.java` in the `azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async` folder. The application generates documents and executes operations to bulk create, upsert, replace, and delete items in Azure Cosmos DB. The following sections review the code in the sample app.

## Bulk execution in Azure Cosmos DB

1. The Azure Cosmos DB's connection strings are read as arguments and assigned to variables defined in /`examples/common/AccountSettings.java` file. These environment variables must be set

```
ACCOUNT_HOST=your account hostname;ACCOUNT_KEY=your account primary key
```

To run the bulk sample, specify its Main Class: 

```
com.azure.cosmos.examples.bulk.async.SampleBulkQuickStartAsync
```

2. The `CosmosAsyncClient` object is initialized by using the following statements:  

  [!code-java[](~/../azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async/SampleBulkQuickStartAsync.java?name=CreateAsyncClient)]


3. The sample creates an async database and container. It then creates multiple documents on which bulk operations will be executed. It adds these documents to a `Flux<Family>` reactive stream object:

  [!code-java[](~/../azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async/SampleBulkQuickStartAsync.java?name=AddDocsToStream)]


4. The sample contains methods for bulk create, upsert, replace, and delete. In each method we map the families documents in the BulkWriter `Flux<Family>` stream to multiple method calls in `CosmosBulkOperations`. These operations are added to another reactive stream object `Flux<CosmosItemOperation>`. The stream is then passed to the `executeBulkOperations` method of the async `container` we created at the beginning, and operations are executed in bulk. See bulk create method below as an example:

  [!code-java[](~/../azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async/SampleBulkQuickStartAsync.java?name=BulkCreateItems)]


5. The `BulkWriter.java` class in the same directory as the sample application demonstrates how to handle rate limiting (429) and timeout (408) errors that occur during bulk execution and how to retry those operations. The following methods also show how to implement local and global throughput control.

  [!code-java[](~/../azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async/SampleBulkQuickStartAsync.java?name=BulkWriterAbstraction)]


6. The sample also includes bulk create methods that illustrate how to add response processing and set execution options:

  [!code-java[](~/../azure-cosmos-java-sql-api-samples/src/main/java/com/azure/cosmos/examples/bulk/async/SampleBulkQuickStartAsync.java?name=BulkCreateItemsWithResponseProcessingAndExecutionOptions)]

## Large-scale ingestion strategy

For large-scale data ingestion, **throughput control and retry policy are the primary levers** for avoiding throttling — not batch size or concurrency alone. Focusing on batch size and concurrency without addressing throughput limits and retries won't reliably prevent 429 (rate limited) responses at scale.

### Choose an ingestion approach

| Scenario | Recommended approach |
| --- | --- |
| Large-scale distributed ingestion across multiple machines | [Apache Spark connector](tutorial-spark-connector.md) |
| Single-machine ingestion requiring fine-grained control | Java SDK bulk executor with throughput control |

For large-scale ingestion, the [Apache Spark connector](tutorial-spark-connector.md) is the preferred choice. It handles distributed computation, automatic retry and backoff, and load balancing across worker nodes without requiring manual tuning of concurrency or batch sizes.

### Java SDK bulk executor with throughput control

If your scenario requires the Java SDK directly, the [azure-cosmos-distributed-bulk-sample](https://github.com/Azure/azure-cosmos-distributed-bulk-sample) provides a reference implementation for production-scale ingestion. It demonstrates the following key settings:

- **Auto-tuned micro-batch sizes**: The sample dynamically adjusts batch sizes from 1 to 100 documents per physical partition to saturate throughput while keeping throttling manageable.
- **Configurable retry count**: Default is 20 retries per batch. Adjust based on your tolerance for transient failures and downstream latency requirements.
- **Concurrent batches per machine**: Default is 8 concurrent batches. The recommended range is 25–100% of available CPU cores on the ingestion machine.

> [!TIP]
> Start with the defaults and monitor 429 (rate limited) response rates. Reduce concurrent batches or add throughput control if excessive throttling occurs.

### Throughput control for shared containers

If multiple workloads share the same container, use [throughput control groups](throughput-control-java.md) to cap the RU/s consumed by bulk ingestion and prevent it from starving other workloads:

> [!NOTE]
> Throughput control requires a supported minimum Azure Cosmos DB Java SDK v4 version. The throughput control APIs are also annotated with `@Beta` and are subject to change. Verify the current version requirements and API status in the [throughput control documentation](throughput-control-java.md) before using the following sample.

```java
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("bulkIngestionGroup")
        .targetThroughputThreshold(0.75) // limit ingestion to 75% of provisioned throughput
        .defaultControlGroup(true)
        .build();

container.enableLocalThroughputControlGroup(groupConfig);
```

To coordinate throughput limits across multiple ingestion machines, use [global throughput control](throughput-control-java.md#global-throughput-control) instead.

### Reference implementations

| Sample | Description |
| --- | --- |
| [azure-cosmos-distributed-bulk-sample](https://github.com/Azure/azure-cosmos-distributed-bulk-sample) | End-to-end distributed ingestion with job tracking, restartable batches, auto-tuned micro-batch sizes, and configurable retry and concurrency settings. |
| [ThroughputControlQuickstartAsync.java](https://github.com/Azure-Samples/azure-cosmos-java-sql-api-samples/blob/main/src/main/java/com/azure/cosmos/examples/throughputcontrol/async/ThroughputControlQuickstartAsync.java) | Local throughput control, global throughput control with a shared RU limit via a metadata container, and priority-based throttling. |

## Performance tips

Consider the following points for better performance when using the bulk executor library:

* For best performance, run your application from an Azure VM in the same region as your Azure Cosmos DB account write region.
* To achieve higher throughput:

   * Set the JVM heap size large enough to avoid memory issues when handling large numbers of documents. Suggested heap size: `max(3 GB, 3 * sizeof(all documents passed to bulk import API in one batch))`.
   * Bulk operations have a preprocessing phase, so you get higher throughput when processing large document sets. For example, importing 10,000,000 documents by running bulk import 10 times with 1,000,000 documents each is more efficient than running it 100 times with 100,000 documents each.

* Instantiate a single `CosmosAsyncClient` object for the entire application within a single virtual machine that corresponds to a specific Azure Cosmos DB container.

* A single bulk operation API execution consumes a large chunk of the client machine's CPU and network I/O by spawning multiple tasks internally. Avoid spawning multiple concurrent tasks within your application process, where each task executes bulk operation API calls. If a single bulk operation API call running on a single virtual machine can't consume your entire container's throughput (if your container's throughput > 1 million RU/s), create separate virtual machines to execute bulk operation API calls concurrently.

## Related content

- [Bulk executor overview](bulk-executor-overview.md)
- [Throughput control groups in Azure Cosmos DB Java SDK v4](throughput-control-java.md)
- [Tutorial: Connect to Azure Cosmos DB for NoSQL by using Spark](tutorial-spark-connector.md)
- [Performance tips for Azure Cosmos DB Java SDK v4](performance-tips-java-sdk-v4.md)
- [Best practices for Azure Cosmos DB Java SDK](best-practice-java.md)
