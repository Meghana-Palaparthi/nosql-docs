---
title: Azure Cosmos DB Shell Visual Studio Code Extension
description: Learn how to use the Azure Cosmos DB Shell Visual Studio Code extension for seamless database interactions and resource management.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell Visual Studio Code extension

The Azure Cosmos DB Shell Visual Studio Code extension provides a seamless, integrated experience for managing and querying your Cosmos DB databases directly from the code editor.

## Installation

### Method 1: VS Code Marketplace (recommended)

1. **Open VS Code**
   - Launch Visual Studio Code on your machine

2. **Open Extensions Marketplace**
   - Click the Extensions icon (Ctrl+Shift+X / Cmd+Shift+X)
   - Or go to View > Extensions

3. **Search for Cosmos DB**
   - Type "Cosmos DB" in the search box
   - Look for "Azure Cosmos DB Shell" by Microsoft

4. **Install the Extension**
   - Click the Install button
   - Wait for installation to complete
   - Reload VS Code if prompted

### Method 2: Install from VSIX file

1. **Download Extension**
   - Download `vscode-cosmosdb-0.33.3.vsix` (or latest version) from the releases

2. **Install Extension**
   - Open VS Code
   - Go to Extensions (Ctrl+Shift+X / Cmd+Shift+X)
   - Click the "..." menu and select "Install from VSIX"
   - Select the downloaded file

3. **Complete Installation**
   - Wait for the installation to finish
   - Reload VS Code if prompted

### Requirements

- **VS Code Version**: 1.85 or later
- **Supported Platforms**: Windows, macOS, Linux
- **Disk Space**: ~50 MB

## Features

### 1. Resource explorer

Explore your Cosmos DB resources in a hierarchical tree view.

**Features:**
- View all databases in your account
- See containers and their properties
- Browse documents within containers
- Display partition key and indexing information

**Access:**
- Click the Cosmos DB icon in the Activity Bar
- Or go to View > Cosmos DB Explorer

### 2. Quick launch

Launch Cosmos DB Shell directly from VS Code.

**Methods:**
- **Command Palette**
  - Press Ctrl+Shift+P (Cmd+Shift+P on macOS)
  - Type "Cosmos DB Shell: Open"
  - Press Enter

- **Explorer Context Menu**
  - Right-click on a database or container
  - Select "Open in Shell"

- **Status Bar**
  - Click the Cosmos DB icon in the status bar

### 3. Command palette integration

Execute Cosmos DB Shell commands directly from the command palette.

**Available Commands:**
- `Cosmos DB Shell: Open` - Launch the shell
- `Cosmos DB Shell: Connect` - Switch accounts
- `Cosmos DB Shell: Disconnect` - Close connection
- `Cosmos DB Shell: Start MCP Server` - Enable MCP
- `Cosmos DB Shell: Stop MCP Server` - Disable MCP
- `Cosmos DB Shell: Settings` - Access extension settings

**Access Commands:**
1. Press Ctrl+Shift+P (Cmd+Shift+P)
2. Type the command name
3. Press Enter

### 4. Azure account integration

Seamlessly integrate with your Azure accounts.

**Features:**
- Automatic authentication with Azure account
- Support for multiple subscriptions
- Managed identity integration
- Azure Entra ID support

**Configuration:**
1. Sign in with your Azure account (prompted on first use)
2. Select subscription and resource group
3. Choose Cosmos DB account to connect

### 5. MCP server integration

Enable Model Context Protocol server for AI integration.

**Access:**
1. Open command palette (Ctrl+Shift+P)
2. Type "Cosmos DB: Start MCP Server"
3. Configure settings in VS Code settings.json

**Configuration:**
```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6128,
  "cosmosDB.shell.MCP.startOnLaunch": true
}
```

## Getting started

### Step 1: Install the extension

