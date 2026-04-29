---
title: "Quickstart - Vector Indexing with Node.js"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Node.js."
ms.reviewer: khelanmodi
ms.devlang: javascript
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-js
  - devx-track-data-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with TypeScript

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-typescript) on GitHub.

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

## Prerequisites

- An Azure subscription ([create one for free](https://azure.microsoft.com/free/))
- Azure DocumentDB vCore cluster with appropriate tier:
  - **IVF**: M10 or higher
  - **HNSW**: M30 or higher
  - **DiskANN**: M30 or higher
- [Azure OpenAI resource](https://learn.microsoft.com/azure/ai-services/openai/how-to/create-resource) with an embeddings model deployed
- [Node.js 18+](https://nodejs.org/)
- [Visual Studio Code](https://code.visualstudio.com/) or your preferred editor

## Set up the project

1. Create and navigate to a new project directory:

```bash
mkdir documentdb-vector-quickstart
cd documentdb-vector-quickstart
```

2. Initialize a TypeScript Node.js project:

```bash
npm init -y
npm install --save typescript ts-node @types/node
npm install --save mongodb openai dotenv @azure/identity
```

3. Create a `tsconfig.json`:

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

4. Create a `.env` file with your credentials:

```env
MONGO_CLUSTER_NAME=your-cluster-name
AZURE_OPENAI_EMBEDDING_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
EMBEDDED_FIELD=DescriptionVector
EMBEDDING_DIMENSIONS=1536
DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
LOAD_SIZE_BATCH=100
```

## Create an IVF index

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
          },
          returnStoredSource: true
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

## Create an HNSW index

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `src/hnsw.ts`:

```typescript
// Similar to IVF above, but with these index options:
cosmosSearchOptions: {
  kind: 'vector-hnsw',
  m: 16,  // Maximum connections per node (2-100, default 16)
  efConstruction: 64,  // Candidate list size during construction (4-1000, default 64)
  similarity: 'COS',
  dimensions: config.embeddingDimensions
}
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity. Higher values (e.g., 32) improve recall but increase memory.
- **efConstruction**: Affects index build time and quality. Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

## Create a DiskANN index

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `src/diskann.ts`:

```typescript
// Similar to IVF and HNSW above, but with these index options:
cosmosSearchOptions: {
  kind: 'vector-diskann',
  maxDegree: 20,  // Maximum edges per node (20-2048)
  lBuild: 10,  // Candidate neighbors evaluated (10-500)
  similarity: 'COS',
  dimensions: config.embeddingDimensions
}
```

Key parameters:
- **maxDegree**: Number of edges per node in the graph. Higher values improve accuracy.
- **lBuild**: Number of candidate neighbors evaluated during construction. Affects index quality.
- **Cluster tier**: Requires M30 or higher.

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

## Clean up resources

When you're done, delete the DocumentDB cluster and OpenAI resource from the Azure Portal to avoid ongoing charges.

## Next steps

- [DocumentDB Vector Search Documentation](https://learn.microsoft.com/azure/cosmos-db/mongodb/vcore/vector-search)
- [Azure OpenAI Embeddings Documentation](https://learn.microsoft.com/azure/ai-services/openai/concepts/understand-embeddings)
- [MongoDB Aggregation Pipeline Reference](https://www.mongodb.com/docs/manual/reference/operator/aggregation/)
- Article 1: Getting Started with Vector Search
- Article 3: Performance Tuning and Optimization
