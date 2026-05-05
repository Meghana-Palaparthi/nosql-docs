---
title: "Quickstart - Vector Indexing with Java"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Java."
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: java
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-java
  - devx-track-data-ai
  - devx-track-java-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with Java

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

This quickstart uses a sample hotel dataset in a JSON file with pre-calculated vectors from the `text-embedding-3-small` model. The dataset includes hotel names, locations, descriptions, and vector embeddings.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-java) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Index Quickstart](includes/prerequisite-quickstart-vector-index.md)]

- [Java 21](/java/openjdk/download) or later

- [Maven 3.6](https://maven.apache.org/download.cgi) or later

## Create data file with vectors

1. Create a new data directory for the hotels data file:

    ```bash
    mkdir data
    ```

1. Copy the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory.

## Set up the project

1. Create a Maven project:

    ```bash
    mvn archetype:generate \
      -DgroupId=com.azure.documentdb \
      -DartifactId=vector-quickstart \
      -DarchetypeArtifactId=maven-archetype-quickstart \
      -DinteractiveMode=false

    cd vector-quickstart
    ```

1. Update `pom.xml` with required dependencies:

    ```xml
    <dependencies>
      <!-- MongoDB Driver -->
      <dependency>
        <groupId>org.mongodb</groupId>
        <artifactId>mongodb-driver-sync</artifactId>
        <version>4.11.1</version>
      </dependency>

      <!-- Azure Identity -->
      <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>
        <version>1.11.1</version>
      </dependency>

      <!-- Azure OpenAI -->
      <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-ai-openai</artifactId>
        <version>1.0.0</version>
      </dependency>

      <!-- JSON processing -->
      <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.0</version>
      </dependency>

      <!-- Logging -->
      <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>2.0.9</version>
      </dependency>
    </dependencies>
    ```

1. Create a `.env` file with your configuration:

    ```env
    MONGO_CLUSTER_NAME=<your-cluster-name>
    AZURE_OPENAI_EMBEDDING_ENDPOINT=<your-azure-openai-endpoint>
    AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
    AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15
    EMBEDDED_FIELD=DescriptionVector
    EMBEDDING_DIMENSIONS=1536
    ```

    Replace the placeholder values with your own information:
    - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
    - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB resource name

## Create a vector index

### [IVF](#tab/tab-ivf)

IVF (Inverted File) is ideal for datasets with fewer than 10,000 documents. It partitions vectors into clusters for fast approximate search.

Create `src/main/java/com/azure/documentdb/IVF.java`:

```java
package com.azure.documentdb;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.MongoCredential;
import com.mongodb.client.AggregateIterable;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.ArrayList;
import java.util.List;

public class IVF {
    private static final String DATABASE_NAME = "Hotels";
    private static final String COLLECTION_NAME = "hotels_ivf";
    private static final String VECTOR_INDEX_NAME = "vectorIndex_ivf";

    public static void main(String[] args) {
        new IVF().run();
        System.exit(0);
    }

    public void run() {
        try (var mongoClient = createMongoClient()) {
            var openAIClient = createOpenAIClient();

            var database = mongoClient.getDatabase(DATABASE_NAME);
            var collection = database.getCollection(COLLECTION_NAME, Document.class);

            // Create vector index
            createVectorIndex(database, collection);

            // Perform vector search
            performVectorSearch(collection, openAIClient);

        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    private MongoClient createMongoClient() {
        var clusterName = System.getenv("MONGO_CLUSTER_NAME");
        var managedIdentityPrincipalId = System.getenv("AZURE_MANAGED_IDENTITY_PRINCIPAL_ID");
        var azureCredential = new DefaultAzureCredentialBuilder().build();

        // Create OIDC credential callback for Azure AD token
        MongoCredential.OidcCallback callback = (MongoCredential.OidcCallbackContext context) -> {
            var token = azureCredential.getToken(
                new com.azure.core.credential.TokenRequestContext()
                    .addScopes("https://ossrdbms-aad.database.windows.net/.default")
            ).block();

            if (token == null) {
                throw new RuntimeException("Failed to obtain Azure AD token");
            }

            return new MongoCredential.OidcCallbackResult(token.getToken());
        };

        var credential = MongoCredential.createOidcCredential(null)
            .withMechanismProperty("OIDC_CALLBACK", callback);

        var connectionString = new ConnectionString(
            String.format("mongodb+srv://%s@%s.mongocluster.cosmos.azure.com/?authMechanism=MONGODB-OIDC&tls=true&retrywrites=false&maxIdleTimeMS=120000",
                managedIdentityPrincipalId, clusterName)
        );

        var settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .credential(credential)
            .build();

        return MongoClients.create(settings);
    }

    private OpenAIClient createOpenAIClient() {
        var endpoint = System.getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT");
        var credential = new DefaultAzureCredentialBuilder().build();

        return new OpenAIClientBuilder()
            .endpoint(endpoint)
            .credential(credential)
            .buildClient();
    }

    private void createVectorIndex(MongoDatabase database, MongoCollection<Document> collection) {
        System.out.println("Creating IVF vector index...");

        // Use the native MongoDB command for DocumentDB vector indexes
        var indexCommand = new Document("createIndexes", COLLECTION_NAME)
            .append("indexes", List.of(
                new Document("name", VECTOR_INDEX_NAME)
                    .append("key", new Document(System.getenv("EMBEDDED_FIELD"), "cosmosSearch"))
                    .append("cosmosSearchOptions", new Document()
                        .append("kind", "vector-ivf")
                        .append("similarity", "COS")
                        .append("dimensions", Integer.parseInt(System.getenv("EMBEDDING_DIMENSIONS")))
                        .append("numLists", 10)  // Number of clusters
                    )
            ));

        try {
            var result = database.runCommand(indexCommand);
            System.out.println("IVF vector index created successfully");
        } catch (Exception e) {
            System.err.println("Error creating index: " + e.getMessage());
            throw new RuntimeException(e);
        }
    }

    private void performVectorSearch(MongoCollection<Document> collection, OpenAIClient openAIClient) {
        System.out.println("Performing vector search...");

        // Create embedding for query
        var query = "quintessential lodging near running trails, eateries, retail";
        var embeddingResponse = openAIClient.getEmbeddingsClient(
            System.getenv("AZURE_OPENAI_EMBEDDING_MODEL")
        ).generateEmbedding(new EmbeddingsOptions(List.of(query)));

        var embedding = embeddingResponse.getValue().getData().get(0).getEmbedding();

        // Build aggregation pipeline for vector search
        var pipeline = List.of(
            new Document("$search", new Document()
                .append("cosmosSearch", new Document()
                    .append("vector", embedding)
                    .append("path", System.getenv("EMBEDDED_FIELD"))
                    .append("k", 5)
                )
            ),
            new Document("$project", new Document()
                .append("score", new Document("$meta", "searchScore"))
                .append("document", "$$ROOT")
            )
        );

        // Execute search
        AggregateIterable<Document> results = collection.aggregate(pipeline);

        System.out.println("\nSearch Results:");
        int count = 0;
        for (Document result : results) {
            count++;
            Document doc = (Document) result.get("document");
            double score = (double) result.get("score");
            System.out.printf("%d. %s, Score: %.4f%n", count, 
                doc.getString("HotelName"), score);
        }
    }
}
```

#### [HNSW](#tab/tab-hnsw)

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `src/main/java/com/azure/documentdb/HNSW.java`:

```java
package com.azure.documentdb;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.mongodb.MongoCredential;
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.AggregateIterable;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.List;

public class HNSW {
    private static final String DATABASE_NAME = "Hotels";
    private static final String COLLECTION_NAME = "hotels_hnsw";
    private static final String VECTOR_INDEX_NAME = "vectorIndex_hnsw";

    public static void main(String[] args) {
        new HNSW().run();
        System.exit(0);
    }

    public void run() {
        try (var mongoClient = createMongoClient()) {
            var openAIClient = createOpenAIClient();
            var database = mongoClient.getDatabase(DATABASE_NAME);
            var collection = database.getCollection(COLLECTION_NAME, Document.class);

            createVectorIndex(database, collection);
            performVectorSearch(collection, openAIClient);

        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    private MongoClient createMongoClient() {
        var clusterName = System.getenv("MONGO_CLUSTER_NAME");
        var managedIdentityPrincipalId = System.getenv("AZURE_MANAGED_IDENTITY_PRINCIPAL_ID");
        var azureCredential = new DefaultAzureCredentialBuilder().build();

        MongoCredential.OidcCallback callback = (MongoCredential.OidcCallbackContext context) -> {
            var token = azureCredential.getToken(
                new com.azure.core.credential.TokenRequestContext()
                    .addScopes("https://ossrdbms-aad.database.windows.net/.default")
            ).block();
            if (token == null) throw new RuntimeException("Failed to obtain Azure AD token");
            return new MongoCredential.OidcCallbackResult(token.getToken());
        };

        var credential = MongoCredential.createOidcCredential(null)
            .withMechanismProperty("OIDC_CALLBACK", callback);

        var connectionString = new ConnectionString(
            String.format("mongodb+srv://%s@%s.mongocluster.cosmos.azure.com/?authMechanism=MONGODB-OIDC&tls=true&retrywrites=false&maxIdleTimeMS=120000",
                managedIdentityPrincipalId, clusterName));

        var settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .credential(credential)
            .build();

        return MongoClients.create(settings);
    }

    private OpenAIClient createOpenAIClient() {
        var endpoint = System.getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT");
        var credential = new DefaultAzureCredentialBuilder().build();
        return new com.azure.ai.openai.OpenAIClientBuilder()
            .endpoint(endpoint).credential(credential).buildClient();
    }

    private void createVectorIndex(MongoDatabase database, MongoCollection<Document> collection) {
        System.out.println("Creating HNSW vector index...");

        var indexCommand = new Document("createIndexes", COLLECTION_NAME)
            .append("indexes", List.of(
                new Document("name", VECTOR_INDEX_NAME)
                    .append("key", new Document(System.getenv("EMBEDDED_FIELD"), "cosmosSearch"))
                    .append("cosmosSearchOptions", new Document()
                        .append("kind", "vector-hnsw")
                        .append("similarity", "COS")
                        .append("dimensions", Integer.parseInt(System.getenv("EMBEDDING_DIMENSIONS")))
                        // Maximum connections per node (2-100, default 16)
                        .append("m", 16)
                        // Candidate list size during construction (4-1000, default 64)
                        .append("efConstruction", 64)
                    )
            ));

        try {
            database.runCommand(indexCommand);
            System.out.println("HNSW vector index created successfully");
        } catch (Exception e) {
            System.err.println("Error creating index: " + e.getMessage());
            throw new RuntimeException(e);
        }
    }

    private void performVectorSearch(MongoCollection<Document> collection, OpenAIClient openAIClient) {
        System.out.println("Performing HNSW vector search...");

        var query = "quintessential lodging near running trails, eateries, retail";
        var embeddingResponse = openAIClient.getEmbeddingsClient(
            System.getenv("AZURE_OPENAI_EMBEDDING_MODEL")
        ).generateEmbedding(new EmbeddingsOptions(List.of(query)));
        var embedding = embeddingResponse.getValue().getData().get(0).getEmbedding();

        var pipeline = List.of(
            new Document("$search", new Document()
                .append("cosmosSearch", new Document()
                    .append("vector", embedding)
                    .append("path", System.getenv("EMBEDDED_FIELD"))
                    .append("k", 5))),
            new Document("$project", new Document()
                .append("score", new Document("$meta", "searchScore"))
                .append("document", "$$ROOT"))
        );

        AggregateIterable<Document> results = collection.aggregate(pipeline);

        System.out.println("\nHNSW Search Results:");
        int count = 0;
        for (Document result : results) {
            count++;
            Document doc = (Document) result.get("document");
            double score = (double) result.get("score");
            System.out.printf("%d. %s, Score: %.4f%n", count,
                doc.getString("HotelName"), score);
        }
    }
}
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity (2–100, default 16). Higher values improve recall but increase memory.
- **efConstruction**: Candidate list size during construction (4–1000, default 64). Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

#### [DiskANN](#tab/tab-diskann)

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `src/main/java/com/azure/documentdb/DiskAnn.java`:

```java
package com.azure.documentdb;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.mongodb.MongoCredential;
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.AggregateIterable;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.List;

public class DiskAnn {
    private static final String DATABASE_NAME = "Hotels";
    private static final String COLLECTION_NAME = "hotels_diskann";
    private static final String VECTOR_INDEX_NAME = "vectorIndex_diskann";

    public static void main(String[] args) {
        new DiskAnn().run();
        System.exit(0);
    }

    public void run() {
        try (var mongoClient = createMongoClient()) {
            var openAIClient = createOpenAIClient();
            var database = mongoClient.getDatabase(DATABASE_NAME);
            var collection = database.getCollection(COLLECTION_NAME, Document.class);

            createVectorIndex(database, collection);
            performVectorSearch(collection, openAIClient);

        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    private MongoClient createMongoClient() {
        var clusterName = System.getenv("MONGO_CLUSTER_NAME");
        var managedIdentityPrincipalId = System.getenv("AZURE_MANAGED_IDENTITY_PRINCIPAL_ID");
        var azureCredential = new DefaultAzureCredentialBuilder().build();

        MongoCredential.OidcCallback callback = (MongoCredential.OidcCallbackContext context) -> {
            var token = azureCredential.getToken(
                new com.azure.core.credential.TokenRequestContext()
                    .addScopes("https://ossrdbms-aad.database.windows.net/.default")
            ).block();
            if (token == null) throw new RuntimeException("Failed to obtain Azure AD token");
            return new MongoCredential.OidcCallbackResult(token.getToken());
        };

        var credential = MongoCredential.createOidcCredential(null)
            .withMechanismProperty("OIDC_CALLBACK", callback);

        var connectionString = new ConnectionString(
            String.format("mongodb+srv://%s@%s.mongocluster.cosmos.azure.com/?authMechanism=MONGODB-OIDC&tls=true&retrywrites=false&maxIdleTimeMS=120000",
                managedIdentityPrincipalId, clusterName));

        var settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .credential(credential)
            .build();

        return MongoClients.create(settings);
    }

    private OpenAIClient createOpenAIClient() {
        var endpoint = System.getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT");
        var credential = new DefaultAzureCredentialBuilder().build();
        return new com.azure.ai.openai.OpenAIClientBuilder()
            .endpoint(endpoint).credential(credential).buildClient();
    }

    private void createVectorIndex(MongoDatabase database, MongoCollection<Document> collection) {
        System.out.println("Creating DiskANN vector index...");

        var indexCommand = new Document("createIndexes", COLLECTION_NAME)
            .append("indexes", List.of(
                new Document("name", VECTOR_INDEX_NAME)
                    .append("key", new Document(System.getenv("EMBEDDED_FIELD"), "cosmosSearch"))
                    .append("cosmosSearchOptions", new Document()
                        .append("kind", "vector-diskann")
                        .append("similarity", "COS")
                        .append("dimensions", Integer.parseInt(System.getenv("EMBEDDING_DIMENSIONS")))
                        // Maximum edges per node (20-2048, default 32)
                        .append("maxDegree", 32)
                        // Candidates evaluated during construction (10-500, default 50)
                        .append("lBuild", 50)
                    )
            ));

        try {
            database.runCommand(indexCommand);
            System.out.println("DiskANN vector index created successfully");
        } catch (Exception e) {
            System.err.println("Error creating index: " + e.getMessage());
            throw new RuntimeException(e);
        }
    }

    private void performVectorSearch(MongoCollection<Document> collection, OpenAIClient openAIClient) {
        System.out.println("Performing DiskANN vector search...");

        var query = "quintessential lodging near running trails, eateries, retail";
        var embeddingResponse = openAIClient.getEmbeddingsClient(
            System.getenv("AZURE_OPENAI_EMBEDDING_MODEL")
        ).generateEmbedding(new EmbeddingsOptions(List.of(query)));
        var embedding = embeddingResponse.getValue().getData().get(0).getEmbedding();

        var pipeline = List.of(
            new Document("$search", new Document()
                .append("cosmosSearch", new Document()
                    .append("vector", embedding)
                    .append("path", System.getenv("EMBEDDED_FIELD"))
                    .append("k", 5))),
            new Document("$project", new Document()
                .append("score", new Document("$meta", "searchScore"))
                .append("document", "$$ROOT"))
        );

        AggregateIterable<Document> results = collection.aggregate(pipeline);

        System.out.println("\nDiskANN Search Results:");
        int count = 0;
        for (Document result : results) {
            count++;
            Document doc = (Document) result.get("document");
            double score = (double) result.get("score");
            System.out.printf("%d. %s, Score: %.4f%n", count,
                doc.getString("HotelName"), score);
        }
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

```java
// Generate embedding for query
var embeddingResponse = openAIClient.getEmbeddingsClient(modelName)
    .generateEmbedding(new EmbeddingsOptions(List.of("your query text")));
var embedding = embeddingResponse.getValue().getData().get(0).getEmbedding();

// Build aggregation pipeline
var pipeline = List.of(
    new Document("$search", new Document()
        .append("cosmosSearch", new Document()
            .append("vector", embedding)
            .append("path", vectorField)
            .append("k", 5)  // Return top 5 results
        )
    ),
    new Document("$project", new Document()
        .append("score", new Document("$meta", "searchScore"))
        .append("document", "$$ROOT")
    )
);

// Execute search
AggregateIterable<Document> results = collection.aggregate(pipeline);
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

The code uses your local developer authentication to access Azure DocumentDB and Azure OpenAI. The authentication relies on [DefaultAzureCredential](/java/api/com.azure.identity.defaultazurecredential) from **azure-identity** to find your Azure credentials in the environment.

## Run the quickstart

### [IVF](#tab/tab-ivf)

```bash
mvn compile
mvn exec:java -Dexec.mainClass="com.azure.documentdb.IVF"
```

#### [HNSW](#tab/tab-hnsw)

```bash
mvn compile
mvn exec:java -Dexec.mainClass="com.azure.documentdb.HNSW"
```

#### [DiskANN](#tab/tab-diskann)

```bash
mvn compile
mvn exec:java -Dexec.mainClass="com.azure.documentdb.DiskAnn"
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
- [MongoDB Java driver documentation](https://www.mongodb.com/docs/drivers/java/sync/current/)
