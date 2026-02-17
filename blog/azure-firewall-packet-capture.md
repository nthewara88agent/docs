---
title: "Azure Firewall Packet Capture: A Complete Guide to Network Troubleshooting"
date: 2026-02-17
author: "Nirmal Thewara"
tags:
  - azure
  - azure-firewall
  - networking
  - troubleshooting
  - packet-capture
  - security
description: "Learn how to use Azure Firewall's packet capture feature to troubleshoot network connectivity issues. This guide covers prerequisites, step-by-step instructions, CLI commands, and best practices."
---

# Azure Firewall Packet Capture: A Complete Guide to Network Troubleshooting

Azure Firewall's packet capture feature is a powerful troubleshooting tool that allows you to capture and analyze network traffic passing through your firewall. This feature became Generally Available (GA) and provides deep visibility into packet-level traffic for debugging connectivity issues.

In this guide, I'll walk you through everything you need to know about Azure Firewall packet capture—from prerequisites to analyzing capture files.

## How Azure Firewall Packet Capture Works

Azure Firewall packet capture intercepts IP packets as they traverse the firewall, saving them to `.pcap` files for analysis. Key characteristics include:

- **Bidirectional capture**: For every packet the firewall processes, you see both incoming and outgoing packets
- **Multi-instance support**: Azure Firewall runs on multiple backend instances, so you'll get one `.pcap` file per instance
- **Filter-based**: You must specify filters (source/destination IPs and ports) to target specific traffic flows
- **Time and count limited**: Captures stop when either the packet count or time limit is reached

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure Firewall                            │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────┐          │
│  │ Incoming │ -> │  Packet Capture  │ -> │ Outgoing │          │
│  │  Packet  │    │    (Filter)      │    │  Packet  │          │
│  └──────────┘    └────────┬─────────┘    └──────────┘          │
│                           │                                      │
│                           v                                      │
│                  ┌─────────────────┐                            │
│                  │  Storage Blob   │                            │
│                  │  (.pcap files)  │                            │
│                  └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before you can run packet captures, ensure you have:

1. **Azure Firewall with Management NIC enabled**
   - Management NIC is enabled by default on Basic SKU and Virtual WAN deployments
   - For Standard or Premium SKU in a virtual network, you must [enable it manually](https://learn.microsoft.com/en-us/azure/firewall/forced-tunneling)

2. **Storage Account** with:
   - Anonymous access enabled on individual containers
   - A container with anonymous read access level
   - A valid SAS URL with **Write** permissions

3. **Appropriate RBAC permissions** on the firewall resource

## Step-by-Step Guide: Azure Portal

### Step 1: Create and Configure Storage Account

```bash
# Create a storage account
az storage account create \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --location australiaeast \
  --sku Standard_LRS \
  --allow-blob-public-access true

# Create a container
az storage container create \
  --name packet-captures \
  --account-name mystorageaccount \
  --public-access container
```

### Step 2: Generate SAS URL

Generate a SAS URL with **Write** permission (and without Read):

```bash
# Generate SAS token with write-only permissions
END_DATE=$(date -u -d "+1 day" '+%Y-%m-%dT%H:%MZ')

az storage container generate-sas \
  --name packet-captures \
  --account-name mystorageaccount \
  --permissions w \
  --expiry $END_DATE \
  --output tsv
```

Combine the SAS token with your container URL:
```
https://mystorageaccount.blob.core.windows.net/packet-captures?<SAS_TOKEN>
```

### Step 3: Configure and Run Packet Capture (Portal)

1. Navigate to your Azure Firewall in the portal
2. Under **Help**, select **Packet Capture**
3. Configure the capture settings:
   - **Packet capture name**: A unique name for your capture
   - **Output SAS URL**: Paste your SAS URL
   - **Maximum packets**: 100 - 90,000
   - **Time limit**: 30 - 1,800 seconds
   - **Protocol**: Any, TCP, UDP, or ICMP
   - **TCP Flags**: SYN, ACK, FIN, RST, PSH, URG (optional)

4. Add at least one filter with:
   - Source IP addresses or subnets
   - Destination IP addresses or subnets
   - Destination ports

5. Click **Start packet capture**

## PowerShell Method

PowerShell provides more flexibility for automation and scripting:

```powershell
# Get the firewall
$azFirewall = Get-AzFirewall -Name "myAzureFirewall" -ResourceGroupName "myResourceGroup"

# Create filter rules
$filter1 = New-AzFirewallPacketCaptureRule `
  -Source "10.0.0.0/24" `
  -Destination "192.168.1.0/24" `
  -DestinationPort "80","443"

$filter2 = New-AzFirewallPacketCaptureRule `
  -Source "10.1.0.0/24" `
  -Destination "172.16.0.0/16" `
  -DestinationPort "22","3389"

# Create packet capture parameters
$captureParams = New-AzFirewallPacketCaptureParameter `
  -DurationInSeconds 300 `
  -NumberOfPacketsToCapture 5000 `
  -SASUrl "https://mystorageaccount.blob.core.windows.net/captures?sv=..." `
  -Filename "my-capture" `
  -Protocol "Any" `
  -Flag "Syn","Ack" `
  -Filter $filter1, $filter2

# Start the packet capture
Invoke-AzFirewallPacketCapture -AzureFirewall $azFirewall -Parameter $captureParams
```

### Stop a Running Capture

```powershell
# Create stop parameters
$stopParams = New-AzFirewallPacketCaptureParameter -Operation "Stop"

# Stop the capture
Invoke-AzFirewallPacketCapture -AzureFirewall $azFirewall -Parameter $stopParams
```

## REST API Method

For maximum flexibility, use the REST API directly with Azure CLI:

```bash
# Start packet capture using REST API
az rest --method post \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/azureFirewalls/{firewallName}/packetCapture?api-version=2024-05-01" \
  --body '{
    "durationInSeconds": 300,
    "numberOfPacketsToCapture": 5000,
    "sasUrl": "https://mystorageaccount.blob.core.windows.net/captures?sv=...",
    "fileName": "azureFirewallCapture",
    "protocol": "Any",
    "flags": [
      {"type": "syn"},
      {"type": "ack"}
    ],
    "filters": [
      {
        "sources": ["10.0.0.0/24"],
        "destinations": ["192.168.1.0/24"],
        "destinationPorts": ["80", "443"]
      }
    ]
  }'
