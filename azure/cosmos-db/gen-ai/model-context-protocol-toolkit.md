---
title: Model Context Protocol (MCP) Toolkit
titleSuffix: Azure Cosmos DB for NoSQL
description: Learn how Azure Cosmos DB MCP Toolkit enables AI agents to interact with Cosmos DB data using natural language. Build AI-first solutions with secure database integration and real-time intelligence. Get started today.
author: sajeetharan
ms.author: sasinnat 
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: feature-guide
ms.date: 05/18/2026
ms.update-cycle: 180-days
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
---

# Model context protocol (MCP) toolkit for Azure Cosmos DB for NoSQL

The Azure Cosmos DB model context protocol (MCP) toolkit ([AzureCosmosDB/MCPToolKit](https://github.com/azurecosmosdb/mcptoolkit)) is an open-source solution that enables AI agents and agentic applications to interact securely with **Azure Cosmos DB** through the **Model Context Protocol (MCP)**. This toolkit provides enterprise-grade authentication, comprehensive database operations, and seamless integration with AI platforms like **Microsoft Foundry**.

The MCP Toolkit helps AI systems work with your Azure Cosmos DB data. It lets AI understand your database using everyday language queries. This functionality makes it perfect for building smart apps that need to read and study database content.

## Key Features


### **Enterprise-Grade Security**
- **Azure Entra ID Authentication** - Token-based security with role-based access control
- **Managed Identity Support** - No credential management required in production
- **Role-Based Access** - Automatic validation of `Mcp.Tool.Executor` permissions
- **Secure Communication** - HTTPS endpoints with proper authentication headers

### **Production-Ready Deployment**
- **Container Apps Hosting** - Scalable, managed container deployment on Azure
- **One-Click Deployment** - Automated infrastructure provisioning with Bicep templates
- **Health Monitoring** - Built-in health checks and logging integration
- **Web Testing Interface** - Browser-based tool for validation and testing

## Capabilities

This toolkit provides:

- **Secure MCP Server**: JWT-authenticated endpoint for AI agents
- **Azure Cosmos DB Integration**: Full CRUD operations, vector search, and schema discovery
- **Foundry Ready**: Optional one-step integration with Foundry projects
- **Enterprise Security**: Microsoft Entra ID, Managed Identity, role-based access control
- **Production Ready**: Container Apps hosting 
- **Local Development**: Docker Compose and .NET dev options

## Available Tools

The toolkit provides seven core tools for interacting with Cosmos DB:

| Tool Name | Description | Use Cases |
|-----------|-------------|-----------|
| **`list_databases`** | Lists all databases in the Cosmos DB account | Database discovery, environment exploration |
| **`list_collections`** | Lists containers within a specific database | Schema exploration, data organization |
| **`get_recent_documents`** | Retrieves the most recent documents (1-20) | Data freshness checks, recent activity analysis |
| **`find_document_by_id`** | Finds a specific document by its ID | Direct record retrieval, data validation |
| **`text_search`** | Full-text search within document properties | Content discovery, keyword-based queries |
| **`vector_search`** | Semantic search using AI embeddings | Similar content discovery, contextual search |
| **`get_approximate_schema`** | Analyzes document structure by sampling | Schema understanding, data modeling |

### Vector Search Requirements
For vector search functionality, you need:
- **Azure OpenAI** service with embedding deployment
- **Vector embeddings** stored in your Cosmos DB documents  
- **Vector indexing policy** configured on your containers

## Prerequisites


Before deploying the Azure Cosmos DB MCP Toolkit:

- **Azure Subscription** with Contributor or Owner access ([Free account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn))
- **Azure CLI** ([Install](/cli/azure/install-azure-cli)) installed and authenticated
- **PowerShell 7+** ([Install](/powershell/scripting/install/installing-powershell)) for deployment scripts
- **Git** ([Install](https://git-scm.com/downloads)) for cloning the repository
- **Docker Desktop** ([Install](https://www.docker.com/products/docker-desktop/)) running for container operations
- **.NET 9.0 SDK** ([Install](https://dotnet.microsoft.com/download/dotnet/9.0)) for building the application
- **Existing Azure Cosmos DB account** with data (the toolkit connects to your existing Cosmos DB)
- **Azure Entra ID** permissions for app registration
- **Azure Container Apps** quota in your region
- **Azure OpenAI** or **Microsoft Foundry project** (required for vector search with embeddings)
- **Optional**: Azure Developer CLI ([Install](https://aka.ms/azure-dev/install)) for `azd up` deployment method 

## Deployment Overview

To deploy and use the Azure Cosmos DB MCP Toolkit:

1. **Clone Repository**: Get the latest main branch

```bash
git clone -b main https://github.com/AzureCosmosDB/MCPToolKit.git
cd MCPToolKit
```

2. **Deploy Infrastructure**: Choose one method:
   - **Option A**: Use the "Deploy to Azure" button for one-click deployment
   - **Option B**: Use Azure Developer CLI with `azd up` for automation

3. **Deploy MCP Server**: Run the automated deployment script

```powershell
.\scripts\Deploy-Cosmos-MCP-Toolkit.ps1 -ResourceGroup "YOUR-RESOURCE-GROUP"
```

4. **Test**: Access the built-in test UI at your Container App URL
5. **Integrate**: Connect with Microsoft Foundry or Visual Studio Code

For detailed deployment instructions, testing guides, and configuration options, see the [Azure Cosmos DB MCP Toolkit README](https://github.com/AzureCosmosDB/MCPToolKit/blob/main/README.md).

## Multi-Resource Group Deployment Support

The MCP Toolkit now supports deploying resources across separate resource groups, enabling enterprise deployments where Cosmos DB, Azure Container Registry (ACR), and Azure Container Apps (ACA) are managed independently.

### When to Use Multiple Resource Groups

- **Enterprise deployments**: Separate resource groups for databases, infrastructure, and applications
- **Existing infrastructure**: Connecting to Cosmos DB in a different resource group than your deployment
- **Organizational policies**: Following your company's resource group organization standards
- **Team ownership**: Different teams owning different resource groups
- **Cost tracking**: Allocating resources and costs by department or project

### Deployment with Separate Resource Groups

Use the deployment script with explicit resource group parameters:

```powershell
# Deploy with Cosmos DB in a different resource group
.\scripts\Deploy-Cosmos-MCP-Toolkit.ps1 `
  -ResourceGroup "app-infrastructure-rg" `
  -CosmosAccountName "my-cosmos-account"
```

### Configuration Parameters

- **`-ResourceGroup`**: Resource group for ACR and Container Apps (required)
- **`-CosmosAccountName`**: Name of your existing Cosmos DB account
- **`-ContainerAppName`**: Name for the container app (optional, defaults to mcp-toolkit-app)
- **`-Location`**: Azure region for deployment (default: eastus)
- **`-EntraAppName`**: Custom name for the Entra app registration (optional)

### RBAC Role Assignment Across Resource Groups

The deployment script automatically validates and assigns the necessary roles:

- **Cosmos DB Data Reader** role to the Container App's managed identity in the Cosmos DB resource group
- **AcrPush** role in the application resource group for container operations
- **Cognitive Services OpenAI User** role for Microsoft Foundry integration

If you lack permissions to assign roles in a resource group, the script provides clear instructions for manual role assignment. You can also use the helper scripts:

```powershell
# Assign role to current user
.\scripts\Assign-Role-To-Current-User.ps1

# Assign role to specific users
.\scripts\Assign-Role-To-Users.ps1 -UserEmails "user1@company.com,user2@company.com"

# Verify role assignments
.\scripts\Verify-Role-Assignments.ps1
```

## Validate the Deployment

### Using the Built-In Test UI

The MCP server includes a web-based test interface accessible at your Container App URL:

1. **Navigate to**: `https://YOUR-CONTAINER-APP.azurecontainerapps.io`

2. **Authenticate**: Select "Sign In with Microsoft Entra ID" (no token copy/paste required)

3. **Test MCP Tools**: Use the interactive forms to explore your Cosmos DB data

### Example Test Scenarios
- **List databases**: Select `list_databases` → Then select "Invoke Tool"
- **Explore containers**: Enter database name → Select `list_collections`  
- **Recent documents**: Enter database + container → Select `get_recent_documents`
- **Search content**: Configure search parameters → Select `text_search`
- **Vector search**: Provide search text and vector property → Select `vector_search`

## Connect to Foundry  

To connect your MCP server to a Foundry project, run the setup script:

```powershell
.\scripts\Setup-AIFoundry-RoleAssignment.ps1 `
  -AIFoundryProjectName "YOUR-PROJECT-NAME" `
  -ResourceGroup "YOUR-RESOURCE-GROUP"
```

This script assigns the necessary roles for Foundry to call your MCP server.

### Configuring the Connection in Foundry

1. Navigate to [Foundry](https://ai.azure.com)
2. Open your AI project
3. Go to **Tools** → **Add Custom Tool**
4. Configure the connection:
   - **Tool Name**: "Azure Cosmos DB"
   - **Endpoint URL**: Your Container App URL (from `deployment-info.json`)
   - **Authentication**: Microsoft Entra ID  
   - **Identity**: Project Identity or User Identity
   - **Client ID**: From your deployment (found in `deployment-info.json`)

### Using with AI Agents

Once connected, you can prompt your AI agents with natural language:
- *"List all databases in my Cosmos DB account"*
- *"Find recent orders in the ecommerce database"*  
- *"Search for products containing 'electronics' in the description"*
- *"What's the schema of the products container?"*

Foundry translates these requests into MCP tool calls, executing them securely against your Cosmos DB.

### Python Client Example

For a complete example of using the MCP server with Foundry agents, see the [Python Client README](https://github.com/AzureCosmosDB/MCPToolKit/blob/main/client/README.md).

## How it Works

:::image type="complex" source="media/model-context-protocol-toolkit/architecture-diagram.svg" lightbox="media/model-context-protocol-toolkit/architecture-diagram.svg" alt-text="Diagram of Foundry connecting to Azure Cosmos DB through MCP Toolkit.":::
  This architecture diagram illustrates a three-tier system for connecting AI agents to Azure Cosmos DB data. Three primary components are arranged horizontally and connected in sequence: Foundry (which hosts AI Agents), the MCP Toolkit (deployed as a Container App) in the center, and Azure Cosmos DB (containing your data). The MCP Toolkit serves as the intermediary layer, enabling Foundry to access and interact with data stored in Azure Cosmos DB.
  
  Microsoft Entra ID authentication (providing security) connects to all three primary components through diagonal lines, forming a hub-and-spoke pattern. This central authentication layer ensures that all communications between the AI agents, the MCP Toolkit, and the database are properly secured and authorized. The diagram demonstrates a secure, layered architecture where the MCP Toolkit acts as a bridge between AI workloads and data storage. This diagram demonstrates the user of Microsoft Entra ID providing unified identity and access management across the entire system.
:::image-end:::

**Key Components:**
- **Foundry**: Hosts AI agents that need database access
- **MCP Toolkit**: Translates AI requests to Cosmos DB operations  
- **Microsoft Entra ID**: Provides enterprise authentication and authorization
- **Cosmos DB**: Your existing database with business data

## Security 

### Authentication
- **JWT Bearer Tokens**: All requests require valid Microsoft Entra ID tokens
- **Audience Validation**: Tokens must be issued for your Microsoft Entra App
- **Managed Identity**: Container App uses managed identity for Cosmos DB access
- **RBAC**: Least-privilege role assignments

### Authentication Flow
1. **Foundry** requests database operation via MCP protocol
2. **MCP Toolkit** validates Microsoft Entra token and role membership
3. **Cosmos DB** access uses managed identity (no stored credentials)
4. **Results** returned securely to AI agent

## Configuration

### Visual Studio Code Integration

To use with GitHub Copilot or other Visual Studio Code MCP clients:

1. Get your MCP server URL from `deployment-info.json`
2. Add to Visual Studio Code settings (`settings.json`):

```json
{
  "mcp.servers": {
    "cosmosdb": {
      "url": "https://YOUR-CONTAINER-APP.azurecontainerapps.io/mcp",
      "headers": {
        "Authorization": "Bearer YOUR-JWT-TOKEN"
      }
    }
  }
}
```

## Considerations

- **Read-only operations** - Toolkit provides query and search capabilities only
- **Azure OpenAI dependency** - Vector search requires Azure OpenAI service
- **Regional deployment** - Best performance when deployed in same region as Cosmos DB
- **Container Apps quotas** - Subject to Azure Container Apps service limits

### Best Practices
- **Deploy close to data** - Use same Azure region as your Azure Cosmos DB for best performance
- **Monitor usage** - Review Container Apps metrics and Azure Cosmos DB RU consumption
- **Secure endpoints** - Use private endpoints for production deployments
- **Regular updates** - Keep the toolkit updated for latest features and security
- **Multi-resource group strategy** - Use separate resource groups when appropriate for your organization's governance model
- **Role-based access** - Leverage Azure RBAC for least-privilege access across resource groups
- **Credential rotation** - Rotate Azure Entra ID application credentials regularly
- **Health monitoring** - Enable Container App monitoring and set up alerts for failed health checks

## Recent Updates

**May 2026 - Release** improvements:
- Multi-resource group deployment support for enterprise scenarios
- Enhanced error handling and troubleshooting guides  
- Azure Developer CLI (`azd up`) deployment option
- Improved role assignment workflows across resource groups
- Support for existing service principals and Entra apps
- Better integration with Azure Container Registry and Azure Container Apps
- Helper scripts for role assignment and verification
- Support for reusing existing Entra app registrations

For a complete list of changes, see the [CHANGELOG](https://github.com/AzureCosmosDB/MCPToolKit/blob/main/CHANGELOG.md).

## Related content

- [Azure Cosmos DB Samples Gallery](https://aka.ms/AzureCosmosDB/Gallery)
- [Vector Search with Azure Cosmos DB for NoSQL](vector-search-overview.md)
- [Tokens](tokens.md)
- [Vector Embeddings](vector-embeddings.md)
- [Retrieval Augmented Generated (RAG)](rag.md)
- [MCPToolKit GitHub Repository](https://github.com/AzureCosmosDB/MCPToolKit)
