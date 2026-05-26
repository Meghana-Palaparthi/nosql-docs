---
title: Azure Backup for Cosmos DB
description: Learn about Azure Backup for Cosmos DB, including prerequisites, supported configurations, and how to troubleshoot common errors.
author: mansinahar
ms.service: azure-cosmos-db
ms.topic: concept-article
ms.date: 05/22/2026
ms.author: mansinahar
appliesto:
  - ✅ NoSQL
  - ✅ MongoDB
---

# Azure Backup for Cosmos DB

Azure Backup for Cosmos DB enables you to take backups of your data and store them in an Azure Backup Vault for extended retention periods. It works alongside continuous backup (point-in-time restore) and allows you to protect your data for compliance, auditing, or disaster recovery scenarios that require retention beyond the standard continuous backup window.

## Prerequisites

For more information about the supported regions, scenarios, and limitations, see [Azure Cosmos DB backup support](/azure/backup/backup-azure-cosmos-db-support-matrix).

## Troubleshoot common errors

When you work with Azure Backup for Cosmos DB, the service validates your requests against several prerequisites. If a validation check fails, the service returns an HTTP 409 response with a substatus code that identifies the specific issue.

The following table lists the common errors you might encounter, grouped by which operations they apply to.

**Operations legend**: E = Enablement, D = Disablement, B = Backup, R = Restore

| Substatus code | Applies to | Description | Resolution |
|---|---|---|---|
| `RequestNotSupportedOnOfflineCosmosDBAccountUserError` | E, B, R | The Azure Cosmos DB account isn't in the Online state. The account might be in a transitional state such as deleting or disabled. | Wait for the account to return to the Online state before retrying. |
| `LongTermProtectionNotEnabledUserError` | D, B | Long-term protection (LTP) hasn't been enabled on this account. Backup operations require LTP to be enabled. | Contact Azure Support to enable long-term protection on your account. For disablement, no action is needed because LTP is already off. |
| `CrossRegionExternalBackupNotAllowedUserError` | B | The Azure Cosmos DB account's write region doesn't match the account's ARM location and cross-region backups aren't supported at this time. This issue can occur if the write region was changed after the account was originally created. | Ensure the Azure Cosmos DB write region matches the ARM location before retrying. |
| `CrossRegionLongTermProtectionNotAllowedUserError` | E | The account's ARM location doesn't match its write region. LTP can only be enabled when these match. | Ensure the account's write region matches the ARM location before enabling LTP. |
| `RequestNotSupportedOnEmptyCosmosDBAccountUserError` | B | No databases or collections were found in the account at the specified backup timestamp. | Verify that your account contains at least one database and collection before retrying. |
| `PartitionLimitExceededUserError` | E, B | Backups are only supported for accounts with fewer than 2,500 partitions. Your account exceeds this limit. | Contact Azure Support for assistance. |
| `IncorrectApiTypeForCosmosDBAccountUserError` | E, R | For enablement: LTP is only supported on NoSQL and MongoDB API accounts. For restore: the target account's API type doesn't match the backup source. | For enablement, use a NoSQL or MongoDB account. For restore, ensure the target account uses the same API type as the source. |
| `RequestNotSupportedOnPeriodicBackupModeCosmosDBAccountUserError` | E | The account uses periodic backup mode. LTP requires continuous backup (PITR) to be enabled. | Migrate the account to continuous backup mode (PITR) before enabling LTP. |
| `RequestNotSupportedOnPPAFEnabledCosmosDBAccountUserError` | E | The account has per-partition automatic failover (PPAF) enabled, which isn't compatible with LTP. | Disable per-partition automatic failover before enabling LTP. |
| `RequestNotSupportedOnMultiRegionCosmosDBAccountUserError` | R | The target account has multiple regions. Restore is only supported on single-region accounts. | Remove additional regions from the target account before retrying. |
| `RestoreNotSupportedOnServerlessCosmosDBAccountUserError` | R | The target account is a serverless account. Restore isn't supported on serverless accounts. | Use a provisioned throughput account as the restore target. |
| `RequestNotSupportedOnNonEmptyCosmosDBAccountUserError` | R | The target account already contains databases. Restore requires an empty account. | Delete all databases from the target account before retrying, or create a new account with no data in it. |
| `AnotherOperationInProgressOnCosmosDBAccountUserError` | R | Another management operation is already in progress on this account. | Wait for the current operation to complete before retrying the restore. |

> [!TIP]
> If you continue to experience issues after following the recommended resolution steps, contact Azure Support with the full error response, including the substatus code and the `x-ms-request-id` header.

## Next steps

- [Back up Azure Cosmos DB using Azure Backup](/azure/backup/backup-azure-cosmos-db)
- [Continuous backup with point-in-time restore in Azure Cosmos DB](continuous-backup-restore-introduction.md)
- [Enable continuous backup](provision-account-continuous-backup.md)
