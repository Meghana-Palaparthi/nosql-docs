---
ms.topic: include
ms.date: 10/13/2025
---

[!INCLUDE[Prerequisites - Azure subscription](prerequisite-azure-subscription.md)]

- An existing Azure DocumentDB cluster. If you don't have a cluster, create a [new cluster](../quickstart-portal.md).
  
  - [Azure role-based access control (Azure RBAC) is enabled](../how-to-connect-role-based-access-control.md#enable-microsoft-entra-id-authentication).
  
  - [Firewall is configured to allow access to your client IP address](../how-to-configure-firewall.md#grant-access-from-your-ip-address).

- An [Azure OpenAI resource](/azure/ai-foundry/openai/how-to/create-resource?view=foundry-classic&pivots=cli#create-a-resource&preserve-view=true).

  - Custom domain is configured.

  - [Azure RBAC is enabled](/azure/developer/ai/keyless-connections).
  
  - `text-embedding-3-small` model is deployed.
  
- [Visual Studio Code](https://code.visualstudio.com/download). Ensure that you have the [Azure DocumentDB extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb).
  
[!INCLUDE[External - Azure CLI prerequisites](~/reusable-content/azure-cli/azure-cli-prepare-your-environment-no-header.md)]
