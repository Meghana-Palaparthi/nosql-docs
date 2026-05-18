---
title: Integrations for AI
description: Learn how to integrate Azure Cosmos DB with AI and large language model (LLM) orchestration frameworks like Semantic Kernel, LangChain, and LlamaIndex for building intelligent applications.
author: jcodella
ms.author: jacodel
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 10/20/2025
ms.update-cycle: 180-days
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
---
# Azure Cosmos DB integrations for AI applications

Azure Cosmos DB for NoSQL integrates with the most widely used AI and LLM orchestration frameworks, providing a single persistence layer for vector search, chat history, semantic caching, agent state, and long-term memory. This article summarizes the available integrations and points to the official connector for each language.

All Azure Cosmos DB connectors support both account-key and Microsoft Entra ID (Managed Identity) authentication unless otherwise noted.

## Integration support at a glance

| Framework | Python | .NET / C# | Java | JavaScript / TypeScript |
|---|---|---|---|---|
| Semantic Kernel | ✅ Vector store | ✅ Vector store  | — | — |
| LangChain | ✅ Vector store, semantic cache, chat history | — | ✅ Embedding store | ✅ Vector store, semantic cache |
| LangGraph | ✅ Checkpointer, node cache, long-term memory | — | — | — |
| Agent Framework | ✅ Workflow checkpoint, history provider | ✅ Checkpoint store, chat history | — | — |
| LlamaIndex | ✅ Vector store, document store, index store, chat store, KV store | — | — | — |
| Spring AI | — | — | ✅ Vector store | — |

## Semantic Kernel

