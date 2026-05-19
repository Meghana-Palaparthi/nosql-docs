---
title: Migrate MongoDB using Azure DocumentDB Migration Extension
description: Use Azure DocumentDB Migration Extension to migrate from existing MongoDB instances to Azure DocumentDB online.
author: sandeepsnairms
ms.author: sandnair
ms.custom: ignite-2025
ms.topic: how-to
ms.date: 05/18/2026
# CustomerIntent: As a database owner, I want to use the Azure DocumentDB Migration Extension so that I can migrate an existing dataset to Azure DocumentDB.
---

# Migrate MongoDB to Azure DocumentDB online using Azure DocumentDB migration extension

In this tutorial, you use the Azure DocumentDB Migration Extension in Visual Studio Code to create and manage migration jobs from an on-premises or cloud instance of MongoDB to Azure DocumentDB. This extension provides a developer-friendly interface for performing migrations without service interruptions. The extension eliminates the need for additional infrastructure and offers secure connectivity, zero-cost usage, and granular control over which databases and collections to migrate.

The focus of this article is on using the extension's integrated workflow to simplify migration steps directly within Visual Studio Code. This approach is ideal for scenarios where you want a streamlined, managed experience with minimal complexity and maximum reliability.

## Prerequisites

[!INCLUDE[Prerequisite - Existing cluster](includes/prerequisite-existing-cluster.md)]

