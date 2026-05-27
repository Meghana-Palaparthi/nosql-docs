---
title: Quickstart - Use Vector Search with Python
description: Learn how to use vector search in Azure DocumentDB with Python. Store and query vector data efficiently in your applications. 
author: seesharprun
ms.author: sidandrews
ms.reviewer: khelanmodi
ms.devlang: python
ms.topic: quickstart-sdk
ms.date: 05/22/2026
ms.collection: ce-skilling-ai-copilot
ms.update-cycle: 180-days
ai-usage: ai-assisted
ms.custom:
  - devx-track-python
  - devx-track-python-ai
  - devx-track-data-ai
# CustomerIntent: As a developer, I want to learn how to use vector search in Python applications with Azure DocumentDB.
---

# Quickstart: Use vector search with Python in Azure DocumentDB

Use vector search in Azure DocumentDB with the Python client library to store and query vector data efficiently.

This quickstart uses a sample hotel dataset in a JSON file with precalculated vectors from the `text-embedding-3-small` model. The dataset includes hotel names, locations, descriptions, and vector embeddings.

Find the [sample code](https://github.com/Azure-Samples/documentdb-samples/tree/main/ai/vector-search-python) on GitHub.

## Prerequisites

[!INCLUDE[Prerequisites - Vector Search Quickstart](includes/prerequisite-quickstart-vector-search-model.md)]

   > [!TIP]
   > To customize Azure OpenAI model parameters before deployment, see the section [Customize Azure OpenAI deployment](#customize-azure-openai-deployment-optional) later in this article.

- [Python](https://www.python.org/downloads/) 3.9 or later.

## Create a data file with vectors

1. Create a new data directory for the hotels data file:

    ```bash
    mkdir data
    ```

1. Copy the `Hotels_Vector.json` [raw data file with vectors](https://raw.githubusercontent.com/Azure-Samples/documentdb-samples/refs/heads/main/ai/data/Hotels_Vector.json) to your `data` directory.

## Create a Python project

1. Create a new directory for your project and open it in Visual Studio Code:

    ```bash
    mkdir vector-search-quickstart
    code vector-search-quickstart
    ```

1. In the terminal, create and activate a virtual environment:

    For Windows:

    ```bash
    python -m venv venv
    venv\\Scripts\\activate
    ```

    For macOS/Linux:

    ```bash
    python -m venv venv
    source venv/bin/activate
    ```

1. Install the required packages:

    ```bash
    pip install pymongo azure-identity openai python-dotenv
    ```

    - `pymongo`: The MongoDB driver for Python.
    - `azure-identity`: The Azure Identity library for passwordless authentication.
    - `openai`: The OpenAI client library to create vectors.
    - `python-dotenv`: The environment variable management from .env files.

1. Create a `.env` file for environment variables in `vector-search-quickstart`:

    ```ini
    # Identity for local developer authentication with Azure CLI
    AZURE_TOKEN_CREDENTIALS=AzureCliCredential

    # Azure OpenAI configuration
    AZURE_OPENAI_EMBEDDING_ENDPOINT= 
    AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
    AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15

    # Azure DocumentDB configuration
    MONGO_CLUSTER_NAME=

    # Data Configuration (defaults should work)
    DATA_FILE_WITH_VECTORS=../data/Hotels_Vector.json
    EMBEDDED_FIELD=DescriptionVector
    EMBEDDING_DIMENSIONS=1536
    EMBEDDING_SIZE_BATCH=16
    LOAD_SIZE_BATCH=50
    ```

    For the passwordless authentication used in this article, replace the placeholder values in the `.env` file with your own information:
    - `AZURE_OPENAI_EMBEDDING_ENDPOINT`: Your Azure OpenAI resource endpoint URL.
    - `MONGO_CLUSTER_NAME`: Your Azure DocumentDB resource name.

    We recommend that you use passwordless authentication, but this type of authentication requires more setup steps. For more information on setting up managed identity and the full range of your authentication options, see [Authenticate Python apps to Azure services by using the Azure SDK for Python](/azure/developer/python/sdk/authentication/overview).

## Create code files for vector search

Continue the project by creating code files for vector search. When you finish, the project structure looks like this example:

```plaintext
├── data/
│   ├── Hotels.json              # Source hotel data (without vectors)
│   └── Hotels_Vector.json       # Hotel data with vector embeddings
└── vector-search-quickstart/
    ├── src/
    │   ├── diskann.py           # DiskANN vector search implementation
    │   ├── hnsw.py              # HNSW vector search implementation
    │   ├── ivf.py               # IVF vector search implementation
    │   └── utils.py              # Shared utility functions
    ├── requirements.txt         # Python dependencies
    ├── .env                     # Environment variables template
```

### [DiskANN](#tab/tab-diskann)

Create an `src` directory for your Python files. Add two files, `diskann.py` and `utils.py`, for the DiskANN index implementation.

```bash
mkdir src    
touch src/diskann.py
touch src/utils.py
```

#### [IVF](#tab/tab-ivf)

Create an `src` directory for your Python files. Add two files, `ivf.py` and `utils.py`, for the Inverted File (IVF) index implementation.

```bash
mkdir src
touch src/ivf.py
touch src/utils.py
```

#### [HNSW](#tab/tab-hnsw)

Create an `src` directory for your Python files. Add two files, `hnsw.py` and `utils.py`, for the Hierarchical Navigable Small World (HNSW) index implementation.

```bash
mkdir src
touch src/hnsw.py
touch src/utils.py
```

----

> [!TIP]
> Unlike some databases, Azure DocumentDB allows you to create and drop vector indexes at any time after you create a container. You don't need to define the vector indexing policy at the time that you create a container.

## Create the code for vector search

### [DiskANN](#tab/tab-diskann)

Paste the following code into the `diskann.py` file.

:::code language="python" source="~/../documentdb-samples/ai/vector-search-python/src/diskann.py" :::

#### [IVF](#tab/tab-ivf)

Paste the following code into the `ivf.py` file.

:::code language="python" source="~/../documentdb-samples/ai/vector-search-python/src/ivf.py" :::

#### [HNSW](#tab/tab-hnsw)

Paste the following code into the `hnsw.py` file.

:::code language="python" source="~/../documentdb-samples/ai/vector-search-python/src/hnsw.py" :::

----

This main module:

- Includes utility functions.

- Creates a configuration object for environment variables.

- Creates clients for Azure OpenAI and Azure DocumentDB.

- Connects to MongoDB, creates a database and collection, inserts data, and creates standard indexes.

- Creates a vector index that uses IVF, HNSW, or DiskANN.

- Creates an embedding for a sample query text by using the OpenAI client. You can change the query at the top of the file.

- Runs a vector search that uses the embedding, and prints the results.

## Create utility functions

Paste the following code into `utils.py`:

:::code language="python" source="~/../documentdb-samples/ai/vector-search-python/src/utils.py" :::

This utility module provides these features:

- `get_clients`: Creates and returns clients for Azure OpenAI and Azure DocumentDB.

- `get_clients_passwordless`: Creates and returns clients for Azure OpenAI and Azure DocumentDB by using passwordless authentication.

- `azure_identity_token_callback`: Gets an Azure AD token used by MongoDB OpenID Connect (OIDC) authentication.

- `read_file_return_json`: Reads a JSON file and returns its contents as an array of objects.

- `write_file_json`: Writes an array of objects to a JSON file.

- `insert_data`: Inserts data in batches into a MongoDB collection, and creates standard indexes on specified fields.

- `drop_vector_indexes`: Drops existing vector indexes on the target vector field.

- `print_search_results`: Prints vector search results, including score and hotel name.

## Authenticate with the Azure CLI

Before you run the application, sign in to the Azure CLI so the app can access Azure resources securely.

```bash
az login
```

The code uses your local developer authentication to access Azure DocumentDB and Azure OpenAI. When you set `AZURE_TOKEN_CREDENTIALS=AzureCliCredential`, this setting tells the function to use the Azure CLI credentials for authentication _deterministically_. The authentication relies on [DefaultAzureCredential](/python/api/azure-identity/azure.identity.defaultazurecredential) from `azure-identity` to find your Azure credentials in the environment. Learn more about how to [Authenticate Python apps to Azure services using the Azure Identity library](/azure/developer/python/sdk/authentication/overview).

## Run the application

To run the Python scripts:

### [DiskANN](#tab/tab-diskann)

```bash
python src/diskann.py
```

#### [IVF](#tab/tab-ivf)

```bash
python src/ivf.py
```

#### [HNSW](#tab/tab-hnsw)

```bash
python src/hnsw.py
```

----

You see the top five hotels that match the vector search query and their similarity scores.

## View and manage data in Visual Studio Code

1. Select the [Azure DocumentDB extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) in Visual Studio Code to connect to your Azure DocumentDB account.

1. View the data and indexes in the **Hotels** database.

    :::image type="content" source="./media/quickstart-nodejs-vector-search/visual-studio-code-documentdb.png" lightbox="./media/quickstart-nodejs-vector-search/visual-studio-code-documentdb.png" alt-text="Screenshot of Azure DocumentDB extension showing the Azure DocumentDB collection.":::

[!INCLUDE[Customize OpenAI deployment](./includes/section-quickstart-openai-configuration-vector-search.md)]

## Clean up resources

When you no longer need them, delete the resource group, Azure DocumentDB cluster, and Azure OpenAI resource to avoid unnecessary costs.

## Related content

- [Vector store in Azure DocumentDB](vector-search.md)
- [Support for geospatial queries](geospatial-support.md)
