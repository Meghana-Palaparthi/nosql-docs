---
title: Quickstart - Vector index with Java
description: Test and compare DiskANN, HNSW, and IVF vector indexes in Azure DocumentDB using Java to select the best algorithm for your vector search workload.
ms.devlang: java
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.custom: sfi-ropc-nochange
ai-usage: ai-generated
author: diberry
ms.author: diberry
ms.service: azure-documentdb
ms.subservice: vector-search
---

# Quickstart: Vector index with Java in Azure DocumentDB

This quickstart compares vector index algorithms (DiskANN, HNSW, IVF) in Azure DocumentDB using Java to help you select the best configuration for your vector search workload. The sample uses the same hotel dataset with pre-calculated vectors as the other quickstarts to demonstrate performance differences across algorithms and similarity functions.



## Prerequisites

[!INCLUDE[Prerequisites](includes/prerequisite-quickstart-vector-index.md)]

- [Java 21 or higher](/java/openjdk/download)

- [Maven 3.6 or higher](https://maven.apache.org/download.cgi)

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

   Verify: Confirm the file exists and is valid JSON:

   ### [Bash](#tab/bash)

   ```bash
   ls -lh data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-Item data\Hotels_Vector.json
   ```

   ---

## Create a Java project

1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir select-algorithm-quickstart
   cd select-algorithm-quickstart
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name select-algorithm-quickstart
   Set-Location select-algorithm-quickstart
   code .
   ```

   ---

2. Create a standard Maven project structure:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p src/main/java/com/azure/documentdb/selectalgorithm
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Path "src\main\java\com\azure\documentdb\selectalgorithm" -Force
   ```

   ---

3. Create a `pom.xml` file in the root directory with the following content:

   ```xml
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>

       <groupId>com.azure.documentdb.samples</groupId>
       <artifactId>select-algorithm-java</artifactId>
       <version>1.0-SNAPSHOT</version>
       <name>Azure DocumentDB Vector Algorithm Comparison</name>

       <properties>
           <maven.compiler.source>21</maven.compiler.source>
           <maven.compiler.target>21</maven.compiler.target>
           <maven.compiler.release>21</maven.compiler.release>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
       </properties>

       <dependencyManagement>
           <dependencies>
               <dependency>
                   <groupId>com.azure</groupId>
                   <artifactId>azure-sdk-bom</artifactId>
                   <version>1.2.29</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
           </dependencies>
       </dependencyManagement>

       <dependencies>
           <dependency>
               <groupId>org.mongodb</groupId>
               <artifactId>mongodb-driver-sync</artifactId>
               <version>5.6.2</version>
           </dependency>
           <dependency>
               <groupId>com.azure</groupId>
               <artifactId>azure-identity</artifactId>
           </dependency>
           <dependency>
               <groupId>com.azure</groupId>
               <artifactId>azure-ai-openai</artifactId>
           </dependency>
           <dependency>
               <groupId>com.fasterxml.jackson.core</groupId>
               <artifactId>jackson-databind</artifactId>
               <version>2.18.2</version>
           </dependency>
           <dependency>
               <groupId>io.github.cdimascio</groupId>
               <artifactId>dotenv-java</artifactId>
               <version>3.0.2</version>
           </dependency>
           <dependency>
               <groupId>org.slf4j</groupId>
               <artifactId>slf4j-simple</artifactId>
               <version>2.0.17</version>
               <scope>runtime</scope>
           </dependency>
       </dependencies>

       <build>
           <plugins>
               <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                   <artifactId>maven-compiler-plugin</artifactId>
                   <version>3.13.0</version>
                   <configuration>
                       <release>21</release>
                   </configuration>
               </plugin>
               <plugin>
                   <groupId>org.codehaus.mojo</groupId>
                   <artifactId>exec-maven-plugin</artifactId>
                   <version>3.1.0</version>
                   <configuration>
                       <mainClass>com.azure.documentdb.selectalgorithm.SelectAlgorithm</mainClass>
                   </configuration>
               </plugin>
           </plugins>
       </build>
   </project>
   ```

   Verify: Run `mvn dependency:resolve` to confirm all dependencies resolve without errors.

4. Create a `.env` filein the project root for environment variables:

   ```bash
   # Azure DocumentDB cluster name for passwordless authentication
   MONGO_CLUSTER_NAME=

   # Azure managed identity principal ID for authentication
   AZURE_MANAGED_IDENTITY_PRINCIPAL_ID=

   # Azure OpenAI endpoint and model configuration
   AZURE_OPENAI_EMBEDDING_ENDPOINT=https://your-openai-resource.openai.azure.com/
   AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small

   # Data file path (relative to project root)
   DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json

   # Vector configuration
   EMBEDDED_FIELD=DescriptionVector
   EMBEDDING_DIMENSIONS=1536
   LOAD_SIZE_BATCH=100

   # Algorithm selection: all, diskann, hnsw, ivf
   ALGORITHM=all

   # Similarity function: COS, L2, IP, all
   SIMILARITY=COS
   ```

   Replace the placeholder values with your Azure resource information:

   - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB cluster name
   - `AZURE_MANAGED_IDENTITY_PRINCIPAL_ID`: Your managed identity principal ID
   - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL

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

   You should see your configuration values including the Azure OpenAI endpoint and cluster name.

   This sample uses passwordless authenticationwith `DefaultAzureCredential`, which requires your identity to have proper RBAC roles assigned. For more information on authentication options, see [Authenticate Java apps to Azure services by using the Azure SDK for Java](/azure/developer/java/sdk/authentication/overview).

## Create code files

When you are done, the project structure should look like this:

```text
select-algorithm-quickstart/
├── data/
│   └── Hotels_Vector.json           # Hotel data with vector embeddings
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── azure/
│                   └── documentdb/
│                       └── selectalgorithm/
│                           ├── SelectAlgorithm.java  # Main comparison logic
│                           └── Utils.java            # Shared utility functions
├── pom.xml                          # Maven dependencies
└── .env                             # Environment variables
```

## Create the algorithm comparison code

### Create utility functions

Create `src/main/java/com/azure/documentdb/selectalgorithm/Utils.java` and paste the following code:

```java
package com.azure.documentdb.selectalgorithm;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.core.http.policy.ExponentialBackoffOptions;
import com.azure.core.http.policy.RetryOptions;
import com.azure.identity.DefaultAzureCredential;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.MongoCredential;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import io.github.cdimascio.dotenv.Dotenv;
import org.bson.Document;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;

// Utility class for shared operations across the vector search comparison sample.
// Uses dotenv-java for cross-platform environment variable loading with .env file fallback.
public class Utils {
    private static Dotenv dotenv;
    private static final ObjectMapper objectMapper = new ObjectMapper();

    // Cached credential instance shared by MongoClient and OpenAIClient
    private static volatile DefaultAzureCredential cachedCredential;

    // Get or create the DefaultAzureCredential instance used for passwordless authentication
    private static DefaultAzureCredential getCredential() {
        if (cachedCredential == null) {
            synchronized (Utils.class) {
                if (cachedCredential == null) {
                    cachedCredential = new DefaultAzureCredentialBuilder().build();
                }
            }
        }
        return cachedCredential;
    }

    // Load environment variables from .env file or system environment
    public static void loadEnv() {
        try {
            dotenv = Dotenv.configure()
                .ignoreIfMissing()
                .load();
        } catch (Exception e) {
            System.err.println("Warning: Could not load .env file, using system environment variables");
        }
    }

    // Get environment variable value from .env file or system environment
    public static String getEnv(String key) {
        if (dotenv != null) {
            String value = dotenv.get(key);
            if (value != null) return value;
        }
        return System.getenv(key);
    }

    // Get environment variable with default value if not found
    public static String getEnv(String key, String defaultValue) {
        String value = getEnv(key);
        return value != null ? value : defaultValue;
    }

    // Create MongoDB client with passwordless authentication using OIDC
    public static MongoClient createMongoClient() {
        var clusterName = Objects.requireNonNull(
            getEnv("MONGO_CLUSTER_NAME"),
            "Environment variable MONGO_CLUSTER_NAME is required");
        var managedIdentityPrincipalId = Objects.requireNonNull(
            getEnv("AZURE_MANAGED_IDENTITY_PRINCIPAL_ID"),
            "Environment variable AZURE_MANAGED_IDENTITY_PRINCIPAL_ID is required");
        var azureCredential = getCredential();

        // OIDC callback that fetches Azure AD tokens for DocumentDB authentication
        MongoCredential.OidcCallback callback = (MongoCredential.OidcCallbackContext context) -> {
            var token = azureCredential.getToken(
                new com.azure.core.credential.TokenRequestContext()
                    .addScopes("https://ossrdbms-aad.database.windows.net/.default")
            ).block();

            if (token == null) {
                throw new RuntimeException(
                    "Failed to obtain Azure AD token for DocumentDB OIDC auth. "
                    + "Verify DefaultAzureCredential is configured correctly.");
            }

            return new MongoCredential.OidcCallbackResult(token.getToken());
        };

        // Create MongoDB credential with OIDC mechanism
        var credential = MongoCredential.createOidcCredential(null)
            .withMechanismProperty("OIDC_CALLBACK", callback);

        // Build connection string for DocumentDB cluster
        var connectionString = new ConnectionString(
            String.format("mongodb+srv://%s@%s.mongocluster.cosmos.azure.com/?authMechanism=MONGODB-OIDC&tls=true&retrywrites=false&maxIdleTimeMS=120000",
                managedIdentityPrincipalId, clusterName)
        );

        // Configure MongoDB client settings with retry and credential
        var settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .credential(credential)
            .retryWrites(true)
            .retryReads(true)
            .build();

        return MongoClients.create(settings);
    }

    // Create Azure OpenAI client with passwordless authentication
    public static OpenAIClient createOpenAIClient() {
        var endpoint = Objects.requireNonNull(
            getEnv("AZURE_OPENAI_EMBEDDING_ENDPOINT"),
            "Environment variable AZURE_OPENAI_EMBEDDING_ENDPOINT is required");
        var credential = getCredential();

        return new OpenAIClientBuilder()
            .endpoint(endpoint)
            .credential(credential)
            .retryOptions(new RetryOptions(
                new ExponentialBackoffOptions()
                    .setMaxRetries(3)
                    .setBaseDelay(Duration.ofSeconds(1))
                    .setMaxDelay(Duration.ofSeconds(30))
            ))
            .buildClient();
    }

    // Load hotel data from JSON file
    public static List<Map<String, Object>> loadHotelData() throws IOException {
        var dataFile = getEnv("DATA_FILE_WITH_VECTORS");
        var filePath = Path.of(dataFile);

        System.out.println("Reading JSON file from " + filePath.toAbsolutePath());
        var jsonContent = Files.readString(filePath);

        return objectMapper.readValue(jsonContent, new TypeReference<List<Map<String, Object>>>() {});
    }

    // Create embedding vector for a text query using Azure OpenAI
    public static List<Double> createEmbedding(OpenAIClient openAIClient, String text) {
        var model = Objects.requireNonNull(
            getEnv("AZURE_OPENAI_EMBEDDING_MODEL"),
            "Environment variable AZURE_OPENAI_EMBEDDING_MODEL is required");
        var options = new EmbeddingsOptions(List.of(text));

        var response = openAIClient.getEmbeddings(model, options);
        return response.getData().get(0).getEmbedding().stream()
                .map(Float::doubleValue)
                .toList();
    }

    // Create vector index configuration options for a specific algorithm and similarity function
    public static Document createVectorIndexOptions(String algorithm, String similarity) {
        var dimensionsStr = getEnv("EMBEDDING_DIMENSIONS");
        var dimensions = dimensionsStr != null ? Integer.parseInt(dimensionsStr) : 1536;

        var options = new Document()
            .append("kind", getVectorKind(algorithm))
            .append("dimensions", dimensions)
            .append("similarity", similarity);

        // Add algorithm-specific tuning parameters
        switch (algorithm.toLowerCase()) {
            case "diskann":
                // DiskANN parameters:
                // maxDegree: maximum edges per node in the graph (higher = more accurate but slower)
                // lBuild: candidates evaluated during index construction (higher = better quality)
                options.append("maxDegree", 32)
                       .append("lBuild", 50);
                break;
            case "hnsw":
                // HNSW parameters:
                // m: number of connections per layer (higher = more accurate but more memory)
                // efConstruction: candidates during index construction (higher = better quality)
                options.append("m", 16)
                       .append("efConstruction", 64);
                break;
            case "ivf":
                // IVF parameters:
                // numLists: number of clusters for partitioning (higher = faster but less accurate)
                options.append("numLists", 1);
                break;
        }

        return options;
    }

    // Create search-time options for a specific algorithm
    public static Document createSearchOptions(String algorithm) {
        var options = new Document();

        // Add algorithm-specific search parameters
        switch (algorithm.toLowerCase()) {
            case "diskann":
                // lSearch: search list size at query time (higher = more accurate but slower)
                options.append("lSearch", 100);
                break;
            case "hnsw":
                // efSearch: candidate list size at query time (higher = more accurate but slower)
                options.append("efSearch", 80);
                break;
            case "ivf":
                // nProbes: number of clusters to search (higher = more accurate but slower)
                options.append("nProbes", 1);
                break;
        }

        return options;
    }

    // Get the vector index kind string for DocumentDB
    private static String getVectorKind(String algorithm) {
        return "vector-" + algorithm.toLowerCase();
    }

    // Partition a list into batches of specified size
    public static <T> List<List<T>> partitionList(List<T> list, int batchSize) {
        var partitions = new ArrayList<List<T>>();
        for (int i = 0; i < list.size(); i += batchSize) {
            partitions.add(list.subList(i, Math.min(i + batchSize, list.size())));
        }
        return partitions;
    }

    // Print formatted comparison table of algorithm results
    public static void printComparisonTable(List<Map<String, Object>> results) {
        System.out.println("\n" + "=".repeat(80));
        System.out.println("Vector Index Algorithm Comparison Results");
        System.out.println("=".repeat(80));
        System.out.printf("%-15s %-15s %-20s%n", "Algorithm", "Similarity", "Avg Latency (ms)");
        System.out.println("-".repeat(80));

        for (var result : results) {
            System.out.printf("%-15s %-15s %-20.2f%n",
                result.get("algorithm"),
                result.get("similarity"),
                result.get("latency"));
        }

        System.out.println("=".repeat(80));
    }
}
```

This utility class provides:

- **Environment variable management**: Loads configuration from `.env` file or system environment
- **Passwordless authentication**: Uses `DefaultAzureCredential` for both MongoDB and Azure OpenAI
- **MongoDB client creation**: Configures OIDC authentication for DocumentDB
- **Azure OpenAI client creation**: Sets up the OpenAI client for embedding generation
- **Data loading**: Reads hotel data from JSON file
- **Embedding generation**: Creates vector embeddings for text queries
- **Index configuration**: Generates algorithm-specific vector index options
- **Search configuration**: Generates algorithm-specific search parameters
- **Results formatting**: Prints comparison table of algorithm performance

### Create main comparison logic

Create `src/main/java/com/azure/documentdb/selectalgorithm/SelectAlgorithm.java` and paste the following code:

```java
package com.azure.documentdb.selectalgorithm;

import com.azure.ai.openai.OpenAIClient;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.model.Indexes;
import org.bson.Document;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

// Main class that compares vector search algorithms across similarity functions.
// Tests DiskANN, HNSW, and IVF with COS, L2, and IP similarity to help select optimal configuration.
public class SelectAlgorithm {
    private static final String SAMPLE_QUERY = "quintessential lodging near running trails, eateries, retail";
    private static final String DATABASE_NAME = "Hotels";
    private static final int NUM_QUERIES = 5;

    public static void main(String[] args) {
        Utils.loadEnv();
        new SelectAlgorithm().run();
        System.exit(0);
    }

    // Main execution flow: test all algorithm/similarity combinations and display results
    public void run() {
        try (var mongoClient = Utils.createMongoClient()) {
            var openAIClient = Utils.createOpenAIClient();

            // Parse algorithm and similarity parameters from environment
            var algorithmParam = Utils.getEnv("ALGORITHM", "all").toLowerCase();
            var similarityParam = Utils.getEnv("SIMILARITY", "COS").toUpperCase();

            var algorithms = getAlgorithms(algorithmParam);
            var similarities = getSimilarities(similarityParam);

            System.out.println("Testing algorithms: " + algorithms);
            System.out.println("Testing similarity functions: " + similarities);
            System.out.println();

            var results = new ArrayList<Map<String, Object>>();
            var database = mongoClient.getDatabase(DATABASE_NAME);

            // Test each algorithm/similarity combination
            for (var algorithm : algorithms) {
                for (var similarity : similarities) {
                    var result = testConfiguration(database, openAIClient, algorithm, similarity);
                    results.add(result);
                }
            }

            // Display comparison table
            Utils.printComparisonTable(results);

        } catch (com.azure.core.exception.HttpResponseException e) {
            System.err.println("Azure service error: " + e.getMessage());
            e.printStackTrace();
        } catch (com.mongodb.MongoException e) {
            System.err.println("MongoDB error: " + e.getMessage());
            e.printStackTrace();
        } catch (Exception e) {
            System.err.println("Unexpected error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    // Parse algorithm parameter into list of algorithms to test
    private List<String> getAlgorithms(String param) {
        if ("all".equals(param)) {
            return List.of("diskann", "hnsw", "ivf");
        }
        return List.of(param);
    }

    // Parse similarity parameter into list of similarity functions to test
    private List<String> getSimilarities(String param) {
        if ("all".equalsIgnoreCase(param)) {
            return List.of("COS", "L2", "IP");
        }
        return List.of(param);
    }

    // Test a specific algorithm/similarity combination and measure performance
    private Map<String, Object> testConfiguration(MongoDatabase database, OpenAIClient openAIClient,
                                                   String algorithm, String similarity) {
        System.out.println("Testing " + algorithm.toUpperCase() + " with " + similarity + " similarity...");

        var collectionName = "hotels_" + algorithm.toLowerCase() + "_" + similarity.toLowerCase();
        var vectorIndexName = "vectorIndex_" + algorithm.toLowerCase() + "_" + similarity.toLowerCase();

        try {
            // Create fresh collection for this algorithm/similarity combination
            var collection = database.getCollection(collectionName, Document.class);
            collection.drop();
            database.createCollection(collectionName);
            System.out.println("  Created collection: " + collectionName);

            // Load and insert hotel data
            var hotelData = Utils.loadHotelData();
            insertDataInBatches(collection, hotelData);

            // Create standard indexes for common query fields
            createStandardIndexes(collection);

            // Create vector index with algorithm-specific configuration
            createVectorIndex(database, collectionName, vectorIndexName, algorithm, similarity);

            // Generate embedding for sample query
            var queryEmbedding = Utils.createEmbedding(openAIClient, SAMPLE_QUERY);

            // Measure average query latency
            var avgLatency = measureSearchLatency(collection, queryEmbedding, algorithm);

            System.out.println("  Average latency: " + String.format("%.2f", avgLatency) + " ms");
            System.out.println();

            // Return results for comparison table
            var result = new HashMap<String, Object>();
            result.put("algorithm", algorithm.toUpperCase());
            result.put("similarity", similarity);
            result.put("latency", avgLatency);
            return result;

        } catch (Exception e) {
            System.err.println("  Error testing " + algorithm + " with " + similarity + ": " + e.getMessage());
            var result = new HashMap<String, Object>();
            result.put("algorithm", algorithm.toUpperCase());
            result.put("similarity", similarity);
            result.put("latency", -1.0);
            return result;
        }
    }

    // Insert hotel data in batches for efficient loading
    private void insertDataInBatches(MongoCollection<Document> collection, List<Map<String, Object>> hotelData) {
        var batchSizeStr = Utils.getEnv("LOAD_SIZE_BATCH");
        var batchSize = batchSizeStr != null ? Integer.parseInt(batchSizeStr) : 100;
        var batches = Utils.partitionList(hotelData, batchSize);

        System.out.println("  Loading data in batches of " + batchSize + "...");

        for (int i = 0; i < batches.size(); i++) {
            var batch = batches.get(i);
            var documents = batch.stream()
                .map(Document::new)
                .toList();

            collection.insertMany(documents);
            if ((i + 1) % 10 == 0 || (i + 1) == batches.size()) {
                System.out.println("    Loaded " + ((i + 1) * batchSize) + " documents");
            }
        }
    }

    // Create standard indexes on common query fields
    private void createStandardIndexes(MongoCollection<Document> collection) {
        collection.createIndex(Indexes.ascending("HotelId"));
        collection.createIndex(Indexes.ascending("Category"));
        collection.createIndex(Indexes.ascending("Description"));
        collection.createIndex(Indexes.ascending("Description_fr"));
    }

    // Create vector index with algorithm-specific configuration
    private void createVectorIndex(MongoDatabase database, String collectionName, String indexName,
                                   String algorithm, String similarity) {
        var embeddedField = Utils.getEnv("EMBEDDED_FIELD");
        var cosmosSearchOptions = Utils.createVectorIndexOptions(algorithm, similarity);

        // Build DocumentDB vector index command
        var indexDefinition = new Document()
            .append("createIndexes", collectionName)
            .append("indexes", List.of(
                new Document()
                    .append("name", indexName)
                    .append("key", new Document(embeddedField, "cosmosSearch"))
                    .append("cosmosSearchOptions", cosmosSearchOptions)
            ));

        database.runCommand(indexDefinition);
        System.out.println("  Created vector index: " + indexName);
    }

    // Measure average query latency across multiple runs
    private double measureSearchLatency(MongoCollection<Document> collection, List<Double> queryEmbedding,
                                        String algorithm) {
        var embeddedField = Utils.getEnv("EMBEDDED_FIELD");
        var searchOptions = Utils.createSearchOptions(algorithm);

        var totalLatency = 0.0;

        // Run query multiple times to get average latency
        for (int i = 0; i < NUM_QUERIES; i++) {
            var cosmosSearch = new Document()
                .append("vector", queryEmbedding)
                .append("path", embeddedField)
                .append("k", 5);

            // Add algorithm-specific search options
            if (!searchOptions.isEmpty()) {
                cosmosSearch.putAll(searchOptions);
            }

            // Build aggregation pipeline for vector search
            var searchStage = new Document("$search", new Document()
                .append("cosmosSearch", cosmosSearch)
            );

            var projectStage = new Document("$project", new Document()
                .append("score", new Document("$meta", "searchScore"))
                .append("HotelName", 1)
            );

            var pipeline = List.of(searchStage, projectStage);

            // Measure query execution time
            var startTime = System.nanoTime();
            var results = collection.aggregate(pipeline);
            results.first();  // Execute query
            var endTime = System.nanoTime();

            totalLatency += (endTime - startTime) / 1_000_000.0;
        }

        return totalLatency / NUM_QUERIES;
    }
}
```

This main class provides:

- **Algorithm comparison logic**: Tests all combinations of algorithms and similarity functions
- **Collection management**: Creates separate collections for each configuration
- **Data loading**: Inserts hotel data in batches
- **Index creation**: Creates both standard and vector indexes
- **Performance measurement**: Measures average query latency
- **Results display**: Outputs comparison table

## Run the code

1. Compile the project:

   ```bash
   mvn clean compile
   ```

   Verify: The build output ends with `BUILD SUCCESS`.

2. Run the comparison for all algorithms with cosine similarity (default):

   ```bash
   mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

3. Run the comparison for a specific algorithm:

   ### [Bash](#tab/bash)

   ```bash
   # Test only DiskANN
   ALGORITHM=diskann mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test only HNSW
   ALGORITHM=hnsw mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test only IVF
   ALGORITHM=ivf mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   # Test only DiskANN
   $env:ALGORITHM="diskann"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test only HNSW
   $env:ALGORITHM="hnsw"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test only IVF
   $env:ALGORITHM="ivf"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ---

4. Run the comparison for all similarity functions:

   ### [Bash](#tab/bash)

   ```bash
   # Test all algorithms with all similarity functions
   ALGORITHM=all SIMILARITY=all mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test DiskANN with all similarity functions
   ALGORITHM=diskann SIMILARITY=all mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   # Test all algorithms with all similarity functions
   $env:ALGORITHM="all"
   $env:SIMILARITY="all"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test DiskANN with all similarity functions
   $env:ALGORITHM="diskann"
   $env:SIMILARITY="all"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ---

5. Run the comparison for a specific similarity function:

   ### [Bash](#tab/bash)

   ```bash
   # Test all algorithms with L2 (Euclidean) distance
   SIMILARITY=L2 mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test all algorithms with IP (inner product)
   SIMILARITY=IP mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   # Test all algorithms with L2 (Euclidean) distance
   $env:SIMILARITY="L2"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"

   # Test all algorithms with IP (inner product)
   $env:SIMILARITY="IP"
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.SelectAlgorithm"
   ```

   ---

The program displays a comparison table showing average latency for each algorithm and similarity function combination:

```text
================================================================================
Vector Index Algorithm Comparison Results
================================================================================
Algorithm       Similarity      Avg Latency (ms)    
--------------------------------------------------------------------------------
DISKANN         COS             42.30               
DISKANN         IP              38.70               
DISKANN         L2              45.10               
HNSW            COS             31.50               
HNSW            IP              29.80               
HNSW            L2              34.20               
IVF             COS             55.60               
IVF             IP              52.10               
IVF             L2              58.90               
================================================================================
```

> [!NOTE]
> The latency values shown above are illustrative. Actual results depend on your DocumentDB cluster configuration, region, network latency, and dataset size.

## Understanding the results

### Algorithm characteristics

**DiskANN** - Disk-based approximate nearest neighbor search
- Good balance of speed and accuracy
- Suitable for large datasets that don't fit in memory
- Parameters: `maxDegree=32` (graph connectivity), `lBuild=50` (build quality), `lSearch=100` (query accuracy)

**HNSW** - Hierarchical Navigable Small World
- Memory-based hierarchical graph
- Excellent for real-time applications requiring low latency
- Parameters: `m=16` (connections per layer), `efConstruction=64` (build quality), `efSearch=80` (query accuracy)

**IVF** - Inverted File Index
- Cluster-based partitioning approach
- Fast search via centroid comparison
- Parameters: `numLists=1` (number of clusters), `nProbes=1` (clusters to search)

### Similarity functions

**COS (Cosine)** - Measures angle between vectors
- Best for text embeddings (like those from OpenAI models)
- Scale-invariant (ignores vector magnitude)
- Range: -1 to 1 (1 = identical direction)

**L2 (Euclidean)** - Measures straight-line distance
- Sensitive to vector magnitude
- Good for embeddings where scale matters
- Range: 0 to infinity (0 = identical)

**IP (Inner Product)** - Dot product of vectors
- Fast to compute
- Can be used with normalized vectors
- Range: -infinity to infinity

### Choosing the right configuration

Use the comparison results to guide your selection:

1. **For real-time applications**: Choose HNSW if latency is critical
2. **For large datasets**: Choose DiskANN if your data exceeds available memory
3. **For fast batch processing**: Choose IVF if you can tolerate slightly lower accuracy
4. **For text embeddings**: Use COS similarity function (most common with OpenAI embeddings)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `MongoTimeoutException` | Verify your connection string in `.env`. Ensure your IP is in the DocumentDB firewall rules. |
| `MongoSecurityException` | Check credentials in connection string. |
| Maven build failures | Run `mvn dependency:resolve` to check for missing dependencies. Ensure Java 17+ is installed. |
| `No plugin found for prefix 'exec'` | Add `exec-maven-plugin` to your `pom.xml` as shown in this article. |

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

- [Vector search overview](vector-search)
- [ENN vector search](enn-vector-search)
- [Product quantization](product-quantization)
- [Quickstart: Vector search with Java](quickstart-java-vector-search)
