---
title: Azure Cosmos DB Shell Security Best Practices
description: Learn security best practices for using Azure Cosmos DB Shell including authentication, credential management, RBAC, and data protection.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: article
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell security best practices

Secure your Azure Cosmos DB Shell deployments with these comprehensive security best practices.

## Authentication methods

### 1. Microsoft Entra ID (recommended)

**Best for:** Development, testing, and production environments

**Advantages:**
- Most secure authentication method
- No credentials stored locally
- Automatic token refresh
- Supports multi-factor authentication (MFA)
- Full audit trail in Azure

**Implementation:**

When you launch Cosmos DB Shell, you're prompted for authentication:

```bash
cosmosdbshell
```

Browser opens automatically for Azure sign-in. Complete the authentication flow, and the shell uses your Entra ID credentials.

**Configuration:**
```bash
cosmosdbshell --auth-method entra-id
```

**Security Benefits:**
- Credentials never stored on disk
- Tokens automatically refreshed
- MFA provides additional protection
- RBAC controls what you can access

### 2. Managed identity (production)

**Best for:** Azure-hosted applications and production environments

**Advantages:**
- No credential management required
- Automatic token refresh
- Works with Azure services (App Service, Functions, AKS)
- No secrets to rotate
- Enhanced security in Azure

**Implementation:**

Enable managed identity on Azure resource:

**For App Service:**
```json
{
  "identity": {
    "type": "SystemAssigned"
  }
}
```

**In Cosmos DB Shell:**
```bash
cosmosdbshell --auth-method managed-identity
```

**Security Configuration:**
1. Enable managed identity on Azure resource
2. Assign Cosmos DB RBAC role to identity
3. Shell automatically uses identity credentials
4. No secrets needed

**Benefits:**
- Zero credential management
- Automatic token rotation
- Audit trail for all operations
- Complies with security standards

### 3. Account key (development only)

**Best for:** Local development and testing only

**Disadvantages:**
- Keys stored locally (security risk)
- Manual rotation required
- Single point of failure
- Cannot use MFA
- Not recommended for production

**Implementation:**

```bash
cosmosdbshell --connection-string "<connection_string>"
```

**Minimal Risk Usage:**
- Development/testing only
- Never commit to source control
- Use environment variables:

```bash
# Set environment variable
export COSMOS_CONNECTION_STRING="your-connection-string"

# Use in script
cosmosdbshell --connection-string "$COSMOS_CONNECTION_STRING"
```

## Credential management

### Best practices

#### 1. Never hardcode credentials

**❌ Bad:**
```bash
#!/bin/bash
cosmosdbshell --connection-string "DefaultEndpointProtocol=https;AccountName=myaccount;..."
```

**✅ Good:**
```bash
#!/bin/bash
cosmosdbshell --connection-string "$COSMOS_CONNECTION_STRING"
```

#### 2. Use environment variables

**Development:**
```bash
# .env file (development only)
COSMOS_CONNECTION_STRING="your-connection-string"

# Load and use
source .env
cosmosdbshell --connection-string "$COSMOS_CONNECTION_STRING"
```

#### 3. Store in secure vaults

**Azure Key Vault:**
```bash
# Store connection string
az keyvault secret set --vault-name myKeyVault \
  --name cosmos-connection-string \
  --value "your-connection-string"

# Retrieve and use
CONNECTION_STRING=$(az keyvault secret show \
  --vault-name myKeyVault \
  --name cosmos-connection-string \
  --query value -o tsv)

cosmosdbshell --connection-string "$CONNECTION_STRING"
```

**Local Development:**
- Use `.env` files (add to `.gitignore`)
- Load via environment variables
- Never commit secrets

#### 4. Rotate keys regularly

**Key Rotation Process:**
1. Generate new key in Azure portal
2. Update applications with new key
3. Delete old key after verification
4. Test all applications work

## RBAC (role-based access control)

### Implement least privilege

**Principle:** Grant minimum permissions needed

**Recommended Roles:**

