---
title: "Quickstart - Vector Indexing with Go"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Go."
ms.reviewer: khelanmodi
ms.devlang: golang
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-go
  - devx-track-data-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with Go

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-go) on GitHub.

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

## Prerequisites

- An Azure subscription ([create one for free](https://azure.microsoft.com/free/))
- Azure DocumentDB vCore cluster with appropriate tier:
  - **IVF**: M10 or higher
  - **HNSW**: M30 or higher
  - **DiskANN**: M30 or higher
- [Azure OpenAI resource](https://learn.microsoft.com/azure/ai-services/openai/how-to/create-resource) with an embeddings model deployed
- [Go 1.21+](https://golang.org/dl/)
- Your preferred code editor

## Set up the project

1. Create and navigate to a new project directory:

```bash
mkdir documentdb-vector-quickstart
cd documentdb-vector-quickstart
```

2. Initialize a Go module:

```bash
go mod init documentdb-quickstart
```

3. Install required packages:

```bash
go get github.com/Azure/azure-sdk-for-go/sdk/azidentity
go get github.com/openai/openai-go/v3
go get go.mongodb.org/mongo-driver
go get github.com/joho/godotenv
```

4. Create a `.env` file with your credentials:

```env
MONGO_CLUSTER_NAME=your-cluster-name
AZURE_OPENAI_EMBEDDING_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
EMBEDDED_FIELD=DescriptionVector
EMBEDDING_DIMENSIONS=1536
DATA_FILE_WITH_VECTORS=data/Hotels_Vector.json
LOAD_SIZE_BATCH=100
```

## Create an IVF index

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

	"github.com/Azure/azure-sdk-for-go/sdk/azidentity"
	"github.com/openai/openai-go/v3"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// CreateIVFVectorIndex creates an IVF (Inverted File) vector index on the specified field
func CreateIVFVectorIndex(ctx context.Context, collection *mongo.Collection, vectorField string, dimensions int) error {
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
	mongoURI := fmt.Sprintf("mongodb+srv://%s.mongocluster.cosmos.azure.com/", os.Getenv("MONGO_CLUSTER_NAME"))
	opts := options.Client().ApplyURI(mongoURI)
	
	mongoClient, err := mongo.Connect(ctx, opts)
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", err)
	}
	defer mongoClient.Disconnect(ctx)

	// Create Azure OpenAI client
	openAIClient := openai.NewClient(
		os.Getenv("AZURE_OPENAI_EMBEDDING_KEY"),
	)

	// Access database and collection
	database := mongoClient.Database("Hotels")
	collection := database.Collection("hotels_ivf")

	// Create IVF vector index
	dimensions, _ := strconv.Atoi(os.Getenv("EMBEDDING_DIMENSIONS"))
	err = CreateIVFVectorIndex(ctx, collection, os.Getenv("EMBEDDED_FIELD"), dimensions)
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

## Create an HNSW index

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `hnsw.go`:

```go
// Similar to ivf.go above, but with these index options:
"cosmosSearchOptions", bson.D{
	// HNSW algorithm configuration
	{"kind", "vector-hnsw"},

	// Vector dimensions must match the embedding model
	{"dimensions", dimensions},

	// Cosine similarity works well with text embeddings
	{"similarity", "COS"},

	// Maximum connections per node in the graph (parameter 'm')
	// Higher values improve recall but increase memory usage and build time
	{"m", 16},

	// Size of the candidate list during construction
	// Higher values improve index quality but slow down building
	{"efConstruction", 64},
}
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity. Higher values (e.g., 32) improve recall but increase memory.
- **efConstruction**: Affects index build time and quality. Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

## Create a DiskANN index

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `diskann.go`:

```go
// Similar to ivf.go and hnsw.go above, but with these index options:
"cosmosSearchOptions", bson.D{
	// DiskANN algorithm configuration
	{"kind", "vector-diskann"},

	// Vector dimensions must match the embedding model
	{"dimensions", dimensions},

	// Vector similarity metric - cosine is good for text embeddings
	{"similarity", "COS"},

	// Maximum degree: number of edges per node in the graph
	// Higher values improve accuracy but increase memory usage
	{"maxDegree", 20},

	// Build parameter: candidates evaluated during index construction
	// Higher values improve index quality but increase build time
	{"lBuild", 10},
}
```

Key parameters:
- **maxDegree**: Number of edges per node in the graph. Higher values improve accuracy.
- **lBuild**: Number of candidate neighbors evaluated during construction. Affects index quality.
- **Cluster tier**: Requires M30 or higher.

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
# Run IVF example
go run ivf.go

# Run HNSW example
go run hnsw.go

# Run DiskANN example
go run diskann.go
```

## Clean up resources

When you're done, delete the DocumentDB cluster and OpenAI resource from the Azure Portal to avoid ongoing charges.

## Next steps

- [DocumentDB Vector Search Documentation](https://learn.microsoft.com/azure/cosmos-db/mongodb/vcore/vector-search)
- [Azure OpenAI Embeddings Documentation](https://learn.microsoft.com/azure/ai-services/openai/concepts/understand-embeddings)
- [MongoDB Go Driver Documentation](https://www.mongodb.com/docs/drivers/go/)
- Article 1: Getting Started with Vector Search
- Article 3: Performance Tuning and Optimization
