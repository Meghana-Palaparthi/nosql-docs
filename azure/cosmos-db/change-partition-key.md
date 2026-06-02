---
title: Change partition key
description: Change partition key in Azure Cosmos DB for NOSQL API.
author: richagaur
ms.author: richagaur
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 12/03/2024
appliesto:
  - ✅ NoSQL
---

# Changing the partition key in Azure Cosmos DB

In the realm of database management, it isn't uncommon for the initially chosen partition key for a container to become inadequate as applications evolve. It can result in suboptimal performance and increased costs for the container. Several factors contributing to this situation include:

- [Cross partition queries](how-to-query-container.md#avoid-cross-partition-queries)
- [Hot partitions](troubleshoot-request-rate-too-large.md?tabs=resource-specific#how-to-identify-the-hot-partition)

To address these issues, Azure Cosmos DB offers the ability to seamlessly change the partition key using the Azure portal.

## Getting started

To change the partition key of a container in Azure Cosmos DB for the NoSQL API using the Azure portal, follow these steps:

1. Navigate to the **Data Explorer** in the Azure Cosmos DB portal and select the container for which you need to change the partition key.
2. Proceed to the **Scale & Settings** option and choose the **Partition Keys** tab.
3. Select the **Change** button to initiate the partition key change process.

![Screenshot of the Change partition key feature in the Data Explorer in an Azure Cosmos DB account.](media/change-partition-key/cosmosdb-change-partition-key.png)

## How the change partition key works

Changing the partition key entails creating a new destination container or selecting an existing destination container within the same database.

If creating a new container using the Azure portal while changing the partition key, all configurations except for the partition key and unique keys are replicated to the destination container.

![Screenshot of create or select destination container screen while changing partition key in an Azure Cosmos DB account.](media/change-partition-key/cosmosdb-change-partition-key-create-container.png)

You can copy data from the source container to the destination container in online or offline manner utilizing the [container copy](container-copy.md#how-does-container-copy-work) jobs.

>[!Note]
>For online mode, you need to complete the change partition key job by selecting the **Complete** button.

Once the copy is complete, you can start using the new container with desired partition key and optionally delete the old container.


## Limitations
- By default, two server-side compute instances, each with 4 vCPUs and 16 GB of memory, are allocated to handle the data copy job per account. The performance of the copy job relies on various [factors](container-copy.md#factors-that-affect-the-rate-of-a-copy-job). To allocate higher SKU server-side compute instances, please reach out to Microsoft support.
- Partition key modification is supported for containers provisioned with less than 1,000,000 RU/s and containing less than 4 TB of data. For containers with over 1,000,000 provisioned throughput or more than 4 TB of data, please contact Microsoft support for assistance with changing the partition key.
- Changing partition key isn't supported for accounts with following capabilities.
   * [Merge partition](merge.md)
- The feature is currently supported only in the documented [regions](container-copy.md#supported-regions).
  
## Next steps

- Explore more about [container copy jobs](container-copy.md).
