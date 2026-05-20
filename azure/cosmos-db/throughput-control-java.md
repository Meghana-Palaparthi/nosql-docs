---
title: Throughput control groups - Java SDK v4
description: Learn how to use throughput control groups to limit RU consumption in the Azure Cosmos DB Java SDK v4.
author: TheovanKraay
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.custom: devx-track-extended-java
ms.topic: how-to
ms.date: 05/14/2026
ms.author: thvankra
appliesto:
  - ✅ NoSQL
---

# Throughput control groups in Azure Cosmos DB Java SDK v4

> [!IMPORTANT]
> The throughput control APIs in the Java SDK are annotated with `@Beta` and are subject to change. Review the [SDK changelog](https://github.com/Azure/azure-sdk-for-java/blob/main/sdk/cosmos/azure-cosmos/CHANGELOG.md) for updates before upgrading.

Throughput control groups allow you to limit the request unit (RU) consumption of your Azure Cosmos DB operations. This is useful when you have multiple applications or workloads sharing the same container and you want to ensure that one workload doesn't consume all available throughput. Throughput control is available in the Azure Cosmos DB Java SDK v4 starting from version 4.13.0.

The Java SDK supports three modes of throughput control:

- **Local throughput control** - Limits RU consumption within a single client instance.
- **Global throughput control** - Limits RU consumption across multiple client instances by coordinating through a metadata container.
- **Server-side throughput control** - Limits RU consumption using server-side throughput buckets.

## Prerequisites

- Azure Cosmos DB Java SDK v4 >= 4.13.0 (for local and global throughput control)
- Azure Cosmos DB Java SDK v4 >= 4.54.0 (for throughput control with Change Feed Processor)
- Azure Cosmos DB Java SDK v4 >= 4.74.0 (for server-side throughput control with throughput buckets)

## Required imports

The code samples in this article use the following imports:

```java
import com.azure.cosmos.CosmosAsyncClient;
import com.azure.cosmos.CosmosAsyncContainer;
import com.azure.cosmos.CosmosAsyncDatabase;
import com.azure.cosmos.CosmosClientBuilder;
import com.azure.cosmos.GlobalThroughputControlConfig;
import com.azure.cosmos.ThroughputControlGroupConfig;
import com.azure.cosmos.ThroughputControlGroupConfigBuilder;
import com.azure.cosmos.models.CosmosItemRequestOptions;
import com.azure.cosmos.models.PartitionKey;
import com.azure.cosmos.models.ChangeFeedProcessor;
import com.azure.cosmos.models.ChangeFeedProcessorBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.fasterxml.jackson.databind.JsonNode;
import java.time.Duration;
```

## Create a throughput control group

Use the `ThroughputControlGroupConfigBuilder` to create a throughput control group configuration. You can set:

| Property | Description |
| --- | --- |
| `groupName` | A unique name for the control group. |
| `targetThroughput` | An absolute RU/s limit for the group (must be > 0). |
| `targetThroughputThreshold` | A percentage (0, 1] of total provisioned throughput to allocate to this group. |
| `defaultControlGroup` | Whether this is the default group for all requests that aren't assigned to a specific group. |
| `priorityLevel` | The priority level (`PriorityLevel.HIGH` or `PriorityLevel.LOW`) for [priority-based execution](priority-based-execution.md). |
| `throughputBucket` | The server-side throughput bucket ID (>= 0) to assign to this group. Used with server-side throughput control. |
| `continueOnInitError` | If `true`, operations fall back to using the container's full throughput if the control group fails to initialize. |

> [!NOTE]
> You must set at least one of `targetThroughput`, `targetThroughputThreshold`, `priorityLevel`, or `throughputBucket` when building a throughput control group. For local and global throughput control, set `targetThroughput` or `targetThroughputThreshold`. For server-side throughput control, set `priorityLevel` or `throughputBucket` (or both).

## Local throughput control

Local throughput control limits the RU consumption within a single `CosmosAsyncClient` instance. This is useful when you want to isolate different types of operations within the same application.

```java
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("localControlGroup")
        .targetThroughputThreshold(0.1) // limit to 10% of provisioned throughput
        .build();

container.enableLocalThroughputControlGroup(groupConfig);
```

After enabling local throughput control, all operations on that container are limited to the configured RU/s within this client instance.

### Multiple local control groups

You can create multiple local control groups on the same container. For example, to separate read and write workloads:

```java
// Limit reads to 25% of provisioned throughput
ThroughputControlGroupConfig readGroup =
    new ThroughputControlGroupConfigBuilder()
        .groupName("reads")
        .targetThroughputThreshold(0.25)
        .defaultControlGroup(true) // default group for all requests
        .build();

container.enableLocalThroughputControlGroup(readGroup);

// Limit writes to a fixed 500 RU/s
ThroughputControlGroupConfig writeGroup =
    new ThroughputControlGroupConfigBuilder()
        .groupName("writes")
        .targetThroughput(500)
        .build();

container.enableLocalThroughputControlGroup(writeGroup);
```

To assign a specific operation to a non-default group, use `CosmosItemRequestOptions`:

```java
CosmosItemRequestOptions requestOptions = new CosmosItemRequestOptions();
requestOptions.setThroughputControlGroupName("writes");

container.createItem(newItem, new PartitionKey(newItem.getPartitionKey()), requestOptions)
    .block();
```

## Global throughput control

Global throughput control coordinates RU consumption across multiple client instances by using a shared metadata container in Azure Cosmos DB. This is useful when you have multiple microservices or application instances accessing the same container and want to enforce a collective throughput limit.

### Set up the control container

The global throughput control feature requires a metadata container to track RU usage across clients. You can create a dedicated database and container for this purpose, or use an existing container.

```java
// Create a dedicated database and container for throughput control metadata
client.createDatabaseIfNotExists("ThroughputControlDatabase").block();
CosmosAsyncDatabase throughputControlDatabase =
    client.getDatabase("ThroughputControlDatabase");

throughputControlDatabase
    .createContainerIfNotExists("ThroughputControlContainer", "/groupId")
    .block();
```

### Enable global throughput control

```java
// Create the throughput control group config
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("globalControlGroup")
        .targetThroughputThreshold(0.25) // limit to 25% of provisioned throughput
        .defaultControlGroup(true)
        .build();

// Create the global control config pointing to the metadata container
GlobalThroughputControlConfig globalControlConfig =
    client.createGlobalThroughputControlConfigBuilder(
            "ThroughputControlDatabase",
            "ThroughputControlContainer")
        .setControlItemRenewInterval(Duration.ofSeconds(5))
        .setControlItemExpireInterval(Duration.ofSeconds(11))
        .build();

// Enable global throughput control on the target container
container.enableGlobalThroughputControlGroup(groupConfig, globalControlConfig);
```

### Configuration options for global throughput control

| Property | Description | Default |
| --- | --- | --- |
| `controlItemRenewInterval` | How often (minimum 5 seconds) each client updates its usage in the metadata container. | 5 seconds |
| `controlItemExpireInterval` | How quickly offline clients are detected and their share redistributed. Must be > 2 × renewInterval + 1. | 11 seconds |

> [!IMPORTANT]
> The `controlItemExpireInterval` must be greater than 2 × `controlItemRenewInterval` + 1 second. Using shorter intervals makes the system more responsive but increases metadata overhead.

## Server-side throughput control

Server-side throughput control uses [throughput buckets](throughput-buckets.md) to enforce RU limits at the server level. This mode requires Azure Cosmos DB Java SDK v4 version 4.74.0 or later.

```java
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("serverSideControlGroup")
        .throughputBucket(2) // assign to server-side bucket 2
        .build();

container.enableServerThroughputControlGroup(groupConfig);
```

> [!NOTE]
> Server-side throughput control requires either `priorityLevel` or `throughputBucket` (or both) to be set in the throughput control group configuration.

For more information on throughput buckets, see [Throughput buckets in Azure Cosmos DB](throughput-buckets.md).

## Error handling

When a request would exceed the allocated RU budget for its control group, the SDK rejects it locally and returns a `CosmosException` with status `429 (Too Many Requests)`, without sending the request to the service. Applications must continue to handle 429 responses as usual (the SDK's built-in retry policy applies).

To handle initialization errors gracefully, set `continueOnInitError` to `true`:

```java
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("resilientGroup")
        .targetThroughputThreshold(0.5)
        .continueOnInitError(true) // fall back to full throughput on init failure
        .build();
```

## Throughput control with Change Feed Processor

You can apply throughput control to the [Change Feed Processor](change-feed-processor.md) to limit RU consumption during change feed processing. This is particularly useful during backfill scenarios where the Change Feed Processor reads large volumes of data. This integration requires Azure Cosmos DB Java SDK v4 version 4.54.0 or later.

To enable throughput control on the Change Feed Processor, configure a throughput control group on the feed container before building the processor:

```java
// Configure throughput control on the feed container
ThroughputControlGroupConfig groupConfig =
    new ThroughputControlGroupConfigBuilder()
        .groupName("cfpControlGroup")
        .targetThroughputThreshold(0.3) // limit to 30% of provisioned throughput
        .defaultControlGroup(true)
        .build();

feedContainer.enableLocalThroughputControlGroup(groupConfig);

// Build the Change Feed Processor — it inherits the throughput control settings
ChangeFeedProcessor changeFeedProcessor = new ChangeFeedProcessorBuilder()
    .hostName("host-1")
    .feedContainer(feedContainer)
    .leaseContainer(leaseContainer)
    .handleChanges(docs -> {
        for (JsonNode doc : docs) {
            // process each change
        }
    })
    .buildChangeFeedProcessor();
```

You can also use global throughput control with the Change Feed Processor to coordinate RU limits across multiple processor instances.

## Related content

- [Throughput control in the Azure Cosmos DB Spark connector](throughput-control-spark.md)
- [Throughput buckets in Azure Cosmos DB](throughput-buckets.md)
- [Change Feed Processor in Azure Cosmos DB](change-feed-processor.md)
- [Priority-based execution in Azure Cosmos DB](priority-based-execution.md)
- [Request units in Azure Cosmos DB](request-units.md)
- [Best practices for Java SDK v4](best-practice-java.md)
- [Performance tips for Java SDK v4](performance-tips-java-sdk-v4.md)
