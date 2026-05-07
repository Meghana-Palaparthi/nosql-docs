---
title: Data plane security reference
titleSuffix: Azure Cosmos DB for MongoDB
description: Learn about data plane actions and built-in roles for role-based access control in Azure Cosmos DB for MongoDB. See which permissions are available and how to use them.
author: seesharprun
ms.author: sidandrews
ms.reviewer: skhera
ms.service: azure-cosmos-db
ms.subservice: mongodb
ms.topic: reference
ms.date: 04/29/2026
appliesto:
  - ✅ MongoDB
---

# Azure Cosmos DB for MongoDB data plane security reference

Azure Cosmos DB for MongoDB exposes a unique set of data actions and roles within its native role-based access control implementation. This article includes a list of those actions and roles with descriptions on what permissions are granted for each resource.

> [!WARNING]
> Azure Cosmos DB for MongoDB's native role-based access control doesn't support the `notDataActions` property. Any action that isn't specified as an allowed `dataAction` is excluded automatically.

## Built-in actions

Here's a list of data actions that can be individually set in a role definition.

| | Description |
| --- | --- |
| **`Microsoft.DocumentDB/databaseAccounts/readMetadata`** | Read some account metadata |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/executeQuery`** | Executes a query against a table |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/executeStoredProcedure`** | Executes a table transaction (procedure) |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/create`** | Creates a new entity (item) |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/read`** | Point reads an individual entity (item) using the row and partition keys |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/replace`** | Entirely replaces an existing entity (item) |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/upsert`** | Creates an entity (item) if it doesn't exist or replaces the entity if it already exists |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/delete`** | Deletes an entity (item) |
| **`Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`** | Read the current throughput |
| **`Microsoft.DocumentDB/databaseAccounts/throughputSettings/write`** | Modify the current throughput |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/write`** | Create or update a table |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/delete`** | Delete a table |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/write`** | Create or update a container |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/delete`** | Delete a container |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/readChangeFeed`** | Read from the container's change feed |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/manageConflicts`** | Manage conflicts for multi-write region accounts (list and delete items from the conflict feed) |

### Data action wildcards

The wildcard (`*`) operator is supported at the `tables`, `containers`, and `entities` levels for actions. Use the wildcard to grant broad access to a specific resource type.

| | Description |
| --- | --- |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/*`** | Perform all operations on tables |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/*`** | Perform all operations on containers |
| **`Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/*`** | Perform all operations on entities (items) |
| **`Microsoft.DocumentDB/databaseAccounts/throughputSettings/*`** | Perform all operations related to throughput |

### Required metadata for actions

The Azure Cosmos DB software development kits (SDKs) issue read-only metadata requests during initialization and to serve specific data requests. These requests fetch various configuration details such as:

- The global configuration of your account, which includes the Azure regions the account is available in
- The partition key of your containers or their indexing policy
- The list of physical partitions that make a container and their addresses
- They don't fetch any of the data that stored in your account

To ensure the best transparency of our permission model, these metadata requests are explicitly covered by the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action. This action must be allowed in every situation where your Azure Cosmos DB account is accessed through one of the Azure Cosmos DB SDKs.

The action can be assigned at any level in an Azure Cosmos DB account's hierarchy including account, database, or container. The actual metadata requests allowed depend on the scope:

- **Account**
  - Listing the databases under the account
  - For each database under the account, the allowed actions at the database scope
- **MongoDB**
  - Reading table metadata
  - Listing the containers under the table
  - For each container under the table, the allowed actions at the container scope
- **Container**
  - Reading container metadata
  - Listing physical partitions under the table
  - Resolving the address of each physical partition

> [!IMPORTANT]
> You can't manage throughput with the `Microsoft.DocumentDB/databaseAccounts/readMetadata` data action.

## Built-in roles

Azure Cosmos DB for MongoDB defines data plane-specific role definitions. These roles are distinct from Azure role-based access control role definitions.

### Cosmos DB Built-in Data Reader

**ID**: `00000000-0000-0000-0000-000000000003`

- **Included actions**
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/read`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/ExecuteQuery`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/ReadChangeFeed`

### Cosmos DB Built-in Data Contributor

**ID**: `00000000-0000-0000-0000-000000000004`

- **Included actions**
  - `Microsoft.DocumentDB/databaseAccounts/readMetadata`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/read`
  - `Microsoft.DocumentDB/databaseAccounts/throughputSettings/write`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/*`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/*`
  - `Microsoft.DocumentDB/databaseAccounts/mongoMI/containers/entities/*`
