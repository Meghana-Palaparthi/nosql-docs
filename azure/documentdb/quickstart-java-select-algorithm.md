---
title: Quickstart - Vector index with Java
description: Test and compare DiskANN, HNSW, and IVF vector indexes in Azure DocumentDB using Java to select the best algorithm for your vector search workload.
author: diberry
ms.author: diberry
ms.reviewer: khelanmodi
ms.devlang: java
ms.topic: quickstart-sdk
ms.date: 05/07/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-extended-java
  - devx-track-extended-java-ai
  - devx-track-data-ai
  - msecd-doc-authoring-1012
# CustomerIntent: As a developer, I want to compare vector index algorithms in Java applications with Azure DocumentDB.
ms.service: azure-documentdb
---

# Quickstart: Vector index with Java in Azure DocumentDB

This quickstart compares vector index algorithms (DiskANN, HNSW, IVF) in Azure DocumentDB using Java to help you select the best configuration for your vector search workload. The sample uses the same hotel dataset with precalculated vectors as the other quickstarts to demonstrate performance differences across algorithms and similarity functions.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-java) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

- [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd) (optional). Use `azd up` to deploy all required Azure resources in one command.

- [Java 21](/java/openjdk/download) or later

- [Maven 3.8 or higher](https://maven.apache.org/download.cgi)

## Create a Java project

1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir select-algorithm-java
   cd select-algorithm-java
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Name select-algorithm-java
   Set-Location select-algorithm-java
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
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>

       <groupId>com.azure.documentdb</groupId>
       <artifactId>select-algorithm-java</artifactId>
       <version>1.0.0</version>
       <packaging>jar</packaging>

       <name>DocumentDB Select Algorithm - Java</name>
       <description>Demonstrates IVF, HNSW, and DiskANN vector search indexes with Azure DocumentDB</description>

       <properties>
           <maven.compiler.source>21</maven.compiler.source>
           <maven.compiler.target>21</maven.compiler.target>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
       </properties>

       <dependencies>
           <dependency>
               <groupId>org.mongodb</groupId>
               <artifactId>mongodb-driver-sync</artifactId>
               <version>5.4.0</version>
           </dependency>
           <dependency>
               <groupId>com.azure</groupId>
               <artifactId>azure-identity</artifactId>
               <version>1.16.0</version>
           </dependency>
           <dependency>
               <groupId>com.azure</groupId>
               <artifactId>azure-ai-openai</artifactId>
               <version>1.0.0-beta.16</version>
           </dependency>
       </dependencies>

       <build>
           <plugins>
               <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                   <artifactId>maven-compiler-plugin</artifactId>
                   <version>3.13.0</version>
                   <configuration>
                       <source>21</source>
                       <target>21</target>
                   </configuration>
               </plugin>
               <plugin>
                   <groupId>org.codehaus.mojo</groupId>
                   <artifactId>exec-maven-plugin</artifactId>
                   <version>3.4.1</version>
                   <configuration>
                       <mainClass>com.azure.documentdb.selectalgorithm.Main</mainClass>
                   </configuration>
               </plugin>
           </plugins>
       </build>

       <profiles>
           <profile>
               <id>compare</id>
               <build>
                   <plugins>
                       <plugin>
                           <groupId>org.codehaus.mojo</groupId>
                           <artifactId>exec-maven-plugin</artifactId>
                           <version>3.4.1</version>
                           <configuration>
                               <mainClass>com.azure.documentdb.selectalgorithm.CompareAll</mainClass>
                           </configuration>
                       </plugin>
                   </plugins>
               </build>
           </profile>
       </profiles>
   </project>
   ```

   Verify that all dependencies resolve without errors by running `mvn dependency:resolve`.

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

   Verify that the file exists and is valid JSON:

   ### [Bash](#tab/bash)

   ```bash
   ls -lh data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   Get-Item data\Hotels_Vector.json
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
$env:DOCUMENTDB_CLUSTER_NAME="<your-cluster-name>"
$env:AZURE_OPENAI_EMBEDDING_ENDPOINT="https://<your-resource>.openai.azure.com"
$env:AZURE_OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
$env:AZURE_DOCUMENTDB_DATABASENAME="Hotels"
$env:DATA_FILE_WITH_VECTORS="data/Hotels_Vector.json"
$env:EMBEDDED_FIELD="DescriptionVector"
$env:EMBEDDING_DIMENSIONS="1536"
```

---

Replace the placeholder values with your Azure resource information:

- `DOCUMENTDB_CLUSTER_NAME`: Your Azure DocumentDB cluster name
- `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL

This sample uses passwordless authentication with `DefaultAzureCredential`, which requires your identity to have proper RBAC roles assigned. For more information on authentication options, see [Authenticate Java apps to Azure services by using the Azure SDK for Java](/azure/developer/java/sdk/authentication/overview).



## Create code files

When you're done, the project structure should look like this:

```text
select-algorithm-java/
├── data/
│   └── Hotels_Vector.json
├── src/main/java/com/azure/documentdb/selectalgorithm/
│   ├── CompareAll.java
│   ├── Main.java
│   └── Utils.java
└── pom.xml
```

## Create the algorithm comparison code

Create the following source files to implement the vector search comparison.

### Create utility functions

Create `src/main/java/com/azure/documentdb/selectalgorithm/Utils.java` and paste the following code:

:::code language="java" source="~/../documentdb-samples/ai/select-algorithm-java/src/main/java/com/azure/documentdb/selectalgorithm/Utils.java" :::

This utility class provides:

- **Environment variable management**: Reads configuration from environment variables with `System.getenv()`.
- **Passwordless authentication**: Uses `DefaultAzureCredential` for both MongoDB and Azure OpenAI.
- **MongoDB client creation**: Configures OIDC authentication for DocumentDB.
- **Azure OpenAI client creation**: Sets up the OpenAI client for embedding generation.
- **Data loading**: Reads hotel data from JSON file.
- **Embedding generation**: Creates vector embeddings for text queries.
- **Index configuration**: Generates algorithm-specific vector index options.
- **Search configuration**: Generates algorithm-specific search parameters.
- **Results formatting**: Prints comparison table of algorithm performance.

> [!NOTE]
> The Java sample configures the DocumentDB connection with `retryWrites=false`, which is required for DocumentDB vector search operations.

### Create main comparison logic

Create the following source files in `src/main/java/com/azure/documentdb/selectalgorithm/`:

#### CompareAll.java

:::code language="java" source="~/../documentdb-samples/ai/select-algorithm-java/src/main/java/com/azure/documentdb/selectalgorithm/CompareAll.java" :::

#### Main.java

:::code language="java" source="~/../documentdb-samples/ai/select-algorithm-java/src/main/java/com/azure/documentdb/selectalgorithm/Main.java" :::


This main comparison logic provides:

- **Algorithm comparison logic**: Tests all combinations of algorithms and similarity functions.
- **Collection management**: Creates separate collections for each configuration.
- **Data loading**: Inserts hotel data in batches.
- **Index creation**: Creates vector indexes for each algorithm and metric combination.
- **Performance measurement**: Measures average query latency.
- **Results display**: Outputs comparison table.

## Run the code

1. Sign in with Azure CLI for passwordless authentication:

   ```azurecli
   az login
   ```

2. Compile the project:

   ```console
   mvn clean compile
   ```

   Verify that the build output ends with `BUILD SUCCESS`.

3. Run the comparison entry point. `Main.java` calls `CompareAll.run()` and always executes all 9 combinations (3 algorithms × 3 metrics):

   ### [Bash](#tab/bash)

   ```bash
   mvn exec:java -Dexec.mainClass="com.azure.documentdb.selectalgorithm.Main"
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   mvn exec:java "-Dexec.mainClass=com.azure.documentdb.selectalgorithm.Main"
   ```

   ---

The program prints output similar to the following:

:::code language="text" source="~/../documentdb-samples/ai/select-algorithm-java/output/compare_all.txt" :::

The **Diff** column shows the score gap between the top-1 and top-2 results. A smaller diff indicates the algorithm found results with more similar relevance scores.



[!INCLUDE[Choosing the right algorithm](includes/choosing-algorithm.md)]

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `MongoTimeoutException` | Verify the `DOCUMENTDB_CLUSTER_NAME` environment variable, and ensure your IP is in the DocumentDB firewall rules. |
| `MongoSecurityException` | Verify your Microsoft Entra token is valid. Run `az login` to refresh your credentials. |
| Maven build failures | Run `mvn dependency:resolve` to check for missing dependencies. Ensure Java 21 or later is installed. |
| `No plugin found for prefix 'exec'` | Add `exec-maven-plugin` to your `pom.xml` as shown in this article. |
| Empty search results | Index may not be ready. The sample retries up to 5 times with 2-second intervals after index creation. If results are still empty, increase the wait time or verify index status with the DocumentDB for VS Code extension. |

## Clean up resources

Remove the database using the DocumentDB for VS Code extension:

1. Install the [DocumentDB for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) extension.
1. Connect to your Azure DocumentDB cluster.
1. Expand the cluster, right-click the **Hotels** database, and select **Drop Database**.

## Related content

- [Vector search overview](./vector-search.md)
- [ENN vector search](./enn-vector-search.md)
- [Product quantization](./product-quantization.md)
