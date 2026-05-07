---
title: Choose and configure vector indexes in Azure DocumentDB with .NET
description: Compare DiskANN, HNSW, and IVF vector search algorithms in Azure DocumentDB using the .NET client library with passwordless authentication.
ms.topic: quickstart
ms.date: 2025-01-28
author: diberry
ms.author: diberry
ms.service: azure-documentdb
ms.subservice: vector-search
---

# Quickstart: Choose and configure vector indexes in Azure DocumentDB with .NET

This article shows you how to compare all three vector search algorithms (DiskANN, HNSW, and IVF) in Azure DocumentDB using the .NET client library. The sample demonstrates how each algorithm performs with different similarity functions (COS, L2, IP) and helps you choose the right configuration for your workload. This quickstart uses a sample hotel dataset in a JSON file with pre-calculated vectors from the `text-embedding-3-small` model.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-dotnet) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites](includes/prerequisite-quickstart-vector-index.md)]

- [.NET 9.0 SDK](https://dotnet.microsoft.com/download/dotnet/9.0) or later. .NET 9.0 is a Standard Term Support (STS) release. Use the latest available .NET SDK for long-term production workloads.

## Create data file with vectors

1. Create a new data directory for the hotels data file:

   **Bash:**

   ```bash
   mkdir data
   ```

   **PowerShell:**

   ```powershell
   New-Item -ItemType Directory -Name data
   ```

2. Download the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory:

   **Bash:**

   ```bash
   curl -o data/Hotels_Vector.json https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json
   ```

   **PowerShell:**

   ```powershell
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json" -OutFile "data/Hotels_Vector.json"
   ```

   Verify the file downloaded successfully:

   ```bash
   ls data/Hotels_Vector.json
   ```

## Create a .NET project

1. Create a new directory for your project and initialize the .NET console application:

   ```bash
   mkdir select-algorithm-dotnet
   cd select-algorithm-dotnet
   dotnet new console --framework net9.0
   ```

   Verify the project was created:

   ```bash
   ls *.csproj
   ```

2. Install the required NuGet packages:

   ```bash
   dotnet add package Azure.AI.OpenAI --version 2.1.0
   dotnet add package Azure.Identity --version 1.17.1
   dotnet add package MongoDB.Driver --version 3.0.0
   dotnet add package Microsoft.Extensions.Configuration --version 9.0.0
   dotnet add package Microsoft.Extensions.Configuration.Binder --version 9.0.0
   dotnet add package Microsoft.Extensions.Configuration.EnvironmentVariables --version 9.0.0
   dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.0
   dotnet add package Microsoft.Extensions.DependencyInjection --version 9.0.0
   dotnet add package Microsoft.Extensions.Logging --version 9.0.0
   dotnet add package Microsoft.Extensions.Logging.Console --version 9.0.0
   ```

   These packages provide:
   - `Azure.AI.OpenAI`: Azure OpenAI client library to create vector embeddings
   - `Azure.Identity`: Azure Identity library for passwordless authentication with DefaultAzureCredential
   - `MongoDB.Driver`: MongoDB driver for .NET to interact with DocumentDB
   - `Microsoft.Extensions.*`: Configuration, dependency injection, and logging infrastructure

   Verify installed packages:

   ```bash
   dotnet list package
   ```

3. Create environment variables for authentication. The sample uses DefaultAzureCredential for passwordless authentication:

   ```bash
   # Set environment variables for passwordless authentication
   export AZURE_OPENAI_EMBEDDING_ENDPOINT="https://<your-openai-resource>.openai.azure.com"
   export AZURE_OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
   export MONGO_CLUSTER_NAME="<your-documentdb-cluster-name>"
   export AZURE_TENANT_ID="<your-tenant-id>"
   export DATA_FILE_WITH_VECTORS="../../data/Hotels_Vector.json"
   ```

   For Windows PowerShell:

   ```powershell
   $env:AZURE_OPENAI_EMBEDDING_ENDPOINT="https://<your-openai-resource>.openai.azure.com"
   $env:AZURE_OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
   $env:MONGO_CLUSTER_NAME="<your-documentdb-cluster-name>"
   $env:AZURE_TENANT_ID="<your-tenant-id>"
   $env:DATA_FILE_WITH_VECTORS="../../data/Hotels_Vector.json"
   ```

   Replace the placeholder values with your own information:
   - `<your-openai-resource>`: Your Azure OpenAI resource name
   - `<your-documentdb-cluster-name>`: Your Azure DocumentDB cluster name
   - `<your-tenant-id>`: Your Microsoft Entra tenant ID

   You should always prefer passwordless authentication. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate .NET apps to Azure services by using the Azure SDK for .NET](/dotnet/azure/sdk/authentication).

