---
title: "Quickstart - Vector Indexing with Node.js"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Node.js."
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: javascript
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-js
  - devx-track-js-ai
  - devx-track-data-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with Node.js

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

This quickstart uses a sample hotel dataset in a JSON file with pre-calculated vectors from the `text-embedding-3-small` model. The dataset includes hotel names, locations, descriptions, and vector embeddings.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-typescript) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Index Quickstart](includes/prerequisite-quickstart-vector-index.md)]

- [Node.js LTS](https://nodejs.org/download/)

- [TypeScript](https://www.typescriptlang.org/download): Install TypeScript globally:

    ```bash
    npm install -g typescript
    ```

## Create data file with vectors

1. Create a new data directory for the hotels data file:

    ```bash
    mkdir data
    ```

1. Copy the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory.

## Set up the project

1. Create and navigate to a new project directory:

    ```bash
    mkdir documentdb-vector-quickstart
    cd documentdb-vector-quickstart
    ```

1. Initialize a TypeScript Node.js project:

    ```bash
    npm init -y
    npm install --save typescript ts-node @types/node
    npm install --save mongodb openai dotenv @azure/identity
    ```

1. Create a `tsconfig.json`:

    ```json
    {
      "compilerOptions": {
        "target": "ES2020",
        "module": "ES2020",
        "lib": ["ES2020"],
        "moduleResolution": "node",
        "esModuleInterop": true,
        "skipLibCheck": true,
        "forceConsistentCasingInFileNames": true,
        "resolveJsonModule": true,
        "strict": true,
        "outDir": "./dist"
      },
      "include": ["src/**/*"],
      "exclude": ["node_modules"]
    }
    ```

1. Create a `.env` file with your credentials:

    ```env
    MONGO_CLUSTER_NAME=
    AZURE_OPENAI_EMBEDDING_ENDPOINT=
    AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
    AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
    EMBEDDED_FIELD=DescriptionVector
    EMBEDDING_DIMENSIONS=1536
    DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
    LOAD_SIZE_BATCH=100
    ```

    Replace the placeholder values in the `.env` file with your own information:
    - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
    - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB resource name

## Create a vector index

### [IVF](#tab/tab-ivf)

IVF (Inverted File) is ideal for datasets with fewer than 10,000 documents. It partitions vectors into clusters for fast approximate search.

Create `src/ivf.ts`:

```typescript
import { MongoClient } from 'mongodb';
import { AzureOpenAI } from 'openai/index.js';
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';
import path from 'path';
import { fileURLToPath } from 'node:url';
import { promises as fs } from 'fs';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

const config = {
  query: "quintessential lodging near running trails, eateries, retail",
  dbName: "Hotels",
  collectionName: "hotels_ivf",
  indexName: "vectorIndex_ivf",
  embeddedField: process.env.EMBEDDED_FIELD!,
  embeddingDimensions: parseInt(process.env.EMBEDDING_DIMENSIONS!, 10),
  deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
};

async function main() {
  const credential = new DefaultAzureCredential();
  
  // Connect to DocumentDB with passwordless authentication
  const dbClient = new MongoClient(
    `mongodb+srv://${process.env.MONGO_CLUSTER_NAME}.mongocluster.cosmos.azure.com/`,
    {
      connectTimeoutMS: 120000,
      tls: true,
      retryWrites: false,
      authMechanism: 'MONGODB-OIDC',
      authMechanismProperties: {
        OIDC_CALLBACK: async (params) => {
          const token = await credential.getToken('https://ossrdbms-aad.database.windows.net/.default');
          return {
            accessToken: token?.token || '',
            expiresInSeconds: (token?.expiresOnTimestamp || 0) - Math.floor(Date.now() / 1000)
          };
        },
        ALLOWED_HOSTS: ['*.azure.com']
      }
    }
  );

  // Create Azure OpenAI client
  const scope = "https://cognitiveservices.azure.com/.default";
  const azureADTokenProvider = getBearerTokenProvider(credential, scope);
  const aiClient = new AzureOpenAI({
    apiVersion: process.env.AZURE_OPENAI_EMBEDDING_API_VERSION!,
    endpoint: process.env.AZURE_OPENAI_EMBEDDING_ENDPOINT!,
    deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
    azureADTokenProvider
  });

  try {
    await dbClient.connect();
    const db = dbClient.db(config.dbName);
    const collection = await db.createCollection(config.collectionName);

    // Create the IVF vector index
    const indexOptions = {
      createIndexes: config.collectionName,
      indexes: [
        {
          name: config.indexName,
          key: {
            [config.embeddedField]: 'cosmosSearch'
          },
          cosmosSearchOptions: {
            kind: 'vector-ivf',
            numLists: 10,  // Number of clusters - tune based on dataset
            similarity: 'COS',  // Cosine similarity for text embeddings
            dimensions: config.embeddingDimensions
          }
        }
      ]
    };
    
    const vectorIndexSummary = await db.command(indexOptions);
    console.log('IVF vector index created successfully');

    // Create embedding for the query
    const createEmbeddedForQueryResponse = await aiClient.embeddings.create({
      model: config.deployment,
      input: [config.query]
    });

    // Perform vector search using $search aggregation
    const searchResults = await collection.aggregate([
      {
        $search: {
          cosmosSearch: {
            vector: createEmbeddedForQueryResponse.data[0].embedding,
            path: config.embeddedField,
            k: 5  // Top 5 results
          }
        }
      },
      {
        $project: {
          score: { $meta: "searchScore" },
          document: "$$ROOT"
        }
      }
    ]).toArray();

    console.log(`\nSearch Results (${searchResults.length} found):`);
    searchResults.forEach((result, index) => {
      console.log(`${index + 1}. ${result.document.HotelName}, Score: ${result.score.toFixed(4)}`);
    });

  } finally {
    await dbClient.close();
  }
}

