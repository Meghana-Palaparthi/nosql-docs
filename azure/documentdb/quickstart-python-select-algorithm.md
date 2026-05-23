---
title: Quickstart - Vector index with Python
description: Compare vector index algorithms and similarity functions using the Python SDK in Azure DocumentDB to optimize search performance for your workload.
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: python
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-python
  - devx-track-python-ai
  - devx-track-data-ai
  - msecd-doc-authoring-1012
# CustomerIntent: As a developer, I want to compare vector index algorithms in Python applications with Azure DocumentDB.
ms.service: azure-documentdb
---

# Quickstart: Vector index with Python in Azure DocumentDB

In this quickstart, you compare three vector index algorithms (DiskANN, HNSW, and IVF) and three similarity functions (cosine, L2, and inner product) to find the optimal configuration for your search workload. This quickstart uses a sample hotel dataset with precalculated embeddings from the `text-embedding-3-small` model.



Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-python) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

- [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd) (optional). Use `azd up` to deploy all required Azure resources in one command.

- [Python](https://www.python.org/downloads/) 3.9 or greater

## Create a Python project

1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p select-algorithm-python
   cd select-algorithm-python
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path select-algorithm-python
   Set-Location select-algorithm-python
   code .
   ```

   ---

2. In the terminal, create and activate a virtual environment:

   For Windows:

   ```powershell
   python -m venv venv
   venv\Scripts\activate
   ```

   For macOS/Linux:

   ```bash
   python -m venv venv
   source venv/bin/activate
   ```

3. Create a `requirements.txt` file with the following content:

   ```text
   pymongo>=4.7.0
   openai>=1.0.0,<2.0.0
   azure-identity>=1.15.0
   tabulate>=0.9.0
   ```

4. Install the required packages:

   ```console
   pip install -r requirements.txt
   ```

   The requirements include:
   - `pymongo`: MongoDB driver for Python (≥4.7 required for OIDC authentication).
   - `openai`: OpenAI client library to create vectors.
   - `azure-identity`: Azure Identity library for passwordless authentication.
   - `tabulate`: Formatted table output for comparison results.

 
5.  Create the `src` directory:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p src
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path src
   ```

   ---

## Create data file with vectors

1. Create a new data directory and download the hotels data file with vectors:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p data
   curl -o data/Hotels_Vector.json https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path data
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json" -OutFile "data/Hotels_Vector.json"
   ```

   ---

   Verify the file was downloaded:

   ### [Bash](#tab/bash)

   ```bash
   ls data/
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-ChildItem data/
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

For the passwordless authentication in this article, replace the placeholder values with your own information:

- `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
- `DOCUMENTDB_CLUSTER_NAME`: Your Azure DocumentDB cluster name

Prefer passwordless authentication, but it requires additional setup. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate Python apps to Azure services by using the Azure SDK for Python](/azure/developer/python/sdk/authentication/overview).

## Create code files

Create the following project structure:

```
select-algorithm-python/
├── data/
│   └── Hotels_Vector.json
├── src/
│   ├── compare_all.py
│   └── utils.py
└── requirements.txt
```

## Create the algorithm comparison code

Create the `src/compare_all.py` file with the following code:

:::code language="python" source="~/../documentdb-samples/ai/select-algorithm-python/src/compare_all.py" :::

This script orchestrates the algorithm comparison by:

- Loading configuration from environment variables.
- Initializing MongoDB and Azure OpenAI clients with passwordless authentication.
- Loading hotel data with precalculated embeddings.
- Testing each algorithm/similarity combination by creating a collection, inserting data, creating an index, and executing a search.
- Measuring and comparing search performance across all configurations.
- Displaying results in a comparison table.

## Create utility functions

Create the `src/utils.py` file with the following code:

:::code language="python" source="~/../documentdb-samples/ai/select-algorithm-python/src/utils.py" :::

The utilities provide essential functions for:

- Passwordless authentication to DocumentDB and Azure OpenAI using DefaultAzureCredential.
- Reading JSON data files with error handling.
- Batch insertion of documents with DocumentDB's 16 MB payload limit in mind.
- Formatted display of comparison results showing algorithm performance.

> [!NOTE]
> The Python sample configures the DocumentDB connection with `retryWrites=false`, which is required for DocumentDB vector search operations.

## Run the code

### Sign in to Azure

```azurecli
az login
```

Create the output directory, then execute the comparison script to run all 9 combinations:

### [Bash](#tab/bash)

```bash
mkdir -p output
python src/compare_all.py
```

### [PowerShell](#tab/powershell)

```powershell
New-Item -ItemType Directory -Force -Path output
python src/compare_all.py
```

---

Expected output:

:::code language="text" source="~/../documentdb-samples/ai/select-algorithm-python/output/compare_all.txt" :::

The **Diff** column shows the score gap between the top-1 and top-2 results. A smaller diff indicates the algorithm found results with more similar relevance scores.


### Understanding the results

The comparison table helps you choose the best configuration for your workload:

- **Algorithm**: The vector index type used for the query (IVF, HNSW, or DiskANN).
- **Metric**: The similarity metric used for scoring (COS, L2, or IP).
- **Top 1 Result and Top 2 Result**: The two highest-ranked hotels returned for each algorithm and metric combination.
- **Score**: The relevance score for each returned result.
- **Diff**: The score gap between the top two results.

[!INCLUDE[Choosing the right algorithm](includes/choosing-algorithm.md)]

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ServerSelectionTimeoutError` | Verify that your environment variables are set in the current shell. Ensure your IP is in the DocumentDB firewall rules. |
| `AuthenticationFailed` | Verify your Microsoft Entra token is valid. Run `az login` to refresh your credentials. |
| `pymongo.errors.OperationFailure` | Ensure the database and collection exist. Check that the vector index was created successfully. |
| `ModuleNotFoundError: No module named 'pymongo'` | Activate your virtual environment and run `pip install "pymongo>=4.7"`. |
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
