---
title: Quickstart - Vector index with TypeScript
description: Compare vector index algorithms and similarity functions using TypeScript in Azure DocumentDB to optimize search performance for your workload.
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: typescript
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-ts
  - devx-track-ts-ai
  - devx-track-data-ai
  - msecd-doc-authoring-1012
# CustomerIntent: As a developer, I want to compare vector index algorithms in TypeScript applications with Azure DocumentDB.
ms.service: azure-documentdb
---

# Quickstart: Vector index with TypeScript in Azure DocumentDB

In this quickstart, you compare three vector index algorithms (DiskANN, HNSW, and IVF) and three similarity functions (cosine, L2, and inner product) to find the optimal configuration for your search workload. This quickstart uses a sample hotel dataset with precalculated embeddings from the `text-embedding-3-small` model.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-typescript) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

- [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd) (optional). Use `azd up` to deploy all required Azure resources in one command.

- [Node.js LTS](https://nodejs.org/download/)

- [TypeScript](https://www.typescriptlang.org/download): Install TypeScript globally:

    ```console
    npm install -g typescript
    ```


## Create the Node.js project

You should end up with the following project structure:

```
select-algorithm-typescript/
├── data/
│   └── Hotels_Vector.json
├── src/
│   ├── compare-all.ts
│   └── utils.ts
├── package.json
└── tsconfig.json
```


1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir select-algorithm-typescript
   cd select-algorithm-typescript
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name select-algorithm-typescript
   Set-Location select-algorithm-typescript
   code .
   ```

   ---

2. Initialize a TypeScript Node.js project:

   ```console
   npm init -y
   npm pkg set type="module"
   ```

   Verify the project was initialized:

   ### [Bash](#tab/bash)

   ```bash
   ls package.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-ChildItem package.json
   ```

   ---

3. Install the required packages:

   ```console
   npm install mongodb openai @azure/identity
   npm install --save-dev typescript @types/node
   ```

   - `mongodb`: MongoDB driver for Node.js.
   - `openai`: OpenAI client library to create vectors.
   - `@azure/identity`: Azure Identity library for passwordless authentication.
   - `typescript`: TypeScript compiler.

   Verify that `npm list` shows all installed packages without errors.

4. Create a `tsconfig.json` file in the project root:

   ```json
   {
     "compilerOptions": {
       "target": "ES2022",
       "module": "Node16",
       "moduleResolution": "Node16",
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true,
       "resolveJsonModule": true,
       "strict": true,
       "rootDir": "./src",
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
       "start": "node dist/compare-all.js"
     }
   }
   ```

6. Create the `src` directory:

   ### [Bash](#tab/bash)

   ```bash
   mkdir src
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name src
   ```

   ---

## Create data file with vectors

1. Create a new data directory for the hotels data file:

   ### [Bash](#tab/bash)

   ```bash
   mkdir data
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name data
   ```

   ---

2. Download the `Hotels_Vector.json` data file with vectors to your `data` directory:

   ### [Bash](#tab/bash)

   ```bash
   curl -o data/Hotels_Vector.json https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json" -OutFile "data/Hotels_Vector.json"
   ```

   ---

   Verify the file was downloaded:

   ### [Bash](#tab/bash)

   ```bash
   ls data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-ChildItem data\Hotels_Vector.json
   ```

   ---

   You should see `Hotels_Vector.json` in the `data` directory.

## Configure environment variables

Set the required environment variables in your current shell session before you run the sample:

### [Bash](#tab/bash)

```bash
export DOCUMENTDB_CLUSTER_NAME=<your-cluster-name>
export AZURE_OPENAI_EMBEDDING_ENDPOINT=https://<your-resource>.openai.azure.com
export AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
export AZURE_DOCUMENTDB_DATABASENAME=Hotels
export DATA_FILE_WITH_VECTORS=data/Hotels_Vector.json
export EMBEDDED_FIELD=DescriptionVector
export EMBEDDING_DIMENSIONS=1536
```

### [PowerShell](#tab/powershell)

```powershell
$env:DOCUMENTDB_CLUSTER_NAME = "<your-cluster-name>"
$env:AZURE_OPENAI_EMBEDDING_ENDPOINT = "https://<your-resource>.openai.azure.com"
$env:AZURE_OPENAI_EMBEDDING_MODEL = "text-embedding-3-small"
$env:AZURE_DOCUMENTDB_DATABASENAME = "Hotels"
$env:DATA_FILE_WITH_VECTORS = "data/Hotels_Vector.json"
$env:EMBEDDED_FIELD = "DescriptionVector"
$env:EMBEDDING_DIMENSIONS = "1536"
```

---

Replace the placeholder values with your own information:

- `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
- `DOCUMENTDB_CLUSTER_NAME`: Your Azure DocumentDB cluster name

Prefer passwordless authentication, but it requires additional setup. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate JavaScript apps to Azure services using the Azure SDK for JavaScript](/azure/developer/javascript/sdk/authentication/overview).

## Create the algorithm comparison code

Create the `src/compare-all.ts` file with the following code:

:::code language="typescript" source="~/../documentdb-samples/ai/select-algorithm-typescript/src/compare-all.ts" :::

This script orchestrates the algorithm comparison by:

- Loading configuration from environment variables.
- Initializing MongoDB and Azure OpenAI clients with passwordless authentication.
- Loading hotel data with precalculated embeddings.
- Testing each algorithm/similarity combination by creating a collection, inserting data, creating an index, and executing a search.
- Measuring and comparing search performance across all configurations.
- Displaying results in a comparison table.

## Create utility functions

Create the `src/utils.ts` file with the following code:

:::code language="typescript" source="~/../documentdb-samples/ai/select-algorithm-typescript/src/utils.ts" :::

The utilities provide essential functions for:

- Passwordless authentication to DocumentDB and Azure OpenAI using DefaultAzureCredential.
- Reading JSON data files.
- Batch insertion of documents with DocumentDB's 16 MB payload limit in mind.
- Formatted display of comparison results showing algorithm performance.

> [!NOTE]
> The Node.js sample configures the DocumentDB connection with `retryWrites=false`, which is required for DocumentDB vector search operations.

## Run the code

1. Sign in with Azure CLI for passwordless authentication:

   ```azurecli
   az login
   ```

2. Execute the comparison script to test all 9 algorithm × similarity combinations:

   ```console
   npm run build
   npm start
   ```

Expected output:

:::code language="text" source="~/../documentdb-samples/ai/select-algorithm-typescript/output/compare_all.txt" :::

The **Diff** column shows the score gap between the top-1 and top-2 results. A smaller diff indicates the algorithm found results with more similar relevance scores.

### Understanding the results

The comparison table demonstrates key behaviors of vector search in DocumentDB:

- **All algorithms return identical results on small datasets.** With 50 documents, every algorithm finds the same matches because the dataset fits entirely in memory regardless of index structure. Algorithm selection becomes important at scale (millions of documents) where tradeoffs in latency, memory, and recall diverge.

- **COS and IP produce identical scores** (0.6184 / 0.5056) because the `text-embedding-3-small` model outputs normalized (unit-length) vectors. For normalized vectors, cosine similarity equals inner product mathematically.

- **L2 (Euclidean distance) scores represent distance.** In this output, the top result has the lower L2 score (0.8736) and the second result is farther away (0.9943).

- **Score separation (Diff column)** shows the gap between the top two results. A smaller diff indicates the algorithm found results with more similar relevance scores.

[!INCLUDE[Choosing the right algorithm](includes/choosing-algorithm.md)]

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `MongoServerSelectionError` | Verify your `DOCUMENTDB_CLUSTER_NAME` environment variable and ensure your IP is in the DocumentDB firewall rules. |
| `MongoServerError: Authentication failed` | Verify your Microsoft Entra token is valid. Run `az login` to refresh your credentials. |
| TypeScript compilation errors | Run `npx tsc --version` to verify TypeScript is installed. Check `tsconfig.json` settings match the values shown in this article. |
| `Cannot find module` errors | Run `npm install` to ensure all dependencies are installed. |
| `Embedding dimension mismatch` | Verify the `AZURE_OPENAI_EMBEDDING_MODEL` environment variable matches the model deployed in your Azure OpenAI resource. |
| Empty search results | The vector index may take a few minutes to build. Wait 2-3 minutes after index creation, then rerun the script. |

## Clean up resources

Remove the database using the DocumentDB for VS Code extension:

1. Install the [DocumentDB for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) extension.
1. Connect to your Azure DocumentDB cluster.
1. Expand the cluster, right-click the **Hotels** database, and select **Drop Database**.

## Related content

- [Vector search overview](./vector-search.md)
- [ENN vector search](./enn-vector-search.md)
- [Product quantization](./product-quantization.md)

