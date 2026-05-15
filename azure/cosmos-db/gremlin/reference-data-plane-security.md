---
title: Data Plane Security Reference
titleSuffix: Azure Cosmos DB for Apache Gremlin
description: Learn about data plane actions and built-in roles for role-based access control in Azure Cosmos DB for Apache Gremlin. See which permissions are available and how to use them.
author: seesharprun
ms.author: sidandrews
ms.reviewer: skhera
ms.service: azure-cosmos-db
ms.subservice: apache-gremlin
ms.topic: reference
ms.date: 04/29/2026
appliesto:
  - ✅ Apache Gremlin
---

# Azure Cosmos DB for Apache Gremlin data plane security reference

Azure Cosmos DB for Apache Gremlin exposes a unique set of data actions and roles within its native role-based access control implementation. This article includes a list of those actions and roles with descriptions on what permissions are granted for each resource.

> [!WARNING]
> The Azure Cosmos DB for Gremlin native role-based access control doesn't support the `notDataActions` property. Any action that isn't specified as an allowed `dataAction` is excluded automatically.

## Built-in actions

You can set the following data actions individually in a role definition.

|Data action | Description |
| --- | --- |
| `Microsoft.DocumentDB/databaseAccounts/readMetadata` | Reads some account metadata. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/executeQuery` | Runs a query against a table. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/executeStoredProcedure` | Runs a table transaction (procedure). |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/create` | Creates a new entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/read` | Point reads an individual entity (item) by using the row and partition keys. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/replace` | Entirely replaces an existing entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/upsert` | Creates an entity (item) if it doesn't exist or replaces the entity if it already exists. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/delete` | Deletes an entity (item). |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read` | Reads the current throughput. |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/write` | Modifies the current throughput. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/write` | Creates or updates a table. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/delete` | Deletes a table. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/write` | Creates or updates a container. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/delete` | Deletes a container. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/readChangeFeed` | Reads from the container's change feed. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/manageConflicts` | Manages conflicts for multi-write region accounts (list and delete items from the conflict feed). |

### Data action wildcards

The wildcard (`*`) operator is supported at the `tables`, `containers`, and `entities` levels for actions. Use the wildcard to grant broad access to a specific resource type.

|Data action wildcards | Description |
| --- | --- |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/*` | Performs all operations on tables. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/*` | Performs all operations on containers. |
| `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/*` | Performs all operations on entities (items). |
| `Microsoft.DocumentDB/databaseAccounts/throughputSettings/*` | Performs all operations related to throughput. |

### Required metadata for actions

The Azure Cosmos DB SDKs issue read-only metadata requests during initialization and to serve specific data requests. These requests fetch various configuration details, such as:

- The global configuration of your account, which includes the Azure regions the account is available in
- The partition key of your containers or their indexing policy
- The list of physical partitions that make a container and their addresses
- They don't fetch any of the data that stored in your account

To ensure the best transparency of our permission model, these metadata requests are explicitly covered by the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action. This action must be allowed in every situation where your Azure Cosmos DB account is accessed through one of the Azure Cosmos DB SDKs.

You can assign the action at any level in an Azure Cosmos DB account's hierarchy, including account, database, or container. The actual metadata requests allowed depend on the scope:

- Account:
  - Lists the databases under the account.
  - Allows the actions at the database scope for each database under the account.
- Gremlin:
  - Reads table metadata.
  - Lists the containers under the table.
  - Allows the actions at the container scope for each container under the table,
- Container:
  - Reads container metadata.
  - Lists physical partitions under the table.
  - Resolves the address of each physical partition.

> [!IMPORTANT]
> You can't manage throughput with the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action.

## Built-in roles

Azure Cosmos DB for Gremlin defines data plane-specific role definitions. The roles are distinct from Azure role-based access control role definitions.

### Cosmos DB Built-in Data Reader

ID: `00000000-0000-0000-0000-000000000003`

- Included actions:
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/read`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/ExecuteQuery`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/ReadChangeFeed`

### Cosmos DB Built-in Data Contributor

ID: `00000000-0000-0000-0000-000000000004`

- Included actions:
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/write`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/*`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/*`
  - `Microsoft.DocumentDB/databaseAccounts/gremlin/containers/entities/*`
