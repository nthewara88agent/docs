# Azure PostgreSQL Hosting Options: A Complete Comparison Guide

*Published: 4 February 2026 | Updated: 5 February 2026*

Choosing the right PostgreSQL hosting option on Azure can be confusing. With multiple services available—some deprecated, some evolving—it's essential to understand what's current and what fits your workload. This guide breaks down all your options based on official Microsoft documentation.

## TL;DR - Quick Decision Guide

| Need | Recommended Option |
|------|-------------------|
| **Standard workloads** (web apps, APIs) | Azure Database for PostgreSQL - Flexible Server |
| **Horizontal scale** (SaaS, multi-tenant) | Flexible Server with Elastic Clusters (Citus) |
| **Ultra-scale, cloud-native** (up to 128TB, 3072 vCores) | Azure HorizonDB (Preview) |
| **Full control** (custom extensions, OS access) | PostgreSQL on Azure VMs |
| **Kubernetes-native, multi-cloud** | PostgreSQL on AKS (with operators) |
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

### 3. Azure HorizonDB 🆕 (Preview - Ultra-Scale Cloud-Native)

**Next-generation fully managed Postgres-compatible database for demanding workloads.**

Announced at Microsoft Ignite 2025, Azure HorizonDB is a cloud-native PostgreSQL-compatible database service designed for modern enterprise workloads requiring extreme scale. It features a new architecture with shared storage, scale-out compute, and tiered caching optimized for cloud applications.