Follow the [Installation](#installation) section above.

### Step 2: Connect to your account

1. Click the Cosmos DB icon in the Activity Bar
2. Sign in with your Azure account (if prompted)
3. Select your subscription and Cosmos DB account
4. The Resource Explorer displays your databases and containers

### Step 3: Open Cosmos DB Shell

1. Press Ctrl+Shift+P (Cmd+Shift+P)
2. Type "Cosmos DB Shell: Open"
3. Press Enter to launch the integrated shell

### Step 4: Start querying

```bash
# Navigate to a database
> cd mydb

# List containers
> ls

# Query documents
> query "SELECT * FROM c"
```

## Working with resources

### Explore databases

**In Resource Explorer:**
1. Expand your Cosmos DB account
2. View all databases
3. Expand a database to see containers
4. Right-click for context menu options

**Available Actions:**
- Create database
- Delete database
- Create container
- Refresh

### Work with containers

**Right-click Container Options:**
- **Open in Shell** - Launch shell in container context
- **Query Documents** - Execute SQL query
- **Create Document** - Add new document
- **Delete Container** - Remove container
- **View Properties** - Display container metadata

### Manage documents

**Document Operations:**
- View document contents
- Create new documents
- Update documents
- Delete documents
- Copy document ID

## Integrated terminal

Use the integrated terminal for shell operations.

### Launch terminal

1. View > Integrated Terminal (Ctrl+`)
2. Or click the Terminal tab at the bottom

### Run commands

```bash
# In the integrated terminal
cosmosdbshell

# Navigate and query
> cd mydb
> ls
> query "SELECT * FROM c"
```

## Settings and configuration

### Access extension settings

1. Open Settings (Ctrl+, / Cmd+,)
2. Search for "Cosmos DB"
3. Modify settings as needed

### Available settings

| Setting | Default | Description |
|---------|---------|-------------|
| `cosmosDB.autoconnect` | `true` | Auto-connect on startup |
| `cosmosDB.shell.MCP.enabled` | `false` | Enable MCP server |
| `cosmosDB.shell.MCP.port` | `6128` | MCP server port |
| `cosmosDB.editor.theme` | `dark` | Editor color theme |
| `cosmosDB.queryTimeout` | `30000` | Query timeout (ms) |

### Example configuration

```json
{
  "cosmosDB.autoconnect": true,
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6128,
  "cosmosDB.queryTimeout": 60000
}
```

## Keyboard shortcuts

| Action | Shortcut |
|--------|----------|
| Open Command Palette | Ctrl+Shift+P (Cmd+Shift+P) |
| Open Integrated Terminal | Ctrl+` |
| Toggle Sidebar | Ctrl+B (Cmd+B) |
| Close Editor | Ctrl+W (Cmd+W) |
| Copy | Ctrl+C (Cmd+C) |
| Paste | Ctrl+V (Cmd+V) |

## Tips and tricks

### 1. Quick access to frequently used containers

1. Click star icon next to container name
2. Container appears in "Favorites" section
3. Quick access without expanding tree

### 2. Multi-container queries

Use multiple terminal tabs to work with different containers:

1. Open terminal (Ctrl+`)
2. Right-click to create new terminal tab
3. Connect to different containers in each tab

### 3. Save query results

Export query results to file:

```bash
# In integrated terminal
query "SELECT * FROM c" > results.json
```

### 4. Format JSON output

Use jq in the integrated terminal:

```bash
query "SELECT * FROM c" | jq '.[]'
```

### 5. Keyboard navigation

- Tab: Move between panels
- Arrow keys: Navigate tree
- Enter: Open selected item
- Delete: Delete selected resource

## Troubleshooting

### Extension won't load

**Issue:** Cosmos DB extension doesn't appear in VS Code

**Solutions:**
1. Verify VS Code version is 1.85 or later
2. Check extension is installed:
   - Go to Extensions (Ctrl+Shift+X)
   - Search "Cosmos DB"
   - Verify "Azure Cosmos DB Shell" by Microsoft is installed

3. Restart VS Code
4. Clear extension cache:
   - Delete `.vscode/extensions` folder
   - Reinstall extension

### Can't connect to account

**Issue:** "Sign in required" or authentication fails

**Solutions:**
1. Open command palette (Ctrl+Shift+P)
2. Type "Azure: Sign Out"
3. Type "Azure: Sign In"
4. Complete browser authentication
5. Select subscription and resource group

### MCP server won't start

**Issue:** MCP server fails to start in extension

**Solutions:**
1. Check port 6128 is not in use:
   ```bash
   # Windows
   netstat -ano | findstr 6128
   
   # macOS/Linux
   lsof -i :6128
   ```

2. Try different port in settings:
   ```json
   "cosmosDB.shell.MCP.port": 6129
   ```

3. Check firewall settings
4. Review Output panel for errors

### Shell doesn't respond

**Issue:** Cosmos DB Shell is frozen or unresponsive

**Solutions:**
1. Try Ctrl+C to interrupt
2. Close and reopen terminal
3. Restart VS Code
4. Check network connectivity
5. Verify Cosmos DB account is accessible

### Extension is slow

**Issue:** Extension is sluggish or resource-heavy

**Solutions:**
1. Disable auto-connect:
   ```json
   "cosmosDB.autoconnect": false
   ```

2. Reduce refresh frequency
3. Close unnecessary explorer items
4. Check VS Code Activity Monitor (Help > About)
5. Disable other extensions if conflicting

## Next steps

- [Command Reference](command-reference.md) - Learn all available commands
- [Quick Start Guide](get-started.md) - Get started with examples
- [Model Context Protocol Setup](model-context-protocol-setup.md) - Enable AI integration
- [Troubleshooting Guide](troubleshooting.md) - Resolve issues
- [Security Best Practices](security.md) - Secure your setup

## Support

- [Azure Support](https://azure.microsoft.com/support/)
- [VS Code Issues](https://github.com/microsoft/vscode/issues)
- [Report Extension Issues](https://github.com/Azure/vscode-cosmosdb/issues)

## See also

- [Azure Cosmos DB Shell Overview](overview.md)
- [Installation Guide](install.md)
- [Azure Cosmos DB Documentation](overview.md)