main().catch(console.error);
```

#### [HNSW](#tab/tab-hnsw)

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `src/hnsw.ts`:

```typescript
import { MongoClient } from 'mongodb';
import { AzureOpenAI } from 'openai/index.js';
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';

const config = {
  query: "quintessential lodging near running trails, eateries, retail",
  dbName: "Hotels",
  collectionName: "hotels_hnsw",
  indexName: "vectorIndex_hnsw",
  embeddedField: process.env.EMBEDDED_FIELD!,
  embeddingDimensions: parseInt(process.env.EMBEDDING_DIMENSIONS!, 10),
  deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
};

async function main() {
  const credential = new DefaultAzureCredential();

  const dbClient = new MongoClient(
    `mongodb+srv://${process.env.MONGO_CLUSTER_NAME}.mongocluster.cosmos.azure.com/`,
    {
      connectTimeoutMS: 120000,
      tls: true,
      retryWrites: false,
      authMechanism: 'MONGODB-OIDC',
      authMechanismProperties: {
        OIDC_CALLBACK: async (params) => {
          const token = await credential.getToken('https://ossrdbms-aad.database.windows.net/.default');
          return {
            accessToken: token?.token || '',
            expiresInSeconds: (token?.expiresOnTimestamp || 0) - Math.floor(Date.now() / 1000)
          };
        },
        ALLOWED_HOSTS: ['*.azure.com']
      }
    }
  );

  const scope = "https://cognitiveservices.azure.com/.default";
  const azureADTokenProvider = getBearerTokenProvider(credential, scope);
  const aiClient = new AzureOpenAI({
    apiVersion: process.env.AZURE_OPENAI_EMBEDDING_API_VERSION!,
    endpoint: process.env.AZURE_OPENAI_EMBEDDING_ENDPOINT!,
    deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
    azureADTokenProvider
  });

  try {
    await dbClient.connect();
    const db = dbClient.db(config.dbName);
    const collection = await db.createCollection(config.collectionName);

    // Create the HNSW vector index
    const indexOptions = {
      createIndexes: config.collectionName,
      indexes: [
        {
          name: config.indexName,
          key: {
            [config.embeddedField]: 'cosmosSearch'
          },
          cosmosSearchOptions: {
            kind: 'vector-hnsw',

            // Maximum connections per node in the graph (2-100, default 16)
            m: 16,

            // Candidate list size during index construction (4-1000, default 64)
            efConstruction: 64,

            // Cosine similarity for text embeddings
            similarity: 'COS',
            dimensions: config.embeddingDimensions
          }
        }
      ]
    };

    await db.command(indexOptions);
    console.log('HNSW vector index created successfully');

    // Generate embedding for the query
    const embeddingResponse = await aiClient.embeddings.create({
      model: config.deployment,
      input: [config.query]
    });

    // Perform vector search
    const searchResults = await collection.aggregate([
      {
        $search: {
          cosmosSearch: {
            vector: embeddingResponse.data[0].embedding,
            path: config.embeddedField,
            k: 5
          }
        }
      },
      {
        $project: {
          score: { $meta: "searchScore" },
          document: "$$ROOT"
        }
      }
    ]).toArray();

    console.log(`\nSearch Results (${searchResults.length} found):`);
    searchResults.forEach((result, index) => {
      console.log(`${index + 1}. ${result.document.HotelName}, Score: ${result.score.toFixed(4)}`);
    });

  } finally {
    await dbClient.close();
  }
}

