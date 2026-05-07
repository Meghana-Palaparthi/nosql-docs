---
title: Connect using role-based access control and Microsoft Entra ID
titleSuffix: Azure Cosmos DB for Apache Cassandra
description: Learn how to set up role-based access control for Azure Cosmos DB for Apache Cassandra accounts and data. Enhance security for your applications with step-by-step guidance.
author: seesharprun
ms.author: sidandrews
ms.reviewer: skhera
ms.service: azure-cosmos-db
ms.subservice: apache-cassandra
ms.topic: how-to
ms.devlang: python
ms.date: 04/29/2026
ms.custom:
  - sfi-image-nochange
  - sfi-ropc-nochange
zone_pivot_groups: azure-interface-portal-cli-powershell-bicep
defaultDevLang: python
dev_langs:
  - python
  - csharp
  - java
ai-usage: ai-assisted
appliesto:
  - ✅ Apache Cassandra
#Customer Intent: As a developer, I want to connect to Azure Cosmos DB for Apache Cassandra using role-based access control, so that I can securely manage access to my database resources.
---

# Connect to Azure Cosmos DB for Apache Cassandra using role-based access control and Microsoft Entra ID

Role-based access control refers to a method to manage access to resources in Azure. This method is based on specific identities being assigned roles that manage what level of access they have to one or more resources. Role-based access control provides a flexible system of fine-grained access management that ensures identities only have the least privileged level of access they need to perform their task.

For more information, see [role-based access control](/azure/role-based-access-control/overview).

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).

- An existing Azure Cosmos DB for Cassandra account.

- One or more existing identities in Microsoft Entra ID.

::: zone pivot="azure-cli,azure-resource-manager-bicep"

[!INCLUDE [Azure CLI prerequisites](~/reusable-content/azure-cli/azure-cli-prepare-your-environment-no-header.md)]

::: zone-end

::: zone pivot="azure-portal"

::: zone-end

::: zone pivot="azure-powershell"

[!INCLUDE [Azure PowerShell prerequisites](~/reusable-content/azure-powershell/azure-powershell-requirements-no-header.md)]

::: zone-end

## Disable key-based authentication

[!INCLUDE[Disable key-based authentication](../includes/disable-key-based-authentication.md)]

## Validate that key-based authentication is disabled

To validate that key-based access is disabled, attempt to use the Azure SDK to connect to Azure Cosmos DB for Cassandra using a resource-owner password credential (ROPC). This attempt should fail. If necessary, code samples for common programming languages are provided here.

```csharp
using Azure.Data.Tables;
using Azure.Core;

string connectionString = "AccountEndpoint=<table-endpoint>;AccountKey=<key>;";

TableServiceClient client = new(connectionString);
```

```python
from azure.data.tables import TableServiceClient

connection_string = "AccountEndpoint=<table-endpoint>;AccountKey=<key>;"

client = TableServiceClient(endpoint, connection_string)
```

```java
import com.azure.data.tables.TableServiceClient;
import com.azure.data.tables.TableServiceClientBuilder;

public class Cassandra{
    public static void main(String[] args){
        TableServiceClient tableServiceClient = new TableServiceClientBuilder()
            .connectionString("AccountEndpoint=<cassandra-endpoint>;AccountKey=<key>;")
            .buildClient();
    }
}
```

## Grant control plane role-based access

[!INCLUDE[Grant control plane role-based access](../includes/grant-control-plane-role-based-access.md)]

## Validate control plane role-based access in code

[!INCLUDE[Validate control plane role-based access](../includes/validate-control-plane-role-based-access.md)]

## Grant data plane role-based access

Data plane access refers to the ability to read and write data within an Azure service without the ability to manage resources in the account. For example, Azure Cosmos DB data plane access could include the ability to:

- Read some account and resource metadata
- Create, read, update, patch, and delete items
- Execute Cassandra queries
- Read from a container's change feed
- Execute stored procedures
- Manage conflicts in the conflict feed