4. Sign in with Azure CLI for passwordless authentication:

   ```bash
   az login
   ```

5. Create an `appsettings.json` configuration file:

   ```bash
   touch appsettings.json
   ```

   Add this content to `appsettings.json`:

   ```json
   {
     "DatabaseName": "Hotels",
     "EmbeddedField": "DescriptionVector",
     "EmbeddingDimensions": 1536,
     "LoadBatchSize": 100,
     "SearchQuery": "quintessential lodging near running trails, eateries, retail",
     "TopK": 5
   }
   ```

## Create code files for vector comparison

Continue the project by creating code files for vector search comparison. When you are done, the project structure should look like this:

```
├── data/
│   └── Hotels_Vector.json            # Hotel data with vector embeddings
└── select-algorithm-dotnet/
    ├── Services/
    │   └── VectorComparisonService.cs # Service to compare vector algorithms
    ├── Utilities/
    │   └── Utils.cs                   # Shared utility functions
    ├── Program.cs                     # Main application entry point
    ├── appsettings.json               # Configuration settings
    ├── global.json                    # .NET SDK version specification
    └── SelectAlgorithm.csproj         # Project file
```

1. Create the directory structure:

   ```bash
   mkdir Services
   mkdir Utilities
   ```

2. Create the code files:

   ```bash
   touch Services/VectorComparisonService.cs
   touch Utilities/Utils.cs
   touch global.json
   ```

## Create code for vector comparison

### Program.cs

Replace the contents of `Program.cs` with this code:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using SelectAlgorithm.Services;
using SelectAlgorithm.Utilities;
using System.Reflection;

namespace SelectAlgorithm;

class Program
{
    static async Task Main(string[] args)
    {
        // Build configuration from appsettings.json and environment variables
        // Environment variables override appsettings.json values
        var configuration = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            .AddEnvironmentVariables()
            .Build();

        // Set up dependency injection container with logging and configuration
        var services = new ServiceCollection()
            .AddLogging(builder => builder
                .AddConsole()
                .SetMinimumLevel(LogLevel.Information))
            .AddSingleton<IConfiguration>(configuration);

        var serviceProvider = services.BuildServiceProvider();
        var logger = serviceProvider.GetRequiredService<ILogger<Program>>();

        try
        {
            // Get required environment variables for Azure services
            // These are required for passwordless authentication with DefaultAzureCredential
            var openAiEndpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_EMBEDDING_ENDPOINT")
                ?? throw new InvalidOperationException("AZURE_OPENAI_EMBEDDING_ENDPOINT not set");
            var openAiModel = Environment.GetEnvironmentVariable("AZURE_OPENAI_EMBEDDING_MODEL")
                ?? throw new InvalidOperationException("AZURE_OPENAI_EMBEDDING_MODEL not set");
            var mongoClusterName = Environment.GetEnvironmentVariable("MONGO_CLUSTER_NAME")
                ?? throw new InvalidOperationException("MONGO_CLUSTER_NAME not set");
            var tenantId = Environment.GetEnvironmentVariable("AZURE_TENANT_ID")
                ?? throw new InvalidOperationException("AZURE_TENANT_ID not set");

            // Get configuration values with defaults from appsettings.json
            var databaseName = configuration["DatabaseName"] ?? "Hotels";
            var embeddedField = configuration["EmbeddedField"] ?? "DescriptionVector";
            if (!int.TryParse(configuration["EmbeddingDimensions"] ?? "1536", out int embeddingDimensions))
                throw new InvalidOperationException("EmbeddingDimensions must be a valid integer");
            if (!int.TryParse(configuration["LoadBatchSize"] ?? "100", out int loadBatchSize))
                throw new InvalidOperationException("LoadBatchSize must be a valid integer");
            var searchQuery = configuration["SearchQuery"] ?? "quintessential lodging near running trails, eateries, retail";
            if (!int.TryParse(configuration["TopK"] ?? "5", out int topK))
                throw new InvalidOperationException("TopK must be a valid integer");

            // Resolve data file path relative to the build output directory
            var dataFileRelative = Environment.GetEnvironmentVariable("DATA_FILE_WITH_VECTORS") ?? "../../data/Hotels_Vector.json";
            var assemblyDir = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location) ?? string.Empty;
            var dataFilePath = Path.GetFullPath(Path.Combine(assemblyDir, dataFileRelative));

            // Control which algorithms and similarity functions to test
            // ALGORITHM options: all, diskann, hnsw, ivf
            // SIMILARITY options: all, COS, L2, IP
            var algorithmFilter = (Environment.GetEnvironmentVariable("ALGORITHM") ?? "all").Trim().ToLower();
            var similarityFilter = (Environment.GetEnvironmentVariable("SIMILARITY") ?? "COS").Trim().ToUpper();

            // Initialize Azure OpenAI and DocumentDB clients with passwordless authentication
            // Uses DefaultAzureCredential which tries multiple authentication methods
            logger.LogInformation("Initializing clients with passwordless authentication...");
            var (aiClient, dbClient) = Utils.GetClientsPasswordless(openAiEndpoint, mongoClusterName, tenantId);

            // Create the comparison service to test vector algorithms
            var comparisonLogger = serviceProvider.GetRequiredService<ILoggerFactory>()
                .CreateLogger<VectorComparisonService>();

            var comparisonService = new VectorComparisonService(
                comparisonLogger,
                aiClient,
                dbClient,
                databaseName,
                embeddedField,
                embeddingDimensions,
                loadBatchSize,
                topK
            );

            // Run the comparison across all selected algorithms and similarity functions
            // This creates collections, indexes, inserts data, and performs searches
            var results = await comparisonService.RunComparisonAsync(
                dataFilePath,
                searchQuery,
                openAiModel,
                algorithmFilter,
                similarityFilter
            );

            // Print a comparison table showing algorithm performance
            if (results.Count > 0)
            {
                Utils.PrintComparisonTable(results);
            }

            logger.LogInformation("\nClosing database connection...");
            // MongoClient does not implement IAsyncDisposable; sync disposal is intentional
            dbClient?.Cluster?.Dispose();
            logger.LogInformation("Database connection closed");
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Application failed");
            Environment.ExitCode = 1;
        }
    }
}
```

This main entry point:
- Loads configuration from appsettings.json and environment variables
- Sets up dependency injection with logging infrastructure
- Initializes Azure OpenAI and DocumentDB clients using passwordless authentication
- Creates a VectorComparisonService to test all algorithms
- Runs the comparison and prints results in a table format

### Services/VectorComparisonService.cs

Add this code to `Services/VectorComparisonService.cs`:

```csharp
using Azure.AI.OpenAI;
using Microsoft.Extensions.Logging;
using MongoDB.Driver;
using MongoDB.Bson;
using SelectAlgorithm.Utilities;

