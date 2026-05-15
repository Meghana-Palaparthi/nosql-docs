---
title: Integrations for AI apps
description: Integrate Azure DocumentDB with AI agent and LLM orchestration frameworks like LangChain, LangGraph, LlamaIndex, Haystack, Semantic Kernel, and CosmosAIGraph.
author: khelanmodi
ms.topic: how-to
ms.date: 05/06/2026
ms.author: khelanmodi
ms.collection:
  - ce-skilling-ai-copilot
---

# Azure DocumentDB integrations for AI applications

Azure DocumentDB integrates with the leading large language model (LLM) orchestration frameworks. Use these integrations to store agent state, embed and retrieve private data, and ground generative AI responses against the same MongoDB-compatible cluster that backs your application data — without introducing a separate vector store.

The following integrations have a how-to in this section:

| Framework | What you build with it | Get started |
| --- | --- | --- |
| **[Haystack](https://haystack.deepset.ai/)** | Composable RAG **pipelines** with a custom `cosmosSearch` retriever component plugged into Haystack 2.x. | [Build RAG pipelines](build-rag-pipelines.md) |
| **[LangChain on Azure](https://github.com/langchain-ai/langchain-azure)** | Full **RAG applications** that use the `langchain-azure-ai` vector store, DiskANN/HNSW/IVF indexes, and chains. | [Build RAG applications](build-rag-applications.md) |
| **[LangGraph](https://langchain-ai.github.io/langgraph/)** | **Stateful agents** with checkpoint persistence so threads survive across turns and process restarts. | [Persist agent state](persist-agent-state.md) |
| **[LlamaIndex](https://www.llamaindex.ai/)** | **Knowledge bases** built from your private documents, queryable through `VectorStoreIndex` and `QueryEngine`s. | [Query a knowledge base](query-knowledge-base.md) |
| **[Semantic Kernel](https://github.com/microsoft/semantic-kernel)** | Vector store connector for Semantic Kernel agents and skills (Python and .NET). | [Azure DocumentDB connector](/semantic-kernel/concepts/vector-store-connectors/out-of-the-box-connectors/azure-cosmosdb-mongodb-connector?pivots=programming-language-python) |
| **[CosmosAIGraph](https://aka.ms/cosmosaigraph)** | OmniRAG reference implementation that combines DiskANN vector / hybrid search with an Apache Jena knowledge graph. | [CosmosAIGraph quickstart](https://github.com/AzureCosmosDB/CosmosAIGraph/tree/main/impl) |

## Language support at a glance

| Framework | Python | TypeScript / JavaScript | .NET | Java |
| --- | :---: | :---: | :---: | :---: |
| LangChain | ✓ ([reference](https://docs.langchain.com/oss/python/integrations/vectorstores/azure_cosmos_db_no_sql)) | ✓ ([reference](https://js.langchain.com/docs/integrations/vectorstores/azure_cosmosdb_mongodb/)) | — | ✓ ([reference](https://docs.langchain4j.dev/integrations/embedding-stores/azure-cosmos-mongo-vcore/)) |
| LangChain on Azure | ✓ | — | — | — |
| LangGraph | ✓ | ✓ | — | — |
| LlamaIndex | ✓ | ✓ | — | — |
| Haystack | ✓ | — | — | — |
| Semantic Kernel | ✓ | — | ✓ | — |
| CosmosAIGraph | ✓ | — | — | — |

For frameworks that aren't yet documented in this section, you can still target Azure DocumentDB directly with any MongoDB driver — vector queries use the `$search` aggregation stage with the `cosmosSearch` operator. See [Vector search in Azure DocumentDB](vector-search.md) for the underlying query syntax.

## Related content

- [Vector search in Azure DocumentDB](vector-search.md)
- [Azure DocumentDB samples gallery](https://aka.ms/AzureCosmosDB/Gallery)
- [Try Azure DocumentDB free for 30 days](https://azure.microsoft.com/try/cosmosdb/)

## Next step

> [!div class="nextstepaction"]
> [Get started for free](free-tier.md)
