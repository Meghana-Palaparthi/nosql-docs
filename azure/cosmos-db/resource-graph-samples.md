---
title: Azure Resource Graph Sample Queries
description: Sample Azure Resource Graph queries for Azure Cosmos DB showing use of resource types and tables to access Azure Cosmos DB related resources and properties.
ms.date: 05/13/2026
ms.topic: sample
author: manishmsfte
ms.author: mansha
ms.service: azure-cosmos-db
ms.custom:
  - subject-resourcegraph-sample
ai-usage: ai-assisted
appliesto:
  - ✅ NoSQL
  - ✅ MongoDB
  - ✅ Apache Cassandra
  - ✅ Apache Gremlin
  - ✅ Table
---

# Azure Resource Graph sample queries for Azure Cosmos DB

This page is a collection of [Azure Resource Graph](/azure/governance/resource-graph/overview) sample queries that you can use to inventory and evaluate Azure Cosmos DB accounts across subscriptions at scale.

## Sample queries

### Count Azure Cosmos DB accounts

```kusto
Resources
| where type =~ 'microsoft.documentdb/databaseaccounts'
| summarize count()
```

This query returns the total number of Azure Cosmos DB accounts across all subscriptions you have access to.

### List Azure Cosmos DB accounts modified in the last 30 days

```kusto
Resources
| where type =~ 'microsoft.documentdb/databaseaccounts'
| extend lastModifiedAt = todatetime(systemData.lastModifiedAt)
| where lastModifiedAt >= ago(30d)
| project subscriptionId, resourceGroup, name, lastModifiedAt
| order by lastModifiedAt desc
```

This query returns Azure Cosmos DB accounts updated in the last 30 days, sorted by the most recent modification time.

### Identify non-compliant Azure Cosmos DB accounts

```kusto
PolicyResources
| where type =~ 'microsoft.policyinsights/policystates'
| extend resourceType = tostring(properties.resourceType), complianceState = tostring(properties.complianceState)
| where resourceType =~ 'microsoft.documentdb/databaseaccounts' and complianceState =~ 'NonCompliant'
| project subscriptionId, resourceId = tostring(properties.resourceId)
| summarize nonCompliantPolicies = count() by subscriptionId, resourceId
| order by nonCompliantPolicies desc
```

This query returns Azure Cosmos DB accounts with a count of non-compliant policies for each account, grouped by subscription and resource ID.

### List Azure Cosmos DB accounts with specific write locations

> [!TIP]
> Replace `'East US'` and `'West US'` with your desired Azure region names.

```kusto
Resources
| where type =~ 'microsoft.documentdb/databaseaccounts'
| project id, name, writeLocations = (properties.writeLocations)
| mv-expand writeLocations
| project id, name, writeLocation = tostring(writeLocations.locationName)
| where writeLocation in ('East US', 'West US')
| distinct id, name
```

## Next steps

- Learn more about the [query language](/azure/governance/resource-graph/concepts/query-language).
- Learn more about how to [explore resources](/azure/governance/resource-graph/concepts/explore-resources).