namespace SelectAlgorithm.Services;

public class VectorComparisonService
{
    private readonly ILogger<VectorComparisonService> _logger;
    private readonly AzureOpenAIClient _aiClient;
    private readonly MongoClient _dbClient;
    private readonly string _databaseName;
    private readonly string _embeddedField;
    private readonly int _embeddingDimensions;
    private readonly int _loadBatchSize;
    private readonly int _topK;

    // All supported algorithms and similarity functions
    private static readonly string[] Algorithms = ["diskann", "hnsw", "ivf"];
    private static readonly string[] Similarities = ["COS", "L2", "IP"];

    // Display labels for algorithms
    private static readonly Dictionary<string, string> AlgorithmLabels = new()
    {
        ["diskann"] = "DiskANN",
        ["hnsw"] = "HNSW",
        ["ivf"] = "IVF"
    };

    public VectorComparisonService(
        ILogger<VectorComparisonService> logger,
        AzureOpenAIClient aiClient,
        MongoClient dbClient,
        string databaseName,
        string embeddedField,
        int embeddingDimensions,
        int loadBatchSize,
        int topK)
    {
        _logger = logger;
        _aiClient = aiClient;
        _dbClient = dbClient;
        _databaseName = databaseName;
        _embeddedField = embeddedField;
        _embeddingDimensions = embeddingDimensions;
        _loadBatchSize = loadBatchSize;
        _topK = topK;
    }