main().catch(console.error);
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity (2–100, default 16). Higher values improve recall but increase memory.
- **efConstruction**: Candidate list size during construction (4–1000, default 64). Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

#### [DiskANN](#tab/tab-diskann)

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `src/diskann.ts`:

```typescript
import { MongoClient } from 'mongodb';
import { AzureOpenAI } from 'openai/index.js';
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';

const config = {
  query: "quintessential lodging near running trails, eateries, retail",
  dbName: "Hotels",
  collectionName: "hotels_diskann",
  indexName: "vectorIndex_diskann",
  embeddedField: process.env.EMBEDDED_FIELD!,
  embeddingDimensions: parseInt(process.env.EMBEDDING_DIMENSIONS!, 10),
  deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
};

async function main() {
  const credential = new DefaultAzureCredential();

  const dbClient = new MongoClient(
    `mongodb+srv://${process.env.MONGO_CLUSTER_NAME}.mongocluster.cosmos.azure.com/`,
    {
      connectTimeoutMS: 120000,
      tls: true,
      retryWrites: false,
      authMechanism: 'MONGODB-OIDC',
      authMechanismProperties: {
        OIDC_CALLBACK: async (params) => {
          const token = await credential.getToken('https://ossrdbms-aad.database.windows.net/.default');
          return {
            accessToken: token?.token || '',
            expiresInSeconds: (token?.expiresOnTimestamp || 0) - Math.floor(Date.now() / 1000)
          };
        },
        ALLOWED_HOSTS: ['*.azure.com']
      }
    }
  );

  const scope = "https://cognitiveservices.azure.com/.default";
  const azureADTokenProvider = getBearerTokenProvider(credential, scope);
  const aiClient = new AzureOpenAI({
    apiVersion: process.env.AZURE_OPENAI_EMBEDDING_API_VERSION!,
    endpoint: process.env.AZURE_OPENAI_EMBEDDING_ENDPOINT!,
    deployment: process.env.AZURE_OPENAI_EMBEDDING_MODEL!,
    azureADTokenProvider
  });

  try {
    await dbClient.connect();
    const db = dbClient.db(config.dbName);
    const collection = await db.createCollection(config.collectionName);

    // Create the DiskANN vector index
    const indexOptions = {
      createIndexes: config.collectionName,
      indexes: [
        {
          name: config.indexName,
          key: {
            [config.embeddedField]: 'cosmosSearch'
          },
          cosmosSearchOptions: {
            kind: 'vector-diskann',

            // Maximum edges per node in the graph (20-2048, default 32)
            maxDegree: 32,

            // Candidates evaluated during construction (10-500, default 50)
            lBuild: 50,

            // Cosine similarity for text embeddings
            similarity: 'COS',
            dimensions: config.embeddingDimensions
          }
        }
      ]
    };

    await db.command(indexOptions);
    console.log('DiskANN vector index created successfully');

    // Generate embedding for the query
    const embeddingResponse = await aiClient.embeddings.create({
      model: config.deployment,
      input: [config.query]
    });

    // Perform vector search
    const searchResults = await collection.aggregate([
      {
        $search: {
          cosmosSearch: {
            vector: embeddingResponse.data[0].embedding,
            path: config.embeddedField,
            k: 5
          }
        }
      },
      {
        $project: {
          score: { $meta: "searchScore" },
          document: "$$ROOT"
        }
      }
    ]).toArray();

    console.log(`\nSearch Results (${searchResults.length} found):`);
    searchResults.forEach((result, index) => {
      console.log(`${index + 1}. ${result.document.HotelName}, Score: ${result.score.toFixed(4)}`);
    });

  } finally {
    await dbClient.close();
  }
}