> ⚠️ **Preview Status:** Azure HorizonDB is currently in limited preview. Apply for early access at [aka.ms/PreviewHorizonDB](https://aka.ms/PreviewHorizonDB).

**Key Features:**
- **Scale-Out Compute:** Up to 3,072 vCores across primary and replica nodes
- **Shared Storage:** Auto-scaling up to 128 TB with sub-millisecond multi-zone commit latencies
- **Performance:** Up to 3x more throughput than open-source Postgres for transactional workloads (per Microsoft's announcement)
- **Zone Redundancy:** Data replicated across availability zones by default
- **Near-Zero Downtime Maintenance:** Transparent maintenance operations
- **Enterprise Security:** Native Entra ID support, Private Endpoints, data encryption
- **Azure Defender Integration:** Additional protection for sensitive data

**AI Capabilities:**
- **DiskANN Vector Index with Filtering:** Advanced query predicate pushdowns directly into vector similarity search, providing performance improvements over pgvector HNSW while maintaining accuracy
- **Built-in AI Model Management:** Seamless integration with generative, embedding, and reranking models from Microsoft Foundry—zero configuration required

**Developer Tooling:**
- **PostgreSQL Extension for VS Code (GA):** GitHub Copilot context-aware of Postgres databases
- **Live Performance Monitoring:** One-click GitHub Copilot debugging from performance dashboard
- **Oracle Migration Tooling (Preview):** GitHub Copilot-powered Oracle-to-PostgreSQL code conversion in VS Code

**Available Regions (Preview):**
- Central US
- West US3
- UK South
- Australia East

**Best For:**
- Mission-critical enterprise applications requiring extreme scale
- AI applications with vector similarity search at scale
- Large-scale migrations from legacy databases (Oracle, etc.)
- Workloads requiring 128TB+ storage or thousands of vCores
- Applications needing sub-millisecond commit latencies across zones

**When to Choose Over Flexible Server:**
- Your workload exceeds Flexible Server's 64 TB storage limit
- You need more than Flexible Server's maximum vCores
- You require the advanced DiskANN vector indexing with predicate pushdown
- You're building AI applications that benefit from built-in model integration

**Considerations:**
- Currently in preview—not recommended for production without engaging Microsoft
- Limited region availability during preview
- Pricing not yet publicly documented

**Reference:** [Announcing Azure HorizonDB](https://techcommunity.microsoft.com/blog/adforpostgresql/announcing-azure-horizondb/4469710)

---

### 4. PostgreSQL on Azure VMs (IaaS) ⚠️ (Full Control)

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

### 5. PostgreSQL on Azure Kubernetes Service (AKS) ⚠️ (Kubernetes-Native)

**Self-managed PostgreSQL on Kubernetes using operators.**

For teams already running workloads on AKS and wanting a unified Kubernetes-native approach, running PostgreSQL on AKS with a Kubernetes operator is a viable option. This is **not** an anti-pattern, but it comes with higher operational responsibility.

**Popular Operators:**
- **[CloudNativePG](https://cloudnative-pg.io/)** — CNCF project, excellent for production
- **[Zalando Postgres Operator](https://github.com/zalando/postgres-operator)** — Battle-tested at Zalando
- **[Crunchy PGO](https://github.com/CrunchyData/postgres-operator)** — Enterprise-grade with Crunchy Data support
- **[Percona Operator](https://www.percona.com/software/percona-operators)** — Open-source, multi-database support

**Key Features:**
- **Kubernetes-Native:** Declarative CRDs, GitOps-friendly, Helm deployments
- **Multi-Cloud Portability:** Same operator works on AKS, EKS, GKE, on-prem
- **Operator-Managed HA:** Automatic failover, streaming replication
- **Integrated Backups:** pgBackRest, Velero, S3-compatible storage
- **Connection Pooling:** PgBouncer sidecars or built-in pooling

**Trade-offs:**

| Aspect | Flexible Server (PaaS) | PostgreSQL on AKS |
|--------|------------------------|-------------------|
| **Management** | Microsoft manages | You manage (with operators) |
| **HA/Failover** | Built-in, zone-redundant | Operator-dependent, you configure |
| **Backups** | Automated, geo-redundant | You configure (pgBackRest, Velero) |
| **Scaling** | Vertical (change tier) | Horizontal read replicas (operator) |
| **Portability** | Azure-only | Multi-cloud, on-prem |
| **SLA** | Up to 99.99% | Your design determines this |
| **Cost** | Predictable per-tier | Potentially cheaper with spot/bin-packing |

**When It Makes Sense:**
- Your entire stack already runs on Kubernetes
- You need multi-cloud or hybrid portability
- Your team has strong Kubernetes operational expertise
- You want unified GitOps deployments for app + database
- Cost optimization via spot nodes and bin-packing

**When It's an Anti-Pattern:**
- You're adopting Kubernetes *just* for the database
- Your team lacks Kubernetes operational experience
- You need a guaranteed SLA without managing HA yourself
- "Just need a database" without the K8s ecosystem benefits

**Best For:**
- Kubernetes-native organizations
- Multi-cloud or hybrid-cloud strategies
- Teams with platform engineering capabilities
- Startups wanting cost-efficient, portable infrastructure

---

### 6. Azure Cosmos DB for PostgreSQL ❌ (Deprecated for New Projects)

**⚠️ Important: This service is no longer supported for new projects.**

Azure Cosmos DB for PostgreSQL was a distributed PostgreSQL service powered by Citus. Microsoft has deprecated this offering and recommends:

1. **Azure Cosmos DB for NoSQL** – For massive distributed scale with 99.999% SLA
2. **Elastic Clusters in Flexible Server** – For sharded PostgreSQL (the Citus use case)

If you have existing Cosmos DB for PostgreSQL clusters, they continue to work, but plan migration to Flexible Server with Elastic Clusters.

---

### 7. Azure Database for PostgreSQL - Single Server ❌ (Retired)

**⚠️ This deployment option has been retired.**

Single Server was the original PaaS offering. It has been replaced by Flexible Server, which provides more features, better performance, and the same (or better) pricing.

If you have Single Server instances, Microsoft recommends migrating to Flexible Server using the [Azure Database Migration Service](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql).

---

## Feature Comparison Matrix

| Feature | Flexible Server | Elastic Clusters | HorizonDB (Preview) | VMs (IaaS) | AKS (Operators) |
|---------|-----------------|------------------|---------------------|------------|-----------------| 
| **Fully Managed** | ✅ | ✅ | ✅ | ❌ | ❌ (Operator-assisted) |
| **Auto Patching** | ✅ | ✅ | ✅ | ❌ | ❌ (You manage) |
| **Zone-Redundant HA** | ✅ | ✅ | ✅ (default) | Manual | Operator-managed |
| **Horizontal Sharding** | ❌ | ✅ (Citus) | ✅ (scale-out) | Manual | Operator-dependent |
| **Max Storage** | 64 TB | Per-node | 128 TB | Disk limits | Disk limits |
| **Max Compute** | Per-tier | Per-node × nodes | 3,072 vCores | VM SKU | Node pool |
| **Stop/Start (Cost Savings)** | ✅ | ❌ | TBD | ✅ (Deallocate) | ✅ (Scale to 0) |
| **Built-in PgBouncer** | ✅ | ✅ | TBD | Manual | Sidecar/Operator |
| **Custom Extensions** | Limited | Limited | TBD | ✅ Any | ✅ Any |
| **OS Access** | ❌ | ❌ | ❌ | ✅ Full | Container-level |
| **Geo-Redundant Backup** | ✅ | ✅ | ✅ | Manual | Manual (pgBackRest) |
| **Vector Index (AI)** | pgvector | pgvector | DiskANN + filtering | Any | Any |
| **Built-in AI Models** | ❌ | ❌ | ✅ (Foundry) | Manual | Manual |
| **Multi-Cloud Portable** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **SLA** | Up to 99.99% | Up to 99.99% | TBD (Preview) | VM SLA | Your design |
| **Production Ready** | ✅ | ✅ | ⚠️ Preview | ✅ | ✅ |

---

## Decision Flowchart

```
                    ┌─────────────────────────┐
                    │  Need PostgreSQL on     │
                    │        Azure?           │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │  Already running on     │
                    │  Kubernetes / AKS?      │
                    └───────────┬─────────────┘
                                │
               ┌────────────────┴────────────────┐
               │ YES                          NO │
               ▼                                 ▼
    ┌──────────────────────┐      ┌──────────────────────────┐
    │  Need multi-cloud    │      │  Need full OS/engine     │
    │  portability?        │      │       control?           │
    └──────────┬───────────┘      └───────────┬──────────────┘
               │                              │
      ┌────────┴────────┐          ┌──────────┴──────────┐
      │ YES         NO  │          │ YES             NO  │
      ▼                 ▼          ▼                     ▼
┌───────────┐   ┌────────────┐  ┌────────────┐  ┌─────────────────┐
│PostgreSQL │   │ Flexible   │  │PostgreSQL  │  │ Need ultra-scale│
│  on AKS   │   │  Server    │  │  on VMs    │  │ (>64TB, 3K+ vC)?│
│(Operators)│   │            │  │            │  └────────┬────────┘
└───────────┘   └────────────┘  └────────────┘           │
                                              ┌──────────┴──────────┐
                                              │ YES             NO  │
                                              ▼                     ▼
                                    ┌─────────────────┐  ┌─────────────────┐
                                    │  HorizonDB      │  │ Will data exceed│
                                    │  (Preview)      │  │ single-node?    │
                                    └─────────────────┘  └────────┬────────┘
                                                                  │
                                                       ┌──────────┴──────────┐
                                                       │ YES             NO  │
                                                       ▼                     ▼
                                              ┌────────────────┐  ┌────────────────┐
                                              │Elastic Clusters│  │Flexible Server │
                                              │    (Citus)     │  │   (Standard)   │
                                              └────────────────┘  └────────────────┘
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

### From Oracle (New)
- Azure HorizonDB offers GitHub Copilot-powered Oracle migration tooling (preview)
- Available in PostgreSQL Extension for VS Code

---

## Pricing Overview

| Option | Billing Model | Cost Optimization Tips |
|--------|---------------|------------------------|
| **Flexible Server** | vCores + Storage + Backup | Use Burstable tier for dev/test; Stop when not in use |
| **Elastic Clusters** | Per-node (vCores + Storage) | Start small, add nodes as needed; Rebalance online |
| **HorizonDB** | TBD (Preview) | Apply for preview access; pricing not yet public |
| **VMs** | VM SKU + Disks + Backup | Reserved instances for production; B-series for dev |
| **AKS** | Node pool VMs + Disks | Spot nodes for non-prod; bin-packing; scale to zero |

Use the [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for detailed estimates.

---

## Conclusion

For most PostgreSQL workloads on Azure, **Azure Database for PostgreSQL - Flexible Server** is the recommended choice. It balances managed convenience with configuration flexibility.

If you're building a multi-tenant SaaS app or need to scale beyond a single node, enable **Elastic Clusters** to get the power of Citus.

For **ultra-scale enterprise workloads** requiring 128TB storage, thousands of vCores, or advanced AI capabilities with DiskANN vector indexing, **Azure HorizonDB** (currently in preview) represents Microsoft's next-generation offering—though you should engage with Microsoft for preview access before committing.

For **Kubernetes-native organizations** already running on AKS with strong platform engineering capabilities, **PostgreSQL on AKS with operators** (CloudNativePG, Zalando, Crunchy PGO) is a solid choice—especially if you need multi-cloud portability.

Only go the **IaaS route (VMs)** if you have specific requirements that PaaS can't meet—and be prepared to handle the operational overhead.

Avoid deprecated options (Cosmos DB for PostgreSQL, Single Server) for new projects.

---

## References

- [Choose the right Azure Database for PostgreSQL hosting option](https://learn.microsoft.com/en-us/azure/postgresql/configure-maintain/overview-postgres-choose-server-options)
- [What is Azure Database for PostgreSQL?](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)
- [Elastic clusters in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-elastic-clusters)
- [Announcing Azure HorizonDB](https://techcommunity.microsoft.com/blog/adforpostgresql/announcing-azure-horizondb/4469710) — Microsoft Tech Community
- [Azure HorizonDB Preview Signup](https://aka.ms/PreviewHorizonDB)
- [Azure Database for PostgreSQL Pricing](https://azure.microsoft.com/pricing/details/postgresql/server/)
- [Migration Service Overview](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)
- [CloudNativePG](https://cloudnative-pg.io/) — CNCF PostgreSQL Operator
- [Zalando Postgres Operator](https://github.com/zalando/postgres-operator)
- [Crunchy PGO](https://github.com/CrunchyData/postgres-operator)

---

*This blog post was researched using Microsoft Learn documentation and official Microsoft announcements. Last updated to include Azure HorizonDB preview announced at Microsoft Ignite 2025.*

