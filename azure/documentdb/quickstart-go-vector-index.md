---
title: "Quickstart - Vector Indexing with Go"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Go."
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: golang
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-go
  - devx-track-data-ai
  - devx-track-go-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with Go

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

This quickstart uses a sample hotel dataset in a JSON file with pre-calculated vectors from the `text-embedding-3-small` model. The dataset includes hotel names, locations, descriptions, and vector embeddings.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-go) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Index Quickstart](includes/prerequisite-quickstart-vector-index.md)]

- [Go](https://go.dev/dl/)

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

1. Initialize a Go module:

    ```bash
    go mod init documentdb-quickstart
    ```

1. Install required packages:

    ```bash
    go get github.com/Azure/azure-sdk-for-go/sdk/azcore
    go get github.com/Azure/azure-sdk-for-go/sdk/azidentity
    go get github.com/openai/openai-go/v3
    go get go.mongodb.org/mongo-driver
    go get github.com/joho/godotenv
    ```

1. Create a `.env` file with your credentials:

    ```env
    MONGO_CLUSTER_NAME=
    AZURE_OPENAI_EMBEDDING_ENDPOINT=
    AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
    AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
    EMBEDDED_FIELD=DescriptionVector
    EMBEDDING_DIMENSIONS=1536
    DATA_FILE_WITH_VECTORS=data/Hotels_Vector.json
    LOAD_SIZE_BATCH=100
    ```

    Replace the placeholder values in the `.env` file with your own information:
    - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
    - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB resource name

## Create a vector index

### [IVF](#tab/tab-ivf)

IVF (Inverted File) is ideal for datasets with fewer than 10,000 documents. It partitions vectors into clusters for fast approximate search.

Create `ivf.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
	"github.com/Azure/azure-sdk-for-go/sdk/azidentity"
	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/azure"
	"github.com/openai/openai-go/v3/option"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// CreateIVFVectorIndex creates an IVF (Inverted File) vector index on the specified field
func CreateIVFVectorIndex(ctx context.Context, collection *mongo.Collection, vectorField string, dimensions int, similarity string) error {
	fmt.Printf("Creating IVF vector index on field '%s'...\n", vectorField)

	// Use the native MongoDB command for DocumentDB vector indexes
	// Note: Must use bson.D for commands to preserve order
	indexCommand := bson.D{
		{"createIndexes", collection.Name()},
		{"indexes", []bson.D{
			{
				{"name", fmt.Sprintf("ivf_index_%s", vectorField)},
				{"key", bson.D{
					{vectorField, "cosmosSearch"}, // DocumentDB vector search index type
				}},
				{"cosmosSearchOptions", bson.D{
					// IVF algorithm configuration
					{"kind", "vector-ivf"},

					// Vector dimensions must match the embedding model
					{"dimensions", dimensions},

					// Cosine similarity is effective for text embeddings
					{"similarity", "COS"},

					// Number of clusters (centroids) to partition vectors into
					// More clusters = faster search but potentially lower recall
					{"numLists", 10},
				}},
			},
		}},
	}

	// Execute the createIndexes command directly
	var result bson.M
	err := collection.Database().RunCommand(ctx, indexCommand).Decode(&result)
	if err != nil {
		if strings.Contains(err.Error(), "not enabled for this cluster tier") {
			fmt.Println("\nIVF indexes require a higher cluster tier.")
			fmt.Println("Try one of these alternatives:")
			fmt.Println("  • Upgrade your DocumentDB cluster to a higher tier")
			fmt.Println("  • Use HNSW instead: go run hnsw.go")
			fmt.Println("  • Use DiskANN instead: go run diskann.go")
		}
		return fmt.Errorf("error creating IVF vector index: %v", err)
	}

	fmt.Println("IVF vector index created successfully")
	return nil
}

// PerformIVFVectorSearch performs a vector search using IVF algorithm
func PerformIVFVectorSearch(ctx context.Context, collection *mongo.Collection, openAIClient openai.Client, queryText, vectorField, modelName string, topK int) ([]SearchResult, error) {
	fmt.Printf("Performing IVF vector search for: '%s'\n", queryText)

	// Generate embedding vector for the search query
	queryEmbedding, err := GenerateEmbedding(ctx, openAIClient, queryText, modelName)
	if err != nil {
		return nil, fmt.Errorf("error generating embedding: %v", err)
	}

	// Construct aggregation pipeline for IVF vector search
	pipeline := []bson.M{
		{
			"$search": bson.M{
				// Use cosmosSearch for vector operations in DocumentDB
				"cosmosSearch": bson.M{
					// Query vector to find similar documents
					"vector": queryEmbedding,

					// Document field containing vectors to search against
					"path": vectorField,

					// Final number of results to return
					"k": topK,
				},
			},
		},
		{
			"$project": bson.M{
				"document": "$$ROOT",
				// Add search score from metadata
				"score": bson.M{"$meta": "searchScore"},
			},
		},
	}

	// Execute the aggregation pipeline
	cursor, err := collection.Aggregate(ctx, pipeline)
	if err != nil {
		return nil, fmt.Errorf("error executing aggregation: %v", err)
	}
	defer cursor.Close(ctx)

	var results []SearchResult
	if err = cursor.All(ctx, &results); err != nil {
		return nil, fmt.Errorf("error decoding results: %v", err)
	}

	return results, nil
}

type SearchResult struct {
	Document interface{} `bson:"document"`
	Score    float64     `bson:"score"`
}

func main() {
	ctx := context.Background()

	// Create Azure credential
	credential, err := azidentity.NewDefaultAzureCredential(nil)
	if err != nil {
		log.Fatalf("Failed to create credential: %v", err)
	}

	// Create MongoDB client with OIDC authentication
	mongoURI := fmt.Sprintf("mongodb+srv://%s.global.mongocluster.cosmos.azure.com/", os.Getenv("MONGO_CLUSTER_NAME"))
	
	oidcCallback := func(ctx context.Context, args *options.OIDCArgs) (*options.OIDCCredential, error) {
		token, err := credential.GetToken(ctx, policy.TokenRequestOptions{
			Scopes: []string{"https://ossrdbms-aad.database.windows.net/.default"},
		})
		if err != nil {
			return nil, err
		}
		return &options.OIDCCredential{AccessToken: token.Token}, nil
	}

	clientOptions := options.Client().
		ApplyURI(mongoURI).
		SetConnectTimeout(30 * time.Second).
		SetAuth(options.Credential{
			AuthMechanism: "MONGODB-OIDC",
			AuthMechanismProperties: map[string]string{
				"TOKEN_RESOURCE": "https://ossrdbms-aad.database.windows.net",
			},
			OIDCMachineCallback: oidcCallback,
		})

	mongoClient, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", err)
	}
	defer mongoClient.Disconnect(ctx)

	// Create Azure OpenAI client with credential-based authentication
	azureOpenAIEndpoint := os.Getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT")
	openAIClient := openai.NewClient(
		option.WithBaseURL(fmt.Sprintf("%s/openai/v1", azureOpenAIEndpoint)),
		azure.WithTokenCredential(credential))

	// Access database and collection
	database := mongoClient.Database("Hotels")
	collection := database.Collection("hotels_ivf")

	// Create IVF vector index
	dimensions, _ := strconv.Atoi(os.Getenv("EMBEDDING_DIMENSIONS"))
	err = CreateIVFVectorIndex(ctx, collection, os.Getenv("EMBEDDED_FIELD"), dimensions, "COS")
	if err != nil {
		log.Fatalf("Failed to create index: %v", err)
	}

	// Perform search
	query := "quintessential lodging near running trails, eateries, retail"
	results, err := PerformIVFVectorSearch(
		ctx,
		collection,
		openAIClient,
		query,
		os.Getenv("EMBEDDED_FIELD"),
		os.Getenv("AZURE_OPENAI_EMBEDDING_MODEL"),
		5,
	)
	if err != nil {
		log.Fatalf("Search failed: %v", err)
	}

	fmt.Printf("\nSearch Results (%d found):\n", len(results))
	for i, result := range results {
		// Extract hotel name from document
		doc := result.Document.(bson.M)
		hotelName := doc["HotelName"].(string)
		fmt.Printf("%d. %s, Score: %.4f\n", i+1, hotelName, result.Score)
	}
}

// Helper function to generate embeddings
func GenerateEmbedding(ctx context.Context, client openai.Client, text, modelName string) ([]float64, error) {
	resp, err := client.Embeddings.New(ctx, openai.EmbeddingNewParams{
		Input: openai.EmbeddingNewParamsInputUnion{
			OfString: openai.String(text),
		},
		Model: modelName,
	})
	if err != nil {
		return nil, fmt.Errorf("failed to generate embedding: %v", err)
	}

	if len(resp.Data) == 0 {
		return nil, fmt.Errorf("no embedding data received")
	}

	embedding := make([]float64, len(resp.Data[0].Embedding))
	for i, v := range resp.Data[0].Embedding {
		embedding[i] = float64(v)
	}

	return embedding, nil
}
```

#### [HNSW](#tab/tab-hnsw)

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `hnsw.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
	"github.com/Azure/azure-sdk-for-go/sdk/azidentity"
	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/azure"
	"github.com/openai/openai-go/v3/option"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// CreateHNSWVectorIndex creates an HNSW vector index on the specified field
func CreateHNSWVectorIndex(ctx context.Context, collection *mongo.Collection, vectorField string, dimensions int, similarity string) error {
	fmt.Printf("Creating HNSW vector index on field '%s'...\n", vectorField)

	indexCommand := bson.D{
		{"createIndexes", collection.Name()},
		{"indexes", []bson.D{
			{
				{"name", fmt.Sprintf("hnsw_index_%s", vectorField)},
				{"key", bson.D{
					{vectorField, "cosmosSearch"},
				}},
				{"cosmosSearchOptions", bson.D{
					// HNSW algorithm configuration
					{"kind", "vector-hnsw"},

					// Vector dimensions must match the embedding model
					{"dimensions", dimensions},

					// Cosine similarity works well with text embeddings
					{"similarity", "COS"},

					// Maximum connections per node (2-100, default 16)
					{"m", 16},

					// Candidate list size during construction (4-1000, default 64)
					{"efConstruction", 64},
				}},
			},
		}},
	}

	var result bson.M
	err := collection.Database().RunCommand(ctx, indexCommand).Decode(&result)
	if err != nil {
		if strings.Contains(err.Error(), "not enabled for this cluster tier") {
			fmt.Println("\nHNSW indexes require M30 or higher cluster tier.")
		}
		return fmt.Errorf("error creating HNSW vector index: %v", err)
	}

	fmt.Println("HNSW vector index created successfully")
	return nil
}

func main() {
	ctx := context.Background()

	credential, err := azidentity.NewDefaultAzureCredential(nil)
	if err != nil {
		log.Fatalf("Failed to create credential: %v", err)
	}

	mongoURI := fmt.Sprintf("mongodb+srv://%s.global.mongocluster.cosmos.azure.com/", os.Getenv("MONGO_CLUSTER_NAME"))

	oidcCallback := func(ctx context.Context, args *options.OIDCArgs) (*options.OIDCCredential, error) {
		token, err := credential.GetToken(ctx, policy.TokenRequestOptions{
			Scopes: []string{"https://ossrdbms-aad.database.windows.net/.default"},
		})
		if err != nil {
			return nil, err
		}
		return &options.OIDCCredential{AccessToken: token.Token}, nil
	}

	clientOptions := options.Client().
		ApplyURI(mongoURI).
		SetConnectTimeout(30 * time.Second).
		SetAuth(options.Credential{
			AuthMechanism: "MONGODB-OIDC",
			AuthMechanismProperties: map[string]string{
				"TOKEN_RESOURCE": "https://ossrdbms-aad.database.windows.net",
			},
			OIDCMachineCallback: oidcCallback,
		})

	mongoClient, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", err)
	}
	defer mongoClient.Disconnect(ctx)

	azureOpenAIEndpoint := os.Getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT")
	openAIClient := openai.NewClient(
		option.WithBaseURL(fmt.Sprintf("%s/openai/v1", azureOpenAIEndpoint)),
		azure.WithTokenCredential(credential))

	database := mongoClient.Database("Hotels")
	collection := database.Collection("hotels_hnsw")

	dimensions, _ := strconv.Atoi(os.Getenv("EMBEDDING_DIMENSIONS"))
	err = CreateHNSWVectorIndex(ctx, collection, os.Getenv("EMBEDDED_FIELD"), dimensions, "COS")
	if err != nil {
		log.Fatalf("Failed to create index: %v", err)
	}

	query := "quintessential lodging near running trails, eateries, retail"
	results, err := PerformIVFVectorSearch(
		ctx, collection, openAIClient, query,
		os.Getenv("EMBEDDED_FIELD"),
		os.Getenv("AZURE_OPENAI_EMBEDDING_MODEL"), 5,
	)
	if err != nil {
		log.Fatalf("Search failed: %v", err)
	}

	fmt.Printf("\nSearch Results (%d found):\n", len(results))
	for i, result := range results {
		doc := result.Document.(bson.M)
		hotelName := doc["HotelName"].(string)
		fmt.Printf("%d. %s, Score: %.4f\n", i+1, hotelName, result.Score)
	}
}
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity (2–100, default 16). Higher values improve recall but increase memory.
- **efConstruction**: Candidate list size during construction (4–1000, default 64). Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

#### [DiskANN](#tab/tab-diskann)

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `diskann.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
	"github.com/Azure/azure-sdk-for-go/sdk/azidentity"
	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/azure"
	"github.com/openai/openai-go/v3/option"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// CreateDiskANNVectorIndex creates a DiskANN vector index on the specified field
func CreateDiskANNVectorIndex(ctx context.Context, collection *mongo.Collection, vectorField string, dimensions int, similarity string) error {
	fmt.Printf("Creating DiskANN vector index on field '%s'...\n", vectorField)

	indexCommand := bson.D{
		{"createIndexes", collection.Name()},
		{"indexes", []bson.D{
			{
				{"name", fmt.Sprintf("diskann_index_%s", vectorField)},
				{"key", bson.D{
					{vectorField, "cosmosSearch"},
				}},
				{"cosmosSearchOptions", bson.D{
					// DiskANN algorithm configuration
					{"kind", "vector-diskann"},

					// Vector dimensions must match the embedding model
					{"dimensions", dimensions},

					// Cosine similarity metric
					{"similarity", "COS"},

					// Maximum degree: edges per node (20-2048, default 32)
					{"maxDegree", 32},

					// Candidates evaluated during construction (10-500, default 50)
					{"lBuild", 50},
				}},
			},
		}},
	}

	var result bson.M
	err := collection.Database().RunCommand(ctx, indexCommand).Decode(&result)
	if err != nil {
		if strings.Contains(err.Error(), "not enabled for this cluster tier") {
			fmt.Println("\nDiskANN indexes require M30 or higher cluster tier.")
		}
		return fmt.Errorf("error creating DiskANN vector index: %v", err)
	}

	fmt.Println("DiskANN vector index created successfully")
	return nil
}

func main() {
	ctx := context.Background()

	credential, err := azidentity.NewDefaultAzureCredential(nil)
	if err != nil {
		log.Fatalf("Failed to create credential: %v", err)
	}

	mongoURI := fmt.Sprintf("mongodb+srv://%s.global.mongocluster.cosmos.azure.com/", os.Getenv("MONGO_CLUSTER_NAME"))

	oidcCallback := func(ctx context.Context, args *options.OIDCArgs) (*options.OIDCCredential, error) {
		token, err := credential.GetToken(ctx, policy.TokenRequestOptions{
			Scopes: []string{"https://ossrdbms-aad.database.windows.net/.default"},
		})
		if err != nil {
			return nil, err
		}
		return &options.OIDCCredential{AccessToken: token.Token}, nil
	}

	clientOptions := options.Client().
		ApplyURI(mongoURI).
		SetConnectTimeout(30 * time.Second).
		SetAuth(options.Credential{
			AuthMechanism: "MONGODB-OIDC",
			AuthMechanismProperties: map[string]string{
				"TOKEN_RESOURCE": "https://ossrdbms-aad.database.windows.net",
			},
			OIDCMachineCallback: oidcCallback,
		})

	mongoClient, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", err)
	}
	defer mongoClient.Disconnect(ctx)

	azureOpenAIEndpoint := os.Getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT")
	openAIClient := openai.NewClient(
		option.WithBaseURL(fmt.Sprintf("%s/openai/v1", azureOpenAIEndpoint)),
		azure.WithTokenCredential(credential))

	database := mongoClient.Database("Hotels")
	collection := database.Collection("hotels_diskann")

	dimensions, _ := strconv.Atoi(os.Getenv("EMBEDDING_DIMENSIONS"))
	err = CreateDiskANNVectorIndex(ctx, collection, os.Getenv("EMBEDDED_FIELD"), dimensions, "COS")
	if err != nil {
		log.Fatalf("Failed to create index: %v", err)
	}

	query := "quintessential lodging near running trails, eateries, retail"
	results, err := PerformIVFVectorSearch(
		ctx, collection, openAIClient, query,
		os.Getenv("EMBEDDED_FIELD"),
		os.Getenv("AZURE_OPENAI_EMBEDDING_MODEL"), 5,
	)
	if err != nil {
		log.Fatalf("Search failed: %v", err)
	}

	fmt.Printf("\nSearch Results (%d found):\n", len(results))
	for i, result := range results {
		doc := result.Document.(bson.M)
		hotelName := doc["HotelName"].(string)
		fmt.Printf("%d. %s, Score: %.4f\n", i+1, hotelName, result.Score)
	}
}
```

Key parameters:
- **maxDegree**: Number of edges per node (20–2048, default 32). Higher values improve accuracy.
- **lBuild**: Candidate neighbors evaluated during construction (10–500, default 50). Affects index quality.
- **Cluster tier**: Requires M30 or higher.

----

## Query with vector search

All three algorithms use the same query pattern with the `$search` aggregation stage:

```go
// Generate embedding for query
queryEmbedding, err := GenerateEmbedding(ctx, client, "your query text", modelName)
if err != nil {
	log.Fatal(err)
}

