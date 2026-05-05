---
title: "Quickstart - Vector Indexing with .NET"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with .NET."
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: csharp
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-dotnet
  - devx-track-data-ai
  - devx-track-dotnet-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with C#

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

This quickstart uses a sample hotel dataset in a JSON file with pre-calculated vectors from the `text-embedding-3-small` model. The dataset includes hotel names, locations, descriptions, and vector embeddings.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-dotnet) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Index Quickstart](includes/prerequisite-quickstart-vector-index.md)]

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later

    - [C# extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)

## Create data file with vectors

1. Create a new data directory for the hotels data file:

    ```bash
    mkdir data
    ```

1. Copy the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory.

## Set up the project

1. Create a new .NET console application:

    ```bash
    dotnet new console -n DocumentDBVectorQuickstart
    cd DocumentDBVectorQuickstart
    ```

1. Add required NuGet packages:

    ```bash
    dotnet add package MongoDB.Driver --version 2.21.0
    dotnet add package Azure.AI.OpenAI --version 1.0.0
    dotnet add package Azure.Identity --version 1.11.1
    ```

    - `MongoDB.Driver`: MongoDB driver for .NET
    - `Azure.AI.OpenAI`: Azure OpenAI client library
    - `Azure.Identity`: Azure Identity library for passwordless authentication

1. Update your `Program.cs` with basic structure:

    ```csharp
    using Azure.AI.OpenAI;
    using Azure.Identity;
    using MongoDB.Bson;
    using MongoDB.Driver;

    // Configuration
    var config = new VectorSearchConfig
    {
        ClusterName = Environment.GetEnvironmentVariable("MONGO_CLUSTER_NAME") ?? "vectorSearch",
        DatabaseName = "Hotels",
        EmbeddedField = Environment.GetEnvironmentVariable("EMBEDDED_FIELD") ?? "DescriptionVector",
        Dimensions = int.Parse(Environment.GetEnvironmentVariable("EMBEDDING_DIMENSIONS") ?? "1536"),
        OpenAIEndpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_EMBEDDING_ENDPOINT") ?? "",
        OpenAIModel = Environment.GetEnvironmentVariable("AZURE_OPENAI_EMBEDDING_MODEL") ?? "text-embedding-3-small",
    };

    // Create clients and run examples
    ```

## Create a vector index

### [IVF](#tab/tab-ivf)

IVF (Inverted File) is ideal for datasets with fewer than 10,000 documents. It partitions vectors into clusters for fast approximate search.

Create `VectorSearchService.cs`:

```csharp
using Azure.AI.OpenAI;
using MongoDB.Bson;
using MongoDB.Driver;

public class VectorSearchService
{
    private readonly MongoDbService _mongoService;
    private readonly AzureOpenAIClient _openAIClient;
    private readonly VectorSearchConfig _config;

    public VectorSearchService(MongoDbService mongoService, AzureOpenAIClient openAIClient, VectorSearchConfig config)
    {
        _mongoService = mongoService;
        _openAIClient = openAIClient;
        _config = config;
    }

    /// <summary>
    /// Creates an IVF (Inverted File) vector index optimized for small datasets
    /// </summary>
    public async Task CreateIVFIndexAsync(string collectionName, string indexName)
    {
        Console.WriteLine("Creating IVF vector index...");

        var searchOptions = new BsonDocument
        {
            ["kind"] = "vector-ivf",
            ["similarity"] = "COS",
            ["dimensions"] = _config.Dimensions,
            ["numLists"] = 10  // Number of clusters to partition vectors
        };

        await _mongoService.CreateVectorIndexAsync(
            _config.DatabaseName,
            collectionName,
            indexName,
            _config.EmbeddedField,
            searchOptions);

        Console.WriteLine("IVF vector index created successfully");
    }

    /// <summary>
    /// Creates an HNSW (Hierarchical Navigable Small World) vector index
    /// </summary>
    public async Task CreateHNSWIndexAsync(string collectionName, string indexName)
    {
        Console.WriteLine("Creating HNSW vector index...");

        var searchOptions = new BsonDocument
        {
            ["kind"] = "vector-hnsw",
            ["similarity"] = "COS",
            ["dimensions"] = _config.Dimensions,
            ["m"] = 16,  // Maximum connections per node (2-100, default 16)
            ["efConstruction"] = 64  // Candidate list size during construction (4-1000, default 64)
        };

        await _mongoService.CreateVectorIndexAsync(
            _config.DatabaseName,
            collectionName,
            indexName,
            _config.EmbeddedField,
            searchOptions);

        Console.WriteLine("HNSW vector index created successfully");
    }

    /// <summary>
    /// Creates a DiskANN vector index for very large datasets
    /// </summary>
    public async Task CreateDiskANNIndexAsync(string collectionName, string indexName)
    {
        Console.WriteLine("Creating DiskANN vector index...");

        var searchOptions = new BsonDocument
        {
            ["kind"] = "vector-diskann",
            ["similarity"] = "COS",
            ["dimensions"] = _config.Dimensions,
            ["maxDegree"] = 32,  // Maximum edges per node (20-2048, default 32)
            ["lBuild"] = 50  // Build parameter (10-500, default 50)
        };

        await _mongoService.CreateVectorIndexAsync(
            _config.DatabaseName,
            collectionName,
            indexName,
            _config.EmbeddedField,
            searchOptions);

        Console.WriteLine("DiskANN vector index created successfully");
    }

    /// <summary>
    /// Performs a vector similarity search using the $search aggregation
    /// </summary>
    public async Task<List<SearchResult>> PerformVectorSearchAsync(
        string collectionName,
        string queryText,
        int topK = 5)
    {
        Console.WriteLine($"Performing vector search for: '{queryText}'");

        // Generate embedding for query
        var embeddingClient = _openAIClient.GetEmbeddingClient(_config.OpenAIModel);
        var embeddingResponse = await embeddingClient.GenerateEmbeddingAsync(queryText);
        var embedding = embeddingResponse.Value.ToFloats().ToArray();

        // Get collection
        var collection = _mongoService.GetCollection(
            _config.DatabaseName,
            collectionName);

        // Build aggregation pipeline for vector search
        var pipeline = new[]
        {
            // Vector similarity search using cosmosSearch
            new BsonDocument("$search", new BsonDocument
            {
                ["cosmosSearch"] = new BsonDocument
                {
                    ["vector"] = new BsonArray(embedding.Select(f => new BsonDouble(f))),
                    ["path"] = _config.EmbeddedField,  // Field containing embeddings
                    ["k"] = topK  // Number of results to return
                }
            }),
            // Project results with similarity scores
            new BsonDocument("$project", new BsonDocument
            {
                ["score"] = new BsonDocument("$meta", "searchScore"),
                ["document"] = "$$ROOT"
            })
        };

        // Execute search
        var results = await collection.AggregateAsync<BsonDocument>(pipeline);
        var searchResults = new List<SearchResult>();

        await results.ForEachAsync(result =>
        {
            var searchResult = new SearchResult
            {
                HotelName = result["document"]["HotelName"].AsString,
                Score = result["score"].AsDouble
            };
            searchResults.Add(searchResult);
        });

        return searchResults;
    }
}

public class VectorSearchConfig
{
    public string ClusterName { get; set; }
    public string DatabaseName { get; set; }
    public string EmbeddedField { get; set; }
    public int Dimensions { get; set; }
    public string OpenAIEndpoint { get; set; }
    public string OpenAIModel { get; set; }
}

public class SearchResult
{
    public string HotelName { get; set; }
    public double Score { get; set; }
}
```

Create `MongoDbService.cs`:

```csharp
using Azure.Identity;
using MongoDB.Bson;
using MongoDB.Driver;

public class MongoDbService
{
    private readonly MongoClient _mongoClient;

    public MongoDbService(string clusterName)
    {
        var credential = new DefaultAzureCredential();
        
        // Create MongoDB client with OIDC authentication
        var mongoUri = $"mongodb+srv://{clusterName}.global.mongocluster.cosmos.azure.com/";
        
        var settings = MongoClientSettings.FromConnectionString(mongoUri);
        settings.ConnectTimeout = TimeSpan.FromSeconds(120);
        settings.UseTls = true;
        settings.RetryWrites = true;
        settings.Credential = MongoCredential.CreateOidcCredential("azure", null)
            .WithMechanismProperty("ENVIRONMENT", "azure");
        
        _mongoClient = new MongoClient(settings);
    }

    public IMongoCollection<BsonDocument> GetCollection(string databaseName, string collectionName)
    {
        return _mongoClient.GetDatabase(databaseName).GetCollection<BsonDocument>(collectionName);
    }

    public async Task CreateVectorIndexAsync(
        string databaseName,
        string collectionName,
        string indexName,
        string vectorField,
        BsonDocument searchOptions)
    {
        var database = _mongoClient.GetDatabase(databaseName);
        var collection = database.GetCollection<BsonDocument>(collectionName);

        // Create the index using the createIndexes command
        var indexCommand = new BsonDocument
        {
            ["createIndexes"] = collectionName,
            ["indexes"] = new BsonArray
            {
                new BsonDocument
                {
                    ["name"] = indexName,
                    ["key"] = new BsonDocument
                    {
                        [vectorField] = "cosmosSearch"
                    },
                    ["cosmosSearchOptions"] = searchOptions
                }
            }
        };

        try
        {
            var result = await database.RunCommandAsync<BsonDocument>(indexCommand);
            Console.WriteLine($"Vector index '{indexName}' created successfully");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error creating index: {ex.Message}");
            throw;
        }
    }
}
```

#### [HNSW](#tab/tab-hnsw)

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

The `CreateHNSWIndexAsync` method in `VectorSearchService.cs` (shown previously) creates an HNSW index with these key parameters in `cosmosSearchOptions`:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `kind` | `vector-hnsw` | HNSW algorithm |
| `m` | `16` | Maximum connections per node (2–100). Higher values improve recall but increase memory. |
| `efConstruction` | `64` | Candidate list size during construction (4–1000). Higher values improve accuracy at cost of build time. |
| `similarity` | `COS` | Cosine similarity for text embeddings |

**Cluster tier**: Requires M30 or higher due to memory overhead.

To create an HNSW index and run a search:

```csharp
await vectorSearchService.CreateHNSWIndexAsync("hotels_hnsw", "vectorIndex_hnsw");
var hnswResults = await vectorSearchService.PerformVectorSearchAsync(
    "hotels_hnsw",
    "quintessential lodging near running trails, eateries, retail",
    5);
```

#### [DiskANN](#tab/tab-diskann)

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

The `CreateDiskANNIndexAsync` method in `VectorSearchService.cs` (shown previously) creates a DiskANN index with these key parameters in `cosmosSearchOptions`:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `kind` | `vector-diskann` | DiskANN algorithm |
| `maxDegree` | `32` | Maximum edges per node (20–2048). Higher values improve accuracy but increase memory. |
| `lBuild` | `50` | Candidates evaluated during construction (10–500). Higher values improve quality. |
| `similarity` | `COS` | Cosine similarity for text embeddings |

**Cluster tier**: Requires M30 or higher.

To create a DiskANN index and run a search:

```csharp
await vectorSearchService.CreateDiskANNIndexAsync("hotels_diskann", "vectorIndex_diskann");
var diskannResults = await vectorSearchService.PerformVectorSearchAsync(
    "hotels_diskann",
    "quintessential lodging near running trails, eateries, retail",
    5);
```

---

## Query with vector search

All three algorithms use the same query pattern with the `$search` aggregation stage:

```csharp
// Generate embedding for query
var embeddingClient = openAIClient.GetEmbeddingClient(modelName);
var embeddingResponse = await embeddingClient.GenerateEmbeddingAsync("your query text");
var embedding = embeddingResponse.Value.ToFloats().ToArray();

// Build aggregation pipeline
var pipeline = new[]
{
    new BsonDocument("$search", new BsonDocument
    {
        ["cosmosSearch"] = new BsonDocument
        {
            ["vector"] = new BsonArray(embedding.Select(f => new BsonDouble(f))),
            ["path"] = vectorField,
            ["k"] = 5  // Return top 5 results
        }
    }),
    new BsonDocument("$project", new BsonDocument
    {
        ["score"] = new BsonDocument("$meta", "searchScore"),
        ["document"] = "$$ROOT"
    })
};

// Execute search
var results = await collection.AggregateAsync<BsonDocument>(pipeline);
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

The code uses your local developer authentication to access Azure DocumentDB and Azure OpenAI. The authentication relies on [DefaultAzureCredential](/dotnet/api/azure.identity.defaultazurecredential) from **Azure.Identity** to find your Azure credentials in the environment.

## Run the quickstart

```bash
# Run the application
dotnet run

# To run specific examples, modify Program.cs:
# await vectorSearchService.CreateIVFIndexAsync("hotels_ivf", "vectorIndex_ivf");
# await vectorSearchService.CreateHNSWIndexAsync("hotels_hnsw", "vectorIndex_hnsw");
# await vectorSearchService.CreateDiskANNIndexAsync("hotels_diskann", "vectorIndex_diskann");
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
- [MongoDB .NET driver documentation](https://www.mongodb.com/docs/drivers/csharp/)
