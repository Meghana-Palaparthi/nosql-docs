---
title: Quickstart - Vector index with .NET
description: Compare DiskANN, HNSW, and IVF vector search algorithms in Azure DocumentDB using the .NET client library with passwordless authentication.
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: csharp
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-dotnet
  - devx-track-dotnet-ai
  - devx-track-data-ai
  - msecd-doc-authoring-1012
# CustomerIntent: As a developer, I want to compare vector index algorithms in .NET applications with Azure DocumentDB.
ms.service: azure-documentdb
---

# Quickstart: Vector index with .NET in Azure DocumentDB

This article explains how to compare all three vector search algorithms (DiskANN, HNSW, and IVF) in Azure DocumentDB using the .NET client library. The sample demonstrates how each algorithm performs with different similarity functions (COS, L2, IP) and helps you choose the right configuration for your workload. This quickstart uses a sample hotel dataset in a JSON file with precalculated vectors from the `text-embedding-3-small` model.



Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-dotnet) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

- [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd) (optional). Use `azd up` to deploy all required Azure resources in one command.

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later.

## Create a .NET project

1. Create a new directory for your project and initialize the .NET console application:

   ### [Bash](#tab/bash)

   ```bash
   mkdir select-algorithm-dotnet
   cd select-algorithm-dotnet
   dotnet new console --framework net8.0 --name SelectAlgorithm --output .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name select-algorithm-dotnet
   Set-Location select-algorithm-dotnet
   dotnet new console --framework net8.0 --name SelectAlgorithm --output .
   ```

   ---

   Verify the project was created:

   ### [Bash](#tab/bash)

   ```bash
   ls SelectAlgorithm.csproj
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-ChildItem SelectAlgorithm.csproj
   ```

   ---

2. Install the required NuGet packages:

   ```console
   dotnet add package Azure.AI.OpenAI --version 2.1.0
   dotnet add package Azure.Identity --version 1.13.2
   dotnet add package MongoDB.Driver --version 3.2.0
   dotnet add package Microsoft.Extensions.Configuration --version 8.0.0
   dotnet add package Microsoft.Extensions.Configuration.Binder --version 8.0.2
   dotnet add package Microsoft.Extensions.Configuration.EnvironmentVariables --version 8.0.0
   dotnet add package Microsoft.Extensions.Configuration.Json --version 8.0.1
   ```

   These packages provide:
   - `Azure.AI.OpenAI`: Azure OpenAI client library to create vector embeddings.
   - `Azure.Identity`: Azure Identity library for passwordless authentication with DefaultAzureCredential.
   - `MongoDB.Driver`: MongoDB driver for .NET to interact with DocumentDB.
   - `Microsoft.Extensions.Configuration*`: Configuration and environment variable binding infrastructure.

   Verify installed packages:

   ```console
   dotnet list package
   ```

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

   Verify the file downloaded successfully:

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

## Configure appsettings.json and environment variable overrides

> [!NOTE]
> .NET uses the standard `IConfiguration` system with `appsettings.json` as the primary configuration source. Environment variables can override any setting using double-underscore (`__`) as the hierarchy separator. The other language quickstarts use flat environment variables (`DOCUMENTDB_CLUSTER_NAME`), but .NET's hierarchical configuration is the idiomatic pattern for this platform.

1. Create an `appsettings.json` configuration file:

   ### [Bash](#tab/bash)

   ```bash
   touch appsettings.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType File -Name appsettings.json
   ```

   ---

2. Add this content to `appsettings.json`:

   ```json
   {
     "DocumentDB": {
       "DatabaseName": "Hotels",
       "ClusterName": "<your-cluster-name>",
       "LoadBatchSize": 100
     },
     "VectorSearch": {
       "Similarity": "",
       "TopK": 5,
       "Query": "luxury hotel near the beach"
     },
     "AzureOpenAI": {
       "Endpoint": "https://<your-resource>.openai.azure.com/",
       "EmbeddingModel": "text-embedding-3-small"
     },
     "DataFiles": {
       "WithVectors": "data/Hotels_Vector.json"
     },
     "Embedding": {
       "EmbeddedField": "DescriptionVector",
       "Dimensions": 1536
     }
   }
   ```

3. Set any environment variable overrides in your current shell session. The sample uses `DefaultAzureCredential` for passwordless authentication, and .NET maps environment variables to `appsettings.json` keys with the `Section__Key` format:

   ### [Bash](#tab/bash)

   ```bash
   export AzureOpenAI__Endpoint="https://<your-resource>.openai.azure.com/"
   export AzureOpenAI__EmbeddingModel="text-embedding-3-small"
   export DocumentDB__ClusterName="<your-cluster-name>"
   export DocumentDB__DatabaseName="Hotels"
   export DataFiles__WithVectors="data/Hotels_Vector.json"
   export Embedding__EmbeddedField="DescriptionVector"
   export Embedding__Dimensions="1536"
   export AZURE_TENANT_ID="<your-tenant-id>"
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   $env:AzureOpenAI__Endpoint="https://<your-resource>.openai.azure.com/"
   $env:AzureOpenAI__EmbeddingModel="text-embedding-3-small"
   $env:DocumentDB__ClusterName="<your-cluster-name>"
   $env:DocumentDB__DatabaseName="Hotels"
   $env:DataFiles__WithVectors="data/Hotels_Vector.json"
   $env:Embedding__EmbeddedField="DescriptionVector"
   $env:Embedding__Dimensions="1536"
   $env:AZURE_TENANT_ID="<your-tenant-id>"
   ```

   ---

