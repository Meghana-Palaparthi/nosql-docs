---
title: Agent Memory Toolkit (Preview)
titleSuffix: Azure Cosmos DB for NoSQL
description: Learn what Agent Memory Toolkit is, what memory types and operations it supports, what Azure services it uses, and where to get it.
author: jcodella
ms.author: jacodel
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: feature-guide
ms.date: 05/20/2026
ms.update-cycle: 180-days
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
ai-usage: ai-assisted 
---

# Agent Memory Toolkit for Azure Cosmos DB (preview)

[!INCLUDE[Preview](../includes/notice-preview.md)]

Agent Memory Toolkit is a Python SDK that helps you add memory to AI agents that use Azure Cosmos DB. It stores raw conversation history and creates higher-value derived memories, such as summaries, facts, and user profiles, so your agent can recall useful context across messages, threads, and sessions.

:::image type="content" source="media/agent-memory-toolkit/overview.png" lightbox="media/agent-memory-toolkit/overview.png" alt-text="Diagram showing an overview of Agent Memory Toolkit with Azure Cosmos DB.":::

## What it supports

Agent Memory Toolkit supports short-term and long-term memory patterns. It can store individual turns during an active conversation and transform those turns into durable artifacts that are easier for an agent to retrieve later.

| Memory type | Description | Common use |
| --- | --- | --- |
| `turn` | Raw conversation records for user, agent, tool, or system messages. | Replay recent conversation history or preserve short-term context. |
| `summary` | A compact summary of one conversation thread. | Recall the main topic, decisions, open issues, and next steps from a long thread. |
| `fact` | A discrete assertion extracted from a thread, such as a preference, requirement, or confirmed decision. | Retrieve fine-grained knowledge across threads by semantic meaning. |
| `user_summary` | A cross-thread profile for one user. | Preserve stable user context, preferences, environment details, and constraints across sessions. |

## Operations

The toolkit supports local development, durable storage, retrieval, semantic search, and memory transformation.

You can run memory processing in two ways. For robust production pipelines, use Azure Durable Functions with the Azure Cosmos DB change feed to process new turns automatically in the background. For lightweight demos, proofs of concept (PoCs), and dev/test workflows, orchestrate memory operations manually from the toolkit in your application code.

| Operation area | Supported capabilities |
| --- | --- |
| Add memories | Add memories to local in-memory storage or directly to Azure Cosmos DB. |
| Upload memories | Push locally collected memories to Azure Cosmos DB. |
| Retrieve memories | Retrieve memories by user, thread, or recent history. |
| Search memories | Search stored memories semantically by using embeddings, and use hybrid search that combines vector and full-text ranking. |
| Generate summaries | Create or update thread summaries from stored turns. |
| Extract facts | Extract discrete facts from conversation history and store them as searchable memory documents. |
| Generate user summaries | Build cross-thread user profiles from memories across a user's conversations. |
| Reconcile facts | Detect duplicate or contradictory facts and preserve an audit trail with soft-deleted superseded records. |

## Azure services

Agent Memory Toolkit uses Azure Cosmos DB as the durable memory store. Azure Cosmos DB stores turns, summaries, facts, and user summaries as JSON documents, and supports query, vector search, full-text search, and hybrid search over those memories.

The toolkit can also use other Azure services depending on your deployment model. Use Azure Durable Functions and the Azure Cosmos DB change feed when you need a robust processing pipeline that reacts to new memories automatically. Use the toolkit directly from your app when you want manual control with less infrastructure.

| Azure service | How the toolkit uses it |
| --- | --- |
| Azure Cosmos DB | Stores memory documents and supports queries, vector search, full-text search, hybrid search, and change feed processing. |
| Azure Cosmos DB change feed | Triggers automatic memory processing when new conversation turns are written. |
| Azure Durable Functions | Runs the memory processing pipeline in a sibling function app for background summaries, fact extraction, and user summaries. |
| Microsoft Foundry | Provides embedding models for semantic search and chat or language models for memory transformation. |

For development and quick testing, the toolkit can also use local in-memory storage. Use Azure Cosmos DB when you need persistence, shared access, semantic search, or the processing pipeline.

## Get the toolkit

Download the toolkit, review setup steps, and get current installation guidance from the Agent Memory Toolkit (<https://aka.ms/agentmemorytoolkit>).

## Related content

- [Agent memories in Azure Cosmos DB for NoSQL](agentic-memories.md)
- [Vector search in Azure Cosmos DB for NoSQL](../vector-search.md)
- [Full-text search in Azure Cosmos DB for NoSQL](full-text-search.md)
- [Hybrid search in Azure Cosmos DB for NoSQL](hybrid-search.md)
- [Semantic Reranker in Azure Cosmos DB for NoSQL](semantic-reranker.md)