[Semantic Kernel](https://github.com/microsoft/semantic-kernel) is Microsoft's open-source SDK for building AI agents and multi-agent systems in Python, .NET, and Java.

### Python

The `CosmosNoSqlStore` and `CosmosNoSqlCollection` classes provide vector store access. See the [Semantic Kernel Python connector documentation](/semantic-kernel/concepts/vector-store-connectors/out-of-the-box-connectors/azure-cosmosdb-nosql-connector?pivots=programming-language-python).

### .NET / C#

The [`Microsoft.SemanticKernel.Connectors.CosmosNoSql`](https://www.nuget.org/packages/Microsoft.SemanticKernel.Connectors.CosmosNoSql) NuGet package provides a vector store connector implementing `Microsoft.Extensions.VectorData`. See the [.NET connector documentation](/semantic-kernel/concepts/vector-store-connectors/out-of-the-box-connectors/azure-cosmosdb-nosql-connector?pivots=programming-language-csharp). *Currently in preview.*

### Java

A native Azure Cosmos DB for NoSQL vector store connector isn't currently available in Semantic Kernel for Java. Java users can integrate via [LangChain4j](#langchain) or [Spring AI](#spring-ai).

## LangChain

[LangChain](https://www.langchain.com/) is a framework for building LLM-powered applications, with implementations across Python, JavaScript, and Java.

### Python

The [`langchain-azure-cosmosdb`](https://pypi.org/project/langchain-azure-cosmosdb/) package is the recommended Python connector. It provides six integrations across LangChain and LangGraph (see the [LangGraph section](#langgraph) below for graph-specific components), each with synchronous and asynchronous variants.

| Functionality | Sync | Async |
|---|---|---|
| Vector store | `AzureCosmosDBNoSqlVectorSearch` | `AsyncAzureCosmosDBNoSqlVectorSearch` |
| Semantic cache | `AzureCosmosDBNoSqlSemanticCache` | `AsyncAzureCosmosDBNoSqlSemanticCache` |
| Chat message history | `CosmosDBChatMessageHistory` | `AsyncCosmosDBChatMessageHistory` |

Capabilities include vector search (DiskANN, Quantized Flat, Flat indexes), full-text (BM25) search, hybrid search, weighted hybrid search, LLM response caching, and persistent chat message history.

### JavaScript / TypeScript

The [`@langchain/azure-cosmosdb`](https://www.npmjs.com/package/@langchain/azure-cosmosdb) package provides `AzureCosmosDBNoSQLVectorStore`, `AzureCosmosDBNoSQLSemanticCache`, and `AzureCosmosDBNoSQLChatMessageHistory`. See the [JS vector store documentation](https://docs.langchain.com/oss/javascript/integrations/vectorstores/azure_cosmosdb_nosql).

### Java

LangChain4j provides an Azure Cosmos DB for NoSQL embedding store. See the [LangChain4j integration documentation](https://docs.langchain4j.dev/integrations/embedding-stores/azure-cosmos-nosql/).

## LangGraph

[LangGraph](https://www.langchain.com/langgraph) extends LangChain with stateful, multi-actor agent workflows.

### Python

LangGraph integration ships in the same [`langchain-azure-cosmosdb`](https://pypi.org/project/langchain-azure-cosmosdb/) package as the LangChain integration above.

| Functionality | Sync | Async |
|---|---|---|
| Graph state persistence (checkpointer) | `CosmosDBSaverSync` | `CosmosDBSaver` |
| Node-level result caching | `CosmosDBCacheSync` | `CosmosDBCache` |
| Long-term memory store | `CosmosDBStore` | `AsyncCosmosDBStore` |

The long-term memory store optionally uses vector search for semantic recall.

## Agent Framework

[Agent Framework](https://github.com/microsoft/agent-framework) is Microsoft's framework for building AI agents and multi-agent workflows, succeeding AutoGen and consolidating capabilities from Semantic Kernel agents.

### Python

The [Azure Cosmos DB package for Agent Framework Python](https://github.com/microsoft/agent-framework/tree/44381c051b3915f8b60a7972641e06c546f5df9d/python/packages/azure-cosmos) provides:

- `CosmosDBWorkflowCheckpointStorage` — workflow checkpoint storage
- `CosmosDBHistoryProvider` — chat history provider
- Database and container setup utilities

### .NET / C#

The [`Microsoft.Agents.AI.CosmosNoSql`](https://github.com/microsoft/agent-framework/tree/main/dotnet/src/Microsoft.Agents.AI.CosmosNoSql) package provides:

- `CheckpointStore` — workflow checkpoint storage
- `ChatHistoryProvider` — chat history management
- `WorkflowExtensions` and `ChatExtensions` — DI and integration helpers

>[!NOTE]
> **AutoGen users:** AutoGen has been merged into Agent Framework. New projects should target Agent Framework directly. The legacy [AutoGen 0.2 Cosmos DB notes](https://microsoft.github.io/autogen/0.2/docs/ecosystem/azure_cosmos_db/) remain available for reference.

## LlamaIndex

[LlamaIndex](https://www.llamaindex.ai/) is a framework for building context-augmented and RAG applications.

### Python

LlamaIndex provides four Azure Cosmos DB for NoSQL integrations across its storage abstractions, allowing Cosmos DB to back the full LlamaIndex storage layer:

| Functionality | Class | Package |
|---|---|---|
| Vector store | `AzureCosmosDBNoSqlVectorSearch` | [`llama-index-vector-stores-azurecosmosnosql`](https://pypi.org/project/llama-index-vector-stores-azurecosmosnosql/) |
| Document store | [`AzureCosmosNoSqlDocumentStore`](https://developers.llamaindex.ai/python/framework-api-reference/storage/docstore/azurecosmosnosql/) | `llama-index-storage-docstore-azurecosmosnosql` |
| Index store | [`AzureCosmosNoSqlIndexStore`](https://developers.llamaindex.ai/python/framework-api-reference/storage/index_store/azurecosmosnosql/) | `llama-index-storage-index-store-azurecosmosnosql` |
| Chat store | [`AzureCosmosNoSqlChatStore`](https://developers.llamaindex.ai/python/framework-api-reference/storage/chat_store/azurecosmosnosql/) | `llama-index-storage-chat-store-azurecosmosnosql` |
| Key-value store | [`AzureCosmosNoSqlKVStore`](https://developers.llamaindex.ai/python/framework-api-reference/storage/kvstore/) | `llama-index-storage-kvstore-azurecosmosnosql` |

The chat store, document store, index store, and KV store all support authentication via connection string, account endpoint + key, or EntraID (`DefaultAzureCredential`). See the [LlamaIndex vector store example](https://developers.llamaindex.ai/python/examples/vector_stores/azurecosmosdbnosqldemo/) for an end-to-end RAG walkthrough.

A native Azure Cosmos DB for NoSQL integration isn't currently available in LlamaIndex.TS, LlamaIndex.NET, or LlamaIndex Java.

## Spring AI

[Spring AI](https://spring.io/projects/spring-ai) brings Spring's programming model to AI engineering in Java.

### Java

Spring AI provides a vector store implementation backed by Azure Cosmos DB for NoSQL. See the [Spring AI vector store documentation](https://docs.spring.io/spring-ai/reference/api/vectordbs/azure-cosmos-db.html).


## Related content

- [Azure Cosmos DB Samples Gallery](https://aka.ms/AzureCosmosDB/Gallery)
- [Vector Search with Azure Cosmos DB for NoSQL](vector-search-overview.md)
- [Tokens](tokens.md)
- [Vector Embeddings](vector-embeddings.md)
- [Retrieval Augmented Generated (RAG)](rag.md)
 