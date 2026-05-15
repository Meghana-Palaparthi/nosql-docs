---
title: Data Plane Security Reference
titleSuffix: Azure Cosmos DB for Apache Cassandra
description: Learn about data plane actions and built-in roles for role-based access control in Azure Cosmos DB for Apache Cassandra. See which permissions are available and how to use them.
author: seesharprun
ms.author: sidandrews
ms.reviewer: skhera
ms.service: azure-cosmos-db
ms.subservice: apache-cassandra
ms.topic: reference
ms.date: 04/29/2026
appliesto:
  - ✅ Apache Cassandra
---

# Azure Cosmos DB for Apache Cassandra data plane security reference

Azure Cosmos DB for Apache Cassandra exposes a unique set of data actions and roles within its native role-based access control implementation. This article includes a list of those actions and roles with descriptions on what permissions are granted for each resource.

> [!WARNING]
> Azure Cosmos DB for Cassandra's native role-based access control doesn't support the `notDataActions` property. Any action that isn't specified as an allowed `dataAction` property is excluded automatically.

## Built-in actions

You can set the following data actions individually in a role definition.

| Data action | Description |
| --- | --- |
| `Microsoft.DocumentDB/databaseAccounts/readMetadata` | Reads some account metadata. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/executeQuery` | Runs a query against a table. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/executeStoredProcedure` | Runs a table transaction (procedure). |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/create` | Creates a new entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/read` | Point reads an individual entity (item) by using the row and partition keys. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/replace` | Entirely replaces an existing entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/upsert` | Creates an entity (item) if it doesn't exist or replaces the entity if it already exists. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/delete` | Deletes an entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read` | Reads the current throughput. |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/write` | Modifies the current throughput. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/write` | Creates or updates a table. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/delete` | Deletes a table. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/write` | Creates or updates a container. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/delete` | Deletes a container. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/readChangeFeed` | Reads from the container's change feed. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/manageConflicts` | Manages conflicts for multi-write region accounts (lists and deletes items from the conflict feed). |

### Data action wildcards

The wildcard (`*`) operator is supported at the `tables`, `containers`, and `entities` levels for actions. Use the wildcard to grant broad access to a specific resource type.

| Data action wildcards | Description |
| --- | --- |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/*` | Performs all operations on tables. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/*` | Performs all operations on containers. |
| `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/*` | Performs all operations on entities (items). |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/*` | Performs all operations related to throughput. |

### Required metadata for actions

The Azure Cosmos DB SDKs issue read-only metadata requests during initialization and to serve specific data requests. These requests fetch various configuration details such as:

- The global configuration of your account, which includes the Azure regions in which the account is available.
- The partition key of your containers or their indexing policy.
- The list of physical partitions that make a container and their addresses.
- They don't fetch any of the data that stored in your account.

To ensure the best transparency of our permission model, these metadata requests are explicitly covered by the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action. This action must be allowed in every situation where your Azure Cosmos DB account is accessed through one of the Azure Cosmos DB SDKs.

You can assign the action at any level in an Azure Cosmos DB account's hierarchy, including account, database, or container. The actual metadata requests allowed depend on the scope:

- Account:
  - Lists the databases under the account.
  - Allows the actions at the database scope for each database under the account.
- Cassandra:
  - Reading table metadata.
  - Lists the containers under the table.
  - Allows the actions at the container scope for each container under the table.
- Container:
  - Reads container metadata.
  - Lists physical partitions under the table.
  - Resolves the address of each physical partition.

> [!IMPORTANT]
> You can't manage throughput with the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action.

## Built-in roles

Azure Cosmos DB for Cassandra defines data plane-specific role definitions. These roles are distinct from Azure role-based access control role definitions.

### Cosmos DB Built-in Data Reader

ID: `00000000-0000-0000-0000-000000000003`

- Included actions:
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/read`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/ExecuteQuery`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/ReadChangeFeed`

### Cosmos DB Built-in Data Contributor

ID: `00000000-0000-0000-0000-000000000004`

- Included actions:
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/write`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/*`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/*`
  - `Microsoft.DocumentDB/databaseAccounts/cassandra/containers/entities/*`
