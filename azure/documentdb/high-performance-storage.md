---
title: Premium SSD v2 Disks - High Performance Storage
description: Learn how to use Premium SSD v2 (high performance) storage in Azure DocumentDB for higher IOPS and bandwidth.
author: suvishodcitus
ms.author: suvishod
ms.topic: feature-guide
ms.date: 05/14/2026
ms.custom:
  - references_regions
zone_pivot_groups: azure-interface-portal-rest-bicep-terraform
ai-usage: ai-assisted
---

# Premium SSD v2 (High performance) storage in Azure DocumentDB

Azure DocumentDB uses **Premium SSD v2** disks to deliver higher performance for I/O-intensive workloads by de-coupling storage capacity from IOPS and bandwidth settings.

With Premium SSD v2 storage on Azure DocumentDB, the maximum configurable IOPS and bandwidth settings are available by default regardless of the storage capacity configured for the cluster. The IOPS and bandwidth capacity of the Compute tier determines the achievable IOPS and bandwidth in the storage layer without the need to scale up storage capacity. 

Only the required storage capacity needs to be selected, while the highest achievable IOPS and bandwidth are auto configured by Azure DocumentDB at no added cost. No extra user intervention is needed to ensure the cluster is set up for optimal performance. The result is a **12x performance boost at no added cost**.

Previously, a jump from 5,000 IOPS to 20,000 IOPS required increasing the size of the disk from 1TB to 20TB, even in the absence of higher storage needs. With Premium SSD v2, 20,000 IOPS can be achieved on the same 1TB disk so long as the cluster's compute tier has the capacity to push and maintain 20,000 IOPS. Moreover, Premium SSD v2 disks can support up to 80,000 IOPS - a 4x increase over Premium SSD.

## Throughput benefits of using Premium SSD v2 disks on Azure DocumentDB (80,000 IOPS)
Consider an application with 2 TB of storage and a periodic increase in traffic volumes detailed below:
- 7,000 IOPS in Month 1
- 10,000 IOPS in Month 2
- 19,000 IOPS in Month 3

Before the introduction of Premium SSD v2 disks, IOPS were directly related to storage capacity. 

Thus, the following increases in disk sizes would have been required to meet the application's growing IOPS requirements:
- 2 TB disk (with a capacity of 7,500 IOPS on SSD v1) to achieve 7,000 IOPS in Month 1
- 8 TB disk (with a capacity of 16,000 IOPS SSD v1) to achieve 10,000 IOPS in Month 2
- 32 TB disk (with a capacity of 20,000 IOPS SSD v1) to achieve 19,000 IOPS in Month 3

The older generation of storage disks had a 20,000 IOPS limit and needed 32 TB of storage capacity to achieve that limit. 

More than 20,000 IOPS could only be achieved by adding more shards to the cluster and scaling out the application's traffic. This would have increased management overhead by introducing logical sharding to the application's architecture to distribute database traffic across a multi shard cluster. 

As the application's requirements grow, larger disks and logical sharding complexities grow, despite needing only 2 TB of storage.

However, with Premium SSD v2 disks, storage capacity is independent of IOPS and bandwidth with 80,000 IOPS available by default for a disk of any size at **no added cost**.

Considering the same application above, its growth can now be sustained on the same 2 TB disk with Premium SSD v2 disks:
- 2 TB disk (with a default capacity of 80,000 IOPS on SSD v2) for 7,000 IOPS in Month 1
- The same 2 TB disk for 10,000 IOPS (with a default capacity of 80,000 IOPS on SSD v2) in Month 2
- Still the same 2 TB disk for 19,000 IOPS (with a default capacity of 80,000 IOPS on SSD v2) in Month 3

What previously needed 32 TB disks, is now achievable on a smaller, 2 TB disk - a 16x cost reduction.

The same 2 TB disk can achieve up to 80,000 IOPS while previously being capped at 7,500 IOPS - a nearly 11x performance boost.

## Bandwidth benefits of using Premium SSD v2 disks on Azure DocumentDB (1200 MB/s)
In addition to storage capacity being decoupled from IOPS on SSD v2 disks, bandwidth (MB/s) limits are also independent of the size of the disk.

Previously, higher bandwidth required scaling up the storage capacity of the disk even in the absence of higher storage requirements.

Consider the same workload that requires 2 TB of storage capacity with an addition need for 900 MB/s of bandwidth:
- Prior to SSD v2, a 2 TB disk had a capacity of 150 MB/s
- 900 MB/s of bandwidth required scaling up the 2 TB disk to 32 TB
- Scaling the disk from 2 TB to 32 TB resulted in a 16x increase in storage costs

