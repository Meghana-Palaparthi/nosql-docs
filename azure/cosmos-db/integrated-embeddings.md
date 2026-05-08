---
title: Integrated Embeddings in Azure Cosmos DB for NoSQL
description: Automatically generate and maintain vector embeddings for your data in Azure Cosmos DB for NoSQL.
author: abhirockzz
ms.author: guabhishek
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.custom:
  - build-2026
ms.topic: concept-article
ms.date: 06/02/2026
ms.update-cycle: 180-days
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
---

# Integrated Embeddings in Azure Cosmos DB for NoSQL

## What are integrated embeddings?

Integrated Embeddings automatically generates and maintains vector embeddings for your data in Azure Cosmos DB. You specify the source properties to embed, the Microsoft Foundry embedding model to use, and the path where the generated embeddings are stored. Azure Cosmos DB detects data changes and generates embeddings asynchronously, writing them back to your items.

Without integrated embedding generation, you typically need to build and operate separate data pipelines that track data changes, call an embedding model for each change, handle errors and retries, and write the generated embeddings back to Azure Cosmos DB. After you configure Integrated Embeddings, Azure Cosmos DB handles this work for you and keeps embeddings up to date as your data changes. You can focus on building AI applications instead of managing embedding pipelines.

> [!IMPORTANT]
> Integrated Embeddings currently supports the following Azure OpenAI embedding models: `text-embedding-3-large`, `text-embedding-3-small`, and `text-embedding-ada-002`.


## Prerequisites

Before you use Integrated Embeddings, you need the following resources and configuration:

