---
title: "VNet Data Gateways: The Complete Guide to Architecture, Setup, HA & Power BI Integration"
date: 2026-03-11
description: "A comprehensive guide to Microsoft's Virtual Network (VNet) Data Gateways — covering architecture, creation, high availability, load balancing, and Power BI integration."
tags:
  - Azure
  - Power BI
  - VNet
  - Data Gateway
  - Networking
  - Microsoft Fabric
author: Nirmal Thewara
---

# VNet Data Gateways: The Complete Guide

Virtual Network (VNet) data gateways let you securely connect Microsoft Cloud services — like Power BI and Microsoft Fabric — to data sources inside an Azure VNet, without managing on-premises gateway infrastructure. This post covers everything you need to know: how they work, how to set them up, how to configure high availability, and how to use them with Power BI.

## What Is a VNet Data Gateway?

A VNet data gateway is a managed gateway that Microsoft injects directly into a delegated subnet in your Azure VNet. Unlike the traditional on-premises data gateway (which requires you to install and maintain software on a VM), the VNet gateway is fully managed by Microsoft. It enables secure connectivity to data sources that are only accessible from within your VNet — such as Azure SQL with private endpoints, or on-premises databases connected via ExpressRoute.

## Architecture: How It Works

Here's the step-by-step data flow when a Power BI report queries a data source behind a VNet:

1. **Query initiation** — The Power BI cloud service (or another supported service) kicks off a query and sends the query, data source details, and credentials to the Microsoft Power Platform VNet service.
2. **Container injection** — The Power Platform VNet service securely injects a container running the VNet data gateway into your delegated subnet.
3. **Query forwarding** — The VNet service sends the query, data source details, and credentials to the gateway container.
4. **Data source connection** — The VNet data gateway connects to the data source using the provided credentials.
5. **Query execution** — The query is executed against the data source.
6. **Results return** — Results are sent back to the gateway container, and the Power Platform VNet service securely pushes the data back to the cloud service.

### Key Networking Details

- When the gateway starts, it **leases an IP from your delegated subnet** — meaning it respects all Network Security Group (NSG) and NAT rules on that subnet.
- The gateway **does not require any Service Endpoints or open ports back to Power BI**. Data returns to Power BI through an internal Microsoft tunnel using Automatic Private IP Addressing (APIPA) that never touches the public internet.
- **All traffic uses the Azure backbone**, including the internal tunnel.

### Hardware Specifications

Each VNet data gateway instance has a fixed capacity of:

- **2 CPU cores**
- **8 GB RAM**

This configuration cannot be scaled or changed per instance. To get more capacity, you add more gateway members to a cluster (covered in the HA section below).

### Region Considerations

- The VNet data gateway **must be created in the home region of your tenant** to work with Power BI.
- However, you **can choose an Azure VNet and subnet from any supported region**.
- Only **metadata** (name, details, encrypted credentials) is stored in the home region. Your actual **data flows through the subnet** in whatever region it's located.

## Creating a VNet Data Gateway

### Prerequisites

- **Power BI Premium capacity** (A4 SKU or higher, or any P SKU) **or a Fabric license** (any SKU)
- The feature must be **supported in your region**
- Cross-tenant creation is **not supported**

### Step 1: Register the Resource Provider

On the Azure portal, register `Microsoft.PowerPlatform` as a resource provider for the subscription containing your VNet:

1. Sign in to the Azure portal
2. Navigate to your subscription
3. Go to **Resource providers**
4. Search for **Microsoft.PowerPlatform** and click **Register**

### Step 2: Delegate a Subnet to Microsoft Power Platform

A user with the `Microsoft.Network/virtualNetworks/subnets/join/action` permission (e.g., the **Network Contributor** role) needs to delegate a subnet:

1. In the Azure portal, **add a new subnet** to your VNet
2. Select **Microsoft.PowerPlatform/vnetaccesslinks** from the subnet delegation dropdown
3. Click **Save**

**Subnet sizing tips:**

- 5 IPs are reserved for basic functionality
- Reserve **1 additional IP per gateway member** you plan to create
- Example: 2 clusters × 3 members each = 6 + 5 = 11 IPs minimum
- Add extra IPs for future growth

**Important subnet rules:**

- Do **not** use the names "gatewaysubnet" or "AzureBastionSubnet" — these are reserved
- Do **not** add an IPv6 address space to the subnet
- The subnet IP range **must not overlap** with 10.0.1.x
- For large datasets, add the **Microsoft.Storage** Service Endpoint to the subnet
- If using Azure Virtual Hub with Propagate Default Route enabled, also add the Microsoft.Storage Service Endpoint
- Gateways within a cluster need to communicate — don't block the subnet's own IP range in NSG rules

### Step 3: Create the Gateway in Power BI

1. Sign in to the Power BI homepage (app.powerbi.com)
2. Click the **settings gear icon** → **Manage connections and gateways**
3. Select **Virtual network (VNet) data gateway** → **New**
4. Choose your license capacity, subscription, resource group, VNet, and delegated subnet
5. Optionally rename the gateway
6. Click **Save**

The gateway now appears in your Virtual network data gateways tab and is ready to use.

## Supported Regions

VNet data gateways are available in a wide range of Azure regions, including:

