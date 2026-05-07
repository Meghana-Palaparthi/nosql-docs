---
title: Azure Cosmos DB Shell
description: Learn about the Azure Cosmos DB Shell, a powerful command-line interface for interacting with Cosmos DB databases with bash-like commands and AI integration.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: overview
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell

Azure Cosmos DB Shell is a powerful, **open-source** command-line interface (CLI) that enables you to interact with your Azure Cosmos DB databases using intuitive bash-like commands. It features optional Model Context Protocol (MCP) server support for AI-powered automation and integrates seamlessly with VS Code through an extension.

## Key features

- **Bash-like Syntax**: Familiar command structure with commands like `cd`, `ls`, `pwd`, `rm`, `mkdir`
- **Database Operations**: Create and manage databases and containers
- **Data Manipulation**: Query, insert, update, and delete documents
- **Pipe Support**: Chain commands together for powerful data transformations
- **JSON Queries**: Execute SQL queries with JSON output
- **Scripting**: Write and execute shell scripts for automated operations
- **MCP Server Integration**: Enable AI assistants to interact with your Cosmos DB resources
- **VS Code Extension**: Seamless integration with Visual Studio Code
- **Open Source**: Community-driven development with transparent contributions

## System requirements

- **Operating System**: Windows, macOS (Intel and Apple Silicon), Linux (x64 and ARM)
- **For NuGet Installation**: .NET SDK 10.0 or later
- **For VS Code Extension**: VS Code 1.85 or later
- **Authentication**: Microsoft Entra ID, Managed Identity, or Account Keys

## Installation options

Choose one of three installation methods:

- **VS Code Extension** (Recommended)
  - Install directly from VS Code Marketplace
  - See: [Visual Studio Code Extension Setup](visual-studio-code.md)

- **NuGet Package**
  - Install as .NET global tool
  - See: [Installation Guide](install.md)

- **Self-Contained Binary**
  - Pre-built binaries for your platform
  - See: [Installation Guide](install.md)

## Quick start

```bash
# Connect to your Cosmos DB account
cosmosdbshell

# Navigate to a database
> cd database_name

# List containers
> ls

# Query documents
> query "SELECT * FROM c WHERE c.name = 'John'"
```

For more examples, see the [Quick Start Guide](get-started.md).

## Use cases

- **Development & Testing**: Quick command-line access during development
- **Database Administration**: Manage databases, containers, and data
- **Data Exploration**: Query and explore your data interactively
- **Automation**: Use in scripts for automated database operations
- **AI Integration**: Enable AI assistants to work with your data via MCP
- **Learning**: Educational tool for learning Cosmos DB concepts

## Release status

Azure Cosmos DB Shell is currently in **private preview**. This is a preview version for testing and feedback purposes only. Your feedback is valuable in helping us improve the tool.

## Available distributions

| Package | Version | Location |
|---------|---------|----------|
| NuGet Package | 1.0.213-preview | [NuGet.org](https://www.nuget.org/packages/CosmosDBShell/1.0.213-preview) |
| VS Code Extension | Latest | [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-cosmosdb) |
| Self-Contained Binaries | 1.0.213-preview | Available for Windows, macOS, Linux |

## Documentation

- [Installation Guide](install.md) - Install using your preferred method
- [Quick Start Guide](get-started.md) - Get started in minutes
- [Command Reference](command-reference.md) - Complete command documentation
- [Model Context Protocol Setup](model-context-protocol-setup.md) - Enable AI integration
- [Visual Studio Code Extension](visual-studio-code.md) - VS Code integration guide
- [Troubleshooting Guide](troubleshooting.md) - Resolve common issues
- [Security Best Practices](security.md) - Secure your credentials and data

## Support and feedback

- [Report Issues](https://github.com) - Found a bug?
- [Submit Feature Requests](https://github.com) - Have a feature idea?
- [View Security Considerations](security.md) - Review security best practices

## Next steps

- [Install Azure Cosmos DB Shell](install.md)
- [Try the Quick Start Guide](get-started.md)
- [Review Security Best Practices](security.md)

## See also

- [Azure Cosmos DB Overview](overview.md)
- [Azure Cosmos DB SDKs](../conceptual-resilient-sdk-applications.md)
- [Data Explorer](../data-explorer.md)