```

### Check Capture Status

```bash
az rest --method post \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/azureFirewalls/{firewallName}/packetCapture?api-version=2024-05-01" \
  --body '{"operation": "Status"}'
```

### Stop Capture

```bash
az rest --method post \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/azureFirewalls/{firewallName}/packetCapture?api-version=2024-05-01" \
  --body '{"operation": "Stop"}'
```

## Supported Filters

| Parameter | Description | Example |
|-----------|-------------|---------|
| **Source IPs** | Source IP addresses or subnets (comma-separated) | `10.0.0.1, 10.0.0.0/24` |
| **Destination IPs** | Destination IP addresses or subnets | `192.168.1.0/24` |
| **Destination Ports** | Specific ports to capture | `80, 443, 3389` |
| **Protocol** | Network protocol | `Any`, `TCP`, `UDP`, `ICMP` |
| **TCP Flags** | Filter by TCP flags (TCP protocol only) | `SYN`, `ACK`, `FIN`, `RST`, `PSH`, `URG` |

### Filter Limitations

- **At least one filter is required** - you cannot capture all traffic
- **No FQDN support** - use DNS to resolve FQDNs to IPs first
- **No IP ranges** - use subnets instead (e.g., `/24`)
- **Maximum 5 IPs/subnets per filter**
- **Cannot use 0.0.0.0/0** - results in empty captures
- **Must specify destination ports** - cannot capture all ports

## Downloading and Analyzing Capture Files

### Download Files

After the capture completes, download the `.pcap` files from your storage container:

```bash
# List capture files
az storage blob list \
  --container-name packet-captures \
  --account-name mystorageaccount \
  --output table

# Download all pcap files
az storage blob download-batch \
  --destination ./captures \
  --source packet-captures \
  --account-name mystorageaccount \
  --pattern "*.pcap"
```

### Merge Multiple Capture Files

Since Azure Firewall runs on multiple instances, you'll have multiple `.pcap` files. Merge them in Wireshark:

1. Open the first `.pcap` file in Wireshark
2. Go to **File** > **Merge**
3. Select additional files
4. Choose **Merge packets chronologically**

### Analyze with Wireshark

Common Wireshark filter examples:

```
# Filter by IP and port
tcp.port==443 && ip.addr==10.0.0.5

# Filter TCP handshake issues
tcp.flags.syn==1 && tcp.flags.ack==0

# Filter HTTP traffic
tcp.port==80 && http