First, you must prepare a role definition with a list of `dataActions` to grant access to read, query, and manage data in Azure Cosmos DB for Cassandra. In this guide, you prepare a built-in and custom role. Then, assign the newly defined role\[s\] to an identity so that your applications can access data in Azure Cosmos DB for Cassandra.

::: zone pivot="azure-cli"

1. List all of the role definitions associated with your Azure Cosmos DB for Cassandra account using `az cosmosdb cassandra role definition list`.

    ```azurecli-interactive
    az cosmosdb cassandra role definition list \
      --account-name <account-name> \
      --resource-group <resource-group>
    ```

1. Review the output and locate the role definition named **Cosmos DB Built-in Data Contributor**. The output contains the unique identifier of the role definition in the `id` property. Record this value as it is required to use in the assignment step later in this guide.

    > [!NOTE]
    > In this example, the `id` value would be `/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-cassandra/cassandraRoleDefinitions/00000000-0000-0000-0000-000000000004`. This example uses fictitious data and your identifier would be distinct from this example.

1. Create a new JSON file named *role-definition.json*. In this file, create a resource definition specifying the data actions you want to allow.

1. Next, use `az cosmosdb cassandra role definition create` to create the role definition. Use the *role-definition.json* as the input for the `--body` argument.

    ```azurecli-interactive
    az cosmosdb cassandra role definition create \
      --account-name <account-name> \
      --resource-group <resource-group> \
      --body @role-definition.json
    ```

1. Now, list all of the role definitions again and record the `id` of your new custom role.

1. Assign the new role using `az cosmosdb cassandra role assignment create`. Use the previously recorded role definition identifier, the unique identifier for your identity, and the scope (account, keyspace, or table).

    ```azurecli-interactive
    az cosmosdb cassandra role assignment create \
      --account-name <account-name> \
      --resource-group <resource-group> \
      --role-definition-id <role-definition-id> \
      --principal-id <principal-id> \
      --scope /
    ```

1. Use `az cosmosdb cassandra role assignment list` to list all role assignments for your account and verify your assignment.

    ```azurecli-interactive
    az cosmosdb cassandra role assignment list \
      --account-name <account-name> \
      --resource-group <resource-group>
    ```

::: zone-end

::: zone pivot="azure-resource-manager-bicep"

1. First, get the resource identifier of the existing Azure Cosmos DB for Cassandra account using `az cosmsodb show` and store it in a variable.

    ```azurecli-interactive
    resourceId=$( \
        az cosmosdb show \
            --resource-group "<name-of-existing-resource-group>" \
            --name "<name-of-existing-table-account>" \
            --query "id" \
            --output tsv \
    )
    
    az rest \
        --method "GET" \
        --url $resourceId/cassandraRoleDefinitions?api-version=2023-04-15
    ```

1. Then, list all of the role definitions associated with your Azure Cosmos DB for Cassandra account using `az rest`. Finally, review the output and locate the role definition named **Cosmos DB Built-in Data Contributor**. The output contains the unique identifier of the role definition in the `id` property. Record this value as it is required to use in the assignment step later in this guide.
    
    ```json
    [
      ...,
      {
        "id": "/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-cassandra/cassandraRoleDefinitions/00000000-0000-0000-0000-000000000004",
        "name": "00000000-0000-0000-0000-000000000004",
        "properties": {
          "assignableScopes": [
            "/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-cassandra"
          ],
          "permissions": [
            {
              "dataActions": [
                "Microsoft.DocumentDB/databaseAccounts/readMetadata",
                "Microsoft.DocumentDB/databaseAccounts/tables/*",
                "Microsoft.DocumentDB/databaseAccounts/tables/containers/entities/*"
              ],
              "notDataActions": []
            }
          ],
          "roleName": "Cosmos DB Built-in Data Contributor",
          "type": "BuiltInRole"
        },
        "type": "Microsoft.DocumentDB/databaseAccounts/cassandraRoleDefinitions"
      }
      ...
    ]
    ```

    > [!NOTE]
    > In this example, the `id` value would be `/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-cassandra/cassandraRoleDefinitions/00000000-0000-0000-0000-000000000004`. This example uses fictitious data and your identifier would be distinct from this example. This example output is truncated.