    // Main method to run comparison across all selected algorithms and similarity functions
    public async Task<List<ComparisonResult>> RunComparisonAsync(
        string dataFilePath,
        string searchQuery,
        string embeddingModel,
        string algorithmFilter,
        string similarityFilter)
    {
        // Determine which algorithm/similarity combinations to test
        var targets = GetTargetCollections(algorithmFilter, similarityFilter);

        _logger.LogInformation("\nVector Algorithm Comparison");
        _logger.LogInformation("   Database: {DatabaseName}", _databaseName);
        _logger.LogInformation("   Algorithms: {AlgorithmFilter}", algorithmFilter);
        _logger.LogInformation("   Similarity: {SimilarityFilter}", similarityFilter);
        _logger.LogInformation("   Collections to query: {Collections}", string.Join(", ", targets.Select(t => t.CollectionName)));
        _logger.LogInformation("   Search query: \"{SearchQuery}\"", searchQuery);

        var db = _dbClient.GetDatabase(_databaseName);
        var data = await Utils.ReadJsonFileAsync<BsonDocument>(dataFilePath);

        // Generate the query embedding once and reuse for all algorithm comparisons
        _logger.LogInformation("Generating query embedding...");
        var embeddingClient = _aiClient.GetEmbeddingClient(embeddingModel);
        var embeddingResponse = await embeddingClient.GenerateEmbeddingAsync(searchQuery);
        var queryEmbedding = embeddingResponse.Value.ToFloats().ToArray();
        _logger.LogInformation("Query embedding: {Dimensions} dimensions", queryEmbedding.Length);

        var comparisonResults = new List<ComparisonResult>();

        // Test each algorithm/similarity combination
        foreach (var target in targets)
        {
            _logger.LogInformation("--- {Algorithm} / {Similarity} ---", AlgorithmLabels[target.Algorithm], target.Similarity);
            _logger.LogInformation("Collection: {CollectionName}", target.CollectionName);

            try
            {
                // Drop and recreate collection for clean comparison
                try
                {
                    await db.DropCollectionAsync(target.CollectionName);
                }
                catch (Exception ex)
                {
                    _logger.LogDebug("Could not drop collection {Name}: {Message}", target.CollectionName, ex.Message);
                }

                await db.CreateCollectionAsync(target.CollectionName);
                _logger.LogInformation("Created collection: {CollectionName}", target.CollectionName);

                var collection = db.GetCollection<BsonDocument>(target.CollectionName);

                // Insert the hotel data with vector embeddings
                var (inserted, failed) = await Utils.InsertDataAsync(collection, data, _loadBatchSize);
                _logger.LogInformation("Inserted: {Inserted}/{Total}", inserted, data.Count);

                // Create the vector index with algorithm-specific parameters
                var indexName = $"vectorIndex_{target.Algorithm}_{target.Similarity.ToLower()}";
                var indexOptions = GetIndexOptions(
                    target.CollectionName,
                    indexName,
                    _embeddedField,
                    _embeddingDimensions,
                    target.Algorithm,
                    target.Similarity
                );
                await db.RunCommandAsync<BsonDocument>(indexOptions);
                _logger.LogInformation("Created vector index: {IndexName}", indexName);

                // Execute the vector search and measure latency
                _logger.LogInformation("Executing vector search...");
                var startTime = DateTime.UtcNow;

                var pipeline = GetSearchPipeline(queryEmbedding, _embeddedField, _topK, target.Algorithm);
                var searchResults = await collection.Aggregate<BsonDocument>(pipeline).ToListAsync();

                var latencyMs = (DateTime.UtcNow - startTime).TotalMilliseconds;

                // Extract hotel names and scores from results
                var results = searchResults.Select(doc => new SearchResult
                {
                    Document = new HotelData
                    {
                        HotelName = doc["document"].AsBsonDocument.GetValue("HotelName", "Unknown").AsString
                    },
                    Score = doc["score"].ToDouble()
                }).ToList();

                comparisonResults.Add(new ComparisonResult
                {
                    CollectionName = target.CollectionName,
                    Algorithm = AlgorithmLabels[target.Algorithm],
                    Similarity = target.Similarity,
                    SearchResults = results,
                    LatencyMs = latencyMs
                });

                _logger.LogInformation("[OK] {ResultCount} results, {LatencyMs}ms", results.Count, latencyMs.ToString("F0"));
            }
            catch (Azure.RequestFailedException ex)
            {
                _logger.LogError("Azure service error (HTTP {Status}): {Message}", ex.Status, ex.Message);
            }
            catch (MongoException ex)
            {
                _logger.LogError("MongoDB error: {Message}", ex.Message);
            }
            catch (Exception ex)
            {
                _logger.LogError("Unexpected error comparing algorithms: {Message}", ex.Message);
            }
        }

        return comparisonResults;
    }

    // Get list of collections to test based on algorithm and similarity filters
    private List<(string CollectionName, string Algorithm, string Similarity)> GetTargetCollections(
        string algorithmEnv,
        string similarityEnv)
    {
        // Support "all" keyword or specific algorithm names
        var algorithms = algorithmEnv.ToLower() == "all"
            ? Algorithms
            : new[] { algorithmEnv.ToLower() };

        // Support "all" keyword or specific similarity functions
        var similarities = similarityEnv.ToUpper() == "ALL"
            ? Similarities
            : new[] { similarityEnv.ToUpper() };

        var targets = new List<(string, string, string)>();

        foreach (var alg in algorithms)
        {
            if (!Algorithms.Contains(alg))
            {
                throw new ArgumentException($"Invalid ALGORITHM '{alg}'. Must be one of: all, {string.Join(", ", Algorithms)}");
            }

            foreach (var sim in similarities)
            {
                if (!Similarities.Contains(sim))
                {
                    throw new ArgumentException($"Invalid SIMILARITY '{sim}'. Must be one of: all, {string.Join(", ", Similarities)}");
                }

                // Collection naming pattern: hotels_{algorithm}_{similarity}
                targets.Add(($"hotels_{alg}_{sim.ToLower()}", alg, sim));
            }
        }

        return targets;
    }

