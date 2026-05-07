---
title: Choose and configure vector indexes in Azure DocumentDB using Python
description: Compare vector index algorithms and similarity functions using the Python SDK in Azure DocumentDB to optimize search performance for your workload.
ms.topic: quickstart
ms.date: 2025-01-30
author: diberry
ms.author: diberry
ms.service: azure-documentdb
ms.subservice: vector-search
---

# Quickstart: Choose and configure vector indexes in Azure DocumentDB using Python

In this quickstart, you compare three vector index algorithms (DiskANN, HNSW, and IVF) and three similarity functions (cosine, L2, and inner product) to find the optimal configuration for your search workload. This quickstart uses a sample hotel dataset with pre-calculated embeddings from the `text-embedding-3-small` model.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-python) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites](includes/prerequisite-quickstart-vector-index.md)]

- [Python](https://www.python.org/downloads/) 3.10 or greater

## Create data file with vectors

1. Create a new data directory and download the hotels data file with vectors:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p data
   curl -o data/Hotels_Vector.json https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/main/data/Hotels_Vector.json
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path data
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/main/data/Hotels_Vector.json" -OutFile "data/Hotels_Vector.json"
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

## Create a Python project

1. Create a new directory for your project and open it in Visual Studio Code:

   ### [Bash](#tab/bash)

   ```bash
   mkdir -p select-algorithm
   cd select-algorithm
   code .
   ```

   ### [PowerShell](#tab/powershell)

   ```powershell
   New-Item -ItemType Directory -Force -Path select-algorithm
   Set-Location select-algorithm
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

3. Install the required packages:

   ```bash
   pip install "pymongo>=4.7" openai==1.55.3 azure-identity==1.15.0 python-dotenv==1.0.0
   ```

   - `pymongo`: MongoDB driver for Python (≥4.7 required for OIDC authentication)
   - `openai`: OpenAI client library to create vectors
   - `azure-identity`: Azure Identity library for passwordless authentication
   - `python-dotenv`: Environment variable management from .env files

   Verify the packages are installed:

   ```bash
   pip list | grep pymongo
   ```

   You should see `pymongo` with a version of 4.7 or greater.

4. Create a `.env` file for environment variables in the project root:

   ```bash
   # Azure OpenAI Embedding Settings
   AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
   AZURE_OPENAI_EMBEDDING_API_VERSION=2024-10-21
   AZURE_OPENAI_EMBEDDING_ENDPOINT=https://<RESOURCE-NAME>.openai.azure.com
   
   # Data File Paths and Vector Configuration
   DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
   EMBEDDED_FIELD=DescriptionVector
   EMBEDDING_DIMENSIONS=1536
   LOAD_SIZE_BATCH=100
   
   # Azure DocumentDB Connection Settings
   MONGO_CLUSTER_NAME=<CLUSTER-NAME>
   
   # Azure DocumentDB Database Name
   AZURE_DOCUMENTDB_DATABASENAME=Hotels
   
   # Algorithm Selection (used by select_algorithm.py)
   # ALGORITHM: "all" | "diskann" | "hnsw" | "ivf"
   ALGORITHM=all
   
   # SIMILARITY: "all" | "COS" | "L2" | "IP"
   SIMILARITY=COS
   ```

   For the passwordless authentication used in this article, replace the placeholder values in the `.env` file with your own information:

   - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL
   - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB cluster name

   You should always prefer passwordless authentication, but it requires additional setup. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate Python apps to Azure services by using the Azure SDK for Python](/azure/developer/python/sdk/authentication/overview).

   Verify the `.env` file was created:

   ```bash
   cat .env
   ```

   You should see your connection string and Azure OpenAI endpoint values.

## Create code files

Create the following project structure:

```
├── data/
│   └── Hotels_Vector.json       # Hotel data with vector embeddings
└── select-algorithm/
    ├── src/
    │   ├── select_algorithm.py  # Main comparison script
    │   └── utils.py             # Shared utility functions
    └── .env                     # Environment variables
```

Create the `src` directory:

### [Bash](#tab/bash)

```bash
mkdir -p src
```

### [PowerShell](#tab/powershell)

```powershell
New-Item -ItemType Directory -Force -Path src
```

---

## Create the algorithm comparison code

Create the `src/select_algorithm.py` file with the following code:

```python
import os
import time
from pathlib import Path
from typing import Any, Literal
import openai
import pymongo.errors
from utils import get_clients_passwordless, read_file_return_json, insert_data, print_comparison_table
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Type definitions for algorithm and similarity options
Algorithm = Literal['diskann', 'hnsw', 'ivf']
Similarity = Literal['COS', 'L2', 'IP']

# Available algorithms and similarity functions
ALGORITHMS: list[Algorithm] = ['diskann', 'hnsw', 'ivf']
SIMILARITIES: list[Similarity] = ['COS', 'L2', 'IP']

# Algorithm display labels for output
ALGORITHM_LABELS = {
    'diskann': 'DiskANN',
    'hnsw': 'HNSW',
    'ivf': 'IVF'
}


def get_index_options(
    collection_name: str,
    index_name: str,
    embedded_field: str,
    dimensions: int,
    algorithm: Algorithm,
    similarity: Similarity
) -> dict[str, Any]:
    """
    Build the index creation command for a specific algorithm and similarity function.
    
    Each algorithm has different tuning parameters:
    - DiskANN: maxDegree (graph connectivity), lBuild (build quality)
    - HNSW: m (graph connectivity), efConstruction (build quality)
    - IVF: numLists (number of clusters)
    """
    base = {
        "createIndexes": collection_name,
        "indexes": [
            {
                "name": index_name,
                "key": {embedded_field: "cosmosSearch"},
                "cosmosSearchOptions": {}
            }
        ]
    }

    # DiskANN: Disk-based approximate nearest neighbor search
    # Best for: Large datasets, memory-constrained environments
    if algorithm == 'diskann':
        base["indexes"][0]["cosmosSearchOptions"] = {
            "kind": "vector-diskann",
            "dimensions": dimensions,
            "similarity": similarity,
            "maxDegree": 32,  # Number of edges per node (higher = better accuracy, more memory)
            "lBuild": 50      # Candidates during build (higher = better quality, slower build)
        }
    # HNSW: Hierarchical Navigable Small World graph
    # Best for: High-accuracy requirements, fast search speed
    elif algorithm == 'hnsw':
        base["indexes"][0]["cosmosSearchOptions"] = {
            "kind": "vector-hnsw",
            "dimensions": dimensions,
            "similarity": similarity,
            "m": 16,              # Number of connections per layer (higher = better recall)
            "efConstruction": 64  # Candidates during construction (higher = better quality)
        }
    # IVF: Inverted File index with clustering
    # Best for: Large datasets, acceptable recall/latency tradeoff
    elif algorithm == 'ivf':
        base["indexes"][0]["cosmosSearchOptions"] = {
            "kind": "vector-ivf",
            "dimensions": dimensions,
            "similarity": similarity,
            "numLists": 1  # Number of clusters (higher = faster search, lower recall)
        }

    return base


def get_search_pipeline(
    query_embedding: list[float],
    embedded_field: str,
    k: int,
    algorithm: Algorithm
) -> list[dict[str, Any]]:
    """
    Build the vector search aggregation pipeline with algorithm-specific search parameters.
    
    Search parameters control the recall/latency tradeoff at query time:
    - DiskANN: lSearch (search list size)
    - HNSW: efSearch (search candidates)
    - IVF: nProbes (clusters to search)
    """
    cosmos_search = {
        "vector": query_embedding,
        "path": embedded_field,
        "k": k  # Number of results to return
    }

    # Add algorithm-specific search parameters
    if algorithm == 'diskann':
        cosmos_search["lSearch"] = 100  # Search list size (higher = better recall, slower)
    elif algorithm == 'hnsw':
        cosmos_search["efSearch"] = 80  # Candidates explored (higher = better recall, slower)
    elif algorithm == 'ivf':
        cosmos_search["nProbes"] = 1    # Clusters searched (higher = better recall, slower)

    # Build aggregation pipeline with vector search and score projection
    return [
        {"$search": {"cosmosSearch": cosmos_search}},
        {"$project": {"score": {"$meta": "searchScore"}, "document": "$$ROOT"}}
    ]


def get_target_collections(
    algorithm_env: str,
    similarity_env: str
) -> list[dict[str, Any]]:
    """
    Generate list of algorithm/similarity combinations to test.
    
    Supports testing:
    - All algorithms with a specific similarity function
    - A specific algorithm with all similarity functions
    - All combinations (9 total: 3 algorithms × 3 similarity functions)
    """
    algorithms = ALGORITHMS if algorithm_env == 'all' else [algorithm_env]
    similarities = SIMILARITIES if similarity_env == 'all' else [similarity_env]

    targets = []

    for alg in algorithms:
        if alg not in ALGORITHMS:
            raise ValueError(f"Invalid ALGORITHM '{alg}'. Must be one of: all, {', '.join(ALGORITHMS)}")

        for sim in similarities:
            if sim not in SIMILARITIES:
                raise ValueError(f"Invalid SIMILARITY '{sim}'. Must be one of: all, {', '.join(SIMILARITIES)}")

            targets.append({
                'collection_name': f"hotels_{alg}_{sim.lower()}",
                'algorithm': alg,
                'similarity': sim
            })

    return targets


def main() -> None:
    """
    Main comparison workflow:
    1. Load configuration from environment variables
    2. Initialize clients for DocumentDB and Azure OpenAI
    3. Load hotel data with pre-calculated embeddings
    4. For each algorithm/similarity combination:
       - Create collection and insert data
       - Create vector index with algorithm-specific parameters
       - Execute vector search with query-time parameters
       - Record results and latency
    5. Display comparison table showing performance across all configurations
    """
    # Load configuration from environment
    db_name = os.getenv('AZURE_DOCUMENTDB_DATABASENAME', 'Hotels')
    embedded_field = os.getenv('EMBEDDED_FIELD', 'DescriptionVector')
    embedding_dimensions = int(os.getenv('EMBEDDING_DIMENSIONS', '1536'))
    data_file = os.getenv('DATA_FILE_WITH_VECTORS', '../../data/Hotels_Vector.json')
    model_name = os.getenv('AZURE_OPENAI_EMBEDDING_MODEL', 'text-embedding-3-small')
    batch_size = int(os.getenv('LOAD_SIZE_BATCH', '100'))
    algorithm_env = os.getenv('ALGORITHM', 'all').strip().lower()
    similarity_env = os.getenv('SIMILARITY', 'COS').strip().upper()
    search_query = 'quintessential lodging near running trails, eateries, retail'

    try:
        # Validate and expand algorithm/similarity combinations
        targets = get_target_collections(algorithm_env, similarity_env)

        print("\nVector Algorithm Comparison")
        print(f"   Database: {db_name}")
        print(f"   Algorithms: {algorithm_env}")
        print(f"   Similarity: {similarity_env}")
        print(f"   Collections to query: {', '.join([t['collection_name'] for t in targets])}")
        print(f'   Search query: "{search_query}"\n')

        # Initialize MongoDB and Azure OpenAI clients with passwordless authentication
        print("\nInitializing MongoDB and Azure OpenAI clients...")
        mongo_client, azure_openai_client = get_clients_passwordless()

        database = mongo_client[db_name]

        # Load hotel data with embeddings
        script_dir = Path(__file__).parent
        data_path = script_dir / '..' / data_file
        print(f"\nLoading data from {data_path}...")
        data = read_file_return_json(str(data_path))
        print(f"Loaded {len(data)} documents")

        # Verify embeddings are present
        documents_with_embeddings = [doc for doc in data if embedded_field in doc]
        if not documents_with_embeddings:
            raise ValueError(f"No documents found with embeddings in field '{embedded_field}'")

        # Generate query embedding using Azure OpenAI
        print('Generating query embedding...')
        embedding_response = azure_openai_client.embeddings.create(
            model=model_name,
            input=[search_query]
        )
        query_embedding = embedding_response.data[0].embedding
        print(f"Query embedding: {len(query_embedding)} dimensions\n")

        # Store results for comparison
        comparison_results = []

        # Test each algorithm/similarity combination
        for target in targets:
            print(f"\n--- {ALGORITHM_LABELS[target['algorithm']]} / {target['similarity']} ---")
            print(f"Collection: {target['collection_name']}")

            try:
                # Drop existing collection to ensure clean test
                try:
                    database.drop_collection(target['collection_name'])
                except Exception as e:
                    print(f"  Note: could not drop existing collection: {e}")

                # Create new collection
                collection = database.create_collection(target['collection_name'])
                print(f"Created collection: {target['collection_name']}")

                # Insert hotel documents with embeddings
                insert_summary = insert_data(collection, documents_with_embeddings, batch_size)
                print(f"Inserted: {insert_summary['inserted']}/{insert_summary['total']}")

                # Create vector index with algorithm-specific parameters
                index_name = f"vectorIndex_{target['algorithm']}_{target['similarity'].lower()}"
                index_options = get_index_options(
                    target['collection_name'],
                    index_name,
                    embedded_field,
                    embedding_dimensions,
                    target['algorithm'],
                    target['similarity']
                )
                database.command(index_options)
                print(f"Created vector index: {index_name}")

                # Execute vector search and measure latency
                print('Executing vector search...')
                start_time = time.time()

                pipeline = get_search_pipeline(query_embedding, embedded_field, 5, target['algorithm'])
                # aggregate() returns a cursor (iterator); list() consumes all pages
                search_results = list(collection.aggregate(pipeline))

                latency_ms = (time.time() - start_time) * 1000

                # Store results for comparison table
                comparison_results.append({
                    'collection_name': target['collection_name'],
                    'algorithm': ALGORITHM_LABELS[target['algorithm']],
                    'similarity': target['similarity'],
                    'search_results': search_results,
                    'latency_ms': latency_ms
                })

                print(f"Success: {len(search_results)} results, {latency_ms:.0f}ms")

            except (pymongo.errors.PyMongoError, openai.APIError) as error:
                print(f"Error with {target['collection_name']}: {error}")

        # Display comparison table if any results were collected
        if comparison_results:
            print_comparison_table(comparison_results)

    except Exception as error:
        print(f"\nApp failed: {error}")
        raise

    finally:
        # Clean up database connection
        print('\nClosing database connection...')
        if 'mongo_client' in locals():
            mongo_client.close()
        print('Database connection closed')


if __name__ == "__main__":
    main()
```

This script orchestrates the algorithm comparison by:

- Loading configuration from environment variables
- Initializing MongoDB and Azure OpenAI clients with passwordless authentication
- Loading hotel data with pre-calculated embeddings
- Testing each algorithm/similarity combination by creating a collection, inserting data, creating an index, and executing a search
- Measuring and comparing search performance across all configurations
- Displaying results in a comparison table

## Create utility functions

Create the `src/utils.py` file with the following code:

```python
import json
import os
import warnings
from typing import Any

# Suppress the PyMongo CosmosDB cluster detection warning
warnings.filterwarnings(
    "ignore",
    message="You appear to be connected to a CosmosDB cluster.*",
)

from pymongo import MongoClient, InsertOne
from pymongo.collection import Collection
from pymongo.errors import BulkWriteError
from azure.identity import DefaultAzureCredential
from pymongo.auth_oidc import OIDCCallback, OIDCCallbackContext, OIDCCallbackResult
from openai import AzureOpenAI
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()


class AzureIdentityTokenCallback(OIDCCallback):
    """
    Callback for MongoDB OIDC authentication using Azure Identity.
    
    DocumentDB requires OIDC tokens for passwordless authentication.
    This callback fetches tokens from Azure AD using DefaultAzureCredential.
    """
    def __init__(self, credential):
        self.credential = credential

    def fetch(self, context: OIDCCallbackContext) -> OIDCCallbackResult:
        # Fetch token for DocumentDB scope
        token = self.credential.get_token(
            "https://ossrdbms-aad.database.windows.net/.default").token
        return OIDCCallbackResult(access_token=token)


def get_clients_passwordless() -> tuple[MongoClient, AzureOpenAI]:
    """
    Initialize MongoDB and Azure OpenAI clients using passwordless authentication.
    
    Uses DefaultAzureCredential which automatically tries multiple authentication methods:
    1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
    2. Managed Identity (when running in Azure)
    3. Azure CLI (az login)
    4. Azure PowerShell
    5. Visual Studio Code
    
    Returns:
        Tuple of (MongoClient, AzureOpenAI) configured for passwordless access
    """
    cluster_name = os.getenv("MONGO_CLUSTER_NAME")
    if not cluster_name:
        raise ValueError(
            "MONGO_CLUSTER_NAME environment variable is required.\n"
            "Create a .env file based on .env.example or set it in your environment."
        )

    # Create credential provider for Azure authentication
    credential = DefaultAzureCredential()

    # Configure OIDC authentication for MongoDB
    auth_properties = {"OIDC_CALLBACK": AzureIdentityTokenCallback(credential)}

    # Create MongoDB client with passwordless authentication
    mongo_client = MongoClient(
        f"mongodb+srv://{cluster_name}.mongocluster.cosmos.azure.com/",
        # 120s connect timeout accommodates cold-start latency on DocumentDB clusters
        connectTimeoutMS=120000,
        tls=True,
        retryWrites=False,
        authMechanism="MONGODB-OIDC",
        authMechanismProperties=auth_properties
    )

    # Get Azure OpenAI configuration
    azure_openai_endpoint = os.getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT")
    if not azure_openai_endpoint:
        raise ValueError(
            "AZURE_OPENAI_EMBEDDING_ENDPOINT environment variable is required.\n"
            "Create a .env file based on .env.example or set it in your environment."
        )

    # Create Azure OpenAI client with passwordless authentication
    azure_openai_client = AzureOpenAI(
        azure_endpoint=azure_openai_endpoint,
        # Token provider fetches Azure AD tokens automatically
        azure_ad_token_provider=lambda: credential.get_token("https://cognitiveservices.azure.com/.default").token,
        # See Azure OpenAI API version lifecycle:
        # /azure/ai-services/openai/api-version-deprecation
        api_version=os.getenv("AZURE_OPENAI_EMBEDDING_API_VERSION", "2023-05-15"),
        timeout=30.0,
        max_retries=3,
    )

    return mongo_client, azure_openai_client


def read_file_return_json(file_path: str) -> list[dict[str, Any]]:
    """
    Read a JSON file and return the parsed data.
    
    Args:
        file_path: Path to the JSON file
        
    Returns:
        List of dictionaries representing the JSON data
        
    Raises:
        FileNotFoundError: If file doesn't exist
        json.JSONDecodeError: If file contains invalid JSON
    """
    try:
        with open(file_path, 'r', encoding='utf-8') as file:
            return json.load(file)
    except FileNotFoundError:
        print(f"Error: File '{file_path}' not found")
        raise
    except json.JSONDecodeError as e:
        print(f"Error: Invalid JSON in file '{file_path}': {e}")
        raise


def insert_data(collection: Collection, data: list[dict[str, Any]], batch_size: int = 100) -> dict[str, int]:
    """
    Insert documents using bulk_write in batches.

    DocumentDB has a 16 MB command payload limit. Batch size of 100 stays well
    within this limit while keeping round-trip overhead reasonable.
    
    Args:
        collection: MongoDB collection to insert into
        data: List of documents to insert
        batch_size: Number of documents per batch (default: 100)
        
    Returns:
        Dictionary with 'total', 'inserted', and 'failed' counts
    """
    total_documents = len(data)
    inserted_count = 0
    failed_count = 0

    print(f"Inserting {total_documents} documents in batches of {batch_size}...")

    # Process documents in batches
    for i in range(0, total_documents, batch_size):
        batch = data[i:i + batch_size]
        batch_num = (i // batch_size) + 1

        try:
            # Use bulk_write with InsertOne operations for efficiency
            operations = [InsertOne(document) for document in batch]
            result = collection.bulk_write(operations, ordered=False)
            inserted_count += result.inserted_count
            print(f"Batch {batch_num} completed: {result.inserted_count} documents inserted")

        except BulkWriteError as e:
            # Handle partial batch failures
            inserted_count += e.details.get('nInserted', 0)
            failed_count += len(batch) - e.details.get('nInserted', 0)
            print(f"Batch {batch_num} had errors: {e.details.get('nInserted', 0)} inserted, {failed_count} failed")

        except Exception as e:
            # Handle complete batch failure
            failed_count += len(batch)
            print(f"Batch {batch_num} failed completely: {e}")

    return {
        'total': total_documents,
        'inserted': inserted_count,
        'failed': failed_count
    }


def print_comparison_table(results: list[dict[str, Any]]) -> None:
    """
    Display comparison results in a formatted table.
    
    Shows:
    - Algorithm and similarity function used
    - Top search result (hotel name)
    - Search score
    - Query latency in milliseconds
    
    Followed by detailed results for each configuration.
    """
    if not results:
        print("No comparison results to display.")
        return

    print("\n" + "=" * 90)
    print("                    Vector Algorithm Comparison Results")
    print("=" * 90)

    # Print table header
    header = (
        f"{'Algorithm':<12} "
        f"{'Similarity':<14} "
        f"{'Top Result':<24} "
        f"{'Score':<12} "
        f"{'Latency(ms)':<14}"
    )
    print(header)
    print("-" * 90)

    # Print summary row for each result
    for r in results:
        top_result = r['search_results'][0] if r['search_results'] else None
        if top_result:
            doc = top_result.get('document', top_result)
            top_name = doc.get('HotelName', 'N/A')[:22]
            top_score = f"{top_result['score']:.4f}"
        else:
            top_name = 'N/A'
            top_score = 'N/A'

        row = (
            f"{r['algorithm']:<12} "
            f"{r['similarity']:<14} "
            f"{top_name:<24} "
            f"{top_score:<12} "
            f"{r['latency_ms']:<14.0f}"
        )
        print(row)

    print("=" * 90)

    # Print detailed results for each configuration
    for r in results:
        print(f"\n--- {r['algorithm']} / {r['similarity']} ({r['collection_name']}) ---")
        if not r['search_results']:
            print("  No results.")
            continue

        for i, item in enumerate(r['search_results'], 1):
            doc = item.get('document', item)
            score = item['score']
            print(f"  {i}. {doc.get('HotelName', 'N/A')}, Score: {score:.4f}")

        print(f"  Latency: {r['latency_ms']:.0f}ms")
```

The utilities provide essential functions for:

- Passwordless authentication to DocumentDB and Azure OpenAI using DefaultAzureCredential
- Reading JSON data files with error handling
- Batch insertion of documents with DocumentDB's 16 MB payload limit in mind
- Formatted display of comparison results showing algorithm performance

## Run the code

Execute the comparison script to test all algorithms with cosine similarity:

```bash
python src/select_algorithm.py
```

The output shows the comparison across all three algorithms:

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

--- DiskANN / COS ---
Collection: hotels_diskann_cos
Created collection: hotels_diskann_cos
Inserting 50 documents in batches of 100...
Batch 1 completed: 50 documents inserted
Inserted: 50/50
Created vector index: vectorIndex_diskann_cos
Executing vector search...
Success: 5 results, 145ms

--- HNSW / COS ---
Collection: hotels_hnsw_cos
Created collection: hotels_hnsw_cos
Inserting 50 documents in batches of 100...
Batch 1 completed: 50 documents inserted
Inserted: 50/50
Created vector index: vectorIndex_hnsw_cos
Executing vector search...
Success: 5 results, 132ms

--- IVF / COS ---
Collection: hotels_ivf_cos
Created collection: hotels_ivf_cos
Inserting 50 documents in batches of 100...
Batch 1 completed: 50 documents inserted
Inserted: 50/50
Created vector index: vectorIndex_ivf_cos
Executing vector search...
Success: 5 results, 128ms

==========================================================================================
                    Vector Algorithm Comparison Results
==========================================================================================
Algorithm    Similarity     Top Result               Score        Latency(ms)   
------------------------------------------------------------------------------------------
DiskANN      COS            Twin Dome Motel          0.8947       145           
HNSW         COS            Twin Dome Motel          0.8947       132           
IVF          COS            Twin Dome Motel          0.8947       128           
==========================================================================================

--- DiskANN / COS (hotels_diskann_cos) ---
  1. Twin Dome Motel, Score: 0.8947
  2. Triple Landscape Hotel, Score: 0.8898
  3. Smile Hotel, Score: 0.8855
  4. Gastronomic Landscape Hotel, Score: 0.8797
  5. Twin Landscape Resort, Score: 0.8772
  Latency: 145ms

--- HNSW / COS (hotels_hnsw_cos) ---
  1. Twin Dome Motel, Score: 0.8947
  2. Triple Landscape Hotel, Score: 0.8898
  3. Smile Hotel, Score: 0.8855
  4. Gastronomic Landscape Hotel, Score: 0.8797
  5. Twin Landscape Resort, Score: 0.8772
  Latency: 132ms

--- IVF / COS (hotels_ivf_cos) ---
  1. Twin Dome Motel, Score: 0.8947
  2. Triple Landscape Hotel, Score: 0.8898
  3. Smile Hotel, Score: 0.8855
  4. Gastronomic Landscape Hotel, Score: 0.8797
  5. Twin Landscape Resort, Score: 0.8772
  Latency: 128ms

Closing database connection...
Database connection closed
```

### Test specific combinations

To override environment variables at the command line:

### [Bash](#tab/bash)

```bash
# Test only DiskANN across all similarity functions
ALGORITHM=diskann SIMILARITY=all python src/select_algorithm.py
```

```bash
# Test all algorithms with L2 distance
ALGORITHM=all SIMILARITY=L2 python src/select_algorithm.py
```

```bash
# Test HNSW with inner product
ALGORITHM=hnsw SIMILARITY=IP python src/select_algorithm.py
```

### [PowerShell](#tab/powershell)

```powershell
# Test only DiskANN across all similarity functions
$env:ALGORITHM="diskann"; $env:SIMILARITY="all"; python src/select_algorithm.py
```

```powershell
# Test all algorithms with L2 distance
$env:ALGORITHM="all"; $env:SIMILARITY="L2"; python src/select_algorithm.py
```

```powershell
# Test HNSW with inner product
$env:ALGORITHM="hnsw"; $env:SIMILARITY="IP"; python src/select_algorithm.py
```

---

> [!NOTE]
> When using `SIMILARITY=all`, the script tests all three similarity functions (COS, L2, IP) for each selected algorithm. Combined with `ALGORITHM=all`, this runs all 9 combinations (3 algorithms × 3 similarity functions). Each combination creates a separate collection, so the full run takes longer.

### Understanding the results

The comparison table helps you choose the best configuration for your workload:

- **Latency**: Query execution time in milliseconds. Lower is better for user-facing search.
- **Score**: Similarity score using the selected function. Higher scores indicate better matches.
- **Top Result**: The highest-scoring hotel for the query. Consistency across algorithms indicates stable results.

Algorithm selection guidelines:

- **DiskANN**: Best for large datasets where memory is limited. Stores index on disk while maintaining good performance.
- **HNSW**: Best for high-accuracy requirements and fast search. Requires more memory but provides excellent recall.
- **IVF**: Best for very large datasets where some recall can be traded for speed. Uses clustering for efficient search.

Similarity function selection:

- **COS (Cosine)**: Best for text embeddings. Normalizes vectors and measures angle between them.
- **L2 (Euclidean)**: Measures straight-line distance. Sensitive to vector magnitude.
- **IP (Inner Product)**: Dot product similarity. Useful when vector magnitude is meaningful.

Tuning parameters:

DiskANN tuning:
- `maxDegree`: Higher values improve accuracy but increase memory usage (default: 32)
- `lBuild`: Higher values improve index quality but slow down index creation (default: 50)
- `lSearch`: Higher values improve recall but slow down queries (default: 100)

HNSW tuning:
- `m`: Number of connections per layer. Higher improves recall (default: 16)
- `efConstruction`: Candidates during build. Higher improves quality (default: 64)
- `efSearch`: Candidates during search. Higher improves recall (default: 80)

IVF tuning:
- `numLists`: Number of clusters. Higher speeds up search but may reduce recall (default: 1)
- `nProbes`: Clusters searched at query time. Higher improves recall but slows queries (default: 1)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ServerSelectionTimeoutError` | Verify your connection string in `.env`. Ensure your IP is in the DocumentDB firewall rules. |
| `AuthenticationFailed` | Check that your connection string includes the correct username and password, or that your Microsoft Entra token is valid. |
| `pymongo.errors.OperationFailure` | Ensure the database and collection exist. Check that the vector index was created successfully. |
| `ModuleNotFoundError: No module named 'pymongo'` | Activate your virtual environment and run `pip install "pymongo>=4.7"`. |
| Empty search results | The vector index may not be ready yet. The script includes retry logic, but large datasets may require longer wait times. |

## Clean up resources

When you're done, you can remove the database using mongosh or the Azure portal.

### [mongosh](#tab/mongosh)

Connect to your DocumentDB cluster and drop the database:

```bash
mongosh "mongodb+srv://<your-cluster-name>.mongocluster.cosmos.azure.com/" --tls --authenticationMechanism MONGODB-OIDC
```

```javascript
use Hotels
db.dropDatabase()
```

### [Azure portal](#tab/portal)

1. Navigate to your DocumentDB resource in the Azure portal.
2. Select **Data Explorer**.
3. Right-click the **Hotels** database and select **Delete Database**.

---

If you created an Azure DocumentDB cluster specifically for this quickstart, you can also delete the entire resource group in the Azure portal to remove all associated resources.

## Next steps

- [Vector search concepts in Azure DocumentDB](concept-vector-search)
- [How to use vector search in Azure DocumentDB](how-to-vector-search)
- [Optimize vector search performance](how-to-optimize-vector-search-performance)
- [Azure DocumentDB Python SDK reference](https://pymongo.readthedocs.io/)
