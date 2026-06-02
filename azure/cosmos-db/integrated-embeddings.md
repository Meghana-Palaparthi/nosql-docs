---
title: Integrated Embeddings in Azure Cosmos DB
description: Automatically generate and maintain vector embeddings for your data in Azure Cosmos DB.
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
ai-usage: ai-assisted
---

# Integrated Embeddings in Azure Cosmos DB for NoSQL (Preview)

[!INCLUDE[Preview](includes/notice-preview.md)]

> [!IMPORTANT]
> As Integrated Embeddings is gradually rolling out across Azure regions, availability may vary, and the feature might not yet be accessible in your subscription or region.

## What are Integrated Embeddings?

Integrated Embeddings automatically generates and maintains vector embeddings for your data in Azure Cosmos DB. You specify the source properties to embed, the Microsoft Foundry embedding model to use, and the path where the generated embeddings are stored. Azure Cosmos DB detects data changes and generates embeddings asynchronously, writing them back to your items.

Without Integrated Embeddings, you typically need to build and operate separate data pipelines that track data changes, call an embedding model for each change, handle errors and retries, and write the generated embeddings back to Azure Cosmos DB. After you configure Integrated Embeddings, Azure Cosmos DB handles this work for you and keeps embeddings up to date as your data changes. You can focus on building AI applications instead of managing embedding pipelines.

> [!IMPORTANT]
> Integrated Embeddings currently supports the following Azure OpenAI embedding models: `text-embedding-3-large`, `text-embedding-3-small`, and `text-embedding-ada-002`.

## Prerequisites

Before you use Integrated Embeddings, you need the following resources and configuration:

