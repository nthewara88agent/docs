# Capability Hosts in Azure AI Foundry

*Published: 2026-02-02*

> **Source:** [Microsoft Learn - Capability Hosts](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/capability-hosts)

## What Is a Capability Host?

A **Capability Host** is a sub-resource that you configure at both the AI Foundry **account level** and **project level**. It tells the Foundry Agent Service **where to store and process agent data**, including:

1. **Conversation history (threads)** — chat messages, agent definitions
2. **File uploads** — documents you upload to agents
3. **Vector stores** — embeddings for retrieval/search

## Why Use Capability Hosts?

By default, if you don't create capability hosts, AI Foundry uses **Microsoft-managed resources** for all storage. When you provision capability hosts, you enable **"Standard Agent Setup"** which provides:

| Benefit | Description |
|---------|-------------|
| **Data sovereignty** | All agent data stays in YOUR Azure subscription |
| **Security control** | Use your own storage accounts, Cosmos DB, AI Search |
| **Compliance** | Meet regulatory requirements by controlling where data lives |

## How It Works

### Configuration Hierarchy

Capability hosts follow a hierarchy where more specific configurations override broader ones:

1. **No capability host** → Microsoft-managed storage (basic setup)
2. **Account-level capability host** → Shared defaults for all projects under the account
3. **Project-level capability host** → Overrides account-level and service defaults for that specific project

### Default Behavior (Microsoft-Managed Resources)

If you don't create an account-level and project-level capability host, the Agent Service automatically uses Microsoft-managed Azure resources for:

- Thread storage (conversation history, agent definitions)
- File storage (uploaded documents)
- Vector search (embeddings and retrieval)

### Bring-Your-Own Resources (Standard Agent Setup)

When you create capability hosts at both the account and project levels, agent data is stored and processed using your Azure resources in your subscription.

> **Note:** For network-secured standard agent setup, deploy all related resources in the same region as your virtual network (VNet).

## Required Properties (Project Capability Host)

To use your own resources for agent data, configure the project capability host with the following properties:

| Property | Purpose | Required Azure Resource | Example Connection Name |
|----------|---------|------------------------|------------------------|
| `threadStorageConnections` | Stores agent definitions, conversation history and chat threads | Azure Cosmos DB | `"my-cosmosdb-connection"` |
| `vectorStoreConnections` | Handles vector storage for retrieval and search | Azure AI Search | `"my-ai-search-connection"` |
| `storageConnections` | Manages file uploads and blob storage | Azure Storage Account | `"my-storage-connection"` |

### Optional Property

| Property | Purpose | Required Azure Resource | When to Use |
|----------|---------|------------------------|-------------|
| `aiServicesConnections` | Use your own model deployments | Azure OpenAI | When you want to use models from your existing Azure OpenAI resource instead of the built-in account level ones |

## Creating Capability Hosts

### Account Capability Host

Use an account capability host to enable Agent Service and (optionally) define defaults that projects can inherit.

```http
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/capabilityHosts/{name}?api-version=2025-06-01

{
  "properties": {
    "capabilityHostKind": "Agents"
  }
}
```

### Project Capability Host

This configuration overrides service defaults and any account-level settings. All agents in this project will use your specified resources.

```http
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/projects/{projectName}/capabilityHosts/{name}?api-version=2025-06-01
```

## Key Constraints

| Constraint | Description |
|------------|-------------|
| **One per scope** | Each account and each project can only have one active capability host. Attempting to create a second with a different name results in a 409 conflict |
| **No updates** | Configuration updates are not supported. You must delete and recreate to change configuration |

## Terraform Example

When deploying AI Foundry with Terraform, capability hosts are created as part of the infrastructure:

```hcl
# Account-level capability host
resource "azapi_resource" "account_capability_host" {
  type      = "Microsoft.CognitiveServices/accounts/capabilityHosts@2025-06-01"
  name      = "agents-host"
  parent_id = azurerm_cognitive_account.ai_foundry.id

  body = jsonencode({
    properties = {
      capabilityHostKind = "Agents"
    }
  })
}

# Project-level capability host
resource "azapi_resource" "project_capability_host" {
  type      = "Microsoft.CognitiveServices/accounts/projects/capabilityHosts@2025-06-01"
  name      = "agents-host"
  parent_id = azapi_resource.ai_foundry_project.id

  body = jsonencode({
    properties = {
      capabilityHostKind        = "Agents"
      threadStorageConnections  = [azapi_resource.cosmos_connection.name]
      vectorStoreConnections    = [azapi_resource.search_connection.name]
      storageConnections        = [azapi_resource.storage_connection.name]
    }
  })

  depends_on = [azapi_resource.account_capability_host]
}
```

## Verifying Configuration

### Get Account Capability Host

```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/capabilityHosts?api-version=2025-06-01
```

### Get Project Capability Host

```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/projects/{projectName}/capabilityHosts?api-version=2025-06-01
```

## Deleting Capability Hosts

> ⚠️ **Warning:** Deleting a capability host will affect all agents that depend on it. Agents will no longer have access to files, threads, and vector stores.

### Delete Account-Level Capability Host

```http
DELETE https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/capabilityHosts/{name}?api-version=2025-06-01
```

### Delete Project-Level Capability Host

```http
DELETE https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/projects/{projectName}/capabilityHosts/{name}?api-version=2025-06-01
```

## Troubleshooting

### HTTP 409 Conflict Errors

**Problem:** Multiple capability hosts per scope

```json
{
  "error": {
    "code": "Conflict",
    "message": "There is an existing Capability Host with name: existing-host..."
  }
}
```

**Solution:**
1. Query the scope to see what already exists
2. Use consistent naming across all requests for the same scope
3. Delete existing and recreate if needed

### Configuration Change Workflow

Since updates aren't supported:

1. Delete the existing capability host
2. Wait for deletion to complete
3. Create a new capability host with the desired configuration

## Related Resources

- [Standard Agent Setup](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/standard-agent-setup)
- [Bring Your Own Storage](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/bring-your-own-azure-storage-foundry)
- [Virtual Networks Configuration](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/virtual-networks)
- [Foundry Account Management REST API](https://learn.microsoft.com/en-us/rest/api/aifoundry/accountmanagement/operation-groups)
