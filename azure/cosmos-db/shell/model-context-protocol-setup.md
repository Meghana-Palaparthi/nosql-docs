---
title: Azure Cosmos DB Shell Model Context Protocol Setup
description: Set up and configure the Model Context Protocol (MCP) server for Azure Cosmos DB Shell to enable AI-powered database interactions.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell model context protocol setup

Enable the Model Context Protocol (MCP) server in Azure Cosmos DB Shell to allow AI assistants and applications to interact with your Cosmos DB resources programmatically.

## What is MCP?

The Model Context Protocol (MCP) is an open standard that enables AI assistants to interact with external tools and systems. When enabled in Cosmos DB Shell, it allows:

- **AI Assistants**: Use AI assistants to query and manage your Cosmos DB data
- **Automated Workflows**: Create workflows that interact with your databases
- **Integration**: Integrate Cosmos DB with other AI-powered applications
- **Data Exploration**: Enable AI to explore and analyze your data

## Prerequisites

- Azure Cosmos DB Shell installed ([Installation Guide](install.md))
- VS Code with Cosmos DB Shell extension (recommended)
- Basic understanding of MCP concepts
- Network connectivity to localhost (MCP server runs locally)

## Enable MCP server

### Step 1: Access VS Code settings

1. Open VS Code
2. Go to Settings (Ctrl+, / Cmd+,)
3. Search for "Cosmos DB"

### Step 2: Configure MCP settings

Add the following settings to your VS Code `settings.json`:

```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6128,
  "cosmosDB.shell.MCP.startOnLaunch": true,
  "cosmosDB.shell.MCP.bindToLocalhost": true
}
```

**Setting Details:**

| Setting | Value | Description |
|---------|-------|-------------|
| `cosmosDB.shell.MCP.enabled` | `true` | Enable MCP server |
| `cosmosDB.shell.MCP.port` | `6128` | Port number (default: 6128) |
| `cosmosDB.shell.MCP.startOnLaunch` | `true` | Auto-start MCP on shell launch |
| `cosmosDB.shell.MCP.bindToLocalhost` | `true` | Bind only to localhost (secure) |

### Step 3: Restart VS Code

Close and reopen VS Code to apply settings.

## Verify MCP server is running

### Check MCP status

1. Open the Cosmos DB Shell output panel
2. Look for confirmation message:
   ```
   MCP Server started on http://localhost:6128
   ```

### Test MCP connection

Open a terminal and test the MCP server:

```bash
# On Windows
curl http://localhost:6128/health

# On macOS/Linux
curl http://localhost:6128/health
```

Expected response:
```json
{
  "status": "healthy",
  "version": "1.0.213-preview"
}
```

## MCP configuration options

### Minimal configuration (recommended)

For most users, minimal configuration is sufficient:

```json
{
  "cosmosDB.shell.MCP.enabled": true
}
```

This uses default settings:
- Port: 6128
- Auto-start: Enabled
- Localhost binding: Enabled

### Advanced configuration

For custom configurations:

```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6129,
  "cosmosDB.shell.MCP.startOnLaunch": false,
  "cosmosDB.shell.MCP.bindToLocalhost": true,
  "cosmosDB.shell.MCP.timeout": 30000,
  "cosmosDB.shell.MCP.maxConnections": 10,
  "cosmosDB.shell.MCP.logLevel": "info"
}
```

**Advanced Settings:**

| Setting | Value | Description |
|---------|-------|-------------|
| `cosmosDB.shell.MCP.timeout` | Milliseconds | Request timeout (default: 30000) |
| `cosmosDB.shell.MCP.maxConnections` | Number | Maximum concurrent connections (default: 10) |
| `cosmosDB.shell.MCP.logLevel` | `debug`/`info`/`warn`/`error` | Logging level (default: `info`) |

## Manual MCP server start

If you disabled auto-start, manually start the MCP server:

### In VS Code

1. Open command palette (Ctrl+Shift+P / Cmd+Shift+P)
2. Type "Cosmos DB: Start MCP Server"
3. Press Enter

### From command line

```bash
cosmosdbshell --mcp --mcp-port 6128
```

## Using MCP with AI assistants

### Example 1: Claude/ChatGPT via tools

Configure your AI client to connect to the MCP server:

```json
{
  "mcpServers": {
    "cosmosdb": {
      "command": "cosmosdbshell",
      "args": ["--mcp"],
      "env": {
        "MCP_PORT": "6128"
      }
    }
  }
}
```

### Example 2: Query via MCP

Once configured, your AI assistant can:

```
"Query all users from the 'users' container in the 'mydb' database"
```

The MCP server translates this to:
```bash
cd /mydb/users
query "SELECT * FROM c"
```

### Example 3: Data analysis

```
"Count documents by status in the users container and show me the breakdown"
```

The MCP server executes:
```bash
query "SELECT c.status, COUNT(*) as count FROM c GROUP BY c.status"
```

## Authentication with MCP