# Filter by source and destination
ip.src==10.0.0.1 && ip.dst==192.168.1.1
```

## Understanding Packet Flow Patterns

Azure Firewall captures **two packets per flow** (incoming and outgoing). Understanding the patterns helps correlate traffic:

### VNet-to-VNet (No SNAT)
| Direction | Source | Destination |
|-----------|--------|-------------|
| Incoming | Client IP | Server IP |
| Outgoing | Client IP | Server IP |

*Layer 3+ remains identical; only Layer 2 headers differ.*

### VNet-to-Internet or SNAT Enabled
| Direction | Source | Destination |
|-----------|--------|-------------|
| Incoming | Client IP | Server IP |
| Outgoing | **Firewall IP** | Server IP |

*Layer 3 source changes to firewall's IP due to SNAT.*

### DNAT Flows (Inbound from Internet)
| Direction | Source | Destination |
|-----------|--------|-------------|
| Incoming | Client Public IP | Firewall Public IP |
| Outgoing | Firewall Private IP | DNATed Private IP |

*Destination IP changes from public to private due to DNAT.*

### Application Rules
| Direction | Source | Destination |
|-----------|--------|-------------|
| Incoming | Client IP | Server IP |
| Outgoing | **Firewall IP** | Server IP |

*The firewall acts as a proxy—L4+ differs. Use HTTP/TLS keys to correlate packets.*

## Use Cases and Best Practices

### Common Use Cases

1. **Troubleshooting connectivity issues** between VMs or to the internet
2. **Debugging SNAT/DNAT** configuration problems
3. **Verifying firewall rules** are working as expected
4. **Investigating security incidents** with forensic evidence
5. **Performance analysis** of network flows

### Best Practices

1. **Be specific with filters** - Narrow down to specific IPs and ports
2. **Include all relevant subnets** - For SNAT scenarios, include both client subnet and AzureFirewallSubnet
3. **Use unique filenames** - Prevents overwriting previous captures
4. **Start with short durations** - 60-300 seconds is usually sufficient
5. **Plan for multiple files** - Each firewall instance creates a separate file
6. **Use appropriate packet limits** - Start with 5,000-10,000 packets

### Scenario: Capturing SNAT Traffic

When troubleshooting SNAT (to Internet or between VNets with SNAT):

```powershell
$filter = New-AzFirewallPacketCaptureRule `
  -Source "10.0.1.0/24", "10.0.0.0/26" `  # Include AzureFirewallSubnet!
  -Destination "203.0.113.0/24" `
  -DestinationPort "443"
```

> **Important**: Include the AzureFirewallSubnet address space in your source filter to capture both the original packet and the SNATed packet.

## Limitations and Considerations

| Limitation | Details |
|------------|---------|
| **Management NIC Required** | Must be enabled for Standard/Premium SKUs |
| **No continuous capture** | Cyclical captures not supported; contact Azure Support for extended captures |
| **Filter required** | Cannot capture all traffic without filters |
| **No wildcard ports** | Must specify at least one destination port |
| **SAS URL expiry** | Ensure SAS token doesn't expire during capture |
| **Success criteria** | Capture is "successful" if at least half of backend instances return data |
| **Empty captures** | May occur if no traffic matches filters |
| **No FQDN filtering** | Must use resolved IP addresses |
| **Azure CLI** | Native CLI support is limited—use PowerShell or REST API |

## Troubleshooting Common Issues

### Capture Fails to Start

1. **Check SAS URL permissions** - Must have Write permission, not Read
2. **Verify container access** - Must be set to Container-level anonymous access
3. **Confirm Management NIC** - Required for packet capture

### Empty Capture Files

1. **Broaden your filters** - Traffic might not match current filters
2. **Check timing** - Ensure traffic occurs during capture window
3. **Verify IP addresses** - Use actual IPs, not FQDNs

### Missing Outgoing Packets

For SNAT scenarios, ensure you include the AzureFirewallSubnet address space in your source filter.

## Summary

Azure Firewall packet capture is an invaluable tool for network troubleshooting. Key takeaways:

- **Enable Management NIC** before attempting captures
- **Configure storage correctly** with Write-only SAS URLs
- **Use specific filters** to target the traffic you need
- **Include firewall subnet** for SNAT scenarios
- **Merge multiple files** since each instance creates its own capture
- **Analyze with Wireshark** for deep packet inspection

For more information, check the [official Microsoft documentation](https://learn.microsoft.com/en-us/azure/firewall/packet-capture).

---

*Have questions or feedback? Feel free to reach out!*
