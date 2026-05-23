---
title: Quickstart - Vector index with Go
description: Compare DiskANN, HNSW, and IVF vector index algorithms using Go to select and tune the optimal index for your workload
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: golang
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-go
  - devx-track-go-ai
  - devx-track-data-ai
  - msecd-doc-authoring-1012
# CustomerIntent: As a developer, I want to compare vector index algorithms in Go applications with Azure DocumentDB.
ms.service: azure-documentdb
---

# Quickstart: Vector index with Go in Azure DocumentDB

This quickstart walks you through building a Go application that compares all three vector index algorithms (DiskANN, HNSW, and IVF) side by side with different similarity functions to help you choose the best configuration for your workload. The sample uses a hotels dataset with precalculated embeddings from the `text-embedding-3-small` model.



Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-go) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

- [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd) (optional). Use `azd up` to deploy all required Azure resources in one command.

- [Go](https://go.dev/doc/install) 1.24 or greater

## Create a Go project

1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir select-algorithm-go
   cd select-algorithm-go
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name select-algorithm-go
   Set-Location select-algorithm-go
   code .
   ```

   ---

2. Initialize a new Go module:

   ```console
   go mod init documentdb-vector-samples
   ```

   Verify the module was initialized:

   ### [Bash](#tab/bash)

   ```bash
   cat go.mod
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-Content go.mod
   ```

   ---

3. Install the required packages:

   ```console
   go get github.com/Azure/azure-sdk-for-go/sdk/azcore@v1.20.0
   go get github.com/Azure/azure-sdk-for-go/sdk/azidentity@v1.13.1
   go get github.com/openai/openai-go/v3@v3.12.0
   go get go.mongodb.org/mongo-driver@v1.17.6
   go mod tidy
   ```

   - `azcore`: Core Azure SDK functionality for Go.
   - `azidentity`: Azure Identity library for passwordless authentication with DefaultAzureCredential.
   - `openai-go/v3`: OpenAI client library with Azure support to generate embeddings.
   - `mongo-driver`: Official MongoDB driver for Go to work with DocumentDB.

   Verify the packages are installed:

   ### [Bash](#tab/bash)

   ```bash
   go list -m all | grep mongo
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   go list -m all | Select-String mongo
   ```

   ---

4. Create a `src` directory:

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

2. Download the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory:

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

For the passwordless authentication in this article, replace the placeholder values in your current shell session with your own information:

- `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
- `DOCUMENTDB_CLUSTER_NAME`: Your Azure DocumentDB cluster name

Prefer passwordless authentication. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate Go apps to Azure services by using the Azure SDK for Go](/azure/developer/go/azure-sdk-authentication).

## Create code files

Create the main application file:

### [Bash](#tab/bash)

```bash
touch src/main.go
```

### [PowerShell](#tab/powershell)

```powershell
New-Item -ItemType File -Path src/main.go
```

---

When you're done, the project structure should look like this:

```text
select-algorithm-go/
├── data/
│   └── Hotels_Vector.json
├── src/
│   ├── compare_all.go
│   ├── main.go
│   └── utils.go
└── go.mod
```

## Create the algorithm comparison code

Create the following source files in the `src` directory.

### src/main.go

:::code language="go" source="~/../documentdb-samples/ai/select-algorithm-go/src/main.go" :::

### src/compare_all.go

:::code language="go" source="~/../documentdb-samples/ai/select-algorithm-go/src/compare_all.go" :::

### src/utils.go

:::code language="go" source="~/../documentdb-samples/ai/select-algorithm-go/src/utils.go" :::

This code provides a vector algorithm comparison application with these key features:

- **Passwordless authentication**: Uses `DefaultAzureCredential` for both Azure OpenAI and DocumentDB via OIDC.
- **Three vector algorithms**: Implements DiskANN, HNSW, and IVF with algorithm-specific tuning parameters.
- **Three similarity functions**: Supports COS (cosine), L2 (Euclidean), and IP (inner product).
- **Single compare-all entry point**: Always runs all 9 algorithm × similarity combinations in one pass.
- **Index lifecycle automation**: Creates, queries, and drops each vector index in sequence.
- **Comparison output**: Generates a formatted table showing the top two results and score gap for each combination.
- **Production-ready patterns**: Includes batched insertion, error handling, and connection pooling.

> [!NOTE]
> The Go sample configures the DocumentDB connection with `retryWrites=false`, which is required for DocumentDB vector search operations.

## Run the code

After setting the environment variables in your shell session, run the application:

```console
go run ./src/
```

The application does the following:

1. Connect to Azure DocumentDB and Azure OpenAI using passwordless authentication
1. Load the hotel data and insert it into the `hotels` collection
1. Generate an embedding for the search query
1. Run all 9 vector index comparisons by creating, querying, and dropping each index in sequence
1. Display a comparison table with the top two results and score gap for each combination
1. Drop the `hotels` collection during cleanup

Expected output:

:::code language="text" source="~/../documentdb-samples/ai/select-algorithm-go/output/compare_all.txt" :::

The **Diff** column shows the score gap between the top-1 and top-2 results. A smaller diff indicates the algorithm found results with more similar relevance scores.

### Understanding the results

The comparison table shows how different algorithms perform on the same dataset with the same query:

- **Algorithm**: DiskANN, HNSW, or IVF
- **Metric**: The similarity metric (COS, L2, or IP)
- **Top 1 Result**: The highest-ranked hotel for that algorithm and metric
- **Score**: The relevance score for the corresponding result
- **Top 2 Result**: The second-highest-ranked hotel for that algorithm and metric
- **Diff**: The score gap between the top two results

[!INCLUDE[Choosing the right algorithm](includes/choosing-algorithm.md)]

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `server selection error` | Verify your environment variables are set correctly. Ensure your IP is in the DocumentDB firewall rules. |
| `authentication failed` | Verify your Microsoft Entra token is valid. Run `az login` to refresh your credentials. |
| `go: module not found` | Run `go mod tidy` to resolve dependencies. |
| Build errors | Ensure Go 1.24+ is installed. Run `go version` to check. |
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
