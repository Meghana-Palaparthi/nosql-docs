---
title: "Quickstart - Vector Indexing with Python"
description: "Learn how to choose and configure IVF, HNSW, and DiskANN vector indexes in Azure DocumentDB with Python."
ms.reviewer: khelanmodi
ms.devlang: python
ms.topic: quickstart-sdk
ms.date: 07/14/2025
ai-usage: ai-assisted
ms.custom:
  - devx-track-python
  - devx-track-data-ai
# CustomerIntent: As a developer, I want to choose and configure the right vector index algorithm for my dataset size in Azure DocumentDB.
---

# Quickstart: Vector indexing in Azure DocumentDB with Python

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/select-algorithm-python) on GitHub.

Learn how to create and use vector indexes in Azure DocumentDB to enable efficient similarity search with LLM embeddings. This quickstart shows how to set up IVF, HNSW, and DiskANN indexes—each optimized for different dataset sizes and performance requirements.

## Prerequisites

- An Azure subscription ([create one for free](https://azure.microsoft.com/free/))
- Azure DocumentDB vCore cluster with appropriate tier:
  - **IVF**: M10 or higher
  - **HNSW**: M30 or higher
  - **DiskANN**: M30 or higher
- [Azure OpenAI resource](https://learn.microsoft.com/azure/ai-services/openai/how-to/create-resource) with an embeddings model deployed
- [Python 3.10+](https://www.python.org/downloads/)
- pip (Python package manager)

## Set up the project

1. Create and navigate to a new project directory:

```bash
mkdir documentdb-vector-quickstart
cd documentdb-vector-quickstart
```

2. Create a virtual environment:

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install required packages:

```bash
pip install pymongo azure-identity openai python-dotenv
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

Create `ivf.py`:

```python
import os
from typing import List, Dict, Any
from pymongo import MongoClient
from pymongo.auth_oidc import OIDCCallback, OIDCCallbackContext, OIDCCallbackResult
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI
from dotenv import load_dotenv

load_dotenv()

class AzureIdentityTokenCallback(OIDCCallback):
    def __init__(self, credential):
        self.credential = credential

    def fetch(self, context: OIDCCallbackContext) -> OIDCCallbackResult:
        token = self.credential.get_token(
            "https://ossrdbms-aad.database.windows.net/.default").token
        return OIDCCallbackResult(access_token=token)

def create_ivf_vector_index(collection, vector_field: str, dimensions: int) -> None:
    """Create an IVF vector index on the specified field"""
    print(f"Creating IVF vector index on field '{vector_field}'...")

    # Use the native MongoDB command for DocumentDB vector indexes
    index_command = {
        "createIndexes": collection.name,
        "indexes": [
            {
                "name": f"ivf_index_{vector_field}",
                "key": {
                    vector_field: "cosmosSearch"  # DocumentDB vector search index type
                },
                "cosmosSearchOptions": {
                    # IVF algorithm configuration
                    "kind": "vector-ivf",
                    
                    # Vector dimensions must match the embedding model
                    "dimensions": dimensions,
                    
                    # Cosine similarity is effective for text embeddings
                    "similarity": "COS",
                    
                    # Number of clusters (centroids) to partition vectors into
                    # More clusters = faster search but potentially lower recall
                    "numLists": 10
                }
            }
        ]
    }

    try:
        # Execute the createIndexes command directly
        result = collection.database.command(index_command)
        print("IVF vector index created successfully")
    except Exception as e:
        print(f"Error creating IVF vector index: {e}")
        raise

def perform_ivf_vector_search(collection,
                              azure_openai_client,
                              query_text: str,
                              vector_field: str,
                              model_name: str,
                              top_k: int = 5) -> List[Dict[str, Any]]:
    """Perform a vector search using IVF algorithm"""
    print(f"Performing IVF vector search for: '{query_text}'")

    try:
        # Generate embedding vector for the search query
        embedding_response = azure_openai_client.embeddings.create(
            input=[query_text],
            model=model_name
        )

        query_embedding = embedding_response.data[0].embedding

        # Construct aggregation pipeline for IVF vector search
        pipeline = [
            {
                "$search": {
                    # Use cosmosSearch for vector operations in DocumentDB
                    "cosmosSearch": {
                        # Query vector to find similar documents
                        "vector": query_embedding,
                        
                        # Document field containing vectors to search against
                        "path": vector_field,
                        
                        # Final number of results to return
                        "k": top_k
                    }
                }
            },
            {
                # Project only the fields we want in the output and add similarity score
                "$project": {
                    "document": "$$ROOT",
                    # Add search score from metadata
                    "score": {"$meta": "searchScore"}
                }
            }
        ]

        # Run the search aggregation pipeline
        results = list(collection.aggregate(pipeline))
        return results

    except Exception as e:
        print(f"Error performing IVF vector search: {e}")
        raise

def main():
    # Create credential and clients
    credential = DefaultAzureCredential()
    
    # Create MongoDB client with OIDC authentication
    mongo_client = MongoClient(
        f"mongodb+srv://{os.getenv('MONGO_CLUSTER_NAME')}.mongocluster.cosmos.azure.com/",
        connectTimeoutMS=120000,
        tls=True,
        retryWrites=False,
        authMechanism="MONGODB-OIDC",
        authMechanismProperties={"OIDC_CALLBACK": AzureIdentityTokenCallback(credential)}
    )

    # Create Azure OpenAI client
    azure_openai_client = AzureOpenAI(
        azure_endpoint=os.getenv("AZURE_OPENAI_EMBEDDING_ENDPOINT"),
        azure_ad_token_provider=lambda: credential.get_token("https://cognitiveservices.azure.com/.default").token,
        api_version=os.getenv("AZURE_OPENAI_EMBEDDING_API_VERSION")
    )

    try:
        # Create collection and index
        database = mongo_client["Hotels"]
        collection = database["hotels_ivf"]
        
        create_ivf_vector_index(
            collection,
            os.getenv("EMBEDDED_FIELD"),
            int(os.getenv("EMBEDDING_DIMENSIONS"))
        )

        # Perform search
        query = "quintessential lodging near running trails, eateries, retail"
        results = perform_ivf_vector_search(
            collection,
            azure_openai_client,
            query,
            os.getenv("EMBEDDED_FIELD"),
            os.getenv("AZURE_OPENAI_EMBEDDING_MODEL"),
            top_k=5
        )

        print(f"\nSearch Results ({len(results)} found):")
        for i, result in enumerate(results, 1):
            print(f"{i}. {result['document']['HotelName']}, Score: {result['score']:.4f}")

    finally:
        mongo_client.close()

if __name__ == "__main__":
    main()
```

## Create an HNSW index

HNSW (Hierarchical Navigable Small World) is ideal for datasets between 10,000 and 50,000 documents. It builds a graph-based index for faster search with better recall.

Create `hnsw.py`:

```python
# Similar to ivf.py above, but with these index options:
"cosmosSearchOptions": {
    # HNSW algorithm configuration
    "kind": "vector-hnsw",
    
    # Vector dimensions must match the embedding model
    "dimensions": dimensions,
    
    # Cosine similarity works well with text embeddings
    "similarity": "COS",
    
    # Maximum connections per node in the graph (parameter 'm')
    # Higher values improve recall but increase memory usage and build time
    "m": 16,
    
    # Size of the candidate list during construction
    # Higher values improve index quality but slow down building
    "efConstruction": 64
}
```

Key differences from IVF:
- **m parameter**: Controls graph connectivity. Higher values (e.g., 32) improve recall but increase memory.
- **efConstruction**: Affects index build time and quality. Higher values improve accuracy at cost of build time.
- **Cluster tier**: Requires M30 or higher due to memory overhead.

## Create a DiskANN index

DiskANN is optimized for very large datasets (50,000+ documents) with efficient disk-based storage.

Create `diskann.py`:

```python
# Similar to ivf.py and hnsw.py above, but with these index options:
"cosmosSearchOptions": {
    # DiskANN algorithm configuration
    "kind": "vector-diskann",
    
    # Vector dimensions must match the embedding model
    "dimensions": dimensions,
    
    # Vector similarity metric - cosine is good for text embeddings
    "similarity": "COS",
    
    # Maximum degree: number of edges per node in the graph
    # Higher values improve accuracy but increase memory usage
    "maxDegree": 20,
    
    # Build parameter: candidates evaluated during index construction
    # Higher values improve index quality but increase build time
    "lBuild": 10
}
```

Key parameters:
- **maxDegree**: Number of edges per node in the graph. Higher values improve accuracy.
- **lBuild**: Number of candidate neighbors evaluated during construction. Affects index quality.
- **Cluster tier**: Requires M30 or higher.

## Query with vector search

All three algorithms use the same query pattern with the `$search` aggregation stage:

```python
# Generate embedding for query
embedding_response = azure_openai_client.embeddings.create(
    input=["your query text"],
    model=model_name
)

# Execute vector search
pipeline = [
    {
        "$search": {
            "cosmosSearch": {
                "vector": embedding_response.data[0].embedding,
                "path": vector_field,
                "k": 5  # Return top 5 similar documents
            }
        }
    },
    {
        "$project": {
            "score": {"$meta": "searchScore"},
            "document": "$$ROOT"
        }
    }
]

results = list(collection.aggregate(pipeline))
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
python ivf.py

# Run HNSW example
python hnsw.py

# Run DiskANN example
python diskann.py
```

## Clean up resources

When you're done, delete the DocumentDB cluster and OpenAI resource from the Azure Portal to avoid ongoing charges.

## Next steps

- [DocumentDB Vector Search Documentation](https://learn.microsoft.com/azure/cosmos-db/mongodb/vcore/vector-search)
- [Azure OpenAI Embeddings Documentation](https://learn.microsoft.com/azure/ai-services/openai/concepts/understand-embeddings)
- [PyMongo Documentation](https://pymongo.readthedocs.io/)
- Article 1: Getting Started with Vector Search
- Article 3: Performance Tuning and Optimization