- Install the [Azure DocumentDB Migration Extension](https://aka.ms/azure-documentdb-migration-extension) on your machine. This automatically installs its prerequisite, the [DocumentDB for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb) extension.

Before starting the migration, prepare your Azure DocumentDB account and your existing MongoDB instance for migration.

**MongoDB instance (source)**

- Complete the premigration assessment to determine if there are incompatibilities and warnings between your source instance and target account.
- Add a user with `readAnyDatabase` and `clusterMonitor` permissions, unless one already exists. You use this credential while creating migration jobs in the extension.

**Azure DocumentDB (target)**

- Gather the Azure DocumentDB [account's credentials](./quickstart-portal.md#get-cluster-credentials).
- Ensure user has `createCollection`, `dropCollection`, `createIndex`, `insert`, and `listCollections` permissions.

### Minimum required permissions

Use the following minimum roles to create and run migration jobs.

| Minimum role | Scope | Applies to connectivity mode | Purpose |
| --- | --- | --- | --- |
| Reader | Subscription | Public and private | List subscriptions and resource groups. Required for each migration job. |
| Azure Database Migration Service Contributor | Resource group | Public and private | Create Azure Database Migration Service (DMS). You don't need to create a new DMS for every migration. One DMS per region is enough. |
| Contributor | Subscription | Public and private | Register DMS in the subscription. This is a one-time activity and can be delegated to another user. |
| User Access Administrator | Virtual network | Private only | Assign the Network Contributor role to the DMS object principal. This is a one-time activity per virtual network and can be delegated to another user. |
| Contributor | Azure DocumentDB | Public and private | Trigger the migration job. |

For provider registration details, see [Register Microsoft.DataMigration resource provider in your subscription](#register-microsoftdatamigration-resource-provider-in-your-subscription).
  
> [!IMPORTANT]
> Microsoft Entra ID authentication isn't currently supported in migration jobs. Use native DocumentDB authentication.

## Perform the migration

For planning guidance on migration sizing, speed, and cutover, see [Migration best practices](./migration-options.md#migration-best-practices).

### Connect to source

1. Open the **DocumentDB for VS Code** extension. 
1. Add the MongoDB server you want to migrate to the **Document DB Connections** list.
1. Select **Add New Connection**.
1. On the navigation bar, select **Connection String**.
1. Paste your connection string: `mongodb://<YOUR_USERNAME>:<YOUR_PASSWORD>@localhost:10260/?tls=true&tlsAllowInvalidCertificates=true&authMechanism=SCRAM-SHA-256`
1. From the DocumentDB Connections, select the connection and expand it to connect.

### Invoke the Migration Extension

You can invoke the Migration Extension from the DocumentDB Connections.

1. Right-click on an expanded (connected) connection. 

1. Select **Data Migration** from the context menu.

    :::image type="content" source="media/how-to-migrate-vs-code-extension/context-menu.png" alt-text="Screenshot of the context menu in Visual Studio Code."  :::

1. From the command palette, select **Migration to Azure DocumentDB**.
    :::image type="content" source="media/how-to-migrate-vs-code-extension/command-palette-migration-tools.png" alt-text="Screenshot of the command palette listing migration tools in Visual Studio Code.":::

1. Then select **Migrate to Azure DocumentDB**.
    :::image type="content" source="media/how-to-migrate-vs-code-extension/command-palette-migrate.png" alt-text="Screenshot of the command palette showing migration option in Visual Studio Code.":::

1. A migration wizard guides you through the process.

### Create a migration job

A migration job is used to migrate a group of collections from the source to the destination Azure DocumentDB. The create migration job wizard has six steps.

#### Step 1: Create job

In this step, you provide the basic details for the job.

- **Job Name**: Provide a user-friendly name to identify the migration job.

- **Migration Mode**: Select the migration mode that's most appropriate for your use case.
    - **Online** migration copies collection data, ensuring updates are also replicated during the process. This method is advantageous with minimal downtime, allowing continuous operations for business continuity. Use this option when ongoing operations are crucial, and reducing downtime is a priority.
    - **Offline** migration captures a snapshot of the database at the beginning, offering a simpler and predictable approach. It works well when using a static copy of the database is acceptable, and real-time updates aren't essential.

    > [!IMPORTANT] 
    > To ensure successful online migrations from MongoDB, ChangeStream must be enabled on the source MongoDB server. Without ChangeStream, any modifications made to the data after the initial migration aren't captured. Therefore, use the online migration mode only if ChangeStream is enabled on your source MongoDB server.

- **Connectivity**: Depending on your organization's security mandate and network setup, choose from **Public** and **Private**. 
    - Use **Public** when the source and target servers are accessible over the internet via public IPs. It enables support for services that require external accessibility.
    - Use **Private** when either the source or target servers are accessible exclusively via private IPs within a virtual network. It enhances security by eliminating exposure to the public internet.

Select **Next** to continue.

:::image type="content" source="media/how-to-migrate-vs-code-extension/create-job.png" alt-text="Screenshot of the create job step in wizard." lightbox="media/how-to-migrate-vs-code-extension/create-job.png":::

#### Step 2: Select target

In this step, you select an existing Azure DocumentDB account and provide its connection string.

1. Select the subscription, resource group, and Azure DocumentDB account from the dropdowns.

1. Provide the connection string to the Azure DocumentDB account.

1. Ensure the IP listed in the screen is allowed on the Azure DocumentDB firewall.

1. Select **Next** to continue.

:::image type="content" source="media/how-to-migrate-vs-code-extension/select-target.png" alt-text="Screenshot of the select target step in wizard." lightbox="media/how-to-migrate-vs-code-extension/select-target.png":::

#### Step 3: Select Database Migration Service(DMS)

Azure Database Migration Service is a service that migrates data to and from Azure data platforms by using cloud infrastructure for data transfer, instead of relying on local resources. Choose an existing Azure Database Migration Service instance from the dropdown or select **Create DMS** to create a new migration service.

> [!IMPORTANT]
> Make sure that the [Microsoft.DataMigration resource provider is registered in your subscription](#register-microsoftdatamigration-resource-provider-in-your-subscription). You only need to do it once per subscription.

Select **Next** to continue.

:::image type="content" source="media/how-to-migrate-vs-code-extension/select-data-migration-service.png" alt-text="Screenshot of the select Database Migration Service step in wizard." lightbox="media/how-to-migrate-vs-code-extension/select-data-migration-service.png":::

#### Step 4: Configure connectivity

This screen depends on the connectivity mode you chose in Step 1. For details on each mode, network architecture, and troubleshooting, see [Migration scenarios](#migration-scenarios) and [Review connectivity](#review-connectivity).

##### Public connectivity

In public connectivity, the migration job connects to your source and target using public internet. To enable communication, you're required to update the source and target firewalls. To enable communication from the DMS servers, add the IP addresses listed in the screen to the source and target firewalls. For more information about the networking model, see [Public connectivity](#public-connectivity-1). For firewall configuration guidance, see [configure Azure DocumentDB cluster](./how-to-configure-firewall.md) firewall.



:::image type="content" source="media/how-to-migrate-vs-code-extension/configure-connectivity-public.png" alt-text="Screenshot of the  public connectivity configuration step in wizard." lightbox="media/how-to-migrate-vs-code-extension/configure-connectivity-public.png":::

##### Private connectivity

In private connectivity, the migration job runs within its virtual network. To securely communicate with your virtual networks, we use virtual network peering. For more information about the networking model, see [Private connectivity](#private-connectivity-1).

1. The tool allows for peering with two virtual networks, one for the source and the other for the target. Depending on your network configuration, select the subscription, resource group, and virtual networks from the dropdowns.

1. In the **DMS Configuration** section, select a CIDR range that doesn't conflict with your virtual networks.

1. Run the PowerShell scripts provided on the screen to enable virtual network integration.

1. Select **Next** to continue.

:::image type="content" source="media/how-to-migrate-vs-code-extension/configure-connectivity-private.png" alt-text="Screenshot of the  private connectivity configuration step in wizard." lightbox="media/how-to-migrate-vs-code-extension/configure-connectivity-private.png":::

#### Step 5: Select collections

In this step, you select the collections to be included in the migration job. Select from the list of collections by using the search options provided. Collections that already exist in the target are automatically marked **Yes** in the **Exists in Target** column.

> [!TIP]
> Make sure to select all collections you want to include as the collections list can't be added once the migration job is created.

Select **Next** to continue.

:::image type="content" source="media/how-to-migrate-vs-code-extension/select-collections.png" alt-text="Screenshot of the select collections step in wizard." lightbox="media/how-to-migrate-vs-code-extension/select-collections.png":::

#### Step 6: Confirm and start

Review the migration job details before selecting **Start Migration**. If the details need to be updated, use the **Edit Details** button.

Once the migration job is successfully created, you're automatically redirected to the **View Existing Jobs** page

> [!TIP]
> The data migration tasks are run on Azure Database Migration Service. Therefore, you aren't required to be connected to the source and target environments during the data migration. The status is updated on the dashboard at frequent intervals.

### Monitor existing migration jobs

Use the **View Existing Jobs** tab to monitor the migration status of initialized jobs. The jobs are listed based on the selected DMS. Use the **Change DMS** button to change your selection.

The status is automatically updated at frequent intervals. Offline jobs automatically complete once the selected collection snapshots are copied to target. However, the online migrations need to be manually cut over.

You can **Pause** a migration job to temporarily halt it at a logical point. When you're ready to continue, select **Resume** to pick up from where the job stopped.

:::image type="content" source="media/how-to-migrate-vs-code-extension/monitor-job.png" alt-text="Screenshot of the view existing jobs screen." lightbox="media/how-to-migrate-vs-code-extension/monitor-job.png" :::

To view the collection-wise status, select a row from the table.

:::image type="content" source="media/how-to-migrate-vs-code-extension/monitor-collections-offline.png" alt-text="Screenshot showing collection-wise status for offline migration." lightbox="media/how-to-migrate-vs-code-extension/monitor-collections-offline.png":::

### Monitor online migrations

Online migrations, unlike offline migrations, don't automatically complete. Instead, they run continuously until they're manually finalized by selecting **Cutover**.

To complete the online migration, follow these steps in the given order:

1. The **Cutover** button is enabled once the initial data load is completed for all collections. At this stage, the job is in the replication phase, continuously copying updates from the source instance to the target instance to keep it up-to-date with the latest changes.

1. When ready to perform the migration cutover, stop all incoming transactions to the source collections being migrated.

1. The **Time Since Last Change** shows the time gap between the last update and current time.

1. Monitor the replication changes in the table and wait until the **Replication Changes Played** metric stabilizes. A stable **Replication Changes Played** metric indicates that all updates from the source are successfully copied to the target.

1. Select **Cutover** when the replication gap is minimal for all collections and the **Replication Changes Played** metric is stable.

1. Manually validate that the row count is the same between the source and target collections.

> [!NOTE]
> Performing the cutover operation without validating that the source and target are synced can result in data loss.

:::image type="content" source="media/how-to-migrate-vs-code-extension/monitor-collections-online.png" alt-text="Screenshot showing collection-wise status for online migration" lightbox="media/how-to-migrate-vs-code-extension/monitor-collections-online.png":::

## Migration scenarios

The Azure DocumentDB Migration Extension supports several source environments, including MongoDB instances running in Azure, on-premises datacenters, and other cloud providers. The extension provides flexible connectivity options to accommodate different network configurations and security requirements.

### Supported source environments

The following table summarizes the supported migration sources:

| Source environment | Description |
| --- | --- |
| Within Azure | MongoDB instances running on Azure Virtual Machines or other Azure-hosted services |
| On-premises | MongoDB servers running in your local data center or private infrastructure |
| Other cloud providers | MongoDB instances hosted on other cloud platforms |

### Public connectivity

In public connectivity mode, the Azure Database Migration Service (DMS) connects to your source and target servers over the public internet. DMS provides static IP addresses that you add to the firewall allow lists on both the source and target servers. DMS uses a shared public virtual network for all migrations within a given region. While this virtual network is shared across customers, each migration job runs on its own isolated private worker node to ensure job-level isolation.

Use public connectivity when:
- Your source and target servers are accessible through public IP addresses.
- Your organization's security policies allow connections over the public internet.
- You need a simpler setup without virtual network configuration.

:::image type="content" source="media/how-to-migrate-vs-code-extension/migrate-using-public-endpoint.png" alt-text="Diagram showing network architecture for public connectivity." lightbox="media/how-to-migrate-vs-code-extension/migrate-using-public-endpoint.png" :::

To enable public connectivity:

1. In the migration wizard, select **Public** as the connectivity mode.

1. Note the static IP addresses displayed in the wizard.

1. Add these IP addresses to the firewall allow list on your source MongoDB server.

1. Add these IP addresses to the [Azure DocumentDB firewall](./how-to-configure-firewall.md).

### Private connectivity

In private connectivity mode, DMS provisions a dedicated private virtual network for each migration job and peers it with your source and target virtual networks. This means every job gets both isolated worker nodes and an isolated network, ensuring that no traffic crosses between jobs and no shared network paths exist between customers.

The extension supports up to two virtual networks:
- **Source virtual network**: The virtual network where your source MongoDB server is accessible.
- **Target virtual network**: The virtual network where your Azure DocumentDB cluster is accessible.

Use private connectivity when:
- Your source or target servers aren't accessible over the public internet.
- Your organization requires all traffic to flow through private networks.
- You need to avoid public internet exposure.

#### From other cloud providers or on-premises

Use your preferred VPN tools to set up network connectivity between Azure and your source environment in another cloud or on-premises. The following topologies are supported depending on your network architecture.

##### Single virtual network

In this topology, the VPN/ExpressRoute gateway and source workloads are in the same virtual network. DMS peers directly with this virtual network and uses `useRemoteGateways` to reach the on-premises or other-cloud source.

:::image type="content" source="media/how-to-migrate-vs-code-extension/private-endpoint-single-virtual-network.png" alt-text="Diagram showing single virtual network topology for private connectivity from on-premises or other cloud." lightbox="media/how-to-migrate-vs-code-extension/private-endpoint-single-virtual-network.png" :::

##### Hub-direct

In this topology, the VPN/ExpressRoute gateway is in a dedicated hub virtual network. DMS peers directly with the hub, so it can use the gateway natively via `useRemoteGateways`.

:::image type="content" source="media/how-to-migrate-vs-code-extension/private-endpoint-hub-direct.png" alt-text="Diagram showing hub-direct topology for private connectivity from on-premises or other cloud." lightbox="media/how-to-migrate-vs-code-extension/private-endpoint-hub-direct.png" :::

##### Hub-spoke with TCP proxy

If DMS can only peer to a spoke virtual network and direct hub peering isn't possible, deploy a [MongoDB migration proxy](https://aka.ms/mongodbmigrationproxy) VM in the DMS-peered spoke. The proxy forwards traffic from DMS to the source MongoDB server through the hub's VPN/ExpressRoute gateway.

:::image type="content" source="media/how-to-migrate-vs-code-extension/private-endpoint-hub-spoke-proxy.png" alt-text="Diagram showing hub-spoke topology with TCP proxy for private connectivity from on-premises or other cloud." lightbox="media/how-to-migrate-vs-code-extension/private-endpoint-hub-spoke-proxy.png" :::

> [!NOTE]
> DMS creates an ephemeral virtual network that peers to your networks. In enterprise hub-and-spoke topologies, virtual network peering is **non-transitive**, DMS can't reach a hub VPN gateway through a spoke virtual network automatically. You need a routing mechanism such as direct hub peering or a TCP proxy deployed in the DMS-peered virtual network. For validation steps, see [Review connectivity](#review-connectivity).

#### From a private endpoint in Azure

Set up private endpoints for the source and target virtual networks.

:::image type="content" source="media/how-to-migrate-vs-code-extension/private-endpoint-source-in-azure.png" alt-text="Diagram showing network architecture for private connectivity within Azure." lightbox="media/how-to-migrate-vs-code-extension/private-endpoint-source-in-azure.png" :::


To enable private connectivity:

1. In the migration wizard, select **Private** as the connectivity mode.

1. Select the subscription, resource group, and virtual network for your source environment.

1. Select the subscription, resource group, and virtual network for your target environment.

1. In the **DMS Configuration** section, select a CIDR range that doesn't conflict with your existing virtual networks.

1. Run the PowerShell scripts provided in the wizard to enable virtual network integration and peering.

> [!IMPORTANT]
> A single virtual network can support only one active migration job at a time  when connectivity mode is private. To run multiple concurrent jobs, use different virtual networks for each job.

## Review connectivity

Before starting a migration job or when diagnosing a failed one, validate that your network is correctly configured for DMS. DMS creates a **dedicated ephemeral virtual network** for each migration job that is **purged as soon as the job completes or fails**. You can't inspect the DMS virtual network afterward, so proactive validation is essential.

### Validation prerequisites

| Tool | Purpose |
|---|---|
| Azure CLI (`az`) | Query Azure networking resources |
| PowerShell or Bash | Run validation scripts |
| `nc` (netcat) or `Test-NetConnection` | Test TCP connectivity |
| Network Contributor or Reader RBAC roles | Read access on relevant virtual networks |

### Public connectivity validation

In public mode, DMS connects to source and target over the public internet using static IPs displayed in the migration wizard.

#### Validate source firewall

Verify that the source MongoDB server allows inbound connections from the DMS static IPs:

```powershell
# PowerShell
Test-NetConnection -ComputerName <SOURCE_PUBLIC_IP_OR_HOSTNAME> -Port 27017
```

```bash
# Bash / Linux
nc -zv <SOURCE_PUBLIC_IP_OR_HOSTNAME> 27017
```

**Expected:** `TcpTestSucceeded : True` or `Connection succeeded`.

#### Validate target firewall

Verify the DMS IPs are added to the Azure DocumentDB firewall:

```bash
az cosmosdb show \
  --subscription "<SUBSCRIPTION>" \
  -g "<RESOURCE_GROUP>" \
  -n "<DOCUMENTDB_ACCOUNT>" \
  --query "publicNetworkAccess"
```

Test target connectivity:

```powershell
Test-NetConnection -ComputerName <DOCUMENTDB_HOSTNAME> -Port 10260
```

#### Validate DNS resolution

```powershell
Resolve-DnsName <SOURCE_HOSTNAME>
Resolve-DnsName <DOCUMENTDB_HOSTNAME>
```

Verify that both hostnames resolve to **public** IPs. A private IP response indicates a private DNS override.

#### Public connectivity checklist

| # | Check | Command | Expected |
|---|---|---|---|
| 1 | Source firewall allows DMS IPs | `Test-NetConnection <source> -Port 27017` | Succeeded |
| 2 | Target firewall allows DMS IPs | Verify in portal or CLI | DMS IPs in allow list |
| 3 | DNS resolves to public IPs | `Resolve-DnsName <hostname>` | Public IP returned |
| 4 | Source MongoDB is listening | `mongosh <connection_string>` | Connected |
| 5 | Target DocumentDB is reachable | `Test-NetConnection <target> -Port 10260` | Succeeded |

### Private connectivity validation

Private connectivity validation has four phases: validate infrastructure, build a simulation environment, test connectivity from the simulation, and clean up.

#### Phase 1: Validate infrastructure

##### Source in Azure

Validate both source and target private endpoints:

```bash
# Check source private endpoints
az network private-endpoint list \
  --subscription "<SUBSCRIPTION>" -g "<SOURCE_RG>" \
  --query "[].{Name:name,Subnet:subnet.id,Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}" \
  -o table

# Check target private endpoints
az network private-endpoint list \
  --subscription "<SUBSCRIPTION>" -g "<TARGET_RG>" \
  --query "[].{Name:name,Subnet:subnet.id,Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}" \
  -o table
```

**Expected:** Status = `Approved` for both.

Check for CIDR overlap between your virtual networks and the DMS CIDR:

```bash
echo "=== Source VNet ==="
az network vnet show --subscription "<SUB>" -g "<SOURCE_RG>" -n "<SOURCE_VNET>" \
  --query "addressSpace.addressPrefixes" -o tsv

echo "=== Target VNet ==="
az network vnet show --subscription "<SUB>" -g "<TARGET_RG>" -n "<TARGET_VNET>" \
  --query "addressSpace.addressPrefixes" -o tsv
```

**Expected:** The DMS CIDR you select in the wizard must not overlap with either virtual network.

Validate that NSGs on source and target subnets allow traffic from the DMS virtual network CIDR:

```bash
az network nsg list \
  --subscription "<SUB>" -g "<SOURCE_RG>" \
  --query "[].{Name:name,Rules:securityRules[?direction=='Inbound'].{Rule:name,Priority:priority,Access:access,Source:sourceAddressPrefix,Port:destinationPortRange}}" \
  -o json
```

**Verify:** No `Deny` rules block the DMS virtual network CIDR on MongoDB ports (default 27017) or DocumentDB ports (default 10260).

Validate private DNS zone configuration:

```bash
# Check private DNS zones
az network private-dns zone list \
  --subscription "<SUB>" \
  --query "[?contains(name,'mongo') || contains(name,'cosmos') || contains(name,'documentdb')].{Zone:name,RecordSets:numberOfRecordSets}" \
  -o table

# Verify DNS zone is linked to relevant VNets
az network private-dns link vnet list \
  --subscription "<SUB>" -g "<DNS_ZONE_RG>" -z "<ZONE_NAME>" \
  --query "[].{Link:name,VNet:virtualNetwork.id,AutoReg:registrationEnabled}" \
  -o table
```

**Azure-to-Azure checklist:**

| # | Check | Expected |
|---|---|---|
| 1 | Source private endpoint status | `Approved` |
| 2 | Target private endpoint status | `Approved` |
| 3 | No CIDR overlap between your virtual networks and the DMS CIDR | Distinct address ranges |
| 4 | NSGs allow DMS CIDR on required ports | No blocking `Deny` rules |
| 5 | Private DNS zones linked to your virtual networks | Zones linked, records resolve |

##### Source on-premises or other cloud

Validate VPN/ExpressRoute tunnel:

```bash
# For VPN Gateway
az network vpn-connection list \
  --subscription "<SUB>" -g "<GATEWAY_RG>" \
  --query "[].{Name:name,Status:connectionStatus,IngressBytes:ingressBytesTransferred,EgressBytes:egressBytesTransferred}" \
  -o table
```

**Expected:** `connectionStatus` = `Connected`, bytes increasing over time.

```bash
# For ExpressRoute
az network express-route show \
  --subscription "<SUB>" -g "<GATEWAY_RG>" -n "<ER_CIRCUIT_NAME>" \
  --query "{Name:name,State:serviceProviderProvisioningState,PeeringState:peerings[0].state}" \
  -o table
```

The DMS virtual network CIDR must be advertised to the remote network, validate BGP route advertisement:

```bash
az network vnet-gateway list-advertised-routes \
  --subscription "<SUB>" -g "<GATEWAY_RG>" -n "<VPN_GATEWAY_NAME>" \
  --peer <REMOTE_BGP_PEER_IP> -o table
```

Verify that the DMS virtual network CIDR appears in the output. If it's missing, set `allowGatewayTransit=true` on your gateway virtual network peering.

Validate routing depending on your topology:

- **Single virtual network (no hub-spoke):** Verify `allowGatewayTransit=true` on your virtual network peering.
- **Hub-direct:** Same as single virtual network, but check the hub virtual network peering.
- **TCP proxy:** If DMS can only peer to a spoke and direct hub peering isn't possible, deploy a [MongoDB migration proxy](https://aka.ms/mongodbmigrationproxy) VM in the DMS-peered virtual network. The proxy forwards traffic from DMS to the source MongoDB server through the existing VPN/ExpressRoute path. Configure the DMS migration job to use the proxy VM's private IP and listen port as the source endpoint.

```bash
# Check VNet peering configuration
az network vnet peering list \
  --subscription "<SUB>" -g "<YOUR_VNET_RG>" --vnet-name "<YOUR_VNET>" \
  --query "[].{Name:name,State:peeringState,AllowGatewayTransit:allowGatewayTransit}" \
  -o table
```

> [!IMPORTANT]
> Virtual network peering is **non-transitive**. If DMS peers to a spoke virtual network, it can't automatically reach the hub's VPN gateway. For DMS traffic to reach on-premises/other-cloud sources, you need either direct hub peering (so DMS can use the VPN gateway natively via `useRemoteGateways`) or a [TCP proxy](https://aka.ms/mongodbmigrationproxy) deployed in the DMS-peered virtual network that forwards traffic to the source through your existing network path.

The on-premises or other-cloud network must have return routes for the DMS virtual network CIDR, validate remote-side return routes. For AWS:

```bash
aws ec2 describe-route-tables --region <REGION> \
  --route-table-ids <ROUTE_TABLE_ID> \
  --query "RouteTables[0].Routes[?GatewayId!=null && starts_with(GatewayId,'vgw')].{Destination:DestinationCidrBlock,Gateway:GatewayId,State:State}" \
  --output table
```

For on-premises routers, verify the DMS CIDR appears in the BGP routing table.

The remote side must also allow **inbound** traffic from the DMS virtual network CIDR on the required ports (default: 27017 for MongoDB).

> [!NOTE]
> DMS creates a new virtual network with a dynamic CIDR for each migration job. Use the specific DMS CIDR shown in the migration wizard when configuring remote firewall rules.

**On-premises/other cloud checklist:**

| # | Check | Expected |
|---|---|---|
| 1 | VPN/ExpressRoute tunnel status | `Connected`, bytes flowing |
| 2 | Azure advertises DMS CIDR via BGP | DMS CIDR in advertised routes |
| 3 | Virtual network peering allows gateway transit | `allowGatewayTransit=true` |
| 4 | Remote side has return route for DMS CIDR | Route present in route table |
| 5 | Remote firewall/SG allows DMS CIDR | Inbound rule on required ports |
| 6 | Target private endpoint status | `Approved` |

#### Phase 2: Build simulation environment

DMS creates an ephemeral virtual network for each migration job that is purged on completion or failure. To test connectivity from the DMS network position, create a simulation virtual network with the same CIDR, peer it to your networks, and deploy a test VM.

> [!NOTE]
> Testing from a VM in your existing gateway or spoke virtual network only confirms that your VPN works. It doesn't prove that traffic originating from the DMS CIDR can be routed correctly. The simulation virtual network replicates the exact network position of DMS, so connectivity tests from it are representative of actual migration traffic.

##### Create a test virtual network

Use the same CIDR you provided (or plan to provide) in the DMS migration wizard:

```bash
SUBSCRIPTION="<YOUR_SUBSCRIPTION>"
TEST_RG="dms-network-test-rg"
LOCATION="<SAME_REGION_AS_DMS>"
DMS_CIDR="<DMS_CIDR_FROM_WIZARD>"
TEST_VNET="dms-test-vnet"
TEST_SUBNET="default"
TEST_SUBNET_CIDR="<SUBNET_WITHIN_DMS_CIDR>"

az group create --subscription "$SUBSCRIPTION" -n "$TEST_RG" -l "$LOCATION"

az network vnet create --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "$TEST_VNET" \
  --address-prefix "$DMS_CIDR" \
  --subnet-name "$TEST_SUBNET" --subnet-prefix "$TEST_SUBNET_CIDR" \
  -l "$LOCATION"
```

##### Peer the test virtual network

Create peerings that mirror what DMS does. Choose the peering that matches your topology.

**For single virtual network or hub-direct (VPN/ExpressRoute scenarios):**

```bash
HUB_RG="<HUB_OR_SINGLE_VNET_RESOURCE_GROUP>"
HUB_VNET="<HUB_OR_SINGLE_VNET_NAME>"

TEST_VNET_ID=$(az network vnet show --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "$TEST_VNET" --query id -o tsv)

HUB_VNET_ID=$(az network vnet show --subscription "$SUBSCRIPTION" \
  -g "$HUB_RG" -n "$HUB_VNET" --query id -o tsv)

# Gateway VNet → Test VNet
az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$HUB_RG" --vnet-name "$HUB_VNET" \
  -n "hub-to-dms-test" --remote-vnet "$TEST_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true --allow-gateway-transit true

# Test VNet → Gateway VNet
az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" --vnet-name "$TEST_VNET" \
  -n "dms-test-to-hub" --remote-vnet "$HUB_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true --use-remote-gateways true
```

**For source in Azure (no VPN needed):**

```bash
SOURCE_RG="<SOURCE_RESOURCE_GROUP>"
SOURCE_VNET="<SOURCE_VNET_NAME>"

SOURCE_VNET_ID=$(az network vnet show --subscription "$SUBSCRIPTION" \
  -g "$SOURCE_RG" -n "$SOURCE_VNET" --query id -o tsv)

TEST_VNET_ID=$(az network vnet show --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "$TEST_VNET" --query id -o tsv)

az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$SOURCE_RG" --vnet-name "$SOURCE_VNET" \
  -n "source-to-dms-test" --remote-vnet "$TEST_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true

az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" --vnet-name "$TEST_VNET" \
  -n "dms-test-to-source" --remote-vnet "$SOURCE_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true
```

**Peer with target virtual network:**

```bash
TARGET_RG="<TARGET_RESOURCE_GROUP>"
TARGET_VNET="<TARGET_VNET_NAME>"

TARGET_VNET_ID=$(az network vnet show --subscription "$SUBSCRIPTION" \
  -g "$TARGET_RG" -n "$TARGET_VNET" --query id -o tsv)

az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$TARGET_RG" --vnet-name "$TARGET_VNET" \
  -n "target-to-dms-test" --remote-vnet "$TEST_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true

az network vnet peering create --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" --vnet-name "$TEST_VNET" \
  -n "dms-test-to-target" --remote-vnet "$TARGET_VNET_ID" \
  --allow-vnet-access true --allow-forwarded-traffic true
```

##### Verify peerings

```bash
az network vnet peering list --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" --vnet-name "$TEST_VNET" \
  --query "[].{Name:name,State:peeringState,UseRemoteGateways:useRemoteGateways}" \
  -o table
```

**Expected:** All peerings show `peeringState` = `Connected`.

##### Deploy a test VM

```bash
az vm create --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "dms-test-vm" \
  --image Ubuntu2204 --size Standard_B2s \
  --vnet-name "$TEST_VNET" --subnet "$TEST_SUBNET" \
  --admin-username azureuser --generate-ssh-keys \
  --public-ip-address "" --no-wait
```

The VM has no public IP. Access it via Azure Bastion, serial console, or `az vm run-command invoke`.

#### Phase 3: Test connectivity from simulation VM

Run the following commands from the test VM using `az vm run-command invoke`.

##### Test DNS resolution

```bash
az vm run-command invoke --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "dms-test-vm" \
  --command-id RunShellScript \
  --scripts "nslookup <SOURCE_HOSTNAME> && echo '---' && nslookup <TARGET_DOCUMENTDB_HOSTNAME>"
```

**Expected:** Both resolve to **private** IPs. If either resolves to a public IP, link the private DNS zone to the test virtual network.

##### Test TCP connectivity

```bash
# Source MongoDB
az vm run-command invoke --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "dms-test-vm" \
  --command-id RunShellScript \
  --scripts "nc -zv -w 10 <SOURCE_PRIVATE_IP_OR_HOSTNAME> 27017 2>&1"

# Target DocumentDB
az vm run-command invoke --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "dms-test-vm" \
  --command-id RunShellScript \
  --scripts "nc -zv -w 10 <TARGET_DOCUMENTDB_HOSTNAME> 10260 2>&1"
```

**Expected:** `Connection … succeeded` for both.

##### Test with mongosh

```bash
az vm run-command invoke --subscription "$SUBSCRIPTION" \
  -g "$TEST_RG" -n "dms-test-vm" \
  --command-id RunShellScript \
  --scripts "
    sudo apt-get update -qq && sudo apt-get install -y -qq gnupg curl
    curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
    echo 'deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse' | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
    sudo apt-get update -qq && sudo apt-get install -y -qq mongosh
    echo '=== Test Source ==='
    mongosh 'mongodb://<USER>:<PASS>@<SOURCE>:27017/?tls=true&tlsAllowInvalidCertificates=true' --eval 'db.runCommand({ping:1})' 2>&1
    echo '=== Test Target ==='
    mongosh 'mongodb://<USER>:<PASS>@<TARGET>:10260/?tls=true&tlsAllowInvalidCertificates=true' --eval 'db.runCommand({ping:1})' 2>&1
  "
```

**Expected:** `{ ok: 1 }` for both. If `nc` succeeds but `mongosh` fails, networking is working correctly and the problem is likely related to authentication credentials or TLS configuration.

##### Interpret results

| Result | Meaning | Action |
|---|---|---|
| Peering state = `Disconnected` | CIDR overlap or virtual network config issue | Check for overlapping address spaces |
| `nc` to source times out | Traffic blocked or not routed | Check firewall rules, return routes, NSGs |
| `nc` succeeds, `mongosh` TLS error | TLS/certificate mismatch | Verify `tls=true` and cert settings |
| `nc` succeeds, `mongosh` auth error | Wrong credentials | Verify username/password and auth DB |
| DNS resolves to public IP | Private DNS zone not linked | Link DNS zone to test virtual network |
| Effective routes show `None` for source CIDR | No route to source | Check VPN gateway/peering config |

> [!TIP]
> If all tests pass but the migration job still fails, the problem likely lies within DMS-managed components rather than your infrastructure. Contact Azure support with these test results to expedite troubleshooting.

#### Phase 4: Clean up

> [!IMPORTANT]
> Clean up the simulation virtual network and VM **before** retrying the migration job. If they remain, DMS fails to create its own virtual network and VM due to IP address conflicts.

```bash
# Remove the test resource group (includes the test VNet and VM)
az group delete --subscription "$SUBSCRIPTION" -n "$TEST_RG" --yes

# Remove peerings on your existing VNets
az network vnet peering delete --subscription "$SUBSCRIPTION" \
  -g "<HUB_RG>" --vnet-name "<HUB_VNET>" -n "hub-to-dms-test" 2>/dev/null

az network vnet peering delete --subscription "$SUBSCRIPTION" \
  -g "<SOURCE_RG>" --vnet-name "<SOURCE_VNET>" -n "source-to-dms-test" 2>/dev/null

az network vnet peering delete --subscription "$SUBSCRIPTION" \
  -g "<TARGET_RG>" --vnet-name "<TARGET_VNET>" -n "target-to-dms-test" 2>/dev/null
```

### Common connectivity issues

| Issue | Symptom | Resolution |
|---|---|---|
| DMS CIDR not advertised via BGP | VPN is up but migration reports connectivity errors | Enable `allowGatewayTransit=true` on gateway virtual network peering |
| Remote firewall blocks DMS CIDR | Other Azure virtual networks reach source, but DMS times out | Add the DMS CIDR from the wizard to remote firewall rules |
| Missing return routes | Traffic leaves Azure but never returns; connection times out | Enable route propagation on remote route table or add static route for DMS CIDR |
| Private endpoint not approved | DNS resolves but connection refused | Approve the private endpoint connection in portal or CLI |
| DNS not resolving correctly | Hostname resolves to public IP instead of private | Link private DNS zone to the virtual network used for migration |
| CIDR overlap | Virtual network peering fails or stays `Disconnected` | Select a DMS CIDR that doesn't conflict with existing virtual networks |
| Single virtual network for multiple private jobs | Second job fails or interferes with the first | Use different virtual networks for each concurrent migration job |

## Register Microsoft.DataMigration resource provider in your subscription

To register the Microsoft.DataMigration resource provider in your subscription, follow these steps:

### Azure portal

1. Go to the Azure portal and navigate to your subscription.

1. In the left-hand menu, select **Resource providers** under **Settings**.

1. Search for **Microsoft.DataMigration** in the search box at the top.

1. If it isn't registered, select it and select the **Register** button.

### Azure CLI

1. Open the Azure Cloud Shell or your local terminal.

1. Run the following command to register the resource provider:

   ```azurecli
   az provider register --namespace Microsoft.DataMigration
   ```

### PowerShell

1. Open the Azure Cloud Shell or your local PowerShell.

1. Run the following command to register the resource provider:

   ```powershell
   Register-AzResourceProvider -ProviderNamespace "Microsoft.DataMigration"
   ```

## FAQ

### Why are views missing in the select collection screen step when Azure DocumentDB supports views?
Azure DocumentDB supports the creation of new views. However, the migration extension doesn't provide support for migrating existing views.

After the migration is finished, you can always recreate the views.

### Which collections and databases are skipped when migrating from MongoDB to Azure DocumentDB?
The following databases and collections are considered internal for MongoDB:

| Category        | Description                          |
|-----------------|--------------------------------------|
| **Databases**   | admin, local, system config           |
| **Collections** | Any collection with prefix `system.`  |



### Does the migration job run locally on my machine?
The migration wizard in VS Code requires network connectivity from your local machine to both the source and target environments. This connectivity is used to enumerate databases and collections and to submit the migration job. Once the job is submitted, you can close VS Code or disconnect from the source and target environments.

Data migration is executed entirely by **Azure Database Migration Service (DMS)**, an Azure-hosted service that manages all data movement. DMS doesn't rely on your local machine or VS Code for job execution, so local connectivity isn't required after job submission.

### Can I rename databases and collections during migration?
The extension doesn't support database and collection renaming during migration.


### How should I configure my source server firewalls to avoid connectivity issues?

The required network configuration depends on the selected connectivity mode:  

- **Public mode:** You must allow the IP addresses displayed in the wizard on both the source and target firewalls to enable communication. 
- **Private mode:** You must enable **virtual network integration** so that the DMS servers can securely communicate with the source and target endpoints within the virtual network.

For detailed validation steps and troubleshooting, see [Review connectivity](#review-connectivity). Also refer to [VS Code connectivity](#does-the-migration-job-run-locally-on-my-machine).

### How many databases and collections can I migrate in a single migration?

You can include unlimited number of collections in a single migration.

### How many migration jobs can I run simultaneously?

You can run **multiple migration jobs** when using **public access**. However, when using **private access**, a single virtual network can support **only one active job at a time**. To run multiple jobs with private access, you need to use **different virtual network** for each job.  

### What type of logs does the extension generate?

The extension records errors, warnings, and other diagnostic logs in the default log directory:

- **Windows** - `C:\Users\<username>\.dmamongo\logs\`
- **Linux** - `~/.dmamongo/logs`
- **macOS** - `/Users/<username>/.dmamongo/logs`

## Next steps

> [!div class="nextstepaction"]
> [Build a Node.js web application](tutorial-nodejs-web-app.md)
