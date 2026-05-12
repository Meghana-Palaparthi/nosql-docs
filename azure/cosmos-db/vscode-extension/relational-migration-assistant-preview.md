---
title: Use the Cosmos DB Relational Migration Assistant in Visual Studio Code (preview)
description: Learn how to use the Relational Migration Assistant to help migrate relational workloads to Azure Cosmos DB from Visual Studio Code.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/12/2026
ms.custom: preview
---

# Use the Cosmos DB Relational Migration Assistant in Visual Studio Code (preview)

## What this feature does

This preview capability helps evaluate relational source schemas and generate migration guidance for Azure Cosmos DB.

## Prerequisites

- Visual Studio Code with Azure Databases extension preview build.
- Access to source relational schema metadata.
- Target Azure Cosmos DB account for validation and migration planning.

## Run an assessment

1. Open the migration assistant flow in the extension.
1. Connect to the source relational metadata or import schema artifacts.
1. Select target Azure Cosmos DB API and scope.
1. Start the assessment.

## Review recommendations

The assessment output typically includes:

- Candidate container boundaries.
- Suggested partition key options.
- Mapping guidance from relational entities to JSON documents.
- Query pattern considerations and potential hot-partition risks.

## Generate migration artifacts

1. Export generated recommendations and mapping output.
1. Review schema and partitioning with application owners.
1. Validate with a pilot dataset before broader migration.

## Validation guidance

- Run representative read and write queries against pilot data.
- Check RU behavior for high-frequency query paths.
- Confirm denormalization choices align with application access patterns.

## Limitations

- Automated recommendations are a starting point, not a complete migration plan.
- Complex transactional workflows usually require application-level redesign for NoSQL patterns.

## Related articles

- [Migrate relational data to Azure Cosmos DB](../migrate-relational-data.md)