- An existing Azure Cosmos DB for NoSQL account with [vector search enabled](vector-search.md#enable-the-vector-indexing-and-search-feature).
- [All versions and deletes change feed mode](change-feed-modes.md?tabs=all-versions-and-deletes#all-versions-and-deletes-change-feed-mode) enabled on the account.
- A Microsoft Foundry resource with a deployed [Azure OpenAI embedding model](/azure/foundry-classic/foundry-models/concepts/models-sold-directly-by-azure?tabs=americas%2Caz-global-standard%2Cglobal-standard&pivots=azure-openai#embeddings).
- A [managed identity](how-to-setup-managed-identity.md) (system-assigned or user-assigned) on the Azure Cosmos DB account, set as the account's [default identity](/cli/azure/cosmosdb#az-cosmosdb-update-optional-parameters). Azure Cosmos DB uses this identity to authenticate to the Microsoft Foundry resource on your behalf.
- A [role assignment](/azure/foundry-classic/openai/how-to/role-based-access-control#add-role-assignment-to-an-azure-openai-resource) on the Microsoft Foundry resource that grants the Azure Cosmos DB managed identity the [Cognitive Services OpenAI User](/azure/foundry-classic/openai/how-to/role-based-access-control#azure-openai-roles) role, so it can make inference API calls to the embedding model.

## Enable Integrated Embeddings

To enable Integrated Embeddings on your Azure Cosmos DB account, [follow these steps](https://aka.ms/enable-integrated-embeddings).

## Policy for Integrated Embeddings

Integrated Embeddings is configured as part of the container vector embedding policy. The existing vector embedding policy defines the vector path, data type, dimensions, and distance function. To have Azure Cosmos DB generate embeddings for a vector path, add an `embeddingSource` object to the corresponding `vectorEmbeddings` entry.

| Property         | Description                                                                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `sourcePaths`    | The item property paths whose values are used as input for embedding generation.                                                               |
| `deploymentName` | The name you assigned to the embedding model deployment in Microsoft Foundry.                                                                  |
| `modelName`      | The underlying embedding model, for example `text-embedding-3-small`.                                                                          |
| `endpoint`       | The endpoint URL of the Microsoft Foundry resource that hosts the deployment, for example `https://<foundry-resource-name>.openai.azure.com/`. |
| `authType`       | The authentication type used to make inference API calls to the embedding model. `Entra` is currently the only supported value.                |

> [!NOTE]
> For each item, Azure Cosmos DB concatenates the string values at the paths listed in `sourcePaths` and sends the result to the embedding model as a single input. This combined input is limited to 8,192 tokens per item, which is the maximum supported by the [Azure OpenAI embedding models](/azure/foundry/openai/how-to/embeddings#best-practice). If the combined input exceeds this limit, only the trailing portion of that input is truncated before it's sent to the embedding model. Truncation applies only to the embedding input. The item stored in Azure Cosmos DB, including any properties not listed in `sourcePaths`, isn't modified.

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

## Get started with Integrated Embeddings

This quickstart walks you through creating a container configured for Integrated Embeddings, inserting items, and verifying that Azure Cosmos DB generates and stores the embeddings. It assumes you have completed the [prerequisites](#prerequisites).

Integrated Embeddings is a preview feature. Support across the Azure Cosmos DB management SDKs, Azure CLI, Azure Resource Manager (ARM), and Bicep will expand over time. For now, you can try the feature in one of the following ways:

- Use the Azure Cosmos DB SDK with key-based authentication.
- Use the Azure Cosmos DB management SDK with Microsoft Entra ID.

### Use the Azure Cosmos DB SDK with key-based authentication

This option uses the Azure Cosmos DB SDK to create the database and container, and an account key for authentication.

#### [Python](#tab/python)

Install the [Azure Cosmos DB Python SDK](https://pypi.org/project/azure-cosmos):

```bash
pip install azure-cosmos
```

#### [Node.js](#tab/nodejs)

Install the [Azure Cosmos DB JavaScript SDK](https://www.npmjs.com/package/@azure/cosmos):

```bash
npm init -y && npm install @azure/cosmos
```

---

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

The following example creates a database and a new container, configures the vector embedding policy with an `embeddingSource`, inserts sample items with a `description` property, and polls them until Azure Cosmos DB adds the generated embeddings to `/embedding`.

`dimensions` is set to `1536`, which matches `text-embedding-3-small` and `text-embedding-ada-002`. Use `3072` for `text-embedding-3-large`.

> [!NOTE]
> This example uses a `quantizedFlat` vector index. To learn about other supported vector index types, see [Vector Indexing Policies](vector-search.md#vector-indexing-policies).

#### [Python](#tab/python)

Save the following script as `integrated_embeddings_quickstart.py`:

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


# Vector embedding policy: tells Cosmos DB which property to embed,
# which Foundry deployment to call, and where to store the generated embedding.
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

# Indexing policy: exclude the embedding path from the standard index
# and add a vector index.
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
        "description": "Azure Cosmos DB offers global distribution with multi-region writes and tunable consistency levels."
    },
    {
        "id": "item-3",
        "description": "Use the change feed to react to data changes in real time without polling."
    }
]


def main():
    client = CosmosClient(COSMOS_ENDPOINT, credential=COSMOS_KEY)

    # Create the database and a new container with the embedding and indexing policies.
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

    # Insert sample items. Azure Cosmos DB picks up the changes asynchronously
    # and generates embeddings in the background.
    for item in sample_items:
        container.upsert_item(item)
        print(f"Inserted item: {item['id']}")

    # Poll each item until Azure Cosmos DB writes the generated embedding back to it.
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

#### [Node.js](#tab/nodejs)

Save the following script as `integrated_embeddings_quickstart.js`:

```javascript
const { CosmosClient } = require("@azure/cosmos");

const COSMOS_ENDPOINT = process.env.COSMOS_ENDPOINT;
const COSMOS_KEY = process.env.COSMOS_KEY;
const DATABASE_NAME = process.env.COSMOS_DATABASE || "integrated-embeddings-db";
const CONTAINER_NAME = process.env.COSMOS_CONTAINER || "integrated-embeddings-items";
const FOUNDRY_ENDPOINT = process.env.FOUNDRY_ENDPOINT;
const FOUNDRY_DEPLOYMENT_NAME = process.env.FOUNDRY_DEPLOYMENT_NAME;
const FOUNDRY_MODEL_NAME = process.env.FOUNDRY_MODEL_NAME;

const EMBEDDING_PATH = "embedding";
const POLL_INTERVAL_MS = 5000;
const POLL_TIMEOUT_MS = 120000;

// Vector embedding policy: tells Cosmos DB which property to embed,
// which Foundry deployment to call, and where to store the generated embedding.
const vectorEmbeddingPolicy = {
  vectorEmbeddings: [
    {
      path: `/${EMBEDDING_PATH}`,
      dataType: "float32",
      dimensions: 1536,
      distanceFunction: "cosine",
      embeddingSource: {
        sourcePaths: ["/description"],
        deploymentName: FOUNDRY_DEPLOYMENT_NAME,
        modelName: FOUNDRY_MODEL_NAME,
        endpoint: FOUNDRY_ENDPOINT,
        authType: "Entra",
      },
    },
  ],
};

// Indexing policy: exclude the embedding path from the standard index
// and add a vector index.
const indexingPolicy = {
  indexingMode: "consistent",
  automatic: true,
  includedPaths: [{ path: "/*" }],
  excludedPaths: [
    { path: '/"_etag"/?' },
    { path: `/${EMBEDDING_PATH}/*` },
  ],
  vectorIndexes: [
    { path: `/${EMBEDDING_PATH}`, type: "quantizedFlat" },
  ],
};

const sampleItems = [
  {
    id: "item-1",
    description: "Azure Cosmos DB for NoSQL supports vector search for AI applications.",
  },
  {
    id: "item-2",
    description: "Azure Cosmos DB offers global distribution with multi-region writes and tunable consistency levels.",
  },
  {
    id: "item-3",
    description: "Use the change feed to react to data changes in real time without polling.",
  },
];

async function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function main() {
  const client = new CosmosClient({ endpoint: COSMOS_ENDPOINT, key: COSMOS_KEY });

  // Create the database and a new container with the embedding and indexing policies.
  const { database } = await client.databases.createIfNotExists({ id: DATABASE_NAME });

  let container;
  try {
    const result = await database.containers.create({
      id: CONTAINER_NAME,
      partitionKey: { paths: ["/id"], kind: "Hash" },
      vectorEmbeddingPolicy,
      indexingPolicy,
    });
    container = result.container;
  } catch (err) {
    if (err.code === 409) {
      throw new Error(
        `Container '${CONTAINER_NAME}' already exists. Use a new container name for this quickstart.`
      );
    }
    throw err;
  }

  // Insert sample items. Azure Cosmos DB picks up the changes asynchronously
  // and generates embeddings in the background.
  for (const item of sampleItems) {
    await container.items.upsert(item);
    console.log(`Inserted item: ${item.id}`);
  }

  // Poll each item until Azure Cosmos DB writes the generated embedding back to it.
  const pending = new Set(sampleItems.map((i) => i.id));
  const deadline = Date.now() + POLL_TIMEOUT_MS;

  while (pending.size > 0 && Date.now() < deadline) {
    for (const id of [...pending]) {
      const { resource } = await container.item(id, id).read();
      const embedding = resource && resource[EMBEDDING_PATH];
      if (embedding) {
        console.log(
          `Generated embedding for ${id} ` +
            `(dimensions: ${embedding.length}, preview: ${JSON.stringify(embedding.slice(0, 3))}...)`
        );
        pending.delete(id);
      }
    }
    if (pending.size > 0) {
      console.log(`Waiting for embeddings: ${[...pending].sort().join(", ")}`);
      await sleep(POLL_INTERVAL_MS);
    }
  }

  if (pending.size > 0) {
    throw new Error(`Embeddings were not generated for: ${[...pending].sort().join(", ")}`);
  }
}

main().catch((err) => {
  console.error(err.message || err);
  process.exit(1);
});
```

Run the script:

```bash
node integrated_embeddings_quickstart.js
```

The output should look similar to this example:

```text
Inserted item: item-1
Inserted item: item-2
Inserted item: item-3
Waiting for embeddings: item-1, item-2, item-3
Generated embedding for item-1 (dimensions: 1536, preview: [0.0049,0.0271,0.0490]...)
Generated embedding for item-2 (dimensions: 1536, preview: [0.0424,0.0374,0.1035]...)
Generated embedding for item-3 (dimensions: 1536, preview: [0.0129,0.0336,-0.0186]...)
```

---

### Use the Azure Management SDK with Microsoft Entra ID

This option uses the Azure Cosmos DB management SDK to create the database and container, and Microsoft Entra ID for authentication.

Install the [Azure Cosmos DB Python SDK](https://pypi.org/project/azure-cosmos/), the [Azure Cosmos DB Python management SDK](https://pypi.org/project/azure-mgmt-cosmosdb/), and the [Azure Identity library](https://pypi.org/project/azure-identity/):

```bash
pip install azure-cosmos azure-identity azure-mgmt-cosmosdb
```

Sign in to the Azure CLI:

```bash
az login
```

Assign the following roles to the identity that runs the script:

- `Cosmos DB Operator` on the Azure Cosmos DB account, to create the database and container through Azure Resource Manager.
- `Cosmos DB Built-in Data Contributor` on the Azure Cosmos DB account, to insert and read items.

> [!NOTE]
>
> - To assign the **Cosmos DB Operator** role, your account needs `Microsoft.Authorization/roleAssignments/write`, included in roles such as **Owner** and **User Access Administrator**.
> - To assign the **Cosmos DB Built-in Data Contributor** role, your account needs `Microsoft.DocumentDB/databaseAccounts/sqlRoleAssignments/write`, included in roles such as **Owner**, **Contributor**, and **DocumentDB Account Contributor**.
>
> For more information, see [Connect to Azure Cosmos DB for NoSQL using role-based access control and Microsoft Entra ID](how-to-connect-role-based-access-control.md).

```bash
# Find your principal ID (for an interactive user)
PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)

# Cosmos DB Operator (Azure RBAC)
az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Cosmos DB Operator" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.DocumentDB/databaseAccounts/<account-name>"

# Cosmos DB Built-in Data Contributor (Azure Cosmos DB RBAC)
az cosmosdb sql role assignment create \
  --account-name "<account-name>" \
  --resource-group "<resource-group>" \
  --scope "/" \
  --principal-id "$PRINCIPAL_ID" \
  --role-definition-id 00000000-0000-0000-0000-000000000002
```

Set the following environment variables for your Azure Cosmos DB account and Microsoft Foundry embedding model deployment:

```bash
export COSMOS_SUBSCRIPTION_ID="<subscription-id>"
export COSMOS_RESOURCE_GROUP="<resource-group-name>"
export COSMOS_ACCOUNT_NAME="<account-name>"
export COSMOS_LOCATION="<azure-region>"
export COSMOS_ENDPOINT="https://<account-name>.documents.azure.com:443/"
export COSMOS_DATABASE="integrated-embeddings-db"
export COSMOS_CONTAINER="integrated-embeddings-items"
export FOUNDRY_ENDPOINT="https://<foundry-resource-name>.openai.azure.com/"
export FOUNDRY_DEPLOYMENT_NAME="text-embedding-3-small"
export FOUNDRY_MODEL_NAME="text-embedding-3-small"
```

Save the following script as `integrated_embeddings_quickstart_mgmt_sdk.py`. The script creates a database and a new container, configures the vector embedding policy with an `embeddingSource`, inserts sample items with a `description` property, and polls them until Azure Cosmos DB adds the generated embeddings to `/embedding`.

The script sets `dimensions` to `1536`, which matches `text-embedding-3-small` and `text-embedding-ada-002`. Use `3072` for `text-embedding-3-large`.

> [!NOTE]
> This example uses a `quantizedFlat` vector index. To learn about other supported vector index types, see [Vector Indexing Policies](vector-search.md#vector-indexing-policies).

```python
import json
import os
import time

from azure.cosmos import CosmosClient
from azure.core.exceptions import HttpResponseError, ResourceNotFoundError
from azure.identity import DefaultAzureCredential
from azure.mgmt.cosmosdb import CosmosDBManagementClient
from azure.mgmt.cosmosdb.models import (
    SqlDatabaseCreateUpdateParameters,
    SqlDatabaseResource,
)


SUBSCRIPTION_ID = os.environ["COSMOS_SUBSCRIPTION_ID"]
RESOURCE_GROUP_NAME = os.environ["COSMOS_RESOURCE_GROUP"]
ACCOUNT_NAME = os.environ["COSMOS_ACCOUNT_NAME"]
LOCATION = os.environ["COSMOS_LOCATION"]
COSMOS_ENDPOINT = os.environ["COSMOS_ENDPOINT"]
DATABASE_NAME = os.environ.get("COSMOS_DATABASE", "integrated-embeddings-db")
CONTAINER_NAME = os.environ.get("COSMOS_CONTAINER", "integrated-embeddings-items")
FOUNDRY_ENDPOINT = os.environ["FOUNDRY_ENDPOINT"]
FOUNDRY_DEPLOYMENT_NAME = os.environ["FOUNDRY_DEPLOYMENT_NAME"]
FOUNDRY_MODEL_NAME = os.environ["FOUNDRY_MODEL_NAME"]

EMBEDDING_PATH = "embedding"
MAX_AUTOSCALE_THROUGHPUT = 1000
POLL_INTERVAL_SECONDS = 5
POLL_TIMEOUT_SECONDS = 120


# Container definition passed to the management SDK. Combines the indexing policy
# (with a vector index on the embedding path) and the vector embedding policy
# (with embeddingSource pointing at the Foundry deployment).
container_body = {
    "location": LOCATION,
    "properties": {
        "resource": {
            "id": CONTAINER_NAME,
            "partitionKey": {"paths": ["/id"], "kind": "Hash"},
            "indexingPolicy": {
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
                    {"path": f"/{EMBEDDING_PATH}", "type": "quantizedFlat"}
                ]
            },
            "vectorEmbeddingPolicy": {
                "vectorEmbeddings": [
                    {
                        "path": f"/{EMBEDDING_PATH}",
                        "dataType": "float32",
                        "dimensions": 1536,
                        "distanceFunction": "cosine",
                        "embeddingSource": {
                            "sourcePaths": ["/description"],
                            "deploymentName": FOUNDRY_DEPLOYMENT_NAME,
                            "modelName": FOUNDRY_MODEL_NAME,
                            "endpoint": FOUNDRY_ENDPOINT,
                            "authType": "Entra"
                        }
                    }
                ]
            }
        },
        "options": {
            "autoscaleSettings": {"maxThroughput": MAX_AUTOSCALE_THROUGHPUT}
        }
    }
}


sample_items = [
    {
        "id": "item-1",
        "description": "Azure Cosmos DB for NoSQL supports vector search for AI applications."
    },
    {
        "id": "item-2",
        "description": "Azure Cosmos DB offers global distribution with multi-region writes and tunable consistency levels."
    },
    {
        "id": "item-3",
        "description": "Use the change feed to react to data changes in real time without polling."
    }
]


def create_database(mgmt):
    print(f"Creating database '{DATABASE_NAME}'...")
    params = SqlDatabaseCreateUpdateParameters(
        location=LOCATION,
        resource=SqlDatabaseResource(id=DATABASE_NAME),
    )
    poller = mgmt.sql_resources.begin_create_update_sql_database(
        resource_group_name=RESOURCE_GROUP_NAME,
        account_name=ACCOUNT_NAME,
        database_name=DATABASE_NAME,
        create_update_sql_database_parameters=params,
    )
    result = poller.result()
    print(f"  Database ready: {result.id}")


def create_container(mgmt):
    print(f"Checking container '{CONTAINER_NAME}'...")
    try:
        mgmt.sql_resources.get_sql_container(
            resource_group_name=RESOURCE_GROUP_NAME,
            account_name=ACCOUNT_NAME,
            database_name=DATABASE_NAME,
            container_name=CONTAINER_NAME,
        )
    except ResourceNotFoundError:
        pass
    else:
        raise RuntimeError(
            f"Container '{CONTAINER_NAME}' already exists. Use a new container name for this quickstart."
        )

    print(f"Creating container '{CONTAINER_NAME}'...")
    body_bytes = json.dumps(container_body).encode("utf-8")
    poller = mgmt.sql_resources.begin_create_update_sql_container(
        resource_group_name=RESOURCE_GROUP_NAME,
        account_name=ACCOUNT_NAME,
        database_name=DATABASE_NAME,
        container_name=CONTAINER_NAME,
        create_update_sql_container_parameters=body_bytes,
        content_type="application/json",
    )
    result = poller.result()
    print(f"  Container ready: {result.id}")


def upsert_and_poll(credential):
    client = CosmosClient(COSMOS_ENDPOINT, credential=credential)
    container = client.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)

    # Insert sample items. Azure Cosmos DB picks up the changes asynchronously
    # and generates embeddings in the background.
    for item in sample_items:
        container.upsert_item(item)
        print(f"Inserted item: {item['id']}")

    # Poll each item until Azure Cosmos DB writes the generated embedding back to it.
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


def main():
    credential = DefaultAzureCredential()
    try:
        mgmt = CosmosDBManagementClient(
            credential=credential,
            subscription_id=SUBSCRIPTION_ID,
        )
        try:
            create_database(mgmt)
            create_container(mgmt)
        except HttpResponseError as ex:
            print(f"ARM call failed: status={ex.status_code} message={ex.message}")
            raise
        finally:
            mgmt.close()

        upsert_and_poll(credential)
    finally:
        credential.close()


if __name__ == "__main__":
    main()
```

Run the script:

```bash
python integrated_embeddings_quickstart_mgmt_sdk.py
```

The output should look similar to this example:

```text
Creating database 'integrated-embeddings-db'...
  Database ready: <database-resource-id>
Checking container 'integrated-embeddings-items'...
Creating container 'integrated-embeddings-items'...
  Container ready: <container-resource-id>
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

Integrated Embeddings is in preview, and the following limitations apply.

- Portal support: [Container vector policies](vector-search.md#container-vector-policies) can be managed in the Azure portal, but the `embeddingSource` configuration isn't supported there yet. Until support is added, use the SDK options shown in the [quickstart](#get-started-with-integrated-embeddings).
- Tooling support: Refer to the [quickstart](#get-started-with-integrated-embeddings) for how to use the feature with the currently supported options, until broader support is available across the Azure Cosmos DB management SDKs, Azure CLI, Azure Resource Manager (ARM), and Bicep.

## Related content

- [VectorDistance system function](/cosmos-db/query/vectordistance)
- [Vector index policies](index-policy.md#vector-indexes)
- [Vector indexing policy examples](how-to-manage-indexing-policy.md#vector-indexing-policy-examples)