    // Create vector index with algorithm-specific parameters
    // DocumentDB uses the cosmosSearch index type for vector indexes
    private BsonDocument GetIndexOptions(
        string collectionName,
        string indexName,
        string embeddedField,
        int dimensions,
        string algorithm,
        string similarity)
    {
        // Base vector index configuration
        var cosmosSearchOptions = new BsonDocument
        {
            ["kind"] = $"vector-{algorithm}",
            ["dimensions"] = dimensions,
            ["similarity"] = similarity
        };

        // Add algorithm-specific tuning parameters
        switch (algorithm)
        {
            case "diskann":
                // DiskANN: disk-based approximate nearest neighbor
                cosmosSearchOptions["maxDegree"] = 32;  // Max edges per node in the graph
                cosmosSearchOptions["lBuild"] = 50;      // Candidates evaluated during index build
                break;
            case "hnsw":
                // HNSW: hierarchical navigable small world graph
                cosmosSearchOptions["m"] = 16;                // Max connections per layer
                cosmosSearchOptions["efConstruction"] = 64;   // Candidates during construction
                break;
            case "ivf":
                // IVF: inverted file index
                cosmosSearchOptions["numLists"] = 1;    // Number of inverted lists (clusters)
                break;
        }

        // Return the createIndexes command document
        return new BsonDocument
        {
            ["createIndexes"] = collectionName,
            ["indexes"] = new BsonArray
            {
                new BsonDocument
                {
                    ["name"] = indexName,
                    ["key"] = new BsonDocument { [embeddedField] = "cosmosSearch" },
                    ["cosmosSearchOptions"] = cosmosSearchOptions
                }
            }
        };
    }

    // Create aggregation pipeline for vector search with algorithm-specific parameters
    private PipelineDefinition<BsonDocument, BsonDocument> GetSearchPipeline(
        float[] queryEmbedding,
        string embeddedField,
        int k,
        string algorithm)
    {
        // Base cosmosSearch configuration
        var cosmosSearch = new BsonDocument
        {
            ["vector"] = new BsonArray(queryEmbedding.Select(f => new BsonDouble(f))),
            ["path"] = embeddedField,
            ["k"] = k  // Number of results to return
        };

        // Add algorithm-specific search parameters
        switch (algorithm)
        {
            case "diskann":
                // DiskANN: lSearch controls search quality vs speed tradeoff
                cosmosSearch["lSearch"] = 100;  // Higher = more accurate but slower
                break;
            case "hnsw":
                // HNSW: efSearch controls candidate list size during search
                cosmosSearch["efSearch"] = 80;  // Higher = more accurate but slower
                break;
            case "ivf":
                // IVF: nProbes controls how many clusters to search
                cosmosSearch["nProbes"] = 1;    // Higher = more accurate but slower
                break;
        }

        // Build the aggregation pipeline: search + project score
        return new BsonDocument[]
        {
            new BsonDocument("$search", new BsonDocument { ["cosmosSearch"] = cosmosSearch }),
            new BsonDocument("$project", new BsonDocument
            {
                ["score"] = new BsonDocument("$meta", "searchScore"),
                ["document"] = "$$ROOT"
            })
        };
    }
}
```

This service:
- Manages the comparison workflow for all algorithms
- Creates collections and indexes for each algorithm/similarity combination
- Inserts data and executes vector searches
- Measures and collects latency metrics
- Configures algorithm-specific parameters for index creation and search

### Utilities/Utils.cs

Add this code to `Utilities/Utils.cs`:

```csharp
using Azure.Core;
using Azure.Identity;
using Azure.AI.OpenAI;
using MongoDB.Driver;
using MongoDB.Driver.Authentication.Oidc;
using MongoDB.Bson;
using System.Text.Json;

namespace SelectAlgorithm.Utilities;

// Custom OIDC token handler for Azure Identity authentication
// MongoDB 3.0+ uses OIDC for passwordless authentication with Azure services
internal sealed class AzureIdentityTokenHandler(TokenCredential credential, string tenantId) : IOidcCallback
{
    // DocumentDB requires this specific Azure scope for authentication
    private readonly string[] scopes = ["https://ossrdbms-aad.database.windows.net/.default"];

