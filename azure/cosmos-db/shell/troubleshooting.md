---
title: Azure Cosmos DB Shell Troubleshooting Guide
description: Troubleshoot common issues with Azure Cosmos DB Shell including installation, authentication, connections, and commands.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: troubleshooting
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell troubleshooting guide

Resolve common issues with Azure Cosmos DB Shell installation, authentication, connections, and command execution.

## Installation issues

### Issue: "Command not found" (NuGet installation)

**Error:**
```
cosmosdbshell: command not found
```

**Causes:**
- .NET SDK not installed
- Tool not installed correctly
- PATH not updated
- Terminal not restarted after installation

**Solutions:**

1. **Verify .NET SDK Installation**
   ```bash
   dotnet --version
   ```
   - Should show version 10.0 or later
   - If not installed, [install .NET SDK](https://dotnet.microsoft.com/download)

2. **Reinstall the Tool**
   ```bash
   dotnet tool uninstall --global CosmosDBShell
   dotnet tool install --global CosmosDBShell --prerelease
   ```

3. **Verify Installation Location**
   ```bash
   dotnet tool list --global | grep CosmosDBShell
   ```
   - Should show: `cosmosdbshell 1.0.213-preview`

4. **Restart Terminal**
   - Close and reopen your terminal
   - Terminal may cache PATH information

5. **Add to PATH Manually (if needed)**
   - **Windows**: Add `%USERPROFILE%\.dotnet\tools` to PATH
   - **macOS/Linux**: `~/.dotnet/tools` should already be in PATH

### Issue: "File not found" (binary installation)

**Error:**
```
cosmosdbshell: No such file or directory
```

**Causes:**
- Archive didn't extract completely
- File not executable (Linux/macOS)
- Wrong file path

**Solutions:**

- **Verify Extraction**
  - Check all files were extracted
  - Look for `cosmosdbshell` executable (or `.exe` on Windows)

- **Make Executable (Linux/macOS)**
  ```bash
  chmod +x ~/cosmosdbshell/cosmosdbshell
  ```

- **Use Full Path**
  ```bash
  /full/path/to/cosmosdbshell
  ```

- **Add to PATH**
  ```bash
  export PATH="$PATH:~/cosmosdbshell"
  ```

### Issue: "File corrupted" during download

**Error:**
```
Archive is corrupted or incomplete
```

**Causes:**
- Incomplete download
- Network interruption
- Wrong file format

**Solutions:**

- **Download Again**
  - Delete corrupted file
  - Re-download from official source
  - Verify file size matches expected

- **Check MD5/SHA Hash**
  - Compare hash of downloaded file
  - Matches hash provided on release page

- **Use Different Browser**
  - Try alternative download method
  - Try different network connection

## Authentication issues

### Issue: "Authentication failed"

**Error:**
```
Authentication failed. Please check your credentials.
```

**Causes:**
- Expired token
- Wrong credentials
- No internet connection
- Account key incorrect

**Solutions:**

- **Try Entra ID (Recommended)**
  - Cosmos DB Shell should prompt for browser authentication
  - Sign in with your Azure account
  - Most reliable method

- **Check Internet Connection**
  ```bash
  ping azure.microsoft.com
  ```
  - Ensure network connectivity

- **Verify Account Key**
  - Go to Azure portal
  - Navigate to Cosmos DB account
  - Copy Primary Connection String
  - Verify format is correct

- **Clear Cached Credentials**
  - **Windows**: `credman` (Credential Manager)
  - **macOS**: Keychain
  - **Linux**: Check ~/.config directory

### Issue: "Unauthorized - insufficient permissions"

**Error:**
```
Authorization failed. Your user does not have permission to perform this operation.
```

**Causes:**
- User lacks necessary RBAC roles
- Role not assigned to account
- Wrong subscription context

**Solutions:**

1. **Check RBAC Roles**
   - Go to Azure portal > Cosmos DB Account
   - Access Control (IAM)
   - Verify your user has "Cosmos DB Account Reader" or higher role

2. **Request Role Assignment**
   - Ask your subscription admin
   - Assign appropriate role (see [Security Best Practices](security.md))

3. **Verify Subscription**
   ```bash
   az account show
   ```
   - Ensure you're in correct subscription
   - Switch if needed: `az account set --subscription <ID>`

### Issue: "Token expired"

**Error:**
```
Token has expired. Please re-authenticate.
```

**Causes:**
- Session timeout
- Token expired after long inactivity
- Time synchronization issue

**Solutions:**

1. **Re-authenticate**
   - Exit shell: `exit`
   - Restart: `cosmosdbshell`
   - Sign in again

2. **Check System Time**
   - Ensure system clock is synchronized
   - **Windows**: Settings > Time & Language > Date & Time
   - **macOS/Linux**: Check system time with `date`

3. **Check Token Refresh Settings**
   - Tokens should auto-refresh
   - If issue persists, contact Azure support

## Connection issues

### Issue: "Connection refused"

**Error:**
```
Connection refused to Cosmos DB account
```

**Causes:**
- Wrong endpoint URL
- Firewall blocking connection
- Network connectivity issue
- Account endpoint incorrect

**Solutions:**

1. **Verify Endpoint URL**
   - Azure portal > Cosmos DB Account
   - Copy correct endpoint URL
   - Format: `https://<account>.documents.azure.com:443/`

2. **Check Firewall**
   - Verify IP allowed in Cosmos DB firewall rules
   - Azure portal > Cosmos DB Account > Firewall and virtual networks
   - Add your IP if needed

3. **Test Network Connectivity**
   ```bash
   ping <account>.documents.azure.com
   ```
   - Should respond with IP address

4. **Check VPN/Proxy**
   - If using VPN, ensure it's connected
   - If using proxy, verify proxy settings

### Issue: "Cannot resolve endpoint"

**Error:**
```
Cannot resolve host: account.documents.azure.com
```

**Causes:**
- DNS resolution failure
- Wrong endpoint name
- Network connectivity
- Firewall blocking DNS

**Solutions:**

1. **Verify Endpoint Spelling**
   - Double-check account name
   - Correct format: `https://<account>.documents.azure.com:443/`

2. **Check DNS**
   ```bash
   nslookup <account>.documents.azure.com
   ```
   - Should return IP address

3. **Try Alternative DNS**
   - Use public DNS (8.8.8.8, 1.1.1.1)
   - Check ISP DNS settings

4. **Test Direct Connection**
   ```bash
   telnet <account>.documents.azure.com 443
   ```
   - Should connect successfully

### Issue: "Timeout connecting to Cosmos DB"

**Error:**
```
Connection timeout. Could not connect to Cosmos DB account.
```

**Causes:**
- Network latency
- Slow internet connection
- Cosmos DB service issue
- Firewall blocking traffic

**Solutions:**

1. **Increase Timeout**
   - Modify connection timeout in settings
   - Default is 30 seconds

2. **Check Network Speed**
   - Test bandwidth with speed test
   - Ensure sufficient bandwidth

3. **Try Different Network**
   - Switch to different Wi-Fi or mobile hotspot
   - Test if issue is network-specific

4. **Check Azure Status**
   - Go to [Azure Status Page](https://status.azure.com)
   - Verify Cosmos DB service is operational

## Command issues

### Issue: "Command not recognized"

**Error:**
```
Unknown command. Type 'help' for available commands.
```

**Causes:**
- Typo in command name
- Command not supported in current version
- Using wrong syntax

**Solutions:**

1. **Check Command Syntax**
   - Use `help` to list available commands
   - Use `help <command>` for specific command help

2. **Verify Command Spelling**
   ```bash
   > help query
   ```
   - Correct: `query`, not `select` or `find`

3. **Check Your Version**
   ```bash
   > version
   ```
   - Ensure you have latest version

### Issue: "Syntax error in query"

**Error:**
```
Syntax error in SQL query
```

**Causes:**
- Invalid SQL syntax
- Missing quotes
- Incorrect JSON path
- Unsupported SQL function

**Solutions:**

1. **Review SQL Syntax**
   - Verify query is valid SQL (not NoSQL)
   - Example: `SELECT * FROM c WHERE c.status = 'active'`
   - Not valid: `db.collection.find({})`

2. **Check String Quotes**
   - Use single quotes in queries
   - String values: `'value'`
   - JSON keys: `"key"`

3. **Validate JSON Path**
   - Correct: `c.address.city`
   - Not valid: `c.address[0].city` (without proper syntax)

4. **Test Query in Azure portal**
   - Verify query works in Data Explorer first
   - Copy working query to shell

### Issue: "No documents returned"

**Error:**
```
CS > query "SELECT * FROM c"
(empty result)
```

**Causes:**
- Container is empty
- Query filter too restrictive
- Wrong container
- Partition key mismatch

**Solutions:**

- **Verify Container Has Data**
  ```bash
  > query "SELECT COUNT(*) as count FROM c"
  ```
  - Should return count > 0

- **Check Container Name**
  ```bash
  > pwd
  ```
  - Verify you're in correct container

- **Simplify Query**
  ```bash
  # Start with simple query
  > query "SELECT TOP 1 * FROM c"
  
  # Then add filters
  > query "SELECT * FROM c WHERE c.status = 'active'"
  ```

- **Review Partition Key**
  - Ensure partition key is included in query
  - For better performance: `SELECT * FROM c WHERE c.id = 'value'`

## VS Code extension issues

### Issue: Extension not appearing

**Error:**
```
Cannot find "Azure Cosmos DB Shell" in Extensions
```

**Causes:**
- VS Code version too old
- Extension not installed
- VS Code cache issue

**Solutions:**

- **Check VS Code Version**
  - Help > About
  - Should be 1.85 or later
  - Update if needed

- **Reinstall Extension**
  - Go to Extensions (Ctrl+Shift+X)
  - Search "Cosmos DB"
  - Click Install

- **Clear VS Code Cache**
  - Close VS Code
  - Delete `.vscode` folder in user directory
  - Restart VS Code

### Issue: Resource Explorer blank

**Error:**
```
Resource Explorer shows no databases or containers
```

**Causes:**
- Not authenticated
- No Cosmos DB accounts in subscription
- Wrong subscription selected
- Account has no databases

**Solutions:**

1. **Verify Authentication**
   - Click Cosmos DB icon in sidebar
   - Should show account selector
   - Sign in if prompted

2. **Check Subscription**
   - Verify correct subscription is selected
   - Switch subscription if needed

3. **Refresh Explorer**
   - Press F5 or click refresh icon
   - Wait for data to load

4. **Check Account Exists**
   - Go to Azure portal
   - Verify Cosmos DB account exists
   - Verify you have access to account

### Issue: MCP server won't start in VS Code

**Error:**
```
Failed to start MCP server
```

**Causes:**
- Port already in use
- Invalid settings
- Cosmos DB Shell not properly installed
- Firewall blocking port

**Solutions:**

1. **Check Port Availability**
   ```bash
   # Windows
   netstat -ano | findstr 6128
   
   # macOS/Linux
   lsof -i :6128
   ```
   - If in use, kill process or use different port

2. **Verify MCP Settings**
   - Open Settings (Ctrl+,)
   - Search "Cosmos DB"
   - Check `cosmosDB.shell.MCP.enabled` is true

3. **Try Different Port**
   ```json
   {
     "cosmosDB.shell.MCP.port": 6129
   }
   ```

4. **Check Firewall**
   - Ensure firewall allows localhost connections
   - Disable temporarily to test

## MCP server issues

### Issue: "MCP server not responding"

**Error:**
```
Cannot connect to MCP server
```

**Causes:**
- MCP server not running
- Wrong port
- Firewall issue
- Connection timeout

**Solutions:**

1. **Verify MCP Server Running**
   ```bash
   curl http://localhost:6128/health
   ```
   - Should return JSON response

2. **Check Configuration**
   - Verify `cosmosDB.shell.MCP.enabled` is true
   - Check correct port in settings

3. **Restart MCP Server**
   - Command Palette (Ctrl+Shift+P)
   - Type "Cosmos DB: Restart MCP Server"

4. **Check Firewall**
   - Ensure localhost:6128 is accessible
   - Disable firewall temporarily to test

### Issue: "MCP port already in use"

**Error:**
```
Cannot bind MCP server to port 6128
```

**Causes:**
- Another process using port
- MCP server already running
- Port conflict

**Solutions:**

1. **Stop Existing MCP Server**
   - Command Palette (Ctrl+Shift+P)
   - Type "Cosmos DB: Stop MCP Server"

2. **Find Process Using Port**
   ```bash
   # Windows
   netstat -ano | findstr 6128
   taskkill /PID <PID> /F
   
   # macOS/Linux
   lsof -i :6128
   kill -9 <PID>
   ```

3. **Use Different Port**
   ```json
   {
     "cosmosDB.shell.MCP.port": 6129
   }
   ```

## Performance issues

### Issue: Shell is slow or unresponsive

**Causes:**
- Large query result set
- Slow network connection
- High system resource usage
- Cosmos DB service issue

**Solutions:**

1. **Limit Query Results**
   ```bash
   > query "SELECT TOP 100 * FROM c"
   ```

2. **Add Filters to Query**
   ```bash
   > query "SELECT * FROM c WHERE c.status = 'active'"
   ```

3. **Check Network**
   - Test bandwidth
   - Switch to different network
   - Check latency with `ping`

4. **Monitor System Resources**
   - Check CPU usage
   - Check memory usage
   - Close unnecessary applications

### Issue: Queries timeout

**Error:**
```
Query timeout. Please try again.
```

**Causes:**
- Large result set
- Complex query
- Slow network
- Cosmos DB service overloaded

**Solutions:**

1. **Increase Timeout Setting**
   - Settings (Ctrl+,)
   - Search "Cosmos DB"
   - Increase `cosmosDB.queryTimeout`

2. **Optimize Query**
   ```bash
   # Instead of SELECT *
   > query "SELECT c.id, c.name FROM c"
   
   # Add filters
   > query "SELECT * FROM c WHERE c.status = 'active'"
   
   # Use TOP
   > query "SELECT TOP 1000 * FROM c"
   ```

3. **Check Cosmos DB**
   - Verify service is operational
   - Check [Azure Status Page](https://status.azure.com)

## Getting help

1. **Check Docs**
   - [Command Reference](command-reference.md)
   - [Quick Start](get-started.md)
   - [Security Best Practices](security.md)

2. **Azure Support**
   - For Cosmos DB service issues: [Create support request](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/newsupportrequest)

## See also

- [Installation Guide](install.md)
- [Command Reference](command-reference.md)
- [Security Best Practices](security.md)
- [Azure Cosmos DB Overview](overview.md)
