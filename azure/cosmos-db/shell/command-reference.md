---
title: Azure Cosmos DB Shell Command Reference
description: Complete reference for all Azure Cosmos DB Shell commands with syntax, options, and examples.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: reference
ms.date: 05/04/2024
---

# Azure Cosmos DB Shell command reference

Complete reference guide for all Azure Cosmos DB Shell commands.

## Navigation commands

Commands for navigating databases and containers.

### `cd` - Change directory

Navigate to a database or container.

**Syntax:**
```bash
cd <path>
```

**Examples:**
```bash
# Navigate to a database
CS > cd mydb
# Navigate to a container
CS > cd users
# Go back one level
CS > cd ..
# Navigate to root
CS > cd /
CS >
```

### `ls` - List

List databases or containers in current location.

**Syntax:**
```bash
ls [options]
```

**Options:**
- `-l`: Long format with details
- `-h`: Human-readable sizes

**Examples:**
```bash
# List databases
CS > ls

# List containers with details
CS > ls -l

# List containers in human-readable format
CS > ls -h
```

### `pwd` - Print working directory

Show the current path.

**Syntax:**
```bash
pwd
```

**Examples:**
```bash
CS > pwd
/mydb/users
```

### `endpoint` - Show endpoint

Display the current Cosmos DB account endpoint.

**Syntax:**
```bash
endpoint
```

**Examples:**
```bash
CS > endpoint
https://myaccount.documents.azure.com:443/
```

## Database management commands

Commands for creating and managing databases.

### `mkdb` - Make database

Create a new database.

**Syntax:**
```bash
mkdb <database_name> [options]
```

**Options:**
- `--throughput <RU/s>`: Provisioned throughput
- `--autoscale <max_RU/s>`: Autoscale max throughput

**Examples:**
```bash
# Create database with default settings
CS > mkdb mydb

# Create database with provisioned throughput
CS > mkdb mydb --throughput 400

# Create database with autoscale
CS > mkdb mydb --autoscale 4000
```

### `rmdb` - Remove database

Delete a database and all its contents.

**Syntax:**
```bash
rmdb <database_name> [--force]
```

**Options:**
- `--force`: Skip confirmation prompt

**Examples:**
```bash
# Delete database (with confirmation)
CS > rmdb tempdb

# Delete database without confirmation
CS > rmdb tempdb --force
```

## Container management commands

Commands for creating and managing containers.

### `mkcon` - Make container

Create a new container.

**Syntax:**
```bash
mkcon <container_name> -pk <partition_key> [options]
```

**Options:**
- `-pk`, `--partition-key`: Partition key path (required)
- `--throughput <RU/s>`: Provisioned throughput
- `--autoscale <max_RU/s>`: Autoscale max throughput
- `--ttl <seconds>`: Time-to-live in seconds
- `--unique-key <path>`: Unique constraint path

**Examples:**
```bash
# Create container with partition key
CS > mkcon users -pk /id

# Create with provisioned throughput
CS > mkcon users -pk /id --throughput 400

# Create with autoscale
CS > mkcon users -pk /id --autoscale 4000

# Create with TTL
CS > mkcon users -pk /id --ttl 86400

# Create with unique key
CS > mkcon users -pk /id --unique-key /email
```

### `rmcon` - Remove container

Delete a container.

**Syntax:**
```bash
rmcon <container_name> [--force]
```

**Options:**
- `--force`: Skip confirmation prompt

**Examples:**
```bash
# Delete container (with confirmation)
CS > rmcon tempcontainer

# Delete without confirmation
CS > rmcon tempcontainer --force
```

## Data manipulation commands

Commands for querying and managing data.

### `query` - Execute query

Execute a SQL query against a container.

**Syntax:**
```bash
query "<SQL_query>" [options]
```

**Options:**
- `--partition-key <value>`: Specify partition key for targeted query
- `--max-items <count>`: Maximum items to return
- `--continuation-token <token>`: Continuation token for pagination

**Examples:**
```bash
# Query all documents
CS > query "SELECT * FROM c"

# Query with filter
CS > query "SELECT * FROM c WHERE c.status = 'active'"

# Query with partition key (faster)
CS > query "SELECT * FROM c WHERE c.id = 'user1'" --partition-key user1

# Query with limit
CS > query "SELECT TOP 10 * FROM c"

# Aggregate query
CS > query "SELECT c.status, COUNT(*) as count FROM c GROUP BY c.status"

# Join query
CS > query "SELECT c.id, c.name, o.orderId FROM c JOIN o IN c.orders"
```

### `create` - Create document

Insert a new document.

**Syntax:**
```bash
create item <JSON_document>
```