- An existing Azure Cosmos DB for NoSQL account with [vector search enabled](vector-search.md#enable-the-vector-indexing-and-search-feature).
- A Microsoft Foundry resource with a deployed [Azure OpenAI embedding model](/azure/foundry-classic/foundry-models/concepts/models-sold-directly-by-azure?tabs=americas%2Caz-global-standard%2Cglobal-standard&pivots=azure-openai#embeddings).
- A [managed identity](how-to-setup-managed-identity.md) on the Azure Cosmos DB account. Azure Cosmos DB uses this identity to authenticate to the Microsoft Foundry resource on your behalf.
- A [role assignment](/azure/foundry-classic/openai/how-to/role-based-access-control#add-role-assignment-to-an-azure-openai-resource) on the Microsoft Foundry resource that grants the Azure Cosmos DB managed identity the [Cognitive Services OpenAI User](/azure/foundry-classic/openai/how-to/role-based-access-control#azure-openai-roles) role, so it can make inference API calls to the embedding model.

## Enable integrated embeddings

Coming soon

## Policy for integrated embeddings

Integrated Embeddings is configured as part of the container vector embedding policy. The existing vector embedding policy defines the vector path, data type, dimensions, and distance function. To have Azure Cosmos DB generate embeddings for a vector path, add an `embeddingSource` object to the corresponding `vectorEmbeddings` entry.

| Property         | Description                                                                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `sourcePaths`    | The item property paths whose values are used as input for embedding generation.                                                               |
| `deploymentName` | The name you assigned to the embedding model deployment in Microsoft Foundry.                                                                  |
| `modelName`      | The underlying embedding model, for example `text-embedding-3-small`.                                                                          |
| `endpoint`       | The endpoint URL of the Microsoft Foundry resource that hosts the deployment, for example `https://<foundry-resource-name>.openai.azure.com/`. |
| `authType`       | The authentication type used to make inference API calls to the embedding model. `Entra` is currently the only supported value.                |

### Example: single source path

This example configures Azure Cosmos DB to generate an embedding from the `/text` property and store it in `/embedding`.

```json
{
  "vectorEmbeddings": [
    {
      "path": "/embedding",
      "dataType": "float32",
      "dimensions": 1536,
      "distanceFunction": "cosine",
      "embeddingSource": {
        "sourcePaths": [
          "/text"
        ],
        "deploymentName": "text-embedding-3-small",
        "modelName": "text-embedding-3-small",
        "endpoint": "https://<foundry-resource-name>.openai.azure.com/",
        "authType": "Entra"
      }
    }
  ]
}
```

### Example: multiple source paths

Use multiple source paths to combine more than one property into a single embedding. This example configures Azure Cosmos DB to generate an embedding from the `/title` and `/description` properties and store it in `/embedding`.

```json
{
  "vectorEmbeddings": [
    {
      "path": "/embedding",
      "dataType": "float32",
      "dimensions": 1536,
      "distanceFunction": "cosine",
      "embeddingSource": {
        "sourcePaths": [
          "/title",
          "/description"
        ],
        "deploymentName": "text-embedding-3-small",
        "modelName": "text-embedding-3-small",
        "endpoint": "https://<foundry-resource-name>.openai.azure.com/",
        "authType": "Entra"
      }
    }
  ]
}
```

### Example: multiple vector paths

Configure more than one vector path in the embedding policy to generate multiple embeddings, each with its own source properties and embedding model.

This example configures Azure Cosmos DB to generate `/desc_embedding` from the `/description` property using `text-embedding-3-large`, and `/title_embedding` from the `/title` property using `text-embedding-3-small`.

```json
{
  "vectorEmbeddings": [
    {
      "path": "/desc_embedding",
      "dataType": "float32",
      "dimensions": 3072,
      "distanceFunction": "cosine",
      "embeddingSource": {
        "sourcePaths": [
          "/description"
        ],
        "deploymentName": "text-embedding-3-large",
        "modelName": "text-embedding-3-large",
        "endpoint": "https://<foundry-resource-name>.openai.azure.com/",
        "authType": "Entra"
      }
    },
    {
      "path": "/title_embedding",
      "dataType": "float32",
      "dimensions": 1536,
      "distanceFunction": "cosine",
      "embeddingSource": {
        "sourcePaths": [
          "/title"
        ],
        "deploymentName": "text-embedding-3-small",
        "modelName": "text-embedding-3-small",
        "endpoint": "https://<foundry-resource-name>.openai.azure.com/",
        "authType": "Entra"
      }
    }
  ]
}
```


## Getting started with integrated embeddings

This quickstart walks you through creating a container configured for Integrated Embeddings, inserting items, and verifying that Azure Cosmos DB generates and stores the embeddings. It assumes you have completed the [prerequisites](#prerequisites).

Install the [Azure Cosmos DB Python SDK](https://github.com/Azure/azure-sdk-for-python):

```bash
pip install azure-cosmos
```

Set the following environment variables for your Azure Cosmos DB account and Microsoft Foundry embedding model deployment:

```bash
export COSMOS_ENDPOINT="https://<account-name>.documents.azure.com:443/"
export COSMOS_KEY="<cosmos-account-key>"
export COSMOS_DATABASE="integrated-embeddings-db"
export COSMOS_CONTAINER="integrated-embeddings-items"
export FOUNDRY_ENDPOINT="https://<foundry-resource-name>.openai.azure.com/"
export FOUNDRY_DEPLOYMENT_NAME="text-embedding-3-small"
export FOUNDRY_MODEL_NAME="text-embedding-3-small"
```

Save the following script as `integrated_embeddings_quickstart.py`. The script creates a database and a new container, configures the vector embedding policy with an `embeddingSource`, inserts sample items with a `description` property, and polls them until Azure Cosmos DB adds the generated embeddings to `/embedding`.

The script sets `dimensions` to `1536`, which matches `text-embedding-3-small` and `text-embedding-ada-002`. Use `3072` for `text-embedding-3-large`.

> [!NOTE]
> This example uses a `quantizedFlat` vector index. To learn about other supported vector index types, see [Vector Indexing Policies](vector-search.md#vector-indexing-policies).

```python
import os
import time

from azure.cosmos import CosmosClient, PartitionKey, exceptions


COSMOS_ENDPOINT = os.environ["COSMOS_ENDPOINT"]
COSMOS_KEY = os.environ["COSMOS_KEY"]
DATABASE_NAME = os.environ.get("COSMOS_DATABASE", "integrated-embeddings-db")
CONTAINER_NAME = os.environ.get("COSMOS_CONTAINER", "integrated-embeddings-items")
FOUNDRY_ENDPOINT = os.environ["FOUNDRY_ENDPOINT"]
FOUNDRY_DEPLOYMENT_NAME = os.environ["FOUNDRY_DEPLOYMENT_NAME"]
FOUNDRY_MODEL_NAME = os.environ["FOUNDRY_MODEL_NAME"]

EMBEDDING_PATH = "embedding"
POLL_INTERVAL_SECONDS = 5
POLL_TIMEOUT_SECONDS = 120


vector_embedding_policy = {
  "vectorEmbeddings": [
    {
      "path": f"/{EMBEDDING_PATH}",
      "dataType": "float32",
      "dimensions": 1536,
      "distanceFunction": "cosine",
      "embeddingSource": {
        "sourcePaths": [
          "/description"
        ],
        "deploymentName": FOUNDRY_DEPLOYMENT_NAME,
        "modelName": FOUNDRY_MODEL_NAME,
        "endpoint": FOUNDRY_ENDPOINT,
        "authType": "Entra"
      }
    }
  ]
}

indexing_policy = {
  "indexingMode": "consistent",
  "automatic": True,
  "includedPaths": [
    {"path": "/*"}
  ],
  "excludedPaths": [
    {"path": "/\"_etag\"/?"},
    {"path": f"/{EMBEDDING_PATH}/*"}
  ],
  "vectorIndexes": [
    {
      "path": f"/{EMBEDDING_PATH}",
      "type": "quantizedFlat"
    }
  ]
}

sample_items = [
  {
    "id": "item-1",
    "description": "Azure Cosmos DB for NoSQL supports vector search for AI applications."
  },
  {
    "id": "item-2",
    "description": "Cosmos DB offers global distribution with multi-region writes and tunable consistency levels."
  },
  {
    "id": "item-3",
    "description": "Use the change feed to react to data changes in real time without polling."
  }
]


def main():
  client = CosmosClient(COSMOS_ENDPOINT, credential=COSMOS_KEY)

  database = client.create_database_if_not_exists(id=DATABASE_NAME)
  try:
    container = database.create_container(
      id=CONTAINER_NAME,
      partition_key=PartitionKey(path="/id"),
      vector_embedding_policy=vector_embedding_policy,
      indexing_policy=indexing_policy,
    )
  except exceptions.CosmosResourceExistsError as err:
    raise RuntimeError(
      f"Container '{CONTAINER_NAME}' already exists. Use a new container name for this quickstart."
    ) from err

  for item in sample_items:
    container.upsert_item(item)
    print(f"Inserted item: {item['id']}")

  pending = {item["id"] for item in sample_items}
  deadline = time.time() + POLL_TIMEOUT_SECONDS
  while pending and time.time() < deadline:
    for item_id in list(pending):
      item = container.read_item(item=item_id, partition_key=item_id)
      embedding = item.get(EMBEDDING_PATH)
      if embedding:
        print(
          f"Generated embedding for {item_id} "
          f"(dimensions: {len(embedding)}, preview: {embedding[:3]}...)"
        )
        pending.remove(item_id)

    if pending:
      print(f"Waiting for embeddings: {sorted(pending)}")
      time.sleep(POLL_INTERVAL_SECONDS)

  if pending:
    raise TimeoutError(f"Embeddings were not generated for: {sorted(pending)}")


if __name__ == "__main__":
  main()
```

Run the script:

```bash
python integrated_embeddings_quickstart.py
```

The output should look similar to this example:

```text
Inserted item: item-1
Inserted item: item-2
Inserted item: item-3
Waiting for embeddings: ['item-1', 'item-2', 'item-3']
Generated embedding for item-1 (dimensions: 1536, preview: [0.0123, -0.0456, 0.0789]...)
Generated embedding for item-2 (dimensions: 1536, preview: [-0.0231, 0.0567, 0.0103]...)
Generated embedding for item-3 (dimensions: 1536, preview: [0.0456, -0.0210, 0.0398]...)
```

## Troubleshoot common issues

The following table lists scenarios you might encounter when using Integrated Embeddings, along with possible causes and how to resolve them.

| What you see                                                 | Possible cause                                                                                                                                      | What to check                                                                                                                                                                                   |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The embedding property is missing from new or updated items. | Azure Cosmos DB might not have processed the item yet, the item might not match the embedding policy, or embedding generation might have failed.    | Confirm that Integrated Embeddings is enabled, the container vector embedding policy includes `embeddingSource`, and the item contains the properties listed in `sourcePaths`.                  |
| Embedding generation takes longer than expected.             | Azure Cosmos DB might be processing a backlog of item changes, or the Microsoft Foundry embedding model deployment might be hitting its rate limit. | Review write volume, Azure Cosmos DB throughput, item size, and the quota on the Microsoft Foundry embedding model deployment.                                                                  |
| Embeddings are generated for some items but not others.      | Some items might be missing the configured source properties, or one or more of those properties might be empty or null.                            | Compare an item that received an embedding with one that didn't. Confirm that the missing item contains every property listed in `sourcePaths` and that those properties have non-empty values. |

## Pricing

Integrated Embeddings is available at no additional cost. You pay only for the underlying services it uses:

- **Microsoft Foundry**: Embedding model inference is billed to your [Microsoft Foundry resource](https://azure.microsoft.com/pricing/details/ai-foundry-models/aoai/#pricing).
- **Azure Cosmos DB**: Request units are consumed when Azure Cosmos DB reads the change feed to detect item changes and writes generated embeddings back to your items.

## Limitations

Integrated Embeddings has the following limitations:

- **Supported models**: Integrated Embeddings supports Azure OpenAI embedding models via Microsoft Foundry. The initial release supports `text-embedding-ada-002`, `text-embedding-3-small`, and `text-embedding-3-large`.
- **Source input size**: The values from all `sourcePaths` are concatenated and sent to the embedding model as a single input. The combined input is capped at 8,192 tokens; longer inputs are truncated before embedding.

## Related content

- [VectorDistance system function](/cosmos-db/query/vectordistance)
- [Vector index policies](index-policy.md#vector-indexes)
- [Vector indexing policy examples](how-to-manage-indexing-policy.md#vector-indexing-policy-examples)