// Execute vector search
pipeline := []bson.M{
	{
		"$search": bson.M{
			"cosmosSearch": bson.M{
				"vector": queryEmbedding,
				"path": vectorField,
				"k": 5,  // Return top 5 similar documents
			},
		},
	},
	{
		"$project": bson.M{
			"score": bson.M{"$meta": "searchScore"},
			"document": "$$ROOT",
		},
	},
}

cursor, err := collection.Aggregate(ctx, pipeline)
defer cursor.Close(ctx)
```

The `$search` stage finds the k nearest neighbors to your query vector. Results are ordered by similarity score (highest first).

> [!IMPORTANT]
> Vectors must be stored as `number[]` (array of numbers) to be indexed. Documents with vectors stored as other types (such as `double[]`) won't be indexed. The maximum supported dimensions depend on the index type: up to 2,000 for IVF/HNSW, and up to 16,000 for DiskANN with product quantization.

## Choose the right algorithm

Select the vector index algorithm based on your dataset size and performance requirements:

| Algorithm | Best For | Cluster Tier | Trade-offs |
|-----------|----------|-------------|------------|
| **IVF** | Small datasets (<10K documents) | M10+ | Fast creation, lower recall at scale |
| **HNSW** | Medium datasets (10K–50K documents) | M30+ | Balanced recall/speed, more memory |
| **DiskANN** | Large datasets (50K+ documents) | M40+ | Best recall at scale, disk-efficient |

### Similarity metrics

Each algorithm supports three similarity metrics:

| Metric | Range | Interpretation | Best For |
|--------|-------|---------------|----------|
| **COS** (Cosine) | 0–1 | Higher = more similar | Text embeddings (normalized vectors) |
| **L2** (Euclidean) | 0+ | Lower = more similar | Spatial data, image embeddings |
| **IP** (Inner Product) | Varies | Higher = more similar | Recommendation systems, normalized embeddings |

### Tuning parameters

#### IVF parameters

| Parameter | Description | Default | Guidance |
|-----------|-------------|---------|----------|
| `numLists` | Number of clusters for partitioning | 1 | Use 1 for <1K docs. For larger datasets, use √N. More lists = faster search but slower creation. |

#### HNSW parameters

| Parameter | Description | Default | Guidance |
|-----------|-------------|---------|----------|
| `m` | Max connections per node | 16 | Higher (up to 64) = better recall, more memory |
| `efConstruction` | Build-time candidate list size | 64 | Higher (up to 500) = better quality, slower build |

#### DiskANN parameters

| Parameter | Description | Default | Guidance |
|-----------|-------------|---------|----------|
| `maxDegree` | Max edges per node | 32 | Higher = better recall, more disk I/O |
| `lBuild` | Build-time search width | 50 | Higher = better quality index, slower build |

### Cluster tier requirements

- **M10/M20**: Sufficient for IVF indexes on small datasets
- **M30**: Required for HNSW indexes (memory-intensive graph structure)
- **M40+**: Required for DiskANN (disk-optimized for large-scale vector search)

> [!NOTE]
> Quantized indexes (HNSW, DiskANN) require a minimum of 1,000 vectors. Below this threshold, the system performs a full scan regardless of index configuration.

### Compare algorithms

Run all three algorithms against the same dataset to compare performance:

```bash
go run compare_all.go
```

This executes 9 searches (3 algorithms × 3 metrics) and displays a comparison table showing latency, similarity scores, and top results for each combination. See the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-go) for the full comparison runner.

## Authenticate with Azure CLI

Sign in to Azure CLI before you run the application so it can access Azure resources securely.

```bash
az login
```

The code uses your local developer authentication to access Azure DocumentDB and Azure OpenAI. The authentication relies on [DefaultAzureCredential](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azidentity#DefaultAzureCredential) from **azidentity** to find your Azure credentials in the environment.

## Run the quickstart

### [IVF](#tab/tab-ivf)

```bash
go run ivf.go
```

#### [HNSW](#tab/tab-hnsw)

```bash
go run hnsw.go
```

#### [DiskANN](#tab/tab-diskann)

```bash
go run diskann.go
```

----

You see the top hotels that match the vector search query and their similarity scores.

## View and manage data in Visual Studio Code

1. Select the [DocumentDB extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) in Visual Studio Code to connect to your Azure DocumentDB account.
1. View the data and indexes in the Hotels database.

## Clean up resources

Delete the resource group, Azure DocumentDB account, and Azure OpenAI resource when you don't need them to avoid extra costs.

## Related content

- [Vector store in Azure DocumentDB](vector-search.md)
- [Azure OpenAI embeddings](/azure/ai-services/openai/concepts/understand-embeddings)
- [MongoDB Go driver documentation](https://www.mongodb.com/docs/drivers/go/)