**Examples:**
```bash
# Create simple document
CS > create item {"id": "user1", "name": "Alice"}

# Create nested document
CS > create item {"id": "user2", "name": "Bob", "address": {"city": "Seattle", "country": "USA"}}

# Create with array
CS > create item {"id": "user3", "name": "Charlie", "tags": ["vip", "premium"]}
```

### `update` - Update document

Update an existing document.

**Syntax:**
```bash
update <JSON_document>
```

**Examples:**
```bash
# Update document
CS > update {"id": "user1", "name": "Alice", "status": "active"}

# Partial update (with JSON merge semantics)
CS > update {"id": "user1", "status": "inactive"}
```

### `rm` - Remove document

Delete a document.

**Syntax:**
```bash
rm <document_id> [--partition-key <value>]
```

**Options:**
- `--partition-key <value>`: Specify partition key for faster deletion

**Examples:**
```bash
# Delete document
CS > rm user1

# Delete with partition key
CS > rm user1 --partition-key user1
```

### `get` - Get document

Retrieve a specific document by ID.

**Syntax:**
```bash
get <document_id> [--partition-key <value>]
```

**Options:**
- `--partition-key <value>`: Specify partition key for faster retrieval

**Examples:**
```bash
# Get document
CS > get user1

# Get with partition key
CS > get user1 --partition-key user1
```

## Utility commands

General utility commands.

### `jq` - JSON processing

Process JSON output using jq syntax.

**Syntax:**
```bash
<command> | jq '<jq_filter>'
```

**Examples:**
```bash
# Pretty print JSON
CS > query "SELECT * FROM c" | jq .

# Extract specific fields
CS > query "SELECT * FROM c" | jq '.[] | {id, name}'

# Filter results
CS > query "SELECT * FROM c" | jq '.[] | select(.status == "active")'

# Transform data
CS > query "SELECT * FROM c" | jq '[.[] | {id, name}]'
```

### `echo` - Display text

Output text to console.

**Syntax:**
```bash
echo "<text>"
```

**Examples:**
```bash
CS > echo "Starting operations..."
Starting operations...
```

### `help` - Show help

Display help information.

**Syntax:**
```bash
help [command]
```

**Examples:**
```bash
# General help
CS > help

# Help for specific command
CS > help query

# Help for container commands
CS > help mkcon
```

### `version` - Show version

Display the shell version.

**Syntax:**
```bash
version
```

**Examples:**
```bash
CS > version
Azure Cosmos DB Shell 1.0.213-preview
```

### `exit` - Exit shell

Exit the Cosmos DB Shell.

**Syntax:**
```bash
exit
```

**Examples:**
```bash
CS > exit
```

## Command chaining and piping

Commands can be chained using pipes (`|`) and redirects.

### Pipe to `jq`
```bash
CS > query "SELECT * FROM c" | jq '.[]'
```

### Pipe to file
```bash
CS > query "SELECT * FROM c" > output.json
```

### Pipe to external command
```bash
CS > query "SELECT * FROM c" | grep "active"
```

### Combine multiple operations
```bash
CS > query "SELECT * FROM c" | jq '.[] | select(.status == "active")' | wc -l
```

## JSON path expressions

Use JSON path expressions in queries for advanced filtering.

**Examples:**
```bash
# Query nested properties
CS > query "SELECT c.address.city FROM c"

# Array operations
CS > query "SELECT * FROM c WHERE ARRAY_CONTAINS(c.tags, 'vip')"

# String functions
CS > query "SELECT * FROM c WHERE STARTSWITH(c.name, 'A')"

# Numeric functions
CS > query "SELECT * FROM c WHERE c.age > 30"
```

## Connection commands

### `connect` - Connect to account

Switch to a different Cosmos DB account.

**Syntax:**
```bash
connect <connection_string|endpoint> [--auth-method <method>]
```

**Auth Methods:**
- `entra-id`: Microsoft Entra ID (default, recommended)
- `managed-identity`: Managed Identity
- `key`: Account key

**Examples:**
```bash
# Connect with Entra ID
CS > connect https://myaccount.documents.azure.com:443/ --auth-method entra-id

# Connect with account key
CS > connect DefaultEndpointProtocol=https;AccountName=myaccount;... --auth-method key
```

## Best practices

- **Use partition keys** for faster queries
- **Limit result sets** with TOP or --max-items
- **Use projections** to reduce data transfer
- **Chain commands** with pipes for complex operations
- **Use jq** for JSON processing and formatting
- **Save complex queries** in script files

## See also

- [Quick Start Guide](get-started.md)
- [Troubleshooting Guide](troubleshooting.md)
- [SQL Query Syntax](../how-to-query-container.md)
- [Azure Cosmos DB Query Reference](../query-cheat-sheet.md)