- **Asia Pacific** — Australia East, Australia Southeast, Central India, East Asia, Indonesia Central, Japan East, Korea Central, Malaysia West, New Zealand North, Southeast Asia, Taiwan North
- **Europe** — France Central, Germany West Central, Italy North, North Europe, Norway East, Poland Central, Spain Central, Sweden Central, Switzerland North, UK South, UK West, West Europe
- **Americas** — Brazil South, Canada Central, Central US, East US, East US 2, Mexico Central, North Central US, South Central US, West Central US, West US, West US 2, West US 3
- **Middle East & Africa** — Israel Central, South Africa North, UAE North

## High Availability and Load Balancing

### Why Clusters Matter

A single VNet data gateway is a single point of failure. By creating a **cluster** with multiple gateway members, you get:

- **High availability** — if one member goes down, others handle the load
- **Load balancing** — queries are distributed randomly across members in the cluster

### Creating a Cluster

**Option 1: During initial creation** — When creating a new VNet data gateway, use the advanced options to set the number of gateways (up to **9 per cluster**).

**Option 2: Edit existing gateway** — Select an existing VNet data gateway → **Settings** → adjust the number of gateways.

### Autopause Behaviour

- VNet data gateway clusters **autopause after inactivity** (default: 30 minutes)
- After autopausing, it takes **2–3 minutes** for the cluster to become available again
- You can increase the autopause interval up to **24 hours**
- **There is no "always on" option** — the gateway will always autopause eventually

You can adjust the autopause interval in the same Settings panel where you configure the number of gateway members.

### Workload Capacity Per Gateway Instance

Each individual gateway instance can handle these concurrent workloads:

- **6 refresh queries** simultaneously (scheduled refresh)
- **15 direct queries** concurrently (interactive reporting / real-time)
- **1 Fabric Pipeline Copy activity** job
- **1 Fabric Copy job** (with multiple tables, the job splits into subjobs — one subjob at a time per instance)
- **6 Lookup, Get metadata, Delete, and Script activities** concurrently
- **Up to 200 other pipeline activities** (Stored Procedure, HDInsight, etc.)

So a 3-member cluster can handle 3× these numbers in parallel.

## Using VNet Data Gateways with Power BI

### Managing the Gateway

You manage VNet data gateways the same way as standard gateways:

- Via the **Power Platform admin center**, or
- Via the **Manage connections and gateways** page in Power BI

You can add/remove gateway admins and create/share data sources just like with standard gateways.

### Connecting to Data Sources

There are three connectivity scenarios:

**1. Private resources in Azure** — Use private endpoints and private DNS zones (or service endpoints) on your data sources. All traffic stays on the Azure backbone.

**2. Private resources outside Azure** — Use ExpressRoute and/or VPNs to extend your VNet to on-premises networks. Traffic stays on the Azure backbone.

**3. Public resources** — Connect to publicly accessible endpoints through the VNet.

### Supported Azure Data Sources (Private Endpoint)

Sources that support secure private endpoint connectivity include:

- Azure SQL and Azure SQL Managed Instance
- Azure Blob Storage, Data Lake Storage Gen1 and Gen2, Table Storage
- Azure Cosmos DB (v1 and v2)
- Azure Synapse Analytics / Workspace
- Azure Databricks and Databricks Workspace
- Azure Database for PostgreSQL
- Azure Data Explorer (Kusto)
- Azure HDInsight (Cluster, Spark, AKS Trino)
- Azure Machine Learning
- Azure AI Search
- Azure Functions
- Azure Key Vault
- Azure Data Factory Workspace
- Azure Data Lake Analytics
- Azure Batch
- Azure Resource Graph
- Snowflake (Azure)

### Supported Sources via Public/ExpressRoute/VPN

The list of supported connectors is extensive — over 150 sources — including SQL Server, PostgreSQL, MySQL, Snowflake, Salesforce, SharePoint, REST/OData, Oracle, Google BigQuery, Amazon Redshift, Databricks, Dataverse, Excel, JSON, XML, Web, and many more.

### Microsoft Entra ID SSO for DirectQuery

For DirectQuery reports, you can enable Single Sign-On (SSO) so that queries execute under the identity of the user interacting with the report:

1. Go to **Manage Gateways** → **Data source settings**
2. Check **Use SSO via Azure AD for DirectQuery queries**

This means each cross-filter, slice, sort, and edit operation runs live against the data source using the actual user's Entra ID identity — great for row-level security scenarios.

### Publishing Reports

Report creators publish their reports to Power BI as usual, then associate the semantic model to the VNet data gateway data source through the gateway connection settings.

## Summary

VNet data gateways eliminate the need to manage on-premises gateway VMs while providing secure, VNet-injected connectivity to your data. Here's the quick recap:

- **Fully managed** — no infrastructure to maintain
- **Secure by design** — traffic stays on the Azure backbone, no open ports required
- **Easy setup** — register the resource provider, delegate a subnet, create the gateway in Power BI
- **Scalable** — up to 9 members per cluster for HA and load balancing
- **Broad compatibility** — 150+ data source connectors, private endpoints, ExpressRoute, VPN
- **Entra ID SSO** — DirectQuery with per-user identity for row-level security

---

**References:**

- <https://learn.microsoft.com/en-us/data-integration/vnet/data-gateway-architecture>
- <https://learn.microsoft.com/en-us/data-integration/vnet/create-data-gateways>
- <https://learn.microsoft.com/en-us/data-integration/vnet/high-availability-load-balancing>
- <https://learn.microsoft.com/en-us/data-integration/vnet/use-data-gateways-sources-power-bi>
