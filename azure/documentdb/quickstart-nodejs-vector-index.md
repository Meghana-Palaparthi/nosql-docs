---
title: Quickstart: Choose and configure vector indexes in Azure DocumentDB with TypeScript
description: Compare vector index algorithms and similarity functions using TypeScript in Azure DocumentDB to optimize search performance for your workload.
ms.topic: quickstart
ms.date: 2025-01-30
author: diberry
ms.author: diberry
ms.service: azure-documentdb
ms.subservice: vector-search
---

# Quickstart: Choose and configure vector indexes in Azure DocumentDB with TypeScript

In this quickstart, you compare three vector index algorithms (DiskANN, HNSW, and IVF) and three similarity functions (cosine, L2, and inner product) to find the optimal configuration for your search workload. This quickstart uses a sample hotel dataset with pre-calculated embeddings from the `text-embedding-3-small` model.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-typescript) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites](includes/prerequisite-quickstart-vector-index.md)]

- [Node.js LTS](https://nodejs.org/download/)
- [TypeScript](https://www.typescriptlang.org/download) 5.x or greater

## Create data file with vectors

1. Create a new data directory for the hotels data file:

   ```bash
   mkdir data
   ```

2. Copy the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory.

## Create a Node.js project

1. Create a new directory for your project and open it in Visual Studio Code:

   ```bash
   mkdir select-algorithm-typescript
   cd select-algorithm-typescript
   code .
   ```

2. Initialize a TypeScript Node.js project:

   ```bash
   npm init -y
   ```

3. Install the required packages:

   ```bash
   npm install mongodb openai @azure/identity
   npm install --save-dev typescript @types/node
   ```

   - `mongodb`: MongoDB driver for Node.js
   - `openai`: OpenAI client library to create vectors
   - `@azure/identity`: Azure Identity library for passwordless authentication
   - `typescript`: TypeScript compiler

4. Create a `tsconfig.json` file in the project root:

   ```json
   {
     "compilerOptions": {
       "target": "ES2022",
       "module": "ES2022",
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

5. Update your `package.json` to include:

   ```json
   {
     "type": "module",
     "scripts": {
       "build": "tsc",
       "start": "node --env-file .env dist/select-algorithm.js"
     }
   }
   ```

6. Create a `.env` file for environment variables in the project root:

   ```bash
   # Azure OpenAI Embedding Settings
   AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
   AZURE_OPENAI_EMBEDDING_ENDPOINT=https://<RESOURCE-NAME>.openai.azure.com

   # Data File Paths and Vector Configuration
   DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
   EMBEDDED_FIELD=DescriptionVector
   EMBEDDING_DIMENSIONS=1536
   LOAD_SIZE_BATCH=100

   # Azure DocumentDB Connection Settings
   MONGO_CLUSTER_NAME=<CLUSTER-NAME>

   # Azure DocumentDB Database Name
   AZURE_DOCUMENTDB_DATABASENAME=Hotels

   # Algorithm Selection (used by select-algorithm.ts)
   # ALGORITHM: "all" | "diskann" | "hnsw" | "ivf"
   ALGORITHM=all

   # SIMILARITY: "all" | "COS" | "L2" | "IP"
   SIMILARITY=COS
   ```

   For the passwordless authentication used in this article, replace the placeholder values in the `.env` file with your own information:

   - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
   - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB cluster name

   You should always prefer passwordless authentication, but it requires additional setup. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate JavaScript apps to Azure services using the Azure SDK for JavaScript](/azure/developer/javascript/sdk/authentication/overview).

## Create code files

Create the following project structure:

```
├── data/
│   └── Hotels_Vector.json       # Hotel data with vector embeddings
└── select-algorithm-typescript/
    ├── src/
    │   ├── select-algorithm.ts  # Main comparison script
    │   └── utils.ts             # Shared utility functions
    ├── tsconfig.json
    ├── package.json
    └── .env                     # Environment variables
```

Create the `src` directory:

```bash
mkdir src
```

## Create the algorithm comparison code

Create the `src/select-algorithm.ts` file with the following code:

```typescript
import path from 'path';
import { readFileReturnJson, getClientsPasswordless, insertData, printComparisonTable } from './utils.js';

import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Validate required environment variables at startup
const requiredEnvVars = [
    'MONGO_CLUSTER_NAME',
    'AZURE_OPENAI_EMBEDDING_ENDPOINT',
    'AZURE_OPENAI_EMBEDDING_MODEL',
    'DATA_FILE_WITH_VECTORS'
];

const missing = requiredEnvVars.filter(v => !process.env[v]);
if (missing.length > 0) {
    console.error(`Missing required environment variables: ${missing.join(', ')}`);
    console.error('See .env.example for required values.');
    process.exit(1);
}

type Algorithm = 'diskann' | 'hnsw' | 'ivf';
type Similarity = 'COS' | 'L2' | 'IP';

const ALGORITHMS: Algorithm[] = ['diskann', 'hnsw', 'ivf'];
const SIMILARITIES: Similarity[] = ['COS', 'L2', 'IP'];

const ALGORITHM_LABELS: Record<Algorithm, string> = {
    diskann: 'DiskANN',
    hnsw: 'HNSW',
    ivf: 'IVF',
};

/**
 * Build the index creation command for a specific algorithm and similarity function.
 *
 * Each algorithm has different tuning parameters:
 * - DiskANN: maxDegree (graph connectivity), lBuild (build quality)
 * - HNSW: m (graph connectivity), efConstruction (build quality)
 * - IVF: numLists (number of clusters)
 */
function getIndexOptions(
    collectionName: string,
    indexName: string,
    embeddedField: string,
    dimensions: number,
    algorithm: Algorithm,
    similarity: Similarity
) {
    const base = {
        createIndexes: collectionName,
        indexes: [
            {
                name: indexName,
                key: { [embeddedField]: 'cosmosSearch' },
                cosmosSearchOptions: {} as Record<string, any>,
            },
        ],
    };

    switch (algorithm) {
        case 'diskann':
            base.indexes[0].cosmosSearchOptions = {
                kind: 'vector-diskann',
                dimensions,
                similarity,
                maxDegree: 32,
                lBuild: 50,
            };
            break;
        case 'hnsw':
            base.indexes[0].cosmosSearchOptions = {
                kind: 'vector-hnsw',
                dimensions,
                similarity,
                m: 16,
                efConstruction: 64,
            };
            break;
        case 'ivf':
            base.indexes[0].cosmosSearchOptions = {
                kind: 'vector-ivf',
                dimensions,
                similarity,
                numLists: 1,
            };
            break;
    }

    return base;
}

/**
 * Build the vector search aggregation pipeline with algorithm-specific search parameters.
 *
 * Search parameters control the recall/latency tradeoff at query time:
 * - DiskANN: lSearch (search list size)
 * - HNSW: efSearch (search candidates)
 * - IVF: nProbes (clusters to search)
 */
function getSearchPipeline(
    queryEmbedding: number[],
    embeddedField: string,
    k: number,
    algorithm: Algorithm
) {
    const cosmosSearch: Record<string, any> = {
        vector: queryEmbedding,
        path: embeddedField,
        k,
    };

    switch (algorithm) {
        case 'diskann':
            cosmosSearch.lSearch = 100;
            break;
        case 'hnsw':
            cosmosSearch.efSearch = 80;
            break;
        case 'ivf':
            cosmosSearch.nProbes = 1;
            break;
    }

    return [
        { $search: { cosmosSearch } },
        { $project: { score: { $meta: "searchScore" }, document: "$$ROOT" } },
    ];
}

/**
 * Determine which collections to create/query based on ALGORITHM and SIMILARITY env vars.
 * Collection naming: hotels_{algorithm}_{similarity}
 */
function getTargetCollections(
    algorithmEnv: string,
    similarityEnv: string
): Array<{ collectionName: string; algorithm: Algorithm; similarity: Similarity }> {
    const algorithms: Algorithm[] =
        algorithmEnv === 'all' ? ALGORITHMS : [algorithmEnv as Algorithm];
    const similarities: Similarity[] =
        similarityEnv === 'all' ? SIMILARITIES : [similarityEnv as Similarity];

    const targets: Array<{ collectionName: string; algorithm: Algorithm; similarity: Similarity }> = [];

    for (const alg of algorithms) {
        if (!ALGORITHMS.includes(alg)) {
            throw new Error(`Invalid ALGORITHM '${alg}'. Must be one of: all, ${ALGORITHMS.join(', ')}`);
        }
        for (const sim of similarities) {
            if (!SIMILARITIES.includes(sim)) {
                throw new Error(`Invalid SIMILARITY '${sim}'. Must be one of: all, ${SIMILARITIES.join(', ')}`);
            }
            targets.push({
                collectionName: `hotels_${alg}_${sim.toLowerCase()}`,
                algorithm: alg,
                similarity: sim,
            });
        }
    }

    return targets;
}

async function main() {
    const { aiClient, dbClient } = getClientsPasswordless();

    try {
        if (!aiClient) {
            throw new Error('Azure OpenAI client is not configured. Please check your environment variables.');
        }
        if (!dbClient) {
            throw new Error('Database client is not configured. Please check your environment variables.');
        }

        const dbName = process.env.AZURE_DOCUMENTDB_DATABASENAME || 'Hotels';
        const embeddedField = process.env.EMBEDDED_FIELD || 'DescriptionVector';
        const embeddingDimensions = parseInt(process.env.EMBEDDING_DIMENSIONS || '1536', 10);
        const dataFile = process.env.DATA_FILE_WITH_VECTORS || '../data/Hotels_Vector.json';
        const deployment = process.env.AZURE_OPENAI_EMBEDDING_MODEL!;
        const batchSize = parseInt(process.env.LOAD_SIZE_BATCH || '100', 10);
        const algorithmEnv = (process.env.ALGORITHM || 'all').trim().toLowerCase();
        const similarityEnv = (process.env.SIMILARITY || 'COS').trim().toUpperCase();
        const searchQuery = 'quintessential lodging near running trails, eateries, retail';

        const targets = getTargetCollections(algorithmEnv, similarityEnv);

        console.log(`\nVector Algorithm Comparison`);
        console.log(`   Database: ${dbName}`);
        console.log(`   Algorithms: ${algorithmEnv}`);
        console.log(`   Similarity: ${similarityEnv}`);
        console.log(`   Collections to query: ${targets.map(t => t.collectionName).join(', ')}`);
        console.log(`   Search query: "${searchQuery}"\n`);

        await dbClient.connect();
        const db = dbClient.db(dbName);

        // Load data once (shared across collections)
        const data = await readFileReturnJson(path.join(__dirname, '..', dataFile));

        // Generate query embedding once (reuse across collections)
        console.log('Generating query embedding...');
        const embeddingResponse = await aiClient.embeddings.create({
            model: deployment,
            input: [searchQuery],
        });
        const queryEmbedding = embeddingResponse.data[0].embedding;
        if (queryEmbedding.length !== embeddingDimensions) {
            throw new Error(
                `Embedding dimension mismatch: expected ${embeddingDimensions}, got ${queryEmbedding.length}. ` +
                `Verify AZURE_OPENAI_EMBEDDING_MODEL matches the configured EMBEDDING_DIMENSIONS.`
            );
        }
        console.log(`Query embedding: ${queryEmbedding.length} dimensions\n`);

        const config = { batchSize };

        const comparisonResults: Array<{
            collectionName: string;
            algorithm: string;
            similarity: string;
            searchResults: any[];
            latencyMs: number;
        }> = [];

        for (const target of targets) {
            console.log(`\n--- ${ALGORITHM_LABELS[target.algorithm]} / ${target.similarity} ---`);
            console.log(`Collection: ${target.collectionName}`);

            try {
                // Create collection (drops existing to ensure clean state)
                try {
                    await db.dropCollection(target.collectionName);
                } catch {
                    // Collection may not exist yet
                }
                const collection = await db.createCollection(target.collectionName);
                console.log('Created collection:', target.collectionName);

                // Insert data
                const insertSummary = await insertData(config, collection, data);
                console.log(`Inserted: ${insertSummary.inserted}/${insertSummary.total}`);

                // Create vector index
                const indexName = `vectorIndex_${target.algorithm}_${target.similarity.toLowerCase()}`;
                const indexOptions = getIndexOptions(
                    target.collectionName,
                    indexName,
                    embeddedField,
                    embeddingDimensions,
                    target.algorithm,
                    target.similarity
                );
                await db.command(indexOptions);
                console.log('Created vector index:', indexName);

                // Run vector search
                console.log('Executing vector search...');
                const startTime = Date.now();

                const pipeline = getSearchPipeline(queryEmbedding, embeddedField, 5, target.algorithm);
                const searchResults = await collection.aggregate(pipeline).toArray();

                const latencyMs = Date.now() - startTime;

                comparisonResults.push({
                    collectionName: target.collectionName,
                    algorithm: ALGORITHM_LABELS[target.algorithm],
                    similarity: target.similarity,
                    searchResults,
                    latencyMs,
                });

                console.log(`Success: ${searchResults.length} results, ${latencyMs}ms`);
            } catch (error) {
                console.error(`Error with ${target.collectionName}:`, (error as Error).message);
            }
        }

        // Print comparison table
        if (comparisonResults.length > 0) {
            printComparisonTable(comparisonResults);
        }
    } catch (error) {
        console.error('App failed:', error);
        process.exitCode = 1;
    } finally {
        console.log('\nClosing database connection...');
        if (dbClient) await dbClient.close();
        console.log('Database connection closed');
    }
}

main().catch(error => {
    console.error('Unhandled error:', error);
    process.exitCode = 1;
});
```

This script orchestrates the algorithm comparison by:

- Loading configuration from environment variables
- Initializing MongoDB and Azure OpenAI clients with passwordless authentication
- Loading hotel data with pre-calculated embeddings
- Testing each algorithm/similarity combination by creating a collection, inserting data, creating an index, and executing a search
- Measuring and comparing search performance across all configurations
- Displaying results in a comparison table

## Create utility functions

Create the `src/utils.ts` file with the following code:

```typescript
import { Collection, Document, MongoClient, OIDCResponse, OIDCCallbackParams } from 'mongodb';
import { AzureOpenAI } from 'openai/index.js';
import { promises as fs } from "fs";
import { AccessToken, DefaultAzureCredential, TokenCredential, getBearerTokenProvider } from '@azure/identity';

export type JsonData = Record<string, any>;

/**
 * Callback for MongoDB OIDC authentication using Azure Identity.
 *
 * DocumentDB requires OIDC tokens for passwordless authentication.
 * This callback fetches tokens from Azure AD using DefaultAzureCredential.
 */
export const AzureIdentityTokenCallback = async (
    params: OIDCCallbackParams,
    credential: TokenCredential
): Promise<OIDCResponse> => {
    const tokenResponse: AccessToken | null = await credential.getToken(
        ['https://ossrdbms-aad.database.windows.net/.default']
    );
    return {
        accessToken: tokenResponse?.token || '',
        expiresInSeconds: (tokenResponse?.expiresOnTimestamp || 0) - Math.floor(Date.now() / 1000)
    };
};

/**
 * Initialize MongoDB and Azure OpenAI clients using passwordless authentication.
 *
 * Uses DefaultAzureCredential which automatically tries multiple authentication methods:
 * 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
 * 2. Managed Identity (when running in Azure)
 * 3. Azure CLI (az login)
 * 4. Azure PowerShell
 * 5. Visual Studio Code
 */
export function getClientsPasswordless(): { aiClient: AzureOpenAI | null; dbClient: MongoClient | null } {
    let aiClient: AzureOpenAI | null = null;
    let dbClient: MongoClient | null = null;

    const endpoint = process.env.AZURE_OPENAI_EMBEDDING_ENDPOINT!;
    const deployment = process.env.AZURE_OPENAI_EMBEDDING_MODEL!;
    const clusterName = process.env.MONGO_CLUSTER_NAME!;

    if (!endpoint || !deployment || !clusterName) {
        throw new Error(
            'Missing required environment variables: AZURE_OPENAI_EMBEDDING_ENDPOINT, ' +
            'AZURE_OPENAI_EMBEDDING_MODEL, MONGO_CLUSTER_NAME'
        );
    }

    console.log(`Using Azure OpenAI Embedding Deployment/Model: ${deployment}`);

    const credential = new DefaultAzureCredential();

    // Azure OpenAI with DefaultAzureCredential
    {
        const scope = "https://cognitiveservices.azure.com/.default";
        const azureADTokenProvider = getBearerTokenProvider(credential, scope);
        aiClient = new AzureOpenAI({
            apiVersion: "2024-10-21",
            endpoint,
            deployment,
            azureADTokenProvider,
            timeout: 30000,
            maxRetries: 3,
        });
    }

    // DocumentDB with DefaultAzureCredential (OIDC)
    {
        dbClient = new MongoClient(
            `mongodb+srv://${clusterName}.mongocluster.cosmos.azure.com/`, {
            connectTimeoutMS: 120000,
            tls: true,
            retryWrites: false,
            maxIdleTimeMS: 120000,
            authMechanism: 'MONGODB-OIDC',
            authMechanismProperties: {
                OIDC_CALLBACK: (params: OIDCCallbackParams) =>
                    AzureIdentityTokenCallback(params, credential),
                ALLOWED_HOSTS: ['*.azure.com']
            }
        });
    }

    return { aiClient, dbClient };
}

/**
 * Read a JSON file and return the parsed data.
 */
export async function readFileReturnJson(filePath: string): Promise<JsonData[]> {
    console.log(`Reading JSON file from ${filePath}`);
    const fileAsString = await fs.readFile(filePath, "utf-8");
    return JSON.parse(fileAsString);
}

/**
 * Insert documents using insertMany in batches.
 *
 * DocumentDB has a 16 MB command payload limit. Batch size of 100 stays well
 * within this limit while keeping round-trip overhead reasonable.
 */
export async function insertData(
    config: { batchSize: number },
    collection: Collection,
    data: Document[]
) {
    console.log(`Processing in batches of ${config.batchSize}...`);
    const totalBatches = Math.ceil(data.length / config.batchSize);

    let inserted = 0;
    let failed = 0;

    for (let i = 0; i < totalBatches; i++) {
        const start = i * config.batchSize;
        const end = Math.min(start + config.batchSize, data.length);
        const batch = data.slice(start, end);

        try {
            const result = await collection.insertMany(batch, { ordered: false });
            inserted += result.insertedCount || 0;
            console.log(`Batch ${i + 1} complete: ${result.insertedCount} inserted`);
        } catch (error: any) {
            if (error?.writeErrors) {
                console.error(`Error in batch ${i + 1}: ${error?.writeErrors.length} failures`);
                failed += error?.writeErrors.length;
                inserted += batch.length - error?.writeErrors.length;
            } else {
                console.error(`Error in batch ${i + 1}:`, error);
                failed += batch.length;
            }
        }

        // Small pause between batches to reduce resource contention
        if (i < totalBatches - 1) {
            await new Promise(resolve => setTimeout(resolve, 100));
        }
    }

    return { total: data.length, inserted, failed };
}

/**
 * Print a side-by-side comparison table of vector search results across collections.
 */
export function printComparisonTable(
    results: Array<{
        collectionName: string;
        algorithm: string;
        similarity: string;
        searchResults: any[];
        latencyMs: number;
    }>
): void {
    console.log('\n' + '='.repeat(90));
    console.log('                    Vector Algorithm Comparison Results');
    console.log('='.repeat(90));

    // Header
    console.log(
        'Algorithm'.padEnd(12) +
        'Similarity'.padEnd(14) +
        'Top Result'.padEnd(24) +
        'Score'.padEnd(12) +
        'Latency(ms)'.padEnd(14)
    );
    console.log('-'.repeat(90));

    for (const r of results) {
        const topResult = r.searchResults[0];
        const topName = topResult
            ? (topResult.document.HotelName as string).substring(0, 22)
            : 'N/A';
        const topScore = topResult ? topResult.score.toFixed(4) : 'N/A';

        console.log(
            r.algorithm.padEnd(12) +
            r.similarity.padEnd(14) +
            topName.padEnd(24) +
            topScore.padEnd(12) +
            r.latencyMs.toFixed(0).padEnd(14)
        );
    }

    console.log('='.repeat(90));

    // Detailed results per collection
    for (const r of results) {
        console.log(`\n--- ${r.algorithm} / ${r.similarity} (${r.collectionName}) ---`);
        if (r.searchResults.length === 0) {
            console.log('  No results.');
            continue;
        }
        for (let i = 0; i < r.searchResults.length; i++) {
            const { document, score } = r.searchResults[i];
            console.log(`  ${i + 1}. ${document.HotelName}, Score: ${score.toFixed(4)}`);
        }
        console.log(`  Latency: ${r.latencyMs.toFixed(0)}ms`);
    }
}
```

The utilities provide essential functions for:

- Passwordless authentication to DocumentDB and Azure OpenAI using DefaultAzureCredential
- Reading JSON data files
- Batch insertion of documents with DocumentDB's 16 MB payload limit in mind
- Formatted display of comparison results showing algorithm performance

## Run the code

Execute the comparison script to test all 9 algorithm × similarity combinations:

```bash
npm run build
npm start
```

The output shows the comparison across all algorithms and similarity metrics:

```
======================================================================
  COMPARE ALL: 3 Algorithms × 3 Similarity Metrics (9 combinations)
======================================================================

Query: "luxury hotel near the beach"
Embedding generated (1536 dimensions)

Running searches (top 3 results)...
  ✓ vector_ivf_cos (created)
    ✗ vector_ivf_cos (dropped)
  ✓ vector_ivf_l2 (created)
    ✗ vector_ivf_l2 (dropped)
  ...

====================================================================================================
  COMPARISON RESULTS
====================================================================================================
Algorithm   Similarity  #1 Result               #1 Score    #2 Result               #2 Score    Diff
----------------------------------------------------------------------------------------------------
IVF         COS         Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
IVF         L2          Ocean Water Resort &..  0.8736      Windy Ocean Motel       0.9943      -0.1208
IVF         IP          Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
HNSW        COS         Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
HNSW        L2          Ocean Water Resort &..  0.8736      Windy Ocean Motel       0.9943      -0.1208
HNSW        IP          Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
DiskANN     COS         Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
DiskANN     L2          Ocean Water Resort &..  0.8736      Windy Ocean Motel       0.9943      -0.1208
DiskANN     IP          Ocean Water Resort &..  0.6184      Windy Ocean Motel       0.5056      0.1128
----------------------------------------------------------------------------------------------------

====================================================================================================
  KEY INSIGHTS
====================================================================================================
  🎯 Highest #1 score:   IVF/COS (0.6184)
  📊 Biggest separation: IVF/COS (diff: 0.1128)
  🔑 All algorithms return the same top results — algorithm choice
     affects performance at scale, not accuracy on small datasets.
  📐 COS and IP produce identical scores (normalized embeddings).
  📏 L2 scores are distances (lower = closer), not similarities.
====================================================================================================

Cleanup: dropped collection "hotels"
Database connection closed
```

### Test individual algorithms

Test a specific algorithm with cosine similarity:

```bash
# IVF (Inverted File Index)
npm run start:ivf

# HNSW (Hierarchical Navigable Small World)
npm run start:hnsw

# DiskANN
npm run start:diskann
```

### Understanding the results

The comparison table demonstrates key behaviors of vector search in DocumentDB:

- **All algorithms return identical results on small datasets.** With 50 documents, every algorithm finds the same matches because the dataset fits entirely in memory regardless of index structure. Algorithm selection becomes important at scale (millions of documents) where tradeoffs in latency, memory, and recall diverge.

- **COS and IP produce identical scores** (0.6184 / 0.5056) because the `text-embedding-3-small` model outputs normalized (unit-length) vectors. For normalized vectors, cosine similarity equals inner product mathematically.

- **L2 (Euclidean distance) scores are inverted.** Higher L2 scores mean *more* distance — the #1 result has the *lowest* score (0.8736 = closest to query). This explains the negative Diff value (-0.1208).

- **Score separation (Diff column)** shows confidence. A larger positive diff means the search clearly distinguishes the best match from the second-best. This metric helps evaluate result quality regardless of the absolute score values.

Algorithm selection guidelines for production:

| Algorithm | Best for | Tradeoff |
|-----------|----------|----------|
| **DiskANN** | Large datasets (millions+) | Stores index on disk, lower memory |
| **HNSW** | High-accuracy requirements | More memory, excellent recall |
| **IVF** | Very large datasets with limited memory | Faster build, possible recall reduction |

Similarity function selection:

| Function | Score meaning | Best for |
|----------|-------------|----------|
| **COS (Cosine)** | Higher = more similar (0–1) | Text embeddings (normalized vectors) |
| **L2 (Euclidean)** | Lower = more similar (distance) | When magnitude matters |
| **IP (Inner Product)** | Higher = more similar | Equivalent to COS for normalized vectors |

Tuning parameters affect the recall/latency tradeoff at both index build and query time:

| Algorithm | Build parameters | Search parameters |
|-----------|-----------------|-------------------|
| **DiskANN** | `maxDegree` (32), `lBuild` (50) | `lSearch` (100) |
| **HNSW** | `m` (16), `efConstruction` (64) | `efSearch` (80) |
| **IVF** | `numLists` (1) | `nProbes` (1) |

Higher build values improve index quality but slow creation. Higher search values improve recall but increase latency.

## Clean up resources

If you created an Azure DocumentDB cluster specifically for this quickstart, you can delete the resource group to remove all associated resources:

```azurecli
az group delete --name <resource-group-name>
```

This command deletes the resource group and all resources within it, including the DocumentDB cluster.

If you want to keep the cluster but remove the test data:

```typescript
import { MongoClient, OIDCCallbackParams } from 'mongodb';
import { DefaultAzureCredential } from '@azure/identity';
import { AzureIdentityTokenCallback } from './utils.js';

const clusterName = "<your-cluster-name>";
const credential = new DefaultAzureCredential();

const client = new MongoClient(
    `mongodb+srv://${clusterName}.mongocluster.cosmos.azure.com/`, {
    connectTimeoutMS: 120000,
    tls: true,
    retryWrites: false,
    authMechanism: 'MONGODB-OIDC',
    authMechanismProperties: {
        OIDC_CALLBACK: (params: OIDCCallbackParams) =>
            AzureIdentityTokenCallback(params, credential),
        ALLOWED_HOSTS: ['*.azure.com']
    }
});

await client.connect();
await client.db("Hotels").dropDatabase();
await client.close();
```

## Next steps

- [Vector search concepts in Azure DocumentDB](concept-vector-search)
- [How to use vector search in Azure DocumentDB](how-to-vector-search)
- [Optimize vector search performance](how-to-optimize-vector-search-performance)
- [MongoDB Node.js driver documentation](https://www.mongodb.com/docs/drivers/node/current/)