With Premium SSD v2 (high performance) disks on Azure DocumentDB:
- The maximum bandwidth of 1200 MB/s is available **at no added cost** regardless of the provisioned storage capacity
- Thus, the same 2 TB disk has 1,200 MB/s of available bandwidth
- 8x performance increase with the same storage capacity

Consider another workload that aims to achieve 1 million writes per second on a single sharded Azure DocumentDB cluster. With Premium SSD v2 (high performance) disks:
- 1200 MB/s is available regardless of the provisioned storage capacity of the disk
- For 1 kB documents, 1 million writes per second will require 1000 MB/s
- With Premium SSD v2 (high performance) disks, 1 million writes can be easily achieved and with room to spare
- Storage capacity does not need to be increased to consume 1,000 MB/s of the available 1,200 MB/s

## Resiliency benefits of using Premium SSD v2 disks on Azure DocumentDB
With a default capacity of 80,000 IOPS and 1,200 MB/s of bandwidth, Premium SSD v2 (high performance) disks provide much more head room for spikes in an application's traffic and higher traffic volumes at steady state.

Moreover, during period spikes, because of the additional head room available, the following resiliency characteristics are improved over the older generation of disks:
- Faster replication between the primary and standby nodes within a replica set
- Faster write latencies because of faster replication between the primary and secondary nodes
- Faster failovers during leader election within a replica set due to faster replication times between the primary and secondary
- Fewer failovers overall due to lower resource contention on the primary node of a replica set

## Guidance

The **maximum performance** for your Azure DocumentDB cluster is now only dependent on the **compute tier** and not the storage size. Start by choosing just the desired storage size needed for the cluster, then select a compute tier that provides the required (IOPS) and throughput (MB/s) for your workload. Tabulated below are the highest achievable and sustainable IOPS and bandwidth limits per compute tier.

## IOPS and throughput caps

The achievable IOPS and bandwidth are only dependent on the compute cluster tier and **not dependent on provisioned storage capacity** when using Premium SSD v2 disks. 

The upper bound IOPS and bandwidth achievable by each compute tier are tabulated below.

| Compute Tier | Max IOPS | Max bandwidth (MB/s) |
|--------------|-------------- |--------------------|
| M30 (2 core) | 3,750 | 85 |
| M40 (4 core) | 6,400 | 145 |
| M50 (8 core) | 12,800 | 290 |
| M60 (16 core) | 25,600 | 600 |
| M80 (32 core) | 51,200 | 865 |
| M200 (64 core) | 80,000 | 1,200 |

Tabulated below is a comparison of the achievable IOPS per storage tier on Azure DocumentDB. The values shown assume a compute cluster tier that is capable of pushing 80,000 IOPS is provisioned (e.g., M200)

| Storage Capacity | Max IOPS before Premium SSD v2 | Max IOPS with Premium SSD v2 | 
|--------------|-------------- |--------------------|
| 32 GB  | 120 | 80,000 |
| 64 GB | 240 | 80,000 | 
| 128 GB | 500 | 80,000 | 
| 256 GB | 1,100 | 80,000 |
| 512 GB | 2,300 | 80,000 |
| 1 TB | 5,000 | 80,000 |
| 2 TB  | 7,500 | 80,000 |
| 4 TB | 7,500 | 80,000 |
| 8 TB | 16,000 | 80,000 |
| 16 TB | 18,000 | 80,000 |
| 32 TB | 20,300 | 80,000 |

Similarly, tabulated below is a comparison of the achievable bandwidth (MB/s) per storage tier on Azure DocumentDB. The values shown assume a compute cluster tier that is capable of pushing 1,200 MB/s is provisioned (e.g., M200)

| Storage Capacity | Max MB/s before Premium SSD v2 | Max MB/s with Premium SSD v2 | 
|--------------|-------------- |--------------------|
| 32 GB  | 25 MB/s | 1,200 MB/s |
| 64 GB | 50 MB/s | 1,200 MB/s | 
| 128 GB | 100 MB/s | 1,200 MB/s | 
| 256 GB | 125 MB/s | 1,200 MB/s |
| 512 GB | 150 MB/s | 1,200 MB/s |
| 1 TB | 200 MB/s | 1,200 MB/s |
| 2 TB  | 250 MB/s | 1,200 MB/s |
| 4 TB | 250 MB/s | 1,200 MB/s |
| 8 TB | 500 MB/s | 1,200 MB/s |
| 16 TB | 750 MB/s | 1,200 MB/s |
| 32 TB | 900 MB/s | 1,200 MB/s |

## Prerequisites

[!INCLUDE[Prerequisite - Existing cluster](includes/prerequisite-existing-cluster.md)]

::: zone pivot="rest-api,azure-resource-manager-bicep"

