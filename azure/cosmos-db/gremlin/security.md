---
title: Secure Your Account
description: Review the fundamentals of securing Azure Cosmos DB for Apache Gremlin from the perspective of data and networking security.
ms.topic: best-practice
ms.date: 04/29/2026
ms.custom:
  - security-horizontal-2026
  - horz-security
ai-usage: ai-generated
---

# Secure your Azure Cosmos DB for Apache Gremlin account

[!INCLUDE[Note - Recommended services](includes/note-recommended-services.md)]

Azure Cosmos DB for Apache Gremlin is a fully managed graph database service that you can use to store, query, and traverse large-scale graph data by using the Gremlin query language.

This article provides guidance on how to best secure your Azure Cosmos DB for Apache Gremlin deployment.

## Network security

- **Disable public network access and use private endpoints only:** Deploy Azure Cosmos DB for NoSQL with a configuration that restricts network access to an Azure-deployed virtual network. The account is exposed through the specific subnet that you configured. Then, disable public network access for the entire account and use private endpoints exclusively for services that connect to the account. For more information, see [Configure virtual network access](../how-to-configure-vnet-service-endpoint.md) and [Configure access from private endpoints](../how-to-configure-private-endpoints.md).
- **Enable network security perimeter (NSP) for network isolation:** Use NSP to restrict access to your Azure Cosmos DB account by defining network boundaries and isolating it from public internet access. For more information, see [Configure network security perimeter](../how-to-configure-nsp.md).

## Identity management

- **Use managed identities to access your account from other Azure services:** Eliminate the need to manage credentials by using an automatically managed identity in Microsoft Entra ID. Use managed identities to securely access Azure Cosmos DB from other Azure services without embedding credentials in your code. For more information, see [Managed identities for Azure resources](/entra/identity/managed-identities-azure-resources/overview).
- **Separate the Azure identities used for data and control plane access:** Use distinct Azure identities for data plane and control plane operations to reduce the risk of privilege escalation and ensure better access control. This separation enhances security by limiting the scope of each identity.

## Transport security

- **Use and enforce Transport Layer Security (TLS) 1.3 for transport security:** Enforce TLS 1.3 to secure data in transit with the latest cryptographic protocols to ensure stronger encryption and improved performance. For more information, see [Minimum TLS enforcement](../self-serve-minimum-tls-enforcement.md).

## Data encryption

- **Encrypt data at rest or in motion by using service-managed keys or customer-managed keys (CMKs):** Protect sensitive data by encrypting it at rest and in transit. Use service-managed keys for simplicity or CMKs for greater control over encryption. For more information, see [Configure customer-managed keys](../how-to-setup-customer-managed-keys.md).

- **Use Always Encrypted to secure data with client-side encryption:** Ensure that sensitive data is encrypted on the client side before the data is sent to Azure Cosmos DB. Use Always Encrypted for an extra layer of security. For more information, see [Always Encrypted](../how-to-always-encrypted.md).

## Backup and restore

- **Enable native continuous backup and restore:** Protect your data by enabling continuous backup so that you can restore your Azure Cosmos DB account to any point in time within the retention period. For more information, see [Continuous backup and restore](../online-backup-and-restore.md).
- **Test backup and recovery procedures:** Verify the effectiveness of backup processes by regularly testing the restoration of databases, containers, and items. For more information, see [Restore a container or database](../how-to-restore-in-account-continuous-backup.md).
