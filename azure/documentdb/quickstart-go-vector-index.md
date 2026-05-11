---
title: Quickstart - Vector index with Go
description: Compare DiskANN, HNSW, and IVF vector index algorithms using Go to select and tune the optimal index for your workload
ms.devlang: golang
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.custom: sfi-ropc-nochange
ai-usage: ai-generated
author: diberry
ms.author: diberry
ms.service: azure-documentdb
---

# Quickstart: Vector index with Go in Azure DocumentDB

This quickstart walks you through building a Go application that compares all three vector index algorithms (DiskANN, HNSW, and IVF) side by side with different similarity functions to help you choose the best configuration for your workload. The sample uses a hotels dataset with pre-calculated embeddings from the `text-embedding-3-small` model.



Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-go) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites](includes/prerequisite-quickstart-vector-index.md)]

- [Go](https://go.dev/doc/install) 1.22 or greater

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

   ```bash
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

   ```bash
   go get github.com/Azure/azure-sdk-for-go/sdk/azcore@v1.20.0
   go get github.com/Azure/azure-sdk-for-go/sdk/azidentity@v1.13.1
   go get github.com/openai/openai-go/v3@v3.12.0
   go get go.mongodb.org/mongo-driver@v1.17.6
   go mod tidy
   ```

   - `azcore`: Core Azure SDK functionality for Go
   - `azidentity`: Azure Identity library for passwordless authentication with DefaultAzureCredential
   - `openai-go/v3`: OpenAI client library with Azure support to generate embeddings
   - `mongo-driver`: Official MongoDB driver for Go to work with DocumentDB

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

4. Create a `.env` file for environment variables in `select-algorithm-go`:

   ```bash
   # Azure OpenAI Embedding Configuration
   AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
   AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
   AZURE_OPENAI_EMBEDDING_ENDPOINT=https://your-openai-resource.openai.azure.com/

   # Data File Configuration
   DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
   EMBEDDED_FIELD=DescriptionVector
   EMBEDDING_DIMENSIONS=1536
   LOAD_SIZE_BATCH=100

   # DocumentDB Configuration
   MONGO_CLUSTER_NAME=your-cluster-name

   # Algorithm Selection
   # ALGORITHM: "all" | "diskann" | "hnsw" | "ivf"
   ALGORITHM=all

   # SIMILARITY: "all" | "COS" | "L2" | "IP"
   SIMILARITY=COS

   # Database name
   AZURE_DOCUMENTDB_DATABASENAME=Hotels
   ```

   For the passwordless authentication used in this article, replace the placeholder values in the `.env` file with your own information:

   - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
   - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB cluster name (not the full connection string, just the name)

   Verify the `.env` file was created:

   ### [Bash](#tab/bash)

   ```bash
   cat .env
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-Content .env
   ```

   ---

   You should always prefer passwordless authentication. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate Go apps to Azure services by using the Azure SDK for Go](/azure/developer/go/azure-sdk-authentication).

## Create code files

Create a `src` directory and add the main application file:

### [Bash](#tab/bash)

```bash
mkdir src
touch src/main.go
```

### [PowerShell](#tab/powershell)

```powershell
New-Item -ItemType Directory -Name src
New-Item -ItemType File -Path src/main.go
```

---

When you're done, the project structure should look like this:

```
├── data/
│   ├── Hotels.json              # Source hotel data (without vectors)
│   └── Hotels_Vector.json       # Hotel data with vector embeddings
└── select-algorithm-go/
    ├── src/
    │   └── main.go              # Main application comparing all algorithms
    ├── go.mod                   # Go module dependencies
    ├── go.sum                   # Dependency checksums
    └── .env                     # Environment configuration
```

## Create the algorithm comparison code

Paste the following code into the `src/main.go` file.

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
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

// Algorithm represents a vector index algorithm type
type Algorithm string
type Similarity string

const (
	DiskANN Algorithm = "diskann"
	HNSW    Algorithm = "hnsw"
	IVF     Algorithm = "ivf"
)

const (
	COS Similarity = "COS"  // Cosine similarity - best for text embeddings
	L2  Similarity = "L2"   // L2 Euclidean distance
	IP  Similarity = "IP"   // Inner product similarity
)

var (
	AllAlgorithms   = []Algorithm{DiskANN, HNSW, IVF}
	AllSimilarities = []Similarity{COS, L2, IP}
)

var AlgorithmLabels = map[Algorithm]string{
	DiskANN: "DiskANN",
	HNSW:    "HNSW",
	IVF:     "IVF",
}

// CollectionTarget defines which collection to create and query
type CollectionTarget struct {
	CollectionName string
	Algorithm      Algorithm
	Similarity     Similarity
}

// SearchResult holds a single vector search result with score
type SearchResult struct {
	Document interface{} `bson:"document"`
	Score    float64     `bson:"score"`
}

// ComparisonResult stores results for one algorithm/similarity combination
type ComparisonResult struct {
	CollectionName string
	Algorithm      string
	Similarity     string
	SearchResults  []SearchResult
	LatencyMs      int64
}

// getEnvOrDefault retrieves an environment variable or returns a default value
func getEnvOrDefault(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

// getTargetCollections determines which collections to create based on env vars
func getTargetCollections(algorithmEnv, similarityEnv string) ([]CollectionTarget, error) {
	algorithmEnv = strings.ToLower(strings.TrimSpace(algorithmEnv))
	similarityEnv = strings.ToUpper(strings.TrimSpace(similarityEnv))

	// Parse algorithm selection
	algorithms := []Algorithm{}
	if algorithmEnv == "all" {
		algorithms = AllAlgorithms
	} else {
		algorithms = []Algorithm{Algorithm(algorithmEnv)}
	}

	// Parse similarity selection
	similarities := []Similarity{}
	if similarityEnv == "all" {
		similarities = AllSimilarities
	} else {
		similarities = []Similarity{Similarity(similarityEnv)}
	}

	// Validate and build target list
	targets := []CollectionTarget{}
	for _, alg := range algorithms {
		validAlg := false
		for _, validAlgorithm := range AllAlgorithms {
			if alg == validAlgorithm {
				validAlg = true
				break
			}
		}
		if !validAlg {
			return nil, fmt.Errorf("invalid ALGORITHM '%s'. Must be one of: all, diskann, hnsw, ivf", alg)
		}

		for _, sim := range similarities {
			validSim := false
			for _, validSimilarity := range AllSimilarities {
				if sim == validSimilarity {
					validSim = true
					break
				}
			}
			if !validSim {
				return nil, fmt.Errorf("invalid SIMILARITY '%s'. Must be one of: all, COS, L2, IP", sim)
			}

			// Generate collection name: hotels_diskann_cos, hotels_hnsw_l2, etc.
			targets = append(targets, CollectionTarget{
				CollectionName: fmt.Sprintf("hotels_%s_%s", alg, strings.ToLower(string(sim))),
				Algorithm:      alg,
				Similarity:     sim,
			})
		}
	}

	return targets, nil
}

// getIndexOptions builds the createIndexes command for a specific algorithm
func getIndexOptions(collectionName, indexName, embeddedField string, dimensions int, algorithm Algorithm, similarity Similarity) bson.D {
	// Base cosmosSearch options for all algorithms
	cosmosSearchOptions := bson.D{
		{"dimensions", dimensions},
		{"similarity", string(similarity)},
	}

	// Add algorithm-specific options
	switch algorithm {
	case DiskANN:
		// DiskANN: disk-based approximate nearest neighbor search
		cosmosSearchOptions = append(bson.D{{"kind", "vector-diskann"}}, cosmosSearchOptions...)
		// maxDegree: maximum number of graph edges per node (higher = better recall, more memory)
		cosmosSearchOptions = append(cosmosSearchOptions, bson.E{"maxDegree", 20})
		// lBuild: candidate list size during index construction (higher = better index quality, slower build)
		cosmosSearchOptions = append(cosmosSearchOptions, bson.E{"lBuild", 10})
	case HNSW:
		// HNSW: hierarchical navigable small world graph
		cosmosSearchOptions = append(bson.D{{"kind", "vector-hnsw"}}, cosmosSearchOptions...)
		// m: number of bidirectional links per node (higher = better recall, more memory)
		cosmosSearchOptions = append(cosmosSearchOptions, bson.E{"m", 16})
		// efConstruction: candidate list size during construction (higher = better index quality, slower build)
		cosmosSearchOptions = append(cosmosSearchOptions, bson.E{"efConstruction", 64})
	case IVF:
		// IVF: inverted file index with clustering
		cosmosSearchOptions = append(bson.D{{"kind", "vector-ivf"}}, cosmosSearchOptions...)
		// numLists: number of clusters to partition vectors into (higher = better accuracy, slower for small datasets)
		cosmosSearchOptions = append(cosmosSearchOptions, bson.E{"numLists", 1})
	}

	// Return the full createIndexes command
	return bson.D{
		{"createIndexes", collectionName},
		{"indexes", []bson.D{
			{
				{"name", indexName},
				{"key", bson.D{{embeddedField, "cosmosSearch"}}},
				{"cosmosSearchOptions", cosmosSearchOptions},
			},
		}},
	}
}

// getSearchPipeline builds the vector search aggregation pipeline
func getSearchPipeline(queryEmbedding []float64, embeddedField string, k int, algorithm Algorithm) []bson.M {
	// Base cosmosSearch configuration
	cosmosSearch := bson.M{
		"vector": queryEmbedding,  // Query vector
		"path":   embeddedField,   // Field containing document vectors
		"k":      k,               // Number of results to return
	}

	// Add algorithm-specific search parameters
	switch algorithm {
	case DiskANN:
		// lSearch: candidate list size during search (higher = better recall, slower search)
		cosmosSearch["lSearch"] = 100
	case HNSW:
		// efSearch: candidate list size during search (higher = better recall, slower search)
		cosmosSearch["efSearch"] = 80
	case IVF:
		// nProbes: number of clusters to search (higher = better recall, slower search)
		cosmosSearch["nProbes"] = 1
	}

	// Build the aggregation pipeline
	return []bson.M{
		{"$search": bson.M{"cosmosSearch": cosmosSearch}},
		{"$project": bson.M{
			"score":    bson.M{"$meta": "searchScore"},  // Include similarity score
			"document": "$$ROOT",                        // Include full document
		}},
	}
}

// getClientsPasswordless creates MongoDB and OpenAI clients using DefaultAzureCredential
func getClientsPasswordless() (*mongo.Client, openai.Client, error) {
	ctx := context.Background()

	// Get DocumentDB cluster name from environment
	clusterName := os.Getenv("MONGO_CLUSTER_NAME")
	if clusterName == "" {
		return nil, openai.Client{}, fmt.Errorf("MONGO_CLUSTER_NAME environment variable is required")
	}

	// Create Azure credential for passwordless authentication
	credential, err := azidentity.NewDefaultAzureCredential(nil)
	if err != nil {
		return nil, openai.Client{}, fmt.Errorf("failed to create Azure credential: %w", err)
	}

	// Build MongoDB connection URI
	mongoURI := fmt.Sprintf("mongodb+srv://%s.mongocluster.cosmos.azure.com/", clusterName)

	// OIDC callback function to provide Azure AD token
	oidcCallback := func(ctx context.Context, args *options.OIDCArgs) (*options.OIDCCredential, error) {
		// Request token with DocumentDB scope
		scope := "https://ossrdbms-aad.database.windows.net/.default"
		token, err := credential.GetToken(ctx, policy.TokenRequestOptions{
			Scopes: []string{scope},
		})
		if err != nil {
			return nil, fmt.Errorf("failed to get token with scope %s: %w", scope, err)
		}

		return &options.OIDCCredential{
			AccessToken: token.Token,
		}, nil
	}

	// Configure MongoDB client with OIDC authentication
	clientOptions := options.Client().
		ApplyURI(mongoURI).
		SetConnectTimeout(30 * time.Second).
		SetServerSelectionTimeout(30 * time.Second).
		SetRetryWrites(true).
		SetAuth(options.Credential{
			AuthMechanism: "MONGODB-OIDC",
			AuthMechanismProperties: map[string]string{
				"TOKEN_RESOURCE": "https://ossrdbms-aad.database.windows.net",
			},
			OIDCMachineCallback: oidcCallback,
		})

	// Connect to MongoDB
	mongoClient, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		return nil, openai.Client{}, fmt.Errorf("failed to connect to MongoDB: %w", err)
	}

	// Get Azure OpenAI endpoint
	azureOpenAIEndpoint := os.Getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT")
	if azureOpenAIEndpoint == "" {
		return nil, openai.Client{}, fmt.Errorf("AZURE_OPENAI_EMBEDDING_ENDPOINT environment variable is required")
	}

	// Create Azure OpenAI client with token-based authentication
	openAIClient := openai.NewClient(
		option.WithBaseURL(fmt.Sprintf("%s/openai/v1", azureOpenAIEndpoint)),
		azure.WithTokenCredential(credential))

	return mongoClient, openAIClient, nil
}

// readFileReturnJSON reads a JSON file and returns parsed data
func readFileReturnJSON(filePath string) ([]map[string]interface{}, error) {
	file, err := os.ReadFile(filePath)
	if err != nil {
		return nil, fmt.Errorf("error reading file '%s': %w", filePath, err)
	}

	var data []map[string]interface{}
	err = json.Unmarshal(file, &data)
	if err != nil {
		return nil, fmt.Errorf("error parsing JSON in file '%s': %w", filePath, err)
	}

	return data, nil
}

// insertData inserts documents in batches and creates standard indexes
func insertData(ctx context.Context, collection *mongo.Collection, data []map[string]interface{}, batchSize int) (int, int, error) {
	totalDocuments := len(data)
	insertedCount := 0
	failedCount := 0

	// Insert in batches to avoid timeout issues
	for i := 0; i < totalDocuments; i += batchSize {
		end := i + batchSize
		if end > totalDocuments {
			end = totalDocuments
		}

		batch := data[i:end]

		// Convert to []interface{} for InsertMany
		documents := make([]interface{}, len(batch))
		for j, doc := range batch {
			documents[j] = doc
		}

		// Insert batch with unordered writes (continue on error)
		result, err := collection.InsertMany(ctx, documents, options.InsertMany().SetOrdered(false))
		if err != nil {
			var bulkErr mongo.BulkWriteException
			if errors.As(err, &bulkErr) {
				// Some documents succeeded
				inserted := len(batch) - len(bulkErr.WriteErrors)
				insertedCount += inserted
				failedCount += len(bulkErr.WriteErrors)
			} else {
				// All documents failed
				failedCount += len(batch)
			}
		} else {
			insertedCount += len(result.InsertedIDs)
		}

		// Rate limiting: pause between batches
		if i+batchSize < totalDocuments {
			time.Sleep(100 * time.Millisecond)
		}
	}

	// Create standard indexes on common query fields
	indexColumns := []string{"HotelId", "Category", "Description", "Description_fr"}
	for _, col := range indexColumns {
		indexModel := mongo.IndexModel{
			Keys: bson.D{{Key: col, Value: 1}},
		}
		_, err := collection.Indexes().CreateOne(ctx, indexModel)
		if err != nil {
			fmt.Printf("Warning: Could not create index on %s: %v\n", col, err)
		}
	}

	return insertedCount, failedCount, nil
}

// generateEmbedding creates a vector embedding for the given text
func generateEmbedding(ctx context.Context, client openai.Client, text, modelName string) ([]float64, error) {
	// Call Azure OpenAI embeddings endpoint
	resp, err := client.Embeddings.New(ctx, openai.EmbeddingNewParams{
		Input: openai.EmbeddingNewParamsInputUnion{
			OfString: openai.String(text),
		},
		Model: modelName,
	})
	if err != nil {
		return nil, fmt.Errorf("failed to generate embedding: %w", err)
	}

	if len(resp.Data) == 0 {
		return nil, fmt.Errorf("no embedding data received")
	}

	// Convert float32 to float64
	embedding := make([]float64, len(resp.Data[0].Embedding))
	for i, v := range resp.Data[0].Embedding {
		embedding[i] = float64(v)
	}

	return embedding, nil
}

// printComparisonTable displays results in a formatted table
func printComparisonTable(results []ComparisonResult) {
	fmt.Println("\n╔══════════════════════════════════════════════════════════════════════════════════╗")
	fmt.Println("║                     Vector Algorithm Comparison Results                         ║")
	fmt.Println("╠══════════════════════════════════════════════════════════════════════════════════╣")

	fmt.Printf("║ %-12s%-14s%-24s%-12s%-14s║\n", "Algorithm", "Similarity", "Top Result", "Score", "Latency(ms)")
	fmt.Println("╠══════════════════════════════════════════════════════════════════════════════════╣")

	// Print summary table
	for _, r := range results {
		topName := "N/A"
		topScore := "N/A"

		if len(r.SearchResults) > 0 {
			topResult := r.SearchResults[0]
			doc := topResult.Document.(bson.D)
			for _, elem := range doc {
				if elem.Key == "HotelName" {
					hotelName := fmt.Sprintf("%v", elem.Value)
					if len(hotelName) > 22 {
						hotelName = hotelName[:22]
					}
					topName = hotelName
					break
				}
			}
			topScore = fmt.Sprintf("%.4f", topResult.Score)
		}

		fmt.Printf("║ %-12s%-14s%-24s%-12s%-14s║\n",
			r.Algorithm,
			r.Similarity,
			topName,
			topScore,
			fmt.Sprintf("%d", r.LatencyMs))
	}

	fmt.Println("╚══════════════════════════════════════════════════════════════════════════════════╝")

	// Print detailed results for each algorithm/similarity combination
	for _, r := range results {
		fmt.Printf("\n--- %s / %s (%s) ---\n", r.Algorithm, r.Similarity, r.CollectionName)
		if len(r.SearchResults) == 0 {
			fmt.Println("  No results.")
			continue
		}
		for i, item := range r.SearchResults {
			doc := item.Document.(bson.D)
			var hotelName string
			for _, elem := range doc {
				if elem.Key == "HotelName" {
					hotelName = fmt.Sprintf("%v", elem.Value)
					break
				}
			}
			fmt.Printf("  %d. %s, Score: %.4f\n", i+1, hotelName, item.Score)
		}
		fmt.Printf("  Latency: %dms\n", r.LatencyMs)
	}
}

func main() {
	// Set environment variables before running, or source your .env file manually:
	//   export $(grep -v '^#' .env | xargs)  # Linux/macOS
	//   Get-Content .env | ForEach-Object { if ($_ -match '^\s*([^#][^=]+)=(.*)') { [System.Environment]::SetEnvironmentVariable($Matches[1].Trim(), $Matches[2].Trim()) } }  # PowerShell

	ctx := context.Background()

	// Load configuration from environment variables
	dbName := getEnvOrDefault("AZURE_DOCUMENTDB_DATABASENAME", "Hotels")
	embeddedField := getEnvOrDefault("EMBEDDED_FIELD", "DescriptionVector")
	embeddingDimensions, err := strconv.Atoi(getEnvOrDefault("EMBEDDING_DIMENSIONS", "1536"))
	if err != nil {
		log.Fatalf("Invalid value for EMBEDDING_DIMENSIONS: %v", err)
	}
	dataFile := getEnvOrDefault("DATA_FILE_WITH_VECTORS", "../../data/Hotels_Vector.json")
	deployment := os.Getenv("AZURE_OPENAI_EMBEDDING_MODEL")
	if deployment == "" {
		log.Fatal("AZURE_OPENAI_EMBEDDING_MODEL environment variable is required")
	}
	batchSize, err := strconv.Atoi(getEnvOrDefault("LOAD_SIZE_BATCH", "100"))
	if err != nil {
		log.Fatalf("Invalid value for LOAD_SIZE_BATCH: %v", err)
	}
	algorithmEnv := getEnvOrDefault("ALGORITHM", "all")
	similarityEnv := getEnvOrDefault("SIMILARITY", "COS")
	searchQuery := "quintessential lodging near running trails, eateries, retail"

	// Determine which collections to create and query
	targets, err := getTargetCollections(algorithmEnv, similarityEnv)
	if err != nil {
		log.Fatal(err)
	}

	collectionNames := []string{}
	for _, t := range targets {
		collectionNames = append(collectionNames, t.CollectionName)
	}

	// Print configuration summary
	fmt.Println("\nVector Algorithm Comparison")
	fmt.Printf("   Database: %s\n", dbName)
	fmt.Printf("   Algorithms: %s\n", algorithmEnv)
	fmt.Printf("   Similarity: %s\n", similarityEnv)
	fmt.Printf("   Collections to query: %s\n", strings.Join(collectionNames, ", "))
	fmt.Printf("   Search query: \"%s\"\n\n", searchQuery)

	// Initialize MongoDB and Azure OpenAI clients with passwordless auth
	fmt.Println("Initializing MongoDB and Azure OpenAI clients...")
	mongoClient, azureOpenAIClient, err := getClientsPasswordless()
	if err != nil {
		log.Fatalf("Failed to initialize clients: %v", err)
	}
	defer mongoClient.Disconnect(context.Background())

	db := mongoClient.Database(dbName)

	// Load hotel data with embeddings
	fmt.Printf("Loading data from %s...\n", dataFile)
	data, err := readFileReturnJSON(dataFile)
	if err != nil {
		log.Fatalf("Failed to load data: %v", err)
	}
	fmt.Printf("Loaded %d documents\n", len(data))

	// Generate embedding for the search query
	fmt.Println("Generating query embedding...")
	queryEmbedding, err := generateEmbedding(ctx, azureOpenAIClient, searchQuery, deployment)
	if err != nil {
		log.Fatalf("Failed to generate embedding: %v", err)
	}
	fmt.Printf("Query embedding: %d dimensions\n\n", len(queryEmbedding))

	// Store results for comparison
	comparisonResults := []ComparisonResult{}

	// Process each algorithm/similarity combination
	for _, target := range targets {
		fmt.Printf("\n━━━ %s / %s ━━━\n", AlgorithmLabels[target.Algorithm], target.Similarity)
		fmt.Printf("Collection: %s\n", target.CollectionName)

		// Drop existing collection to ensure clean state
		if err := db.Collection(target.CollectionName).Drop(ctx); err != nil {
			log.Printf("Warning: failed to drop collection %s: %v (may not exist)", target.CollectionName, err)
		}

		collection := db.Collection(target.CollectionName)
		fmt.Printf("Created collection: %s\n", target.CollectionName)

		// Insert hotel data
		inserted, failed, err := insertData(ctx, collection, data, batchSize)
		if err != nil {
			fmt.Printf("Error inserting data: %v\n", err)
			continue
		}
		fmt.Printf("Inserted: %d/%d\n", inserted, len(data))
		if failed > 0 {
			fmt.Printf("Failed: %d\n", failed)
		}

		// Create vector index for this algorithm
		indexName := fmt.Sprintf("vectorIndex_%s_%s", target.Algorithm, strings.ToLower(string(target.Similarity)))
		indexOptions := getIndexOptions(
			target.CollectionName,
			indexName,
			embeddedField,
			embeddingDimensions,
			target.Algorithm,
			target.Similarity,
		)

		var result bson.M
		err = db.RunCommand(ctx, indexOptions).Decode(&result)
		if err != nil {
			fmt.Printf("Error creating vector index: %v\n", err)
			continue
		}
		fmt.Printf("Created vector index: %s\n", indexName)

		// Execute vector search
		fmt.Println("Executing vector search...")
		startTime := time.Now()

		pipeline := getSearchPipeline(queryEmbedding, embeddedField, 5, target.Algorithm)
		cursor, err := collection.Aggregate(ctx, pipeline)
		if err != nil {
			fmt.Printf("Error performing vector search: %v\n", err)
			continue
		}

		var searchResults []SearchResult
		for cursor.Next(ctx) {
			var result SearchResult
			if err := cursor.Decode(&result); err != nil {
				fmt.Printf("Warning: Could not decode result: %v\n", err)
				continue
			}
			searchResults = append(searchResults, result)
		}
		cursor.Close(ctx)

		latencyMs := time.Since(startTime).Milliseconds()

		// Store results for comparison
		comparisonResults = append(comparisonResults, ComparisonResult{
			CollectionName: target.CollectionName,
			Algorithm:      AlgorithmLabels[target.Algorithm],
			Similarity:     string(target.Similarity),
			SearchResults:  searchResults,
			LatencyMs:      latencyMs,
		})

		fmt.Printf("[OK] %d results, %dms\n", len(searchResults), latencyMs)
	}

	// Display comparison table
	if len(comparisonResults) > 0 {
		printComparisonTable(comparisonResults)
	}

	fmt.Println("\nDone.")
}
```

This code provides a complete vector algorithm comparison application with these key features:

- **Passwordless authentication**: Uses `DefaultAzureCredential` for both Azure OpenAI and DocumentDB via OIDC
- **Three vector algorithms**: Implements DiskANN, HNSW, and IVF with algorithm-specific tuning parameters
- **Three similarity functions**: Supports COS (cosine), L2 (Euclidean), and IP (inner product)
- **Flexible configuration**: Use environment variables to compare all algorithms or test specific combinations
- **Performance measurement**: Tracks query latency for each algorithm/similarity pair
- **Comparison output**: Generates a formatted table showing results side by side
- **Production-ready patterns**: Includes batched insertion, error handling, and connection pooling

## Run the code

Before running the code, source your `.env` file to load environment variables into your shell session.

### [Bash](#tab/bash)

```bash
export $(grep -v '^#' .env | xargs)
```

### [PowerShell](#tab/powershell)

```powershell
Get-Content .env | ForEach-Object {
    if ($_ -match '^\s*([^#][^=]+)=(.*)') {
        [System.Environment]::SetEnvironmentVariable($Matches[1].Trim(), $Matches[2].Trim())
    }
}
```

---

After sourcing the environment variables, run the application:

```bash
go run src/main.go
```

The application will:

1. Connect to Azure DocumentDB and Azure OpenAI using passwordless authentication
2. Create separate collections for each algorithm/similarity combination
3. Insert the hotel data into each collection
4. Create a vector index on each collection with algorithm-specific parameters
5. Generate an embedding for the search query
6. Execute vector searches across all collections
7. Display a comparison table with results and latencies

Expected output:

```
Vector Algorithm Comparison
   Database: Hotels
   Algorithms: all
   Similarity: COS
   Collections to query: hotels_diskann_cos, hotels_hnsw_cos, hotels_ivf_cos
   Search query: "quintessential lodging near running trails, eateries, retail"

Initializing MongoDB and Azure OpenAI clients...
Loading data from ../data/Hotels_Vector.json...
Loaded 50 documents
Generating query embedding...
Query embedding: 1536 dimensions

━━━ DiskANN / COS ━━━
Collection: hotels_diskann_cos
Created collection: hotels_diskann_cos
Inserted: 50/50
Created vector index: vectorIndex_diskann_cos
Executing vector search...
[OK] 5 results, 42ms

━━━ HNSW / COS ━━━
Collection: hotels_hnsw_cos
Created collection: hotels_hnsw_cos
Inserted: 50/50
Created vector index: vectorIndex_hnsw_cos
Executing vector search...
[OK] 5 results, 38ms

━━━ IVF / COS ━━━
Collection: hotels_ivf_cos
Created collection: hotels_ivf_cos
Inserted: 50/50
Created vector index: vectorIndex_ivf_cos
Executing vector search...
[OK] 5 results, 35ms

╔══════════════════════════════════════════════════════════════════════════════════╗
║                     Vector Algorithm Comparison Results                         ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ Algorithm   Similarity    Top Result              Score       Latency(ms)      ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ DiskANN     COS           Secret Point Motel       0.8562      42              ║
║ HNSW        COS           Secret Point Motel       0.8562      38              ║
║ IVF         COS           Secret Point Motel       0.8562      35              ║
╚══════════════════════════════════════════════════════════════════════════════════╝

--- DiskANN / COS (hotels_diskann_cos) ---
  1. Secret Point Motel, Score: 0.8562
  2. Countryside Hotel, Score: 0.8457
  3. Downtown Modern Hotel, Score: 0.8398
  4. Old Century Hotel, Score: 0.8321
  5. Save-the-Light Deluxe Inn, Score: 0.8298
  Latency: 42ms

--- HNSW / COS (hotels_hnsw_cos) ---
  1. Secret Point Motel, Score: 0.8562
  2. Countryside Hotel, Score: 0.8457
  3. Downtown Modern Hotel, Score: 0.8398
  4. Old Century Hotel, Score: 0.8321
  5. Save-the-Light Deluxe Inn, Score: 0.8298
  Latency: 38ms

--- IVF / COS (hotels_ivf_cos) ---
  1. Secret Point Motel, Score: 0.8562
  2. Countryside Hotel, Score: 0.8457
  3. Downtown Modern Hotel, Score: 0.8398
  4. Old Century Hotel, Score: 0.8321
  5. Save-the-Light Deluxe Inn, Score: 0.8298
  Latency: 35ms

Done.
```

## Understanding the results

The comparison table shows how different algorithms perform on the same dataset with the same query:

- **Algorithm**: DiskANN, HNSW, or IVF
- **Similarity**: The distance metric (COS, L2, or IP)
- **Top Result**: The highest-scoring hotel from the search
- **Score**: Similarity score (higher is better for COS and IP, lower is better for L2)
- **Latency**: Query execution time in milliseconds

### Choosing the right algorithm

Use this comparison to select the best algorithm for your workload:

**DiskANN** (disk-based approximate nearest neighbor):
- Best for: Large datasets that don't fit in memory
- Pros: Memory efficient, good recall with high dimensions
- Cons: Requires disk I/O, slower build time
- Tune: Increase `maxDegree` and `lBuild` for better accuracy, increase `lSearch` for better recall

**HNSW** (hierarchical navigable small world):
- Best for: High-speed queries with excellent recall
- Pros: Fastest queries, excellent recall, stable performance
- Cons: Higher memory usage than DiskANN
- Tune: Increase `m` and `efConstruction` for better index quality, increase `efSearch` for better recall

**IVF** (inverted file index):
- Best for: Large datasets with good clustering properties
- Pros: Fast queries, low memory overhead
- Cons: Recall depends on `numLists` and `nProbes` tuning
- Tune: Increase `numLists` for larger datasets, increase `nProbes` for better recall

### Choosing the right similarity function

The similarity function should match your embedding model and use case:

- **COS (Cosine similarity)**: Best for text embeddings and most OpenAI models. Measures angle between vectors (range: -1 to 1, higher is more similar)
- **L2 (Euclidean distance)**: Measures straight-line distance between vectors (lower is more similar). Good for spatial data
- **IP (Inner product)**: Measures alignment between vectors. Good when vector magnitudes are meaningful

For the `text-embedding-3-small` model used in this quickstart, **COS (cosine similarity) is recommended** because OpenAI embeddings are normalized and optimized for cosine similarity.

## Experiment with different configurations

You can compare different combinations by setting environment variables:

**Compare all algorithms with cosine similarity (default):**

```bash
# .env file
ALGORITHM=all
SIMILARITY=COS
```

**Compare all algorithms with all similarity functions (9 collections):**

```bash
# .env file
ALGORITHM=all
SIMILARITY=all
```

**Test only DiskANN with all similarity functions:**

```bash
# .env file
ALGORITHM=diskann
SIMILARITY=all
```

**Test only cosine similarity across all algorithms:**

```bash
# .env file
ALGORITHM=all
SIMILARITY=COS
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `server selection error` | Verify your connection string in `.env`. Ensure your IP is in the DocumentDB firewall rules. |
| `authentication failed` | Check credentials in connection string. Ensure `DefaultAzureCredential` is configured (run `az login`). |
| `go: module not found` | Run `go mod tidy` to resolve dependencies. |
| Build errors | Ensure Go 1.22+ is installed. Run `go version` to check. |
| Empty search results | The vector index may not be ready yet. The code includes retry logic, but larger datasets may need more time. |

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

1. Navigate to your DocumentDB resource in the Azure portal
2. Select **Data Explorer**
3. Right-click the **Hotels** database and select **Delete Database**

---

## Related content

- [Vector search overview](./vector-search.md)
- [ENN vector search](./enn-vector-search.md)
- [Product quantization](./product-quantization.md)