1. Create a new Bicep file to define your role definition. Name the file *data-plane-role-definition.bicep*. Add these `dataActions` to the definition:

    | | Description |
    | --- | --- |
    | **`Microsoft.DocumentDB/databaseAccounts/readMetadata`** | |
    | **`Microsoft.DocumentDB/databaseAccounts/tables/*`** | |
    | **`Microsoft.DocumentDB/databaseAccounts/tables/containers/entities/*`** | |

    ```bicep
    metadata description = 'Create RBAC definition for data plane access to Azure Cosmos DB for Cassandra.'
    
    @description('Name of the Azure Cosmos DB for Cassandra account.')
    param accountName string
    
    @description('Name of the role definition.')
    param roleDefinitionName string = 'API for Cassandra Data Plane Owner'
        
    resource account 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' existing = {
      name: accountName
    }
    
    resource definition 'Microsoft.DocumentDB/databaseAccounts/cassandraRoleDefinitions@2023-04-15' = {
      name: guid('table-role-definition', account.id)
      parent: account
      properties: {
        roleName: roleDefinitionName
        type: 'CustomRole'
        assignableScopes: [
          account.id
        ]
        permissions: [
          {
            dataActions: [
              'Microsoft.DocumentDB/databaseAccounts/readMetadata'
              'Microsoft.DocumentDB/databaseAccounts/tables/*'
              'Microsoft.DocumentDB/databaseAccounts/tables/containers/entities/*'
            ]
          }
        ]
      }
    }
    
    output definitionId string = definition.id
    ```

    > [!TIP]
    > In Azure Cosmos DB's native implementation of role-based access control, **scope** refers to the granularity of resources within an account for which you want permission applied. At the highest level, you can scope a data plane role-based access control assignment to the entire account using the largest scope. This scope includes all databases and containers within the account:
    >
    > ```output
    > /subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.DocumentDB/databaseAccounts/<account-name>/
    > ```
    >
    > Or, you can scope your data plane role assignment to a specific database:
    >
    > ```output
    > /subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.DocumentDB/databaseAccounts/<account-name>/dbs/<database-name>
    > ```
    >
    > Finally, you can scope the assignment to a single container, the most granular scope:
    >
    > ```output
    > /subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.DocumentDB/databaseAccounts/<account-name>/dbs/<database-name>/colls/<container-name>
    > ```
    >
    > In many cases, you can use the relative scope instead of the fully qualified scope. For example, you can use this relative scope to grant data plane role-based access control permissions to a specific database and container from an Azure CLI command:
    >
    > ```output
    > /dbs/<database-name>/colls/<container-name>
    > ```
    >
    > You can also grant universal access to all databases and containers using the relative scope:
    >
    > ```output
    > /
    > ```
    >

1. Create a new Bicep parameters file named *`data-plane-role-definition.bicepparam`*. In this parameters file, assign the name of your existing Azure Cosmos DB for Cassandra account to the `accountName` parameter.

    ```bicep
    using './data-plane-role-definition.bicep'
    
    param accountName = '<name-of-existing-table-account>'
    ```

1. Deploy the Bicep template using `az deployment group create`. Specify the name of the Bicep template, parameters file, and Azure resource group.

    ```azurecli-interactive
    az deployment group create \
        --resource-group "<name-of-existing-resource-group>" \
        --parameters data-plane-role-definition.bicepparam \
        --template-file data-plane-role-definition.bicep
    ```