    // OIDC tokens expire after approximately 1 hour
    // MongoDB driver handles token refresh automatically via this callback
    public OidcAccessToken GetOidcAccessToken(OidcCallbackParameters parameters, CancellationToken cancellationToken)
    {
        AccessToken token = credential.GetToken(
            new TokenRequestContext(scopes, tenantId: tenantId),
            cancellationToken
        );

        return new OidcAccessToken(token.Token, token.ExpiresOn - DateTimeOffset.UtcNow);
    }

    public async Task<OidcAccessToken> GetOidcAccessTokenAsync(OidcCallbackParameters parameters, CancellationToken cancellationToken)
    {
        AccessToken token = await credential.GetTokenAsync(
            new TokenRequestContext(scopes, parentRequestId: null, tenantId: tenantId),
            cancellationToken
        );

        return new OidcAccessToken(token.Token, token.ExpiresOn - DateTimeOffset.UtcNow);
    }
}

public static class Utils
{
    // Initialize Azure OpenAI and DocumentDB clients with passwordless authentication
    // Uses DefaultAzureCredential which tries multiple authentication methods in order:
    // 1. Environment variables, 2. Managed identity, 3. Visual Studio, 4. Azure CLI, 5. Azure PowerShell
    public static (AzureOpenAIClient aiClient, MongoClient dbClient) GetClientsPasswordless(
        string openAiEndpoint,
        string mongoClusterName,
        string tenantId)
    {
        // Create credential with tenant ID for multi-tenant scenarios
        var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
        {
            TenantId = tenantId
        });

        // Initialize Azure OpenAI client with automatic retry policy
        var options = new AzureOpenAIClientOptions();
        // Default retry policy (3 retries with exponential backoff) is applied automatically
        var aiClient = new AzureOpenAIClient(new Uri(openAiEndpoint), credential, options);

        // Build DocumentDB connection string with OIDC authentication
        var connectionString = $"mongodb+srv://{mongoClusterName}.mongocluster.cosmos.azure.com/?tls=true&authMechanism=MONGODB-OIDC&retrywrites=false&maxIdleTimeMS=120000";
        var settings = MongoClientSettings.FromUrl(MongoUrl.Create(connectionString));
        settings.UseTls = true;
        settings.RetryWrites = false;
        settings.MaxConnectionIdleTime = TimeSpan.FromMinutes(2);
        settings.Credential = MongoCredential.CreateOidcCredential(new AzureIdentityTokenHandler(credential, tenantId));
        settings.Freeze();

        var dbClient = new MongoClient(settings);

        return (aiClient, dbClient);
    }

    // Read JSON file and deserialize to list of objects
    // Console.WriteLine is used intentionally in Utils methods for demo output,
    // keeping utility methods simple without requiring ILogger dependency injection
    public static async Task<List<T>> ReadJsonFileAsync<T>(string filePath)
    {
        Console.WriteLine($"Reading JSON file from {filePath}");
        var jsonContent = await File.ReadAllTextAsync(filePath);
        var options = new JsonSerializerOptions { PropertyNameCaseInsensitive = true };
        return JsonSerializer.Deserialize<List<T>>(jsonContent, options) ?? new List<T>();
    }

    // Insert data in batches with progress logging
    // Returns count of successfully inserted and failed documents
    public static async Task<(int inserted, int failed)> InsertDataAsync<T>(
        IMongoCollection<T> collection,
        List<T> data,
        int batchSize) where T : class
    {
        Console.WriteLine($"Processing in batches of {batchSize}...");
        int totalBatches = (int)Math.Ceiling((double)data.Count / batchSize);
        int inserted = 0;
        int failed = 0;

        for (int i = 0; i < totalBatches; i++)
        {
            var batch = data.Skip(i * batchSize).Take(batchSize).ToList();
            try
            {
                await collection.InsertManyAsync(batch, new InsertManyOptions { IsOrdered = false });
                inserted += batch.Count;
                Console.WriteLine($"Batch {i + 1} complete: {batch.Count} inserted");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error in batch {i + 1}: {ex.Message}");
                failed += batch.Count;
            }

            // Small delay between batches to avoid throttling
            if (i < totalBatches - 1)
            {
                await Task.Delay(100);
            }
        }

        // Create standard indexes on commonly queried fields
        var indexColumns = new[] { "HotelId", "Category", "Description", "Description_fr" };
        foreach (var col in indexColumns)
        {
            await collection.Indexes.CreateOneAsync(
                new CreateIndexModel<T>(Builders<T>.IndexKeys.Ascending(col))
            );
        }

        return (inserted, failed);
    }

    // Print comparison results in a formatted table
    public static void PrintComparisonTable(List<ComparisonResult> results)
    {
        Console.WriteLine("\n" + new string('=', 90));
        Console.WriteLine("                     Vector Algorithm Comparison Results");
        Console.WriteLine(new string('=', 90));

        Console.WriteLine(
            "Algorithm".PadRight(14) +
            "Similarity".PadRight(14) +
            "Top Result".PadRight(26) +
            "Score".PadRight(14) +
            "Latency(ms)"
        );
        Console.WriteLine(new string('-', 90));

        // Print summary table with top result from each algorithm
        foreach (var r in results)
        {
            var topResult = r.SearchResults.FirstOrDefault();
            var topName = topResult != null
                ? (topResult.Document.HotelName?.Length > 24
                    ? topResult.Document.HotelName.Substring(0, 24)
                    : topResult.Document.HotelName ?? "N/A")
                : "N/A";
            var topScore = topResult != null ? topResult.Score.ToString("F4") : "N/A";

            Console.WriteLine(
                r.Algorithm.PadRight(14) +
                r.Similarity.PadRight(14) +
                topName.PadRight(26) +
                topScore.PadRight(14) +
                r.LatencyMs.ToString("F0")
            );
        }

        Console.WriteLine(new string('=', 90));

        // Print detailed results for each algorithm
        foreach (var r in results)
        {
            Console.WriteLine($"\n--- {r.Algorithm} / {r.Similarity} ({r.CollectionName}) ---");
            if (r.SearchResults.Count == 0)
            {
                Console.WriteLine("  No results.");
                continue;
            }
            for (int i = 0; i < r.SearchResults.Count; i++)
            {
                var item = r.SearchResults[i];
                Console.WriteLine($"  {i + 1}. {item.Document.HotelName}, Score: {item.Score:F4}");
            }
            Console.WriteLine($"  Latency: {r.LatencyMs:F0}ms");
        }
    }
}