[!INCLUDE[External - Azure CLI prerequisites](~/reusable-content/azure-cli/azure-cli-prepare-your-environment-no-header.md)]

::: zone-end

::: zone pivot="azure-portal"

::: zone-end

::: zone pivot="azure-terraform"

- [Terraform 1.2.0](https://developer.hashicorp.com/terraform/tutorials/azure-get-started/install-cli) or later.

::: zone-end

## Create a cluster with Premium SSD v2 (high performance) storage

Configure a cluster using **Premium SSD v2** (high performance) storage as part of the cluster creation step.

::: zone pivot="azure-portal"

1. Sign in to the **Azure portal** (<https://portal.azure.com>).

1. From the Azure portal menu or the **Home page**, select **Create a resource**.

1. On the **New** page, search for and select **Azure DocumentDB**.

    :::image type="content" source="media/high-performance-storage/select-azure-documentdb.png" alt-text="Screenshot of the Azure portal search feature to locate Azure DocumentDB.":::

1. On the **Create Azure DocumentDB cluster** page and within the **Basics** section, select the **Configure** option within the **Cluster tier** section.

    :::image type="content" source="media/high-performance-storage/select-configure-option.png" alt-text="Screenshot of the options available to configure an Azure DocumentDB cluster.":::

1. On the **Configure** page, choose the cluster tier and storage size as required. Select the storage type as **Premium SSD v2** to enable high-performance storage, then select Save to apply the changes.

    :::image type="content" source="media/high-performance-storage/enable-premium-storage.png" alt-text="Screenshot of the configuration option specific to premium SSD v2 disks in Azure DocumentDB.":::

1. Fill in the remaining details and then select **Review + create**.

1. Review the settings you provide, and then select **Create**. It takes a few minutes to create the cluster. Wait for the resource deployment is complete.

1. Finally, select **Go to resource** to navigate to the Azure DocumentDB cluster in the portal.

:::image type="content" source="media/high-performance-storage/go-to-resource.png" alt-text="Screenshot of the deployment completion step with an option to navigate to the new Azure DocumentDB cluster.":::

::: zone-end

::: zone pivot="azure-resource-manager-bicep"

1. Open a new terminal.

1. Sign in to Azure CLI.

1. Create a new Bicep file to define your role definition. Name the file *main.bicep*.

1. Add this template to the file's content. Replace the `<cluster-name>`, `<location>`, `<username>`, and `<password>` placeholders with appropriate values.

    ```bicep
    resource cluster 'Microsoft.DocumentDB/mongoClusters@2025-09-01' = {
      name: '<cluster-name>'
      location: '<location>'
      properties: {
        administrator: {
          userName: '<username>'
          password: '<password>'
        }
        serverVersion: '8.0'
        storage: {
          sizeGb: 32
          type: 'PremiumSSDv2'
        }
        compute: {
          tier: 'M30'
        }
        sharding: {
          shardCount: 1
        }
        highAvailability: {
          targetMode: 'Disabled'
        }
      }
    }
    ```

1. Deploy the Bicep template using [`az deployment group create`](/cli/azure/deployment/group#az-deployment-group-create). Specify the name of the Bicep template and replace the `<resource-group>` placeholder with the name of your target Azure resource group.

    ```azurecli-interactive
    az deployment group create \
        --resource-group "<resource-group>" \
        --template-file main.bicep
    ```

1. Wait for the deployment to complete. Review the output from the deployment.

::: zone-end

::: zone pivot="azure-terraform"

1. Open a new terminal.

1. Sign in to Azure CLI.

1. Check your target Azure subscription.

    ```azurecli-interactive
    az account show
    ```

1. Define your cluster in a new Terraform file. Name the file *cluster.`tf`*.

1. Add this resource configuration to the file's content. Replace the `<cluster-name>`, `<resource-group>`, and `<location>` placeholders with appropriate values.

    ```terraform
    variable "admin_username" {
      type        = string
      description = "Administrator username for the cluster."
      sensitive   = true
    }
    
    variable "admin_password" {
      type        = string
      description = "Administrator password for the cluster."
      sensitive   = true
    }
    
    terraform {
      required_providers {
        azurerm = {
          source  = "hashicorp/azurerm"
          version = "~> 4.0"
        }
      }
    }
    
    provider "azurerm" {
      features {}
    }
    
    data "azurerm_resource_group" "existing" {
      name = "<resource-group>"
    }
    
    resource "azurerm_mongo_cluster" "cluster" {
      name                   = "<cluster-name>"
      resource_group_name    = data.azurerm_resource_group.existing.name
      location               = "<location>"
      administrator_username = var.admin_username
      administrator_password = var.admin_password
      shard_count            = "1"
      compute_tier           = "M30"
      high_availability_mode = "Disabled"
      storage_size_in_gb     = "32"
      storage_type           = "PremiumSSDv2"
      version                = "8.0"
    }
    ```

    > [!TIP]
    > For more information on options using the `azurerm_mongo_cluster` resource, see [`azurerm` provider documentation in Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mongo_cluster#arguments-reference).

1. Initialize the Terraform deployment.

    ```azurecli-interactive
    terraform init --upgrade
    ```

1. Create an execution plan and save it to a file named *cluster.tfplan*. Provide values when prompted for the `admin_username` and `admin_password` variables.

    ```azurecli-interactive
    ARM_SUBSCRIPTION_ID=$(az account show --query id --output tsv) terraform plan --out "cluster.tfplan"
    ```

    > [!NOTE]
    > This command sets the `ARM_SUBSCRIPTION_ID` environment variable temporarily. This setting is required for the `azurerm` provider starting with version 4.0 For more information, see [subscription ID in `azurerm`](https://registry.terraform.io/providers/hashicorp/azurerm/4.0.0/docs/guides/4.0-upgrade-guide#specifying-subscription-id-is-now-mandatory).

1. Apply the execution plan to deploy the cluster to Azure.

    ```azurecli-interactive
    ARM_SUBSCRIPTION_ID=$(az account show --query id --output tsv) terraform apply "cluster.tfplan"
    ```

1. Wait for the deployment to complete. Review the output from the deployment.

::: zone-end

::: zone pivot="rest-api"

1. Open a new terminal.

1. Sign in to Azure CLI.

1. Create a new JSON file named *cluster.json*.

1. Add this document to the file's content. Replace the `<location>`, `<username>`, and `<password>` placeholders with appropriate values.

    ```json
    {
      "location": "<location>",
      "properties": {
        "administrator": {
          "userName": "<username>",
          "password": "<password>"
        },
        "serverVersion": "8.0",
        "storage": {
          "sizeGb": 32,
          "type": "PremiumSSDv2"
        },
        "compute": {
          "tier": "M30"
        },
        "sharding": {
          "shardCount": 1
        },
        "highAvailability": {
          "targetMode": "Disabled"
        }
      }
    }
    ```

1. Use the `az rest` Azure CLI command to create a new cluster with the configuration specified in the JSON file. Specify the name of the JSON file as the `body` of the request and replace the following placeholders:

    | | Description |
    | --- | --- |
    | **`<subscription-id>`** | The unique identifier of your target Azure subscription |
    | **`<resource-group>`** | The name of your target Azure resource group |
    | **`<cluster-name>`** | The unique name of your new Azure DocumentDB cluster |

    ```azurecli-interactive
    az rest \
        --method "GET" \
        --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.DocumentDB/mongoClusters/<cluster-name>/users?api-version=2025-09-01" \
        --body @cluster.json
    ```

    > [!TIP]
    > Use `az account show` to get the unique identifier of your target Azure subscription. 

1. Wait for the deployment to complete. Review the output from the deployment.

::: zone-end

## Current limitations of Premium SSD v2 storage (high performance)

- Customer-managed keys (CMK) aren't supported with Premium SSD v2 storage.

- Storage capacity settings on Premium SSD v2 disks can be adjusted up to four times within a 24-hour period. For newly created clusters, a maximum of three storage capacity adjustments can be made during the first 24 hours. 
  
- Replication from Premium SSD to Premium SSD v2 is supported only for migration scenarios. Ongoing replication isn't supported because Premium SSD can't match the performance of Premium SSD v2 and may result in higher latency.

- Online migration from Premium SSD to Premium SSD v2 isn't currently supported. To upgrade from Premium SSD to Premium SSD V2, you can perform a point-in-time-restore to a new server using Premium SSD v2. Alternatively, you can create a read replica from a Premium SSD server to a Premium SSD v2 server and promote it after replication completes.

- If you perform any operation that requires disk hydration the following error might occur. This error occurs because Premium SSD v2 disks don't support any operation while the disk is still hydrating.
  - Error message: Unable to complete the operation because the disk is still being hydrated. Retry after some time.
  - Operations that can trigger this behavior include:
      - Performing compute scaling, storage scaling, enabling high availability (HA) in quick succession.
      - This also includes service-triggered failovers to guarantee high availability.
      - Using PITR (point-in-time-restore) to create a new cluster and immediately enabling High Availability while the disk is still being hydrated.
  - As a best practice, when using Premium SSD v2 disks, space out these operations or complete them sequentially, ensuring disk hydration finishes between actions.

## Related content

- [Compare Azure DocumentDB to MongoDB](compare-mongodb-atlas.md)
- [Review supported Mongo API surface area in Azure DocumentDB](compatibility-query-language.md)