1. Review the output from the deployment. The output contains the unique identifier of the role definition in the `properties.outputs.definitionId.value` property. Record this value as it is required to use in the assignment step later in this guide.

    ```json
    {
      "properties": {
        "outputs": {
          "definitionId": {
            "type": "String",
            "value": "/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourcegroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-table-account/cassandraRoleDefinitions/dddddddd-9999-0000-1111-eeeeeeeeeeee"
          }
        }
      }
    }
    ```

    > [!NOTE]
    > In this example, the `id` value would be `/subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourcegroups/msdocs-identity-example/providers/Microsoft.DocumentDB/databaseAccounts/msdocs-identity-example-table-account/cassandraRoleDefinitions/dddddddd-9999-0000-1111-eeeeeeeeeeee`. This example uses fictitious data and your identifier would be distinct from this example. This example is a subset of the typical JSON outputted from the deployment for clarity.

1. Create another Bicep file to assign a role to an identity. Name this file *data-plane-role-assignment.bicep*.

    ```bicep
    metadata description = 'Assign RBAC role for data plane access to Azure Cosmos DB for Cassandra.'
    
    @description('Name of the Azure Cosmos DB for Cassandra account.')
    param accountName string
    
    @description('Id of the role definition to assign to the targeted principal in the context of the account.')
    param roleDefinitionId string
    
    @description('Id of the identity/principal to assign this role in the context of the account.')
    param identityId string
    
    resource account 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' existing = {
      name: accountName
    }
    
    resource assignment 'Microsoft.DocumentDB/databaseAccounts/tableRoleAssignments@2023-04-15' = {
      name: guid(roleDefinitionId, identityId, account.id)
      parent: account
      properties: {
        principalId: identityId
        roleDefinitionId: roleDefinitionId
        scope: account.id
      }
    }
    
    output id string = assignment.id
    ```

1. Create a new Bicep parameters file named *`data-plane-role-assignment.bicepparam`*. In this parameters file; assign the name of your existing Azure Cosmos DB for Cassandra account to the `accountName` parameter, the previously recorded role definition identifiers to the `roleDefinitionId` parameter, and the unique identifier for your identity to the `identityId` parameter.

    ```bicep
    using './data-plane-role-assignment.bicep'
    
    param accountName = '<name-of-existing-table-account>'
    param roleDefinitionId = '<id-of-new-role-definition>'
    param identityId = '<id-of-existing-identity>'
    ```

1. Deploy this Bicep template using `az deployment group create`.

    ```azurecli-interactive
    az deployment group create \
        --resource-group "<name-of-existing-resource-group>" \
        --parameters data-plane-role-assignment.bicepparam \
        --template-file data-plane-role-assignment.bicep
    ```

1. Repeat these steps to grant access to the account from any other identities you would like to use.

    > [!TIP]
    > You can repeat these steps for as many identities as you'd like. Typically, these steps are at least repeated to allow developers access to an account using their human identity and to allow applications access to data using a managed identity.

::: zone-end

::: zone pivot="azure-powershell"

1. List all of the role definitions associated with your Azure Cosmos DB for Cassandra account using `Get-AzCosmosDBCassandraRoleDefinition`.

    ```azurepowershell-interactive
    Get-AzCosmosDBCassandraRoleDefinition \
      -AccountName <account-name> \
      -ResourceGroupName <resource-group>
    ```

1. Review the output and locate the role definition named **Cosmos DB Built-in Data Contributor**. The output contains the unique identifier of the role definition in the `Id` property. Record this value as it is required to use in the assignment step later in this guide.

1. Create a new role definition using `New-AzCosmosDBCassandraRoleDefinition` and a JSON file describing the permissions you want to allow.

    ```azurepowershell-interactive
    New-AzCosmosDBCassandraRoleDefinition \
      -AccountName <account-name> \
      -ResourceGroupName <resource-group> \
      -InputObject (Get-Content -Raw -Path ./role-definition.json | ConvertFrom-Json)
    ```

1. List all of the role definitions again and record the `id` of your new custom role.

1. Assign the new role using `New-AzCosmosDBCassandraRoleAssignment`. Use the previously recorded role definition identifier, the unique identifier for your identity, and the scope (account, keyspace, or table).

    ```azurepowershell-interactive
    New-AzCosmosDBCassandraRoleAssignment \
      -AccountName <account-name> \
      -ResourceGroupName <resource-group> \
      -RoleDefinitionId <role-definition-id> \
      -PrincipalId <principal-id> \
      -Scope /
    ```

