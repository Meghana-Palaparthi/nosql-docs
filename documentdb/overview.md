---
title: DocumentDB Overview
description: Learn how DocumentDB in Azure provides an open-source and MongoDB-compatible platform with advanced indexing, vector search, and geospatial capabilities built on PostgreSQL.
ms.date: 11/07/2025
ms.update-cycle: 180-days
ai-usage: ai-generated
---

# What is DocumentDB (in Azure)

DocumentDB in Azure is a fully permissive and open-source platform for document data stores built on the PostgreSQL engine. Use DocumentDB to handle Binary JSON (BSON) documents, advanced indexing, vector search queries, and geospatial operations with the flexibility of the MIT license. This overview explains the core capabilities, design goals, and how DocumentDB helps you build MongoDB-compatible applications in Azure.

## Core design goals and capabilities

The platform handles BSON document parsing and manipulation at all levels of nesting, enabling flexible document storage and iteration. The platform provides advanced indexing capabilities including single field, multi-key, compound, text, and geospatial indexes. Vector search queries are powered by the pg_vector PostgreSQL extension, enabling AI and machine learning applications.

The platform uses SCRAM (Salted Challenge Response Authentication Mechanism) for authentication and uses the PostGIS extension for geospatial queries. Full Decimal128 support is powered by Intel Floating Point Math Library, and regex support utilizes the PCRE2 Project. The architecture consists of two primary components: pg_documentdb_core (a custom PostgreSQL extension optimizing BSON datatype support) and pg_documentdb_api (the data plane implementing CRUD operations, query functionality, and index management).

## Common operational concerns

Choose appropriate indexing strategies to balance query performance with write throughput, applying the platform's support for single field, compound, text, and geospatial indexes. For vector search workloads, configure pg_vector appropriately to optimize similarity search queries. Consider the tradeoffs between document nesting depth and query complexity when designing your document schemas.

## Scenarios

The DocumentDB platform is designed for MongoDB-compatible applications requiring advanced features such as vector search for AI workloads, geospatial queries for location-based services, and full-text search capabilities. The platform supports operational databases, document-oriented applications, and AI/ML feature stores requiring BSON document handling with PostgreSQL's reliability and extensibility.

## Implementations

The DocumentDB open-source platform is implemented in services that use its MongoDB-compatible capabilities while providing managed infrastructure and Azure integration.

### Azure DocumentDB

Azure DocumentDB is a fully managed MongoDB-compatible database service built on the DocumentDB open-source platform. It provides developers with a familiar vCore architecture for building modern applications with native Azure integrations and low total cost of ownership. The service offers an integrated vector database for generative AI applications, enabling efficient indexing and querying without external integrations. Azure DocumentDB features deep integration with Azure products such as Azure Monitor and Azure CLI, provides flexible vertical and horizontal scaling with optional high availability, and supports automatic sharding with no downtime. The service uses the permissive MIT-licensed DocumentDB platform, ensuring developers have complete freedom to use, modify, and distribute their applications without commercial licensing restrictions.

For more information about Azure DocumentDB, see [Azure DocumentDB documentation](/azure/documentdb).
