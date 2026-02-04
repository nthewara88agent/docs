# Azure PostgreSQL Hosting Options: A Complete Comparison Guide

*Published: 4 February 2026*

Choosing the right PostgreSQL hosting option on Azure can be confusing. With multiple services available—some deprecated, some evolving—it's essential to understand what's current and what fits your workload. This guide breaks down all your options based on official Microsoft documentation.

## TL;DR - Quick Decision Guide

| Need | Recommended Option |
|------|-------------------|
| **Standard workloads** (web apps, APIs) | Azure Database for PostgreSQL - Flexible Server |
| **Horizontal scale** (SaaS, multi-tenant) | Flexible Server with Elastic Clusters (Citus) |
| **Full control** (custom extensions, OS access) | PostgreSQL on Azure VMs |
| **Massive distributed scale** | Azure Cosmos DB for NoSQL (not PostgreSQL) |

---

## The Current Landscape (2026)

Microsoft has consolidated PostgreSQL offerings over time. Here's what's available today:

### 1. Azure Database for PostgreSQL - Flexible Server ✅ (Recommended)

**The primary PaaS option for PostgreSQL on Azure.**

Flexible Server is the fully managed PostgreSQL service that Microsoft recommends for new deployments. It provides enterprise-grade security, high availability, and automated maintenance while giving you control over database configuration.

**Key Features:**
- **Compute Tiers:** Burstable, General Purpose, and Memory Optimized
- **High Availability:** Zone-redundant HA across availability zones, or same-zone HA
- **Storage:** Up to 64 TB with auto-grow capability
- **Backups:** Automated daily backups with up to 35-day retention (zone-redundant storage)
- **Stop/Start:** Pause compute to save costs during dev/test
- **Built-in PgBouncer:** Connection pooling included
- **Maintenance Windows:** Choose when patching occurs
- **PostgreSQL Versions:** Supports recent community versions (check docs for specifics)

**Best For:**
- Web applications and APIs
- Development and testing environments
- Production workloads requiring managed HA
- Applications migrating from on-premises PostgreSQL

**Pricing Model:** Pay for compute (vCores) + storage + backup storage. Hourly billing with the ability to stop/start.

---

### 2. Elastic Clusters (Citus Extension) ✅ (For Horizontal Scale)

**Horizontal sharding for PostgreSQL, built on Flexible Server.**