| Role | Use Case | Permissions |
|------|----------|-------------|
| `Cosmos DB Account Reader` | Read-only access | View data, metadata |
| `Cosmos DB Built-in Data Reader` | Query operations | Execute queries, read documents |
| `Cosmos DB Built-in Data Contributor` | Full data access | Create, read, update, delete documents |

**Assign Roles:**

```bash
# Get resource ID
RESOURCE_ID=$(az cosmosdb show \
  --resource-group myResourceGroup \
  --name myaccount \
  --query id -o tsv)

# Assign role to user
az role assignment create \
  --role "Cosmos DB Built-in Data Reader" \
  --assignee "<user-principal-id>" \
  --scope "$RESOURCE_ID"
```

### Scope access by container

Create custom roles with limited container access:

```bash
# Custom role for specific container
cat > custom-role.json << 'EOF'
{
  "Name": "Cosmos DB Container Reader",
  "Description": "Read-only access to specific container",
  "Actions": [
    "Microsoft.DocumentDB/databaseAccounts/readonlyKeys/action",
    "Microsoft.DocumentDB/databaseAccounts/read/action"
  ],
  "DataActions": [
    "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read"
  ],
  "NotDataActions": []
}
EOF
```

## Data protection

### 1. Encryption in transit

**TLS 1.2+ (Default)**

Cosmos DB Shell uses TLS 1.2 or later:

```bash
# Verify TLS version
cosmosdbshell --tls-version 1.2
```

**Configuration:**
```bash
# Force TLS 1.3
cosmosdbshell --min-tls-version 1.3
```

### 2. Encryption at rest

**Enable Encryption at Rest:**

Azure handles encryption automatically, but verify:

1. Go to Azure portal > Cosmos DB Account
2. Settings > Encryption at Rest
3. Verify encryption is enabled

**Verify:**
```bash
# Check encryption settings
az cosmosdb show --resource-group myResourceGroup \
  --name myaccount \
  --query "encryption.keyWrapperProperties" -o table
```

### 3. Customer-managed keys (CMK)

**For Enhanced Security:**

1. Create Azure Key Vault
2. Generate encryption key
3. Configure Cosmos DB to use CMK

**Setup:**
```bash
# Create key vault
az keyvault create --resource-group myResourceGroup \
  --name myKeyVault

# Create key
az keyvault key create --vault-name myKeyVault \
  --name cosmos-key

# Configure Cosmos DB (via Portal or Terraform)
```

## Network security

### 1. IP firewall

**Enable Firewall:**

1. Azure portal > Cosmos DB Account > Firewall
2. Add your IP address(es)
3. Save changes

**Command Line:**
```bash
# Add IP to firewall
az cosmosdb update --resource-group myResourceGroup \
  --name myaccount \
  --ip-range-filter "203.0.113.0/24"
```

### 2. Virtual endpoints

**Restrict to VNet:**

```bash
# Enable virtual endpoint
az cosmosdb update --resource-group myResourceGroup \
  --name myaccount \
  --enable-virtual-network true
```

### 3. Private endpoints

**For Maximum Network Isolation:**

1. Create private endpoint in Azure portal
2. Configure DNS settings
3. Shell connects via private network

**Benefits:**
- No internet exposure
- Restricted network access
- Compliant with enterprise policies

## MCP server security

### 1. Local-only binding (default)

**Configuration:**
```json
{
  "cosmosDB.shell.MCP.bindToLocalhost": true
}
```

**Why Important:**
- Prevents remote access
- Only local processes can connect
- Reduces attack surface
- **Recommended:** Keep enabled

### 2. Authentication for MCP

**Entra ID Authentication:**
```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.authMethod": "entra-id"
}
```

### 3. MCP port security

**Secure Configuration:**
```json
{
  "cosmosDB.shell.MCP.enabled": true,
  "cosmosDB.shell.MCP.port": 6128,
  "cosmosDB.shell.MCP.bindToLocalhost": true,
  "cosmosDB.shell.MCP.tlsEnabled": true,
  "cosmosDB.shell.MCP.tlsCertPath": "/path/to/cert.pem"
}
```

### 4. Firewall rules for MCP