// Model classes for comparison results
public class ComparisonResult
{
    public string CollectionName { get; set; } = string.Empty;
    public string Algorithm { get; set; } = string.Empty;
    public string Similarity { get; set; } = string.Empty;
    public List<SearchResult> SearchResults { get; set; } = new();
    public double LatencyMs { get; set; }
}

public class SearchResult
{
    public HotelData Document { get; set; } = new();
    public double Score { get; set; }
}

public class HotelData
{
    public string? HotelName { get; set; }
}
```

This utility class provides:
- Passwordless authentication setup for Azure OpenAI and DocumentDB
- OIDC token handler for automatic token refresh
- JSON file reading and deserialization
- Batch data insertion with error handling
- Results formatting and display

### global.json

Add this code to `global.json`:

```json
{
  "sdk": {
    "version": "9.0.200",
    "rollForward": "latestFeature"
  }
}
```

This file specifies the .NET SDK version requirements for the project.

## Run the code

1. Build the project:

   ```bash
   dotnet build
   ```

2. Run the application to compare all algorithms with COS similarity (default):

   ```bash
   dotnet run
   ```

   The application creates three collections (`hotels_diskann_cos`, `hotels_hnsw_cos`, `hotels_ivf_cos`), inserts data, creates vector indexes, and performs searches on each.

3. To compare all algorithms with all similarity functions, set environment variables:

   ```bash
   export ALGORITHM=all
   export SIMILARITY=all
   dotnet run
   ```

   This creates nine collections (3 algorithms x 3 similarity functions) and compares all combinations.

4. To test a specific algorithm with a specific similarity function:

   ```bash
   export ALGORITHM=diskann
   export SIMILARITY=COS
   dotnet run
   ```

### Expected output

The application displays progress logs and a comparison table. Results vary based on data and server load:

```
Vector Algorithm Comparison
   Database: Hotels
   Algorithms: all
   Similarity: COS
   Collections to query: hotels_diskann_cos, hotels_hnsw_cos, hotels_ivf_cos
   Search query: "quintessential lodging near running trails, eateries, retail"

Generating query embedding...
Query embedding: 1536 dimensions

--- DiskANN / COS ---
Collection: hotels_diskann_cos
Created collection: hotels_diskann_cos
Inserted: 50/50
Created vector index: vectorIndex_diskann_cos
Executing vector search...
[OK] 5 results, 45ms

--- HNSW / COS ---
Collection: hotels_hnsw_cos
Created collection: hotels_hnsw_cos
Inserted: 50/50
Created vector index: vectorIndex_hnsw_cos
Executing vector search...
[OK] 5 results, 38ms

--- IVF / COS ---
Collection: hotels_ivf_cos
Created collection: hotels_ivf_cos
Inserted: 50/50
Created vector index: vectorIndex_ivf_cos
Executing vector search...
[OK] 5 results, 52ms