The MCP server uses the same authentication as Cosmos DB Shell:

- **Entra ID** (Recommended)
  - Authenticates using Azure identity
  - Most secure method
  - Requires appropriate RBAC roles

- **Managed Identity** (Production)
  - Automatically uses VM or App Service managed identity
  - Best for Azure-hosted applications
  - Requires identity to have Cosmos DB permissions

- **Account Key** (Development)
  - Uses account key from connection string
  - Quick for testing
  - Less secure, avoid in production

## Security considerations

### MCP server security best practices

- **Local Binding Only** (Default)
  ```json
  "cosmosDB.shell.MCP.bindToLocalhost": true
  ```
  - Restricts MCP server to localhost
  - Prevents remote access
  - **Recommended**: Keep this enabled

- **Network Isolation**
  - Run MCP server in isolated network
  - Use firewall rules to block external access
  - Don't expose port 6128 to public internet

- **Authentication**
  - Use Entra ID for authentication
  - Implement RBAC least-privilege access
  - Rotate account keys regularly

- **Audit and Monitoring**
  - Enable MCP server logging
  - Monitor MCP server activity
  - Review logs regularly for suspicious activity

### Connection security

**Secure Configuration:**
```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.bindToLocalhost": true,
  "cosmosDB.shell.MCP.logLevel": "info"
}
```

## Troubleshooting

### MCP server won't start

**Issue:** MCP server fails to start

**Solutions:**
1. Check port availability:
   ```bash
   # Windows
   netstat -ano | findstr 6128
   
   # macOS/Linux
   lsof -i :6128
   ```

2. Try different port:
   ```json
   "cosmosDB.shell.MCP.port": 6129
   ```

3. Check firewall settings
4. Review VS Code output panel for error messages

### Connection refused

**Issue:** AI client can't connect to MCP server

**Solutions:**
1. Verify MCP is enabled and started
2. Check correct port number (default: 6128)
3. Ensure localhost binding is enabled
4. Check firewall allows localhost connections

### Authentication failures

**Issue:** MCP server can't authenticate to Cosmos DB

**Solutions:**
1. Verify Cosmos DB Shell authentication works:
   ```bash
   cosmosdbshell
   ```

2. Check Entra ID permissions
3. Verify managed identity has correct roles
4. Test with account key in development

### Performance issues

**Issue:** MCP server is slow or unresponsive

**Solutions:**
1. Reduce `maxConnections` if too many requests:
   ```json
   "cosmosDB.shell.MCP.maxConnections": 5
   ```

2. Increase timeout for slow queries:
   ```json
   "cosmosDB.shell.MCP.timeout": 60000
   ```

3. Check network connectivity
4. Monitor system resources (CPU, memory, disk)

### Port already in use

**Issue:** Port 6128 is already in use

**Solutions:**
1. Find and stop the process using port:
   ```bash
   # Windows
   netstat -ano | findstr 6128
   taskkill /PID <PID> /F
   
   # macOS/Linux
   lsof -i :6128
   kill -9 <PID>
   ```

2. Change MCP port:
   ```json
   "cosmosDB.shell.MCP.port": 6129
   ```

## View MCP logs

### Enable debug logging

```json
{
  "cosmosDB.shell.MCP.logLevel": "debug"
}
```

### Access logs

Logs are displayed in VS Code Output panel:
1. Open Output panel (View > Output)
2. Select "Cosmos DB Shell" from dropdown
3. Review MCP server logs

## Stop MCP server

### In VS Code

1. Open command palette (Ctrl+Shift+P / Cmd+Shift+P)
2. Type "Cosmos DB: Stop MCP Server"
3. Press Enter

### From command line

```bash
# Kill the MCP server process
# The shell will exit when you type:
exit
```

## Advanced scenarios

### MCP with multiple Cosmos DB accounts

Create separate MCP instances for each account:

```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6128,
  "cosmosDB.shell.MCP.accounts": [
    {
      "name": "production",
      "endpoint": "https://prod.documents.azure.com:443/",
      "port": 6128
    },
    {
      "name": "staging",
      "endpoint": "https://staging.documents.azure.com:443/",
      "port": 6129
    }
  ]
}
```

### MCP with custom AI agents

Integrate MCP with custom AI agents:

```python
import requests

# Query Cosmos DB via MCP server
response = requests.post(
    'http://localhost:6128/query',
    json={
        'database': 'mydb',
        'container': 'users',
        'query': 'SELECT * FROM c WHERE c.status = "active"'
    }
)

results = response.json()
print(results)
```

## Next steps

- [Quick Start Guide](get-started.md) - Get started with basic commands
- [Command Reference](command-reference.md) - Learn all available commands
- [Security Best Practices](security.md) - Review security guidelines
- [Troubleshooting Guide](troubleshooting.md) - Resolve common issues

## See also

- [Azure Cosmos DB Shell Overview](overview.md)
- [Visual Studio Code Extension Guide](visual-studio-code.md)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io)