**Block External Access:**
```bash
# Windows Firewall
netsh advfirewall firewall add rule name="Block MCP" \
  dir=in action=block protocol=tcp localport=6128

# Linux/macOS (iptables)
sudo iptables -A INPUT ! -i lo -p tcp --dport 6128 -j DROP
```

## Audit and monitoring

### 1. Enable audit logging

**Azure Audit Logs:**
```bash
# View audit logs
az monitor activity-log list \
  --resource-group myResourceGroup \
  --query "[?resourceId=='<cosmos-resource-id>']"
```

### 2. Monitor MCP activity

**Enable MCP Logging:**
```json
{
  "cosmosDB.shell.MCP.logLevel": "info"
}
```

**Review Logs:**
- VS Code Output panel
- Application Insights
- Azure Monitor

### 3. Query metrics

**Monitor Query Performance:**
```bash
# Review query metrics
CS > query "SELECT * FROM c" --show-metrics
```

**Metrics to Track:**
- Request units (RU) used
- Execution time
- Item count

## Compliance and standards

### 1. Data residency

**Ensure Data Stays in Region:**

```bash
# Deploy Cosmos DB in specific region
az cosmosdb create --resource-group myResourceGroup \
  --name myaccount \
  --locations regionName="East US" \
  --databases only
```

### 2. Compliance standards

**Supported Compliance:**
- SOC 2 Type II
- ISO 27001
- GDPR
- HIPAA
- PCI DSS

**Verify Compliance:**
1. Azure portal > Compliance
2. Download compliance reports
3. Share with auditors

### 3. Data retention

**Set Time-to-Live (TTL) for Sensitive Data:**

```bash
CS > mkcon sensitive_data -pk /id --ttl 2592000
```

This deletes documents after 30 days.

## Security checklist

- [ ] Use Entra ID for authentication
- [ ] Enable MFA on Azure account
- [ ] Implement least-privilege RBAC
- [ ] Enable IP firewall
- [ ] Use encrypted connections (TLS 1.2+)
- [ ] Enable audit logging
- [ ] Monitor MCP server activity
- [ ] Store credentials in Key Vault
- [ ] Rotate keys regularly
- [ ] Use private endpoints in production
- [ ] Enable encryption at rest
- [ ] Review compliance requirements
- [ ] Document security policies
- [ ] Train users on security practices

## Common security mistakes

### ❌ Storing keys in code

```bash
# DON'T do this
cosmosdbshell --connection-string "DefaultEndpointProtocol=https;AccountKey=abc123"
```

### ✅ Use environment variables

```bash
# DO this
export COSMOS_CONNECTION_STRING="..."
cosmosdbshell --connection-string "$COSMOS_CONNECTION_STRING"
```

### ❌ Using account key in production

```bash
# DON'T use keys in production
cosmosdbshell --auth-method key
```

### ✅ Use managed identity

```bash
# DO use managed identity
cosmosdbshell --auth-method managed-identity
```

### ❌ Leaving MCP open

```json
// DON'T expose MCP publicly
{
  "cosmosDB.shell.MCP.bindToLocalhost": false
}
```

### ✅ Restrict MCP access

```json
// DO restrict to localhost
{
  "cosmosDB.shell.MCP.bindToLocalhost": true
}
```

## Security resources

- [Azure Security Best Practices](/azure/security/fundamentals/best-practices-and-patterns)
- [Cosmos DB Security](/azure/cosmos-db/database-security)
- [Key Vault Best Practices](/azure/key-vault/general/best-practices)
- [RBAC in Cosmos DB](/azure/cosmos-db/role-based-access-control)

## Next steps

- [Installation Guide](install.md) - Get started securely
- [Command Reference](command-reference.md) - Learn available commands
- [Quick Start Guide](get-started.md) - Safe examples
- [MCP Setup Guide](model-context-protocol-setup.md) - Secure MCP configuration

## Support

For security concerns or vulnerability reports:
- [Report Security Issues](https://azure.microsoft.com/support/)
- [Azure Security Advisory](https://msrc.microsoft.com/)

## See also

- [Azure Cosmos DB Overview](overview.md)
- [Azure Cosmos DB Security](security.md)
- [Compliance in Azure](../compliance.md)