Replace the placeholder values with your own information:
- `<your-resource>`: Your Azure OpenAI resource name
- `<your-cluster-name>`: Your Azure DocumentDB cluster name
- `<your-tenant-id>`: Your Microsoft Entra tenant ID

These environment variables override the matching values in `appsettings.json`. For example, `DocumentDB__ClusterName` overrides `DocumentDB:ClusterName`, `DocumentDB__DatabaseName` overrides `DocumentDB:DatabaseName`, and `AzureOpenAI__Endpoint` overrides `AzureOpenAI:Endpoint`.

Prefer passwordless authentication. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate .NET apps to Azure services by using the Azure SDK for .NET](/dotnet/azure/sdk/authentication).

## Create code files

Continue the project by creating code files for vector search comparison. When you're done, the project structure should look like this:

```
select-algorithm-dotnet/
├── data/
│   └── Hotels_Vector.json
├── Models/
│   ├── Configuration.cs
│   └── HotelData.cs
├── Utilities/
│   └── AzureIdentityTokenHandler.cs
├── appsettings.json
├── CompareAll.cs
├── Program.cs
├── SelectAlgorithm.csproj
└── Utils.cs
```

1. Create the directory structure:

   ### [Bash](#tab/bash)

   ```bash
   mkdir Models
   mkdir Utilities
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name Models
   New-Item -ItemType Directory -Name Utilities
   ```

   ---

2. Create the code files:

   ### [Bash](#tab/bash)

   ```bash
   touch CompareAll.cs
   touch Utils.cs
   touch Models/Configuration.cs
   touch Models/HotelData.cs
   touch Utilities/AzureIdentityTokenHandler.cs
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType File -Name CompareAll.cs
   New-Item -ItemType File -Name Utils.cs
   New-Item -ItemType File -Path Models\Configuration.cs
   New-Item -ItemType File -Path Models\HotelData.cs
   New-Item -ItemType File -Path Utilities\AzureIdentityTokenHandler.cs
   ```

   ---

## Create the algorithm comparison code

Create the following source files to implement the vector search comparison.

### Program.cs

Replace the contents of `Program.cs` with this code:

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/Program.cs" :::

This main entry point:
- Loads configuration from appsettings.json and environment variables.
- Sets up dependency injection with logging infrastructure.
- Initializes Azure OpenAI and DocumentDB clients using passwordless authentication.
- Calls `CompareAll.Run()` to execute the flat project entry point.
- Runs the comparison and prints results in a table format.

### CompareAll.cs

Add this code to `CompareAll.cs`:

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/CompareAll.cs" :::

This service:
- Manages the comparison workflow for all algorithms
- Creates collections and indexes for each algorithm/similarity combination
- Inserts data and executes vector searches
- Measures and collects latency metrics
- Configures algorithm-specific parameters for index creation and search

### Supporting files

Create the following supporting files in the project:

#### Utils.cs

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/Utils.cs" :::

#### Utilities/AzureIdentityTokenHandler.cs

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/Utilities/AzureIdentityTokenHandler.cs" :::

#### Models/Configuration.cs

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/Models/Configuration.cs" :::

#### Models/HotelData.cs

:::code language="csharp" source="~/../documentdb-samples/ai/select-algorithm-dotnet/Models/HotelData.cs" :::

These supporting files provide:
- Passwordless authentication setup for Azure OpenAI and DocumentDB.
- OIDC token handler for automatic token refresh.
- JSON file reading and deserialization.
- Batch data insertion with error handling.
- Results formatting and display.

> [!NOTE]
> The .NET sample configures the DocumentDB connection with `retryWrites=false`, which is required for DocumentDB vector search operations.

## Run the code

1. Sign in with Azure CLI for passwordless authentication:

   ```azurecli
   az login
   ```

2. Build the project:

   ```console
   dotnet build
   ```

3. Create the output directory:

   ### [Bash](#tab/bash)

   ```bash
   mkdir output
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path output
   ```

   ---

4. Run the flat `SelectAlgorithm.csproj` entry point to compare all 9 algorithm × similarity combinations:

   ```console
   dotnet run
   ```

   The application loads the sample data once, then creates and tests all 9 algorithm × similarity combinations sequentially.

### Expected output

The application displays progress logs and a comparison table:

:::code language="text" source="~/../documentdb-samples/ai/select-algorithm-dotnet/output/compare_all.txt" :::

The **Diff** column shows the score gap between the top-1 and top-2 results. A smaller diff indicates the algorithm found results with more similar relevance scores.


[!INCLUDE[Choosing the right algorithm](includes/choosing-algorithm.md)]

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `TimeoutException` during connection | Verify your environment variables are set correctly. Ensure your IP is in the DocumentDB firewall rules. |
| `AuthenticationException` | Verify your Microsoft Entra token is valid. Run `az login` to refresh your credentials. |
| Build errors with .NET version | Ensure you have .NET 8.0 or later installed. Run `dotnet --version` to check. |
| `BsonSerializationException` | Ensure your model classes match the document structure in the collection. |
| Empty search results | The vector index may take a few minutes to build. Wait 2-3 minutes after index creation, then rerun the script. |
| `IndexOptionsConflict` (code 85) | DocumentDB doesn't allow multiple vector indexes of the same kind on the same field. Drop the existing index before creating a new one. |

## Clean up resources

Remove the database using the DocumentDB for VS Code extension:

1. Install the [DocumentDB for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) extension.
1. Connect to your Azure DocumentDB cluster.
1. Expand the cluster, right-click the **Hotels** database, and select **Drop Database**.

## Related content

- [Vector search overview](./vector-search.md)
- [ENN vector search](./enn-vector-search.md)
- [Product quantization](./product-quantization.md)