1. List all role assignments for your account and verify your assignment.

    ```azurepowershell-interactive
    Get-AzCosmosDBCassandraRoleAssignment \
      -AccountName <account-name> \
      -ResourceGroupName <resource-group>
    ```

::: zone-end

::: zone pivot="azure-portal"

> [!WARNING]
> Managing data plane role-based access control isn't supported in the Azure portal.

::: zone-end

## Validate data plane role-based access in code

Validate that you correctly granted access using application code and the Azure SDK.

```csharp
using Azure.Identity;
using Azure.Core;
using Cassandra;
using System.Security.Authentication;

// Acquire token using managed identity
ManagedIdentityCredential defaultCredential = new ManagedIdentityCredential();
var scopes = new string[] { "https://cosmos.azure.com/.default" };
CancellationTokenSource cts = new CancellationTokenSource();

var accessToken = await defaultCredential.GetTokenAsync(new TokenRequestContext(scopes), cts.Token);
var credentialString = accessToken.Token;

// Set up SSL options
var options = new SSLOptions(SslProtocols.Tls12, true, ValidateServerCertificate);
options.SetHostNameResolver((ipAddress) => "<account-name>.cassandra.cosmos.azure.com");

// Connect to Cassandra
Cluster cluster = Cluster.Builder()
    .WithCredentials("<account-name>", credentialString)
    .WithPort(10350)
    .AddContactPoint("<account-name>.cassandra.cosmos.azure.com")
    .WithSSL(options)
    .Build();

ISession session = cluster.Connect("<keyspace-name>");
```

```python
from azure.identity import ManagedIdentityCredential
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider
import ssl

# Acquire token using managed identity
credential = ManagedIdentityCredential()
access_token = credential.get_token("https://cosmos.azure.com/.default")
credential_string = access_token.token

# Set up SSL context
ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
ssl_context.verify_mode = ssl.CERT_NONE

auth_provider = PlainTextAuthProvider(username="<account-name>", password=credential_string)
cluster = Cluster(["<account-name>.cassandra.cosmos.azure.com"], port=10350, auth_provider=auth_provider, ssl_context=ssl_context)
session = cluster.connect("<keyspace-name>")
```

```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenRequestContext;
import com.azure.core.credential.AccessToken;
import com.azure.core.credential.TokenCredential;
import com.datastax.oss.driver.api.core.CqlSession;
import java.net.InetSocketAddress;
import javax.net.ssl.SSLContext;

// Acquire token using managed identity
TokenCredential credential = new DefaultAzureCredentialBuilder().build();
TokenRequestContext requestContext = new TokenRequestContext().addScopes("https://cosmos.azure.com/.default");
AccessToken accessToken = credential.getToken(requestContext).block();
String credentialString = accessToken.getToken();

// Set up SSL context (assume sslContext is properly configured)
try (CqlSession session = CqlSession.builder()
        .addContactPoint(new InetSocketAddress("<account-name>.cassandra.cosmos.azure.com", 10350))
        .withAuthCredentials("<account-name>", credentialString)
        .withSslContext(sslContext)
        .withKeyspace("<keyspace-name>")
        .withLocalDatacenter("<azure-region>")
        .build()) {
}
```

## Related content

- [Secure your Azure Cosmos DB for Apache Cassandra account](security.md)
- [Azure Cosmos DB for Apache Cassandra control plane roles](/azure/role-based-access-control/built-in-roles?context=/azure/cosmos-db/context/context#databases)
- [Azure Cosmos DB for Apache Cassandra control plane permissions](/azure/role-based-access-control/permissions/databases?context=/azure/cosmos-db/context/context#microsoftdocumentdb)
- [Azure Cosmos DB for Apache Cassandra data plane roles](reference-data-plane-security.md#built-in-roles)
- [Azure Cosmos DB for Apache Cassandra data plane permissions](reference-data-plane-security.md#built-in-actions)