==========================================================================================
                     Vector Algorithm Comparison Results
==========================================================================================
Algorithm     Similarity    Top Result                Score         Latency(ms)
------------------------------------------------------------------------------------------
DiskANN       COS           Historic Downtown Inn      0.8342        45
HNSW          COS           Historic Downtown Inn      0.8342        38
IVF           COS           Historic Downtown Inn      0.8342        52
==========================================================================================

--- DiskANN / COS (hotels_diskann_cos) ---
  1. Historic Downtown Inn, Score: 0.8342
  2. Mountain Trail Lodge, Score: 0.7891
  3. Riverside Retreat, Score: 0.7654
  4. Urban Fitness Suites, Score: 0.7210
  5. Lakeside Wellness Resort, Score: 0.7045
  Latency: 45ms

--- HNSW / COS (hotels_hnsw_cos) ---
  1. Historic Downtown Inn, Score: 0.8342
  2. Mountain Trail Lodge, Score: 0.7891
  3. Riverside Retreat, Score: 0.7654
  4. Urban Fitness Suites, Score: 0.7210
  5. Lakeside Wellness Resort, Score: 0.7045
  Latency: 38ms

--- IVF / COS (hotels_ivf_cos) ---
  1. Historic Downtown Inn, Score: 0.8342
  2. Mountain Trail Lodge, Score: 0.7891
  3. Riverside Retreat, Score: 0.7654
  4. Urban Fitness Suites, Score: 0.7210
  5. Lakeside Wellness Resort, Score: 0.7045
  Latency: 52ms
```

## Algorithm comparison guidance

Use this guidance to choose the right vector search algorithm for your workload:

| Algorithm | Best for | Index creation | Search speed | Memory usage | Accuracy |
|-----------|----------|---------------|--------------|--------------|----------|
| **DiskANN** | Large datasets, disk-based storage | Slow | Fast | Low (disk-based) | High |
| **HNSW** | Real-time search, high throughput | Medium | Fastest | High (memory-intensive) | Very high |
| **IVF** | Cost-sensitive, approximate search | Fast | Medium | Low | Medium |

### Similarity functions

| Function | Formula | Best for |
|----------|---------|----------|
| **COS** (Cosine) | Angle between vectors | Text embeddings, normalized vectors |
| **L2** (Euclidean) | Distance between points | Image embeddings, coordinate data |
| **IP** (Inner Product) | Dot product | Recommendation systems, unnormalized data |

### Tuning parameters

Each algorithm has tuning parameters that control the accuracy/performance tradeoff:

**DiskANN:**
- `maxDegree`: Higher values (32-64) improve accuracy but increase memory
- `lBuild`: Higher values (50-100) improve index quality but slow build time
- `lSearch`: Higher values (100-200) improve search accuracy but slow queries

**HNSW:**
- `m`: Higher values (16-48) improve accuracy but increase memory
- `efConstruction`: Higher values (64-200) improve index quality but slow build time
- `efSearch`: Higher values (80-200) improve search accuracy but slow queries

**IVF:**
- `numLists`: More lists improve speed but may reduce accuracy
- `nProbes`: Higher values (1-10) improve accuracy but slow queries

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `TimeoutException` during connection | Verify your connection string and environment variables. Ensure your IP is in the DocumentDB firewall rules. |
| `AuthenticationException` | Check that `DefaultAzureCredential` can acquire a token. Run `az login` to refresh your credentials. |
| Build errors with .NET version | Ensure you have .NET 9.0 or later installed. Run `dotnet --version` to check. |
| `BsonSerializationException` | Ensure your model classes match the document structure in the collection. |
| Empty search results | The vector index might not be ready yet. The sample includes retry logic, but if you still see empty results, wait a few seconds and retry. |
| `IndexOptionsConflict` (code 85) | DocumentDB doesn't allow multiple vector indexes of the same kind on the same field. Drop the existing index before creating a new one. |

## Clean up resources

When you're done, you can remove the database using mongosh or the Azure portal.

### [mongosh](#tab/mongosh)

Connect to your DocumentDB cluster and drop the database:

```bash
mongosh "<your-connection-string>"
use Hotels
db.dropDatabase()
```

### [Azure portal](#tab/portal)

1. Navigate to your DocumentDB resource in the Azure portal.
2. Select **Data Explorer**.
3. Right-click the **Hotels** database and select **Delete Database**.

---

## Next steps

- [Vector search concepts in Azure DocumentDB](concepts-vector-search)
- [How to configure vector indexes](how-to-vector-search)
- [Tune vector search performance](how-to-tune-vector-search)
- [Best practices for production workloads](best-practices-vector-search)