Elastic Clusters is a feature of Azure Database for PostgreSQL that enables the open-source [Citus extension](https://www.citusdata.com/). It allows you to distribute data across multiple PostgreSQL nodes, enabling horizontal scale for high-throughput workloads.

**Key Features:**
- **Sharding Models:** Row-based sharding (hash distribution) or schema-based sharding
- **Coordinator + Workers:** Queries route through coordinator, parallelized across workers
- **Online Rebalancing:** Add nodes and redistribute data without downtime
- **Load Balancing:** Connect via port 7432 for automatic load balancing across nodes
- **Built-in PgBouncer:** Available on port 8432 for pooled, load-balanced connections

**Architecture:**
```
┌─────────────────────────────────────────────┐
│              Your Application               │
│                    │                        │
│            Port 5432 (Coordinator)          │
│            Port 7432 (Load Balanced)        │
│            Port 8432 (PgBouncer LB)         │
└─────────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Node 1  │  │ Node 2  │  │ Node 3  │
   │(Coord)  │  │(Worker) │  │(Worker) │
   │Shard A-D│  │Shard E-H│  │Shard I-L│
   └─────────┘  └─────────┘  └─────────┘
```

**Best For:**
- Multi-tenant SaaS applications
- Real-time analytics dashboards
- High-throughput transactional systems
- Time-series data at scale
- Workloads exceeding single-node capacity

**When to Choose Over Standard Flexible Server:**
- Your data exceeds what a single node can handle
- You need to parallelize queries across multiple nodes
- You're building a multi-tenant app where each tenant's data can be isolated

---

### 3. PostgreSQL on Azure VMs (IaaS) ⚠️ (Full Control)

**Self-managed PostgreSQL on Azure Virtual Machines.**

For scenarios where you need complete control over the PostgreSQL engine, operating system, or require unsupported extensions, running PostgreSQL on Azure VMs is the IaaS option.

**Key Features:**
- **Full OS Access:** SSH into your VM, install any packages
- **Any PostgreSQL Version:** Run any version, including forks (e.g., EnterpriseDB)
- **Custom Extensions:** Install extensions not available in Flexible Server
- **Custom HA:** Implement your own clustering (Patroni, repmgr, etc.)

**Trade-offs:**

| Aspect | Flexible Server (PaaS) | VMs (IaaS) |
|--------|------------------------|------------|
| **OS Patching** | Automatic | You manage |
| **PostgreSQL Patching** | Automatic | You manage |
| **Backups** | Built-in, automated | You configure |
| **High Availability** | Built-in zone-redundant | You architect & implement |
| **Disaster Recovery** | Built-in geo-restore | You design & test |
| **Monitoring** | Built-in metrics & alerts | You configure |
| **SLA** | Up to 99.99% | Based on VM SLA |

**Best For:**
- Workloads requiring unsupported extensions
- Custom PostgreSQL builds or forks
- Existing VM-based infrastructure you're migrating as-is
- Organizations with strong DBA expertise wanting full control

**Cost Considerations:**
- VM compute + Premium SSD/disk storage
- Backup storage (Azure Backup or custom)
- Monitoring (Azure Monitor, third-party)
- Licensing for commercial PostgreSQL distributions (if applicable)

---

### 4. Azure Cosmos DB for PostgreSQL ❌ (Deprecated for New Projects)

**⚠️ Important: This service is no longer supported for new projects.**

Azure Cosmos DB for PostgreSQL was a distributed PostgreSQL service powered by Citus. Microsoft has deprecated this offering and recommends:

1. **Azure Cosmos DB for NoSQL** – For massive distributed scale with 99.999% SLA
2. **Elastic Clusters in Flexible Server** – For sharded PostgreSQL (the Citus use case)

If you have existing Cosmos DB for PostgreSQL clusters, they continue to work, but plan migration to Flexible Server with Elastic Clusters.

---

### 5. Azure Database for PostgreSQL - Single Server ❌ (Retired)

**⚠️ This deployment option has been retired.**

Single Server was the original PaaS offering. It has been replaced by Flexible Server, which provides more features, better performance, and the same (or better) pricing.

If you have Single Server instances, Microsoft recommends migrating to Flexible Server using the [Azure Database Migration Service](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql).

---

## Feature Comparison Matrix

| Feature | Flexible Server | Elastic Clusters | VMs (IaaS) |
|---------|-----------------|------------------|------------|
| **Fully Managed** | ✅ | ✅ | ❌ |
| **Auto Patching** | ✅ | ✅ | ❌ |
| **Zone-Redundant HA** | ✅ | ✅ | Manual |
| **Horizontal Sharding** | ❌ | ✅ (Citus) | Manual |
| **Stop/Start (Cost Savings)** | ✅ | ❌ | ✅ (Deallocate) |
| **Built-in PgBouncer** | ✅ | ✅ | Manual |
| **Custom Extensions** | Limited | Limited | ✅ Any |
| **OS Access** | ❌ | ❌ | ✅ Full |
| **Geo-Redundant Backup** | ✅ | ✅ | Manual |
| **SLA** | Up to 99.99% | Up to 99.99% | VM SLA |

---

## Decision Flowchart

```
                    ┌─────────────────────────┐
                    │  Need PostgreSQL on     │
                    │        Azure?           │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │  Need full OS/engine    │
                    │       control?          │
                    └───────────┬─────────────┘
                                │
                   ┌────────────┴────────────┐
                   │ YES                 NO  │
                   ▼                         ▼
          ┌────────────────┐     ┌───────────────────────┐
          │ PostgreSQL on  │     │ Will data exceed      │
          │   Azure VMs    │     │ single-node capacity? │
          └────────────────┘     └───────────┬───────────┘
                                             │
                                ┌────────────┴────────────┐
                                │ YES                 NO  │
                                ▼                         ▼
                    ┌────────────────────┐    ┌────────────────────┐
                    │  Elastic Clusters  │    │  Flexible Server   │
                    │     (Citus)        │    │    (Standard)      │
                    └────────────────────┘    └────────────────────┘
```

---

## Migration Paths

### From On-Premises PostgreSQL
- **Recommended:** Use [Azure Database Migration Service](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql) for minimal downtime
- **Alternative:** `pg_dump` and `pg_restore` for offline migrations

### From Single Server (Retired)
- Migrate to Flexible Server using Azure DMS
- Plan ahead—Single Server is fully retired

### From Cosmos DB for PostgreSQL (Deprecated)
- Migrate to Flexible Server with Elastic Clusters enabled
- Citus extension provides the same sharding capabilities

---

## Pricing Overview

| Option | Billing Model | Cost Optimization Tips |
|--------|---------------|------------------------|
| **Flexible Server** | vCores + Storage + Backup | Use Burstable tier for dev/test; Stop when not in use |
| **Elastic Clusters** | Per-node (vCores + Storage) | Start small, add nodes as needed; Rebalance online |
| **VMs** | VM SKU + Disks + Backup | Reserved instances for production; B-series for dev |

Use the [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for detailed estimates.

---

## Conclusion

For most PostgreSQL workloads on Azure, **Azure Database for PostgreSQL - Flexible Server** is the recommended choice. It balances managed convenience with configuration flexibility.

If you're building a multi-tenant SaaS app or need to scale beyond a single node, enable **Elastic Clusters** to get the power of Citus.

Only go the **IaaS route (VMs)** if you have specific requirements that PaaS can't meet—and be prepared to handle the operational overhead.

Avoid deprecated options (Cosmos DB for PostgreSQL, Single Server) for new projects.

---

## References

- [Choose the right Azure Database for PostgreSQL hosting option](https://learn.microsoft.com/en-us/azure/postgresql/configure-maintain/overview-postgres-choose-server-options)
- [What is Azure Database for PostgreSQL?](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)
- [Elastic clusters in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-elastic-clusters)
- [Azure Database for PostgreSQL Pricing](https://azure.microsoft.com/pricing/details/postgresql/server/)
- [Migration Service Overview](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)

---

*This blog post was researched using Microsoft Learn documentation and published to help clarify Azure PostgreSQL hosting options.*