main().catch(console.error);
```

Key parameters:
- **maxDegree**: Number of edges per node (20–2048, default 32). Higher values improve accuracy.
- **lBuild**: Candidate neighbors evaluated during construction (10–500, default 50). Affects index quality.
- **Cluster tier**: Requires M30 or higher.

----

## Query with vector search

All three algorithms use the same query pattern with the `$search` aggregation stage:

```typescript
// Generate embedding for query
const embedding = await aiClient.embeddings.create({
  model: config.deployment,
  input: ["your query text"]
});

// Execute vector search
const results = await collection.aggregate([
  {
    $search: {
      cosmosSearch: {
        vector: embedding.data[0].embedding,
        path: config.embeddedField,
        k: 5  // Return top 5 similar documents
      }
    }
  },
  {
    $project: {
      score: { $meta: "searchScore" },
      document: "$$ROOT"
    }
  }
]).toArray();
```

The `$search` stage finds the k nearest neighbors to your query vector. Results are ordered by similarity score (highest first).

## Choose the right algorithm

| Algorithm | Dataset Size | Cluster Tier | Query Speed | Accuracy | Memory |
|-----------|--------------|--------------|-------------|----------|--------|
| IVF | < 10K docs | M10+ | Very fast | Good | Low |
| HNSW | 10K-50K docs | M30+ | Fast | Excellent | Medium |
| DiskANN | 50K+ docs | M30+ | Medium | Excellent | Low (disk-based) |

**Selection guidelines:**
- **IVF**: Start here for small datasets. Simple and resource-efficient.
- **HNSW**: Choose for medium datasets where recall is important. Best recall rates.
- **DiskANN**: Required for datasets exceeding 50,000 documents. Balances accuracy and resource usage.

## Authenticate with Azure CLI

Sign in to Azure CLI before you run the application so it can access Azure resources securely.

```bash
az login
```

The code uses your local developer authentication to access Azure DocumentDB and Azure OpenAI. The authentication relies on [DefaultAzureCredential](/javascript/api/@azure/identity/defaultazurecredential) from **@azure/identity** to find your Azure credentials in the environment.

## Run the quickstart

```bash
# Compile TypeScript
npx tsc

# Run IVF example
node dist/ivf.js

# Run HNSW example
node dist/hnsw.js

# Run DiskANN example
node dist/diskann.js
```

You see the top hotels that match the vector search query and their similarity scores.

## View and manage data in Visual Studio Code

1. Select the [DocumentDB extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) in Visual Studio Code to connect to your Azure DocumentDB account.
1. View the data and indexes in the Hotels database.

## Clean up resources

Delete the resource group, Azure DocumentDB account, and Azure OpenAI resource when you don't need them to avoid extra costs.

## Related content

- [Vector store in Azure DocumentDB](vector-search.md)
- [Azure OpenAI embeddings](/azure/ai-services/openai/concepts/understand-embeddings)
- [MongoDB aggregation pipeline reference](https://www.mongodb.com/docs/manual/reference/operator/aggregation/)
