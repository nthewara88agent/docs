---
title: "Self-Hosted PostgreSQL on AKS: A Complete Production Guide with CloudNativePG, Operators, and Barman Backup"
date: 2026-03-09
description: "The definitive guide to running production-grade self-hosted PostgreSQL on Azure Kubernetes Service (AKS) using CloudNativePG, with deep coverage of operators, high availability, Barman backup to Azure Blob Storage, storage optimization, security, monitoring, and day-2 operations."
tags: ["aks", "kubernetes", "postgresql", "cloudnativepg", "barman", "database", "azure", "cnpg"]
---

# Self-Hosted PostgreSQL on AKS: A Complete Production Guide with CloudNativePG, Operators, and Barman Backup

Running PostgreSQL on Kubernetes has matured dramatically. What was once a risky proposition — "friends don't let friends run databases on Kubernetes" — has become a well-trodden path with robust tooling, proven operators, and [official Microsoft documentation](https://learn.microsoft.com/azure/aks/postgresql-ha-overview) backing it up. In 2026, self-hosted PostgreSQL on AKS is a legitimate production choice, and this guide will show you exactly how to do it right.

## Table of Contents

- [1. Introduction — Why Self-Host PostgreSQL on AKS?](#1-introduction--why-self-host-postgresql-on-aks)
- [2. PostgreSQL Operators Landscape](#2-postgresql-operators-landscape)
- [3. CloudNativePG Deep Dive](#3-cloudnativepg-deep-dive)
- [4. Storage Considerations for PostgreSQL on AKS](#4-storage-considerations-for-postgresql-on-aks)
- [5. High Availability & Failover](#5-high-availability--failover)
- [6. Backup & Recovery with Barman](#6-backup--recovery-with-barman)
- [7. Connection Pooling with PgBouncer](#7-connection-pooling-with-pgbouncer)
- [8. Security](#8-security)
- [9. Monitoring & Observability](#9-monitoring--observability)
- [10. Day-2 Operations](#10-day-2-operations)
- [11. Cost Optimization](#11-cost-optimization)
- [12. Complete Reference Architecture](#12-complete-reference-architecture)
- [13. Summary & Recommendations](#13-summary--recommendations)

---

## 1. Introduction — Why Self-Host PostgreSQL on AKS?

Azure Database for PostgreSQL Flexible Server is excellent — managed patching, built-in HA, automated backups. For many workloads, it's the right answer. But there are compelling reasons to self-host:

**Custom extensions and version control.** Need `pgvector` at a specific version, `TimescaleDB`, `PostGIS`, `pg_partman`, or a custom C extension? Managed services limit which extensions you can install and when you can upgrade. Self-hosted gives you full control over the PostgreSQL binary, extensions, and configuration.

**Compliance and data sovereignty.** Some regulatory frameworks require that database processes run within specific network boundaries, on dedicated hardware, or with specific encryption configurations that managed services can't accommodate. Running on dedicated AKS node pools with Azure Dedicated Host gives you hardware-level isolation.

**Cost at scale.** A 64-vCPU, 256-GB Flexible Server instance costs significantly more than equivalent compute on AKS nodes. At scale (multiple large databases), self-hosting on reserved VM instances can cut costs by 40-60%.

**Multi-cloud portability.** CloudNativePG manifests work identically on AKS, EKS, GKE, and bare-metal Kubernetes. If you're building for multi-cloud or hybrid scenarios, self-hosting avoids vendor lock-in to a specific managed database service.

**Kubernetes-native operations.** If your team already operates everything via GitOps (ArgoCD/Flux), having PostgreSQL as a Kubernetes-native resource with declarative YAML fits naturally into existing workflows. No separate management plane, no portal clicking.

**What this guide covers:** We'll walk through the entire stack — choosing an operator, deploying CloudNativePG, configuring storage, achieving high availability across AKS availability zones, setting up Barman backups to Azure Blob Storage, connection pooling, security hardening, monitoring, day-2 operations, cost optimization, and a complete reference architecture with production-ready manifests.

---

## 2. PostgreSQL Operators Landscape

### Why You Need an Operator

Don't deploy PostgreSQL as a raw StatefulSet. A bare StatefulSet gives you ordered pod creation and stable network identities, but it knows nothing about PostgreSQL. It won't:

- Configure streaming replication between primary and replicas
- Handle automatic failover when the primary dies
- Manage WAL archiving and backup schedules
- Orchestrate rolling updates with minimal downtime
- Handle certificate rotation or connection pooling

A PostgreSQL operator encapsulates all this operational knowledge as code. It watches your declarative cluster spec and continuously reconciles the actual state to match — the Kubernetes way.

### CloudNativePG (CNPG)

[CloudNativePG](https://cloudnative-pg.io/) is a [CNCF Sandbox project](https://www.cncf.io/projects/cloudnativepg/) and the leading PostgreSQL operator for Kubernetes in 2026. It's the operator [Microsoft uses in their official AKS PostgreSQL HA documentation](https://learn.microsoft.com/azure/aks/deploy-postgresql-ha).

**Architecture & Philosophy:**
- **No external dependencies.** CNPG doesn't require etcd, Consul, or any external consensus store for leader election. It uses Kubernetes primitives (leases) natively.
- **One operator, one CRD.** The `Cluster` CRD is the single source of truth. Define your desired state, and the operator makes it happen.
- **Direct integration with Kubernetes.** Pods are managed directly (not via StatefulSet), giving the operator fine-grained control over failover, fencing, and instance lifecycle.
- **Barman Cloud built-in.** Backup and WAL archiving to object storage (Azure Blob, S3, GCS) is a first-class feature, not a bolt-on.
- **Kubernetes-native TLS.** Automatic certificate management for server and replication certificates, with optional cert-manager integration.

**Key Features (v1.25+):**
- Declarative PostgreSQL 13-18 clusters (primary + replicas)
- Automated failover with configurable quorum
- Continuous WAL archiving and scheduled base backups
- Point-in-time recovery (PITR)
- Built-in PgBouncer connection pooling
- Rolling updates with configurable maintenance windows
- Online volume expansion
- Prometheus metrics endpoint on every instance
- Fencing and hibernation for maintenance scenarios
- Plugin architecture for extensibility (Barman Cloud plugin)

### Zalando PostgreSQL Operator (Patroni-based)

The [Zalando Postgres Operator](https://github.com/zalando/postgres-operator) was one of the first PostgreSQL operators for Kubernetes, battle-tested at Zalando's scale.

**Architecture:**
- Uses [Patroni](https://github.com/patroni/patroni) for HA and leader election
- Runs PostgreSQL inside [Spilo](https://github.com/zalando/spilo) containers (Patroni + PostgreSQL + WAL-E/WAL-G)
- Relies on Kubernetes endpoints or etcd for Patroni's DCS (Distributed Configuration Store)
- Manages clusters via a `postgresql` CRD

**When to choose Zalando:**
- You're already invested in Patroni and understand its operational model
- You need WAL-G for backup (rather than Barman)
- Your team has existing Spilo expertise
- You want the connection pooler as a separate deployment (uses their own connection pooler or external PgBouncer)

**Trade-offs:** More moving parts (Patroni + DCS + Spilo), less active development compared to CNPG, and the operator itself is less deeply integrated with Kubernetes primitives.

### Crunchy Data PGO (PostgreSQL Operator)

[Crunchy PGO](https://github.com/CrunchyData/postgres-operator) (v5+) is an enterprise-focused operator backed by Crunchy Data.

**Architecture:**
- Uses Patroni for HA (similar to Zalando)
- Manages PostgreSQL clusters via `PostgresCluster` CRD
- Built-in pgBackRest for backup and restore
- Includes PgBouncer integration
- pgMonitor for monitoring

**When to choose Crunchy PGO:**
- You want enterprise support from Crunchy Data
- pgBackRest is your preferred backup tool
- You need features like pgAdmin integration or the Crunchy Bridge UI
- Your organization requires a commercial support contract

**Trade-offs:** More opinionated, heavier footprint, some features require Crunchy's commercial offerings.

### Operator Comparison

| Feature | CloudNativePG | Zalando Operator | Crunchy PGO |
|---|---|---|---|
| **CNCF Status** | Sandbox ✅ | Community | Community |
| **HA Mechanism** | Kubernetes-native (no external DCS) | Patroni + DCS | Patroni + DCS |
| **Backup Tool** | Barman Cloud | WAL-E/WAL-G | pgBackRest |
| **Connection Pooler** | Built-in PgBouncer | Separate deployment | Built-in PgBouncer |
| **TLS Management** | Built-in + cert-manager | Manual / Spilo | Built-in |
| **Azure Blob Backup** | Native (Barman Cloud) | Via WAL-G config | Via pgBackRest |
| **Workload Identity** | `inheritFromAzureAD` | Manual config | Manual config |
| **Microsoft Docs** | [Official AKS guide](https://learn.microsoft.com/azure/aks/postgresql-ha-overview) | ❌ | ❌ |
| **Pod Management** | Direct (not StatefulSet) | StatefulSet | StatefulSet |
| **Active Development** | Very active (weekly releases) | Moderate | Active |
| **License** | Apache 2.0 | MIT | Apache 2.0 |

### Recommendation

**Use CloudNativePG** unless you have a specific reason not to. It's the most Kubernetes-native option, has the best Azure integration (Microsoft literally uses it in their docs), requires the fewest external dependencies, and has the most active development community. The rest of this guide focuses on CNPG.

---

## 3. CloudNativePG Deep Dive

### Installation on AKS

Install the CloudNativePG operator via Helm:

```bash
# Add the CloudNativePG Helm repository
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

# Install the operator
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace \
  --set monitoring.podMonitorEnabled=true \
  --set monitoring.grafanaDashboard.create=true
```

Also install the `kubectl cnpg` plugin via [Krew](https://krew.sigs.k8s.io/) — it's invaluable for day-to-day operations:

```bash
kubectl krew install cnpg

# Verify installation
kubectl cnpg version
```

### The Cluster CRD

The `Cluster` CRD is the heart of CloudNativePG. Here's a minimal production cluster:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production
  namespace: cnpg-database
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  bootstrap:
    initdb:
      database: appdb
      owner: app
      dataChecksums: true

  storage:
    storageClass: managed-csi-premium
    size: 100Gi

  resources:
    requests:
      memory: "8Gi"
      cpu: 2
    limits:
      memory: "8Gi"

  postgresql:
    parameters:
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      max_connections: "200"
```

### Instance Management

CNPG manages pods **directly** — not through a StatefulSet. This gives the operator fine-grained control:

- **Primary election:** The operator designates one instance as primary and configures all others as streaming replicas.
- **Failover:** When the primary fails, the operator promotes the most up-to-date replica. No external tool (Patroni, etcd) is involved.
- **Fencing:** The operator can fence (isolate) misbehaving instances without destroying them, allowing investigation.

Each instance gets:
- A dedicated PVC for PGDATA
- An optional dedicated PVC for WAL (`walStorage`)
- A service account for workload identity
- TLS certificates (auto-managed or user-provided)

### Declarative Configuration

All PostgreSQL configuration is declarative in the `Cluster` spec:

```yaml
postgresql:
  parameters:
    shared_buffers: "4GB"
    effective_cache_size: "12GB"
    work_mem: "64MB"
    maintenance_work_mem: "1GB"
    max_connections: "300"
    wal_compression: "lz4"
    max_wal_size: "6GB"
    checkpoint_timeout: "15min"
    checkpoint_completion_target: "0.9"
    random_page_cost: "1.1"
    effective_io_concurrency: "64"
    log_checkpoints: "on"
    log_min_duration_statement: "1000"
    pg_stat_statements.max: "10000"
    pg_stat_statements.track: "all"
  pg_hba:
    - host all all all scram-sha-256
```

Changes to `postgresql.parameters` that require a restart trigger an automated rolling restart — replicas first, then a controlled switchover to update the (former) primary.

### PostgreSQL Version Management

CNPG manages versions through the container image tag:

```yaml
# Minor version upgrade — rolling restart
imageName: ghcr.io/cloudnative-pg/postgresql:17.4  # was 17.3

# Major version — requires import/logical replication (see Day-2 Operations)
imageName: ghcr.io/cloudnative-pg/postgresql:18.0
```

Minor version updates are applied via rolling restart automatically. The operator updates replicas first, performs a switchover, then updates the old primary — achieving near-zero downtime.

### Resource Requests and Limits Best Practices

For PostgreSQL workloads, set **Guaranteed QoS** (requests == limits for memory):

```yaml
resources:
  requests:
    memory: "8Gi"
    cpu: 2
  limits:
    memory: "8Gi"
    # CPU limit deliberately omitted — avoid CPU throttling
```

**Why no CPU limit?** PostgreSQL is sensitive to CPU throttling. Setting a CPU limit can cause query latency spikes when the CFS scheduler throttles the process. Set a CPU request for scheduling purposes, but omit the limit (or set it high) to allow bursting.

**Memory sizing guideline:** `shared_buffers` should be ~25% of pod memory. `effective_cache_size` should be ~75%. Leave headroom for OS page cache and connection overhead.

---

## 4. Storage Considerations for PostgreSQL on AKS

Storage is the most critical infrastructure decision for database workloads. Get this wrong, and no amount of PostgreSQL tuning will save you.

### Azure Disk Types

| Disk Type | IOPS (max) | Throughput (max) | Latency | Best For |
|---|---|---|---|---|
| **Premium SSD (P-series)** | 20,000 | 900 MB/s | Single-digit ms | Most PostgreSQL workloads |
| **Premium SSD v2** | 80,000 | 1,200 MB/s | Sub-millisecond | High-performance OLTP |
| **Ultra Disk** | 160,000 | 4,000 MB/s | Sub-millisecond | Extreme I/O requirements |

**Premium SSD v2** is the sweet spot for PostgreSQL on AKS. Unlike Premium SSD (where IOPS and throughput are tied to disk size), [Premium SSD v2 lets you independently configure capacity, IOPS, and throughput](https://learn.microsoft.com/azure/aks/use-premium-v2-disks). You can provision a 100-GiB disk with 10,000 IOPS without paying for a 1-TiB P30.

### Storage Class Configuration

**Premium SSD v2 StorageClass (recommended for production):**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-v2-postgresql
parameters:
  cachingMode: None          # Required for Premium SSD v2
  skuName: PremiumV2_LRS
  DiskIOPSReadWrite: "5000"  # Adjust per workload
  DiskMBpsReadWrite: "200"   # Adjust per workload
provisioner: disk.csi.azure.com
reclaimPolicy: Retain        # CRITICAL for databases — never Delete
volumeBindingMode: WaitForFirstConsumer  # Zone-aware binding
allowVolumeExpansion: true
```

**Premium SSD StorageClass (simpler, good default):**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-postgresql
parameters:
  cachingMode: ReadOnly      # Beneficial for read-heavy workloads
  skuName: Premium_LRS
provisioner: disk.csi.azure.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

> **Critical:** Always use `reclaimPolicy: Retain` for database storage. The default `Delete` policy destroys the Azure Disk when the PVC is deleted — that's your data gone forever.

### WAL Volume Separation

For high-throughput workloads, separate PostgreSQL WAL (Write-Ahead Log) onto a dedicated volume. WAL writes are sequential and latency-sensitive; data writes are more random. Separating them prevents I/O contention:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production
spec:
  # ... other config ...

  storage:
    storageClass: premium-ssd-v2-postgresql
    size: 200Gi

  walStorage:
    storageClass: premium-ssd-v2-wal
    size: 50Gi
```

Create a separate StorageClass for WAL with higher IOPS:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-v2-wal
parameters:
  cachingMode: None
  skuName: PremiumV2_LRS
  DiskIOPSReadWrite: "10000"   # Higher IOPS for WAL
  DiskMBpsReadWrite: "400"
provisioner: disk.csi.azure.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

That said, Microsoft's own AKS CNPG guide notes that for many workloads, keeping data and WAL on the same volume with adequate IOPS is simpler and provides better throughput tiers on managed disks. **Separate WAL only when benchmarks show I/O contention.**

### IOPS Sizing Guidelines

| Workload Type | Recommended IOPS | Recommended Throughput |
|---|---|---|
| Light OLTP (< 100 TPS) | 3,000 - 5,000 | 125 - 200 MB/s |
| Medium OLTP (100-500 TPS) | 5,000 - 10,000 | 200 - 400 MB/s |
| Heavy OLTP (500+ TPS) | 10,000 - 30,000 | 400 - 800 MB/s |
| Analytics / Reporting | 5,000 - 15,000 | 400 - 1,200 MB/s |

### Volume Expansion

CloudNativePG supports online volume expansion. Simply update the `size` in your Cluster spec:

```yaml
storage:
  size: 400Gi  # Was 200Gi — expanded online
```

The operator requests PVC expansion, and the Azure Disk CSI driver handles the resize without pod restart (for Premium SSD v2 and Premium SSD with CSI driver v1.27+).

### Anti-Patterns to Avoid

- ❌ **Using `reclaimPolicy: Delete`** — One accidental `kubectl delete pvc` and your data is gone
- ❌ **Using `ReadWriteMany` (Azure Files/NFS)** — PostgreSQL requires block storage with `ReadWriteOnce`. Never run PostgreSQL on Azure Files
- ❌ **Ignoring `volumeBindingMode: WaitForFirstConsumer`** — Without this, the disk might be provisioned in a different zone than the pod, causing scheduling failures
- ❌ **Using Standard HDD/SSD** — The latency will kill your database performance
- ❌ **Over-provisioning Premium SSD for IOPS** — If you need 5,000 IOPS, use Premium SSD v2 instead of a 1-TiB P30

---

## 5. High Availability & Failover

### How CloudNativePG Handles HA

CloudNativePG provides built-in HA without any external tooling. No Patroni, no etcd, no Consul — just the operator and Kubernetes primitives.

**Streaming Replication Architecture:**

```
┌──────────────────────────────────────────────────┐
│                  CNPG Operator                     │
│          (watches Cluster CRD, manages pods)       │
└──────────────┬───────────────────┬─────────────────┘
               │                   │
    ┌──────────▼──────────┐       │
    │   Primary (Zone 1)  │       │
    │   pg-production-1   │       │
    │   ┌──────────────┐  │       │
    │   │  PostgreSQL   │──┼───────┼──── Streaming Replication ────┐
    │   │  (read-write) │  │       │                               │
    │   └──────────────┘  │       │                               │
    │   PVC: 200Gi        │       │                               │
    └─────────────────────┘       │                               │
                                  │                               │
    ┌──────────▼──────────┐    ┌──▼──────────────────┐
    │  Replica (Zone 2)   │    │  Replica (Zone 3)   │
    │  pg-production-2    │    │  pg-production-3    │
    │  ┌──────────────┐   │    │  ┌──────────────┐   │
    │  │  PostgreSQL   │   │    │  │  PostgreSQL   │   │
    │  │  (read-only)  │   │    │  │  (read-only)  │   │
    │  └──────────────┘   │    │  └──────────────┘   │
    │  PVC: 200Gi         │    │  PVC: 200Gi         │
    └─────────────────────┘    └─────────────────────┘
```

**Services created automatically:**
- `pg-production-rw` — Always points to the primary (read-write)
- `pg-production-ro` — Load-balances across replicas (read-only)
- `pg-production-r` — All instances (primary + replicas)

### Automatic Failover

When the primary pod fails:

1. The operator detects the failure via health checks
2. It selects the most up-to-date replica (lowest replication lag)
3. Promotes the replica to primary (`pg_promote()`)
4. Updates the `-rw` service endpoint to point to the new primary
5. Reconfigures remaining replicas to follow the new primary

This typically completes in **10-30 seconds**. With `alpha.cnpg.io/failoverQuorum: "true"`, failover requires a quorum of instances to be reachable, preventing split-brain scenarios.

### Synchronous vs Asynchronous Replication

```yaml
postgresql:
  synchronous:
    method: any    # "any" or "first"
    number: 1      # Number of synchronous replicas
```

- **Asynchronous (default):** Primary doesn't wait for replicas to confirm writes. Fastest, but potential data loss on failover (RPO > 0).
- **Synchronous (`number: 1`):** Primary waits for at least one replica to confirm each commit. Zero data loss on failover (RPO = 0), but adds ~1-2ms write latency.

**Recommendation:** Use synchronous replication with `number: 1` for production workloads where data durability matters. The latency impact is minimal with AKS zones (intra-region latency < 2ms).

### Topology Spread and Anti-Affinity

Spread PostgreSQL instances across availability zones for zone-level resilience:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          cnpg.io/cluster: pg-production

  affinity:
    nodeSelector:
      workload: postgres
    tolerations:
      - key: "workload"
        operator: "Equal"
        value: "postgres"
        effect: "NoSchedule"
```

This ensures:
- Each PostgreSQL instance runs in a different AZ
- Instances only schedule on nodes labeled `workload=postgres`
- A zone failure takes out at most one instance

### Dedicated Node Pool for PostgreSQL

Create a dedicated AKS user node pool for PostgreSQL workloads. This prevents noisy-neighbor issues and allows you to right-size the VM SKU:

```bash
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name postgres \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --zones 1 2 3 \
  --labels workload=postgres \
  --node-taints workload=postgres:NoSchedule \
  --max-pods 30 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5
```

**Why Standard_D4s_v3?** 4 vCPUs, 16 GiB RAM — a solid baseline for PostgreSQL. It supports Premium Storage and has good memory-to-CPU ratio. Scale up to `Standard_D8s_v3` or `Standard_E8s_v3` (memory-optimized) for larger databases.

### Pod Disruption Budgets

CNPG automatically creates a PDB that ensures at least one instance is always available during voluntary disruptions (node upgrades, scale-downs). You can verify this:

```bash
kubectl get pdb -n cnpg-database
# NAME                   MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# pg-production          1               N/A               1
```

### RTO and RPO

| Configuration | RPO | RTO |
|---|---|---|
| Async replication + Barman backup | Seconds to minutes (WAL gap) | 10-30s (failover) |
| Sync replication (`number: 1`) + Barman | **Zero** (committed transactions) | 10-30s (failover) |
| Single instance + Barman backup | Minutes to hours (last WAL archived) | Minutes (PITR from backup) |

---

## 6. Backup & Recovery with Barman

### Barman Cloud Overview

[Barman](https://pgbarman.org/) is the most widely used PostgreSQL backup tool. CloudNativePG integrates Barman Cloud directly into the operator, enabling:

- **Continuous WAL archiving** to object storage (Azure Blob, S3, GCS)
- **Scheduled base backups** via cron expressions
- **Point-in-time recovery (PITR)** to any second within your retention window
- **Authentication via Azure Workload Identity** — no storage keys needed

### Setting Up Azure Blob Storage for Backups

First, create the storage account and container:

```bash
# Variables
export RESOURCE_GROUP="rg-cnpg"
export STORAGE_ACCOUNT="cnpgbackups$(openssl rand -hex 4)"
export CONTAINER_NAME="backups"
export LOCATION="australiaeast"

# Create storage account (ZRS for zone redundancy)
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_ZRS \
  --kind StorageV2

# Create backup container
az storage container create \
  --name $CONTAINER_NAME \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

### Configuring Workload Identity for Backup

CNPG uses Azure Workload Identity to authenticate to Blob Storage without secrets. This is the recommended approach:

```bash
# Create user-assigned managed identity
az identity create \
  --name mi-cnpg-backup \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Get identity details
export IDENTITY_CLIENT_ID=$(az identity show \
  --name mi-cnpg-backup \
  --resource-group $RESOURCE_GROUP \
  --query clientId -o tsv)

export IDENTITY_OBJECT_ID=$(az identity show \
  --name mi-cnpg-backup \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Assign Storage Blob Data Contributor role
export STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $IDENTITY_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $STORAGE_ID

# Get AKS OIDC issuer
export OIDC_ISSUER=$(az aks show \
  --name myAKSCluster \
  --resource-group $RESOURCE_GROUP \
  --query oidcIssuerProfile.issuerUrl -o tsv)

# Create federated credential
# Note: CNPG creates a service account with the same name as the cluster
az identity federated-credential create \
  --name cnpg-fedcred \
  --identity-name mi-cnpg-backup \
  --resource-group $RESOURCE_GROUP \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:cnpg-database:pg-production" \
  --audience api://AzureADTokenExchange
```

### Complete Cluster with Backup Configuration

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production
  namespace: cnpg-database
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  inheritedMetadata:
    labels:
      azure.workload.identity/use: "true"

  bootstrap:
    initdb:
      database: appdb
      owner: app
      dataChecksums: true

  storage:
    storageClass: premium-ssd-v2-postgresql
    size: 200Gi

  resources:
    requests:
      memory: "8Gi"
      cpu: 2
    limits:
      memory: "8Gi"

  postgresql:
    synchronous:
      method: any
      number: 1
    parameters:
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      wal_compression: "lz4"
      max_wal_size: "6GB"
      checkpoint_timeout: "15min"
      checkpoint_completion_target: "0.9"
      pg_stat_statements.max: "10000"
      pg_stat_statements.track: "all"
    pg_hba:
      - hostssl all all all scram-sha-256

  serviceAccountTemplate:
    metadata:
      annotations:
        azure.workload.identity/client-id: "<YOUR_IDENTITY_CLIENT_ID>"
      labels:
        azure.workload.identity/use: "true"

  backup:
    barmanObjectStore:
      destinationPath: "https://<STORAGE_ACCOUNT>.blob.core.windows.net/backups"
      azureCredentials:
        inheritFromAzureAD: true
    retentionPolicy: "30d"

  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          cnpg.io/cluster: pg-production

  affinity:
    nodeSelector:
      workload: postgres
```

### Scheduled Backups

Create a `ScheduledBackup` resource for automated base backups:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: pg-production-backup
  namespace: cnpg-database
spec:
  schedule: "0 0 2 * * *"    # Daily at 2:00 AM
  backupOwnerReference: self
  cluster:
    name: pg-production
  method: barmanObjectStore
  immediate: true              # Take a backup immediately on creation
```

### On-Demand Backup

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pg-production-backup-manual
  namespace: cnpg-database
spec:
  method: barmanObjectStore
  cluster:
    name: pg-production
```

```bash
# Or via the kubectl plugin
kubectl cnpg backup pg-production -n cnpg-database
```

### Point-in-Time Recovery (PITR)

Restore to a specific point in time by creating a new cluster from backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production-restored
  namespace: cnpg-database
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  bootstrap:
    recovery:
      source: pg-production-backup
      recoveryTarget:
        targetTime: "2026-03-09T10:30:00Z"  # Restore to this exact time

  storage:
    storageClass: premium-ssd-v2-postgresql
    size: 200Gi

  externalClusters:
    - name: pg-production-backup
      barmanObjectStore:
        destinationPath: "https://<STORAGE_ACCOUNT>.blob.core.windows.net/backups"
        azureCredentials:
          inheritFromAzureAD: true

  serviceAccountTemplate:
    metadata:
      annotations:
        azure.workload.identity/client-id: "<YOUR_IDENTITY_CLIENT_ID>"
      labels:
        azure.workload.identity/use: "true"
```

### Backup Retention Policies

```yaml
backup:
  retentionPolicy: "30d"   # Keep backups for 30 days
```

CNPG automatically cleans up base backups and WAL files older than the retention period.

### Cross-Region Backup for DR

For disaster recovery, replicate backups to a secondary region using [Azure Storage Object Replication](https://learn.microsoft.com/azure/storage/blobs/object-replication-overview) or configure a second `barmanObjectStore` destination in a different region:

```bash
# Create DR storage account in a secondary region
az storage account create \
  --name cnpgbackupsdr \
  --resource-group $RESOURCE_GROUP \
  --location southeastasia \
  --sku Standard_ZRS \
  --kind StorageV2

# Enable object replication between accounts
az storage account or-policy create \
  --account-name cnpgbackupsdr \
  --resource-group $RESOURCE_GROUP \
  --source-account $STORAGE_ACCOUNT \
  --destination-account cnpgbackupsdr \
  --source-container backups \
  --destination-container backups
```

### Verifying Backups

Always verify your backups. Use the kubectl plugin:

```bash
# List backups
kubectl cnpg backup list pg-production -n cnpg-database

# Check backup status
kubectl get backups -n cnpg-database

# Check WAL archiving status
kubectl cnpg status pg-production -n cnpg-database
```

---

## 7. Connection Pooling with PgBouncer

### Why Connection Pooling Matters on Kubernetes

In Kubernetes, application pods are ephemeral. They scale up/down, restart, and get rescheduled frequently. Without connection pooling:

- Each pod opening 10 connections × 50 pods = 500 PostgreSQL backend connections
- PostgreSQL performance degrades significantly above ~200-300 connections
- Each backend connection consumes ~10 MB of memory
- Connection establishment takes ~5ms per connection (TLS handshake)

PgBouncer sits between your applications and PostgreSQL, maintaining a pool of reusable connections and multiplexing application connections onto a smaller set of database connections.

### CloudNativePG Built-in PgBouncer

CNPG includes native PgBouncer pooler integration. Simply add a `Pooler` resource:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pg-production-pooler-rw
  namespace: cnpg-database
spec:
  cluster:
    name: pg-production
  instances: 2                    # Number of PgBouncer pods
  type: rw                        # "rw" for read-write, "ro" for read-only

  pgbouncer:
    poolMode: transaction         # transaction | session | statement
    parameters:
      max_client_conn: "1000"
      default_pool_size: "25"
      min_pool_size: "5"
      reserve_pool_size: "5"
      reserve_pool_timeout: "3"
      server_idle_timeout: "300"
      server_lifetime: "3600"
      log_connections: "1"
      log_disconnections: "1"

  template:
    metadata:
      labels:
        app: pg-pooler
    spec:
      containers:
        - name: pgbouncer
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

**Also create a read-only pooler for read replicas:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pg-production-pooler-ro
  namespace: cnpg-database
spec:
  cluster:
    name: pg-production
  instances: 2
  type: ro

  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "2000"
      default_pool_size: "50"
```

### Pool Mode Selection

| Mode | Description | Use Case |
|---|---|---|
| **Transaction** | Connection returned to pool after each transaction | Most applications (Django, Rails, microservices) |
| **Session** | Connection held for entire client session | Apps using prepared statements, LISTEN/NOTIFY, temp tables |
| **Statement** | Connection returned after each statement | Simple query workloads only (rare) |

**Use `transaction` mode** unless your application requires session-level features. It provides the best connection multiplexing ratio.

### Sizing PgBouncer

- **`default_pool_size`**: Number of server connections per user/database pair. Start with 25, increase if you see `cl_waiting` > 0.
- **`max_client_conn`**: Maximum client connections PgBouncer accepts. Set this high (1000+) — client connections are cheap.
- **`reserve_pool_size`**: Extra connections available when the pool is exhausted. Set to 5-10.
- **Instances**: Run at least 2 PgBouncer pods for availability. Scale based on connection count.

Applications connect to the Pooler service instead of the Cluster service:

```
# Read-write through pooler
pg-production-pooler-rw.cnpg-database.svc:5432

# Read-only through pooler
pg-production-pooler-ro.cnpg-database.svc:5432
```

---

## 8. Security

### TLS Encryption

CloudNativePG manages TLS certificates automatically. By default, the operator creates a self-signed CA and issues certificates for:

- Server TLS (client-to-server encryption)
- Streaming replication (inter-node encryption)
- Client authentication (certificate-based auth)

Certificates are automatically rotated 7 days before their 90-day expiry.

### cert-manager Integration

For production, integrate with [cert-manager](https://cert-manager.io/) for certificate management:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production
spec:
  certificates:
    serverCASecret: pg-server-ca
    serverTLSSecret: pg-server-tls
    clientCASecret: pg-client-ca
    replicationTLSSecret: pg-replication-tls
```

With cert-manager, you can use your organization's CA or integrate with Azure Key Vault for certificate storage.

### Authentication

**SCRAM-SHA-256 (recommended):**

```yaml
postgresql:
  pg_hba:
    - hostssl all all all scram-sha-256
```

**Certificate-based authentication (most secure):**

```yaml
postgresql:
  pg_hba:
    - hostssl all all all cert
```

### Network Policies

Restrict network access to PostgreSQL pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pg-production-network-policy
  namespace: cnpg-database
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: pg-production
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow connections from application namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: app-namespace
      ports:
        - protocol: TCP
          port: 5432
    # Allow replication between cluster members
    - from:
        - podSelector:
            matchLabels:
              cnpg.io/cluster: pg-production
      ports:
        - protocol: TCP
          port: 5432
    # Allow Prometheus scraping
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9187
  egress:
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow replication between cluster members
    - to:
        - podSelector:
            matchLabels:
              cnpg.io/cluster: pg-production
      ports:
        - protocol: TCP
          port: 5432
    # Allow backup to Azure Blob Storage
    - to: []
      ports:
        - protocol: TCP
          port: 443
```

### Azure Workload Identity for Blob Storage

As shown in the backup section, CNPG uses `inheritFromAzureAD: true` to authenticate to Azure Blob Storage via Workload Identity. This eliminates the need for storage account keys or SAS tokens.

The key components:
1. **User-assigned managed identity** with `Storage Blob Data Contributor` role on the storage account
2. **Federated credential** linking the AKS OIDC issuer to the managed identity
3. **Service account template** in the Cluster spec with the workload identity annotation

### Secrets Management

For database user passwords, use the [External Secrets Operator](https://external-secrets.io/) with Azure Key Vault:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-user-pass
  namespace: cnpg-database
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: db-user-pass
  data:
    - secretKey: username
      remoteRef:
        key: pg-app-username
    - secretKey: password
      remoteRef:
        key: pg-app-password
```

### Pod Security Standards

Ensure PostgreSQL pods run with least privilege:

```yaml
# The CNPG operator sets these automatically, but verify:
securityContext:
  runAsNonRoot: true
  runAsUser: 26        # postgres user
  fsGroup: 26
  seccompProfile:
    type: RuntimeDefault
```

---

## 9. Monitoring & Observability

### Built-in Prometheus Metrics

Every CloudNativePG instance exposes a metrics endpoint on port `9187`. The operator can create `PodMonitor` resources automatically (or you create them manually as [recommended in recent CNPG versions](https://learn.microsoft.com/azure/aks/deploy-postgresql-ha)):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pg-production
  namespace: cnpg-database
  labels:
    cnpg.io/cluster: pg-production
spec:
  selector:
    matchLabels:
      cnpg.io/cluster: pg-production
  podMetricsEndpoints:
    - port: metrics
```

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|---|---|---|
| `cnpg_pg_replication_lag` | Replication lag in bytes | > 100MB (warn), > 1GB (critical) |
| `cnpg_pg_stat_activity_count` | Active connections | > 80% of `max_connections` |
| `cnpg_pg_stat_bgwriter_buffers_checkpoint` | Checkpoint buffer writes | Sudden spikes |
| `cnpg_pg_database_size_bytes` | Database size | > 80% of PVC capacity |
| `cnpg_pg_stat_archiver_failed_count` | Failed WAL archives | > 0 |
| `cnpg_pg_stat_archiver_last_archive_age` | Time since last WAL archive | > 300s |
| `cnpg_pg_stat_user_tables_seq_scan` | Sequential scans | High for large tables |
| `pg_stat_statements` queries | Slow query tracking | > 1s avg execution |

### Grafana Dashboards

CloudNativePG provides [official Grafana dashboards](https://grafana.com/grafana/dashboards/20417-cloudnativepg/). When installing the Helm chart with `monitoring.grafanaDashboard.create=true`, dashboards are automatically provisioned.

For Azure Managed Grafana, Microsoft's AKS guide sets up monitoring infrastructure with:

```bash
# Create Azure Managed Grafana
az grafana create \
  --name grafana-cnpg \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --zone-redundancy Enabled

# Create Azure Monitor workspace
az monitor account create \
  --name amw-cnpg \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Link to AKS cluster during creation
az aks create \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id $AMW_RESOURCE_ID \
  --grafana-resource-id $GRAFANA_RESOURCE_ID \
  # ... other flags
```

### Alerting Rules

Example PrometheusRule for critical PostgreSQL alerts:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pg-production-alerts
  namespace: cnpg-database
spec:
  groups:
    - name: postgresql.rules
      rules:
        - alert: PostgreSQLReplicationLagHigh
          expr: cnpg_pg_replication_lag > 1073741824  # 1GB
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PostgreSQL replication lag > 1GB"

        - alert: PostgreSQLConnectionsHigh
          expr: cnpg_pg_stat_activity_count / cnpg_pg_settings_setting{name="max_connections"} > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL connections > 80% of max"

        - alert: PostgreSQLWALArchiveFailing
          expr: cnpg_pg_stat_archiver_failed_count > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PostgreSQL WAL archiving is failing"

        - alert: PostgreSQLStorageNearlyFull
          expr: cnpg_pg_database_size_bytes / cnpg_pg_volume_size_bytes > 0.85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL storage > 85% full"
```

### pg_stat_statements for Query Performance

Enable query tracking in the Cluster spec (already included in our examples):

```yaml
postgresql:
  parameters:
    pg_stat_statements.max: "10000"
    pg_stat_statements.track: "all"
```

Query the top slow queries:

```sql
SELECT
  queryid,
  calls,
  mean_exec_time::numeric(10,2) as avg_ms,
  total_exec_time::numeric(10,2) as total_ms,
  rows,
  query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Log Management

CNPG outputs PostgreSQL logs as JSON to stdout, which integrates naturally with:
- **Azure Monitor Container Insights** (collected automatically)
- **Fluentd/Fluent Bit** → Azure Log Analytics
- **Loki** for Grafana-native log aggregation

Configure useful logging parameters:

```yaml
postgresql:
  parameters:
    log_checkpoints: "on"
    log_lock_waits: "on"
    log_min_duration_statement: "1000"   # Log queries > 1s
    log_statement: "ddl"                  # Log DDL statements
    log_temp_files: "1024"               # Log temp files > 1MB
    log_autovacuum_min_duration: "1s"
```

---

## 10. Day-2 Operations

### Scaling Replicas

Simply update the `instances` field:

```yaml
spec:
  instances: 5  # Was 3 — added 2 read replicas
```

The operator creates new pods, initializes them from a base backup or pg_basebackup, and starts streaming replication. Scale down by reducing the count — the operator removes replicas gracefully.

### PostgreSQL Major Version Upgrades

Major version upgrades (e.g., 16 → 17) require data migration. CNPG supports two approaches:

**1. Import from a running cluster (recommended):**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production-v17
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  bootstrap:
    initdb:
      import:
        type: microservice
        databases:
          - appdb
        source:
          externalCluster: pg-production-v16

  externalClusters:
    - name: pg-production-v16
      connectionParameters:
        host: pg-production-rw.cnpg-database.svc
        user: postgres
        dbname: appdb
      password:
        name: pg-production-superuser
        key: password
```

**2. Logical replication:** Set up logical replication from the old cluster to the new one, then cutover when they're in sync.

### Minor Version Updates

Update the image tag — CNPG handles the rolling restart:

```yaml
imageName: ghcr.io/cloudnative-pg/postgresql:17.4  # Was 17.3
```

The operator updates replicas first, performs a controlled switchover, then updates the former primary.

### Configuration Changes

Update `postgresql.parameters` in the Cluster spec. CNPG classifies parameters as:

- **No restart required** (e.g., `work_mem`, `effective_cache_size`): Applied via `ALTER SYSTEM` + `pg_reload_conf()` — no downtime.
- **Restart required** (e.g., `shared_buffers`, `max_connections`): Triggers a rolling restart — replicas first, then switchover.

### Troubleshooting Common Issues

**Check cluster status:**
```bash
kubectl cnpg status pg-production -n cnpg-database
```

**Check operator logs:**
```bash
kubectl logs -n cnpg-system -l app.kubernetes.io/name=cloudnative-pg --tail=100
```

**Check instance logs:**
```bash
kubectl logs pg-production-1 -n cnpg-database --tail=100
```

**Common issues:**
- **Pod stuck in Pending**: Check node pool capacity, affinity rules, and PVC binding (zone mismatch)
- **Replication lag growing**: Check network, storage IOPS, and heavy write workloads
- **WAL archiving failing**: Verify workload identity, storage account access, and network policies allowing HTTPS egress
- **OOM kills**: Increase memory limits, reduce `shared_buffers`, or check for memory leaks in extensions

### Emergency Failover

Force a manual switchover (e.g., for maintenance):

```bash
# Controlled switchover — promotes a specific replica
kubectl cnpg promote pg-production pg-production-2 -n cnpg-database

# Or via annotation
kubectl annotate cluster pg-production \
  cnpg.io/switchoverTarget=pg-production-2 \
  -n cnpg-database
```

---

## 11. Cost Optimization

### Right-Sizing Compute

| VM SKU | vCPUs | Memory | Cost/mo (Pay-as-you-go, AustraliaEast) | Best For |
|---|---|---|---|---|
| Standard_D2s_v3 | 2 | 8 GiB | ~$120 | Dev/test |
| Standard_D4s_v3 | 4 | 16 GiB | ~$240 | Small-medium OLTP |
| Standard_D8s_v3 | 8 | 32 GiB | ~$480 | Medium-large OLTP |
| Standard_E4s_v3 | 4 | 32 GiB | ~$310 | Memory-intensive workloads |
| Standard_E8s_v3 | 8 | 64 GiB | ~$620 | Large databases |

**Memory-optimized (E-series)** VMs give PostgreSQL more shared_buffers and effective_cache_size per dollar. Use them for databases larger than available RAM.

### Storage Cost Comparison

| Disk Type | 256 GiB Cost/mo | 5,000 IOPS Cost/mo | Notes |
|---|---|---|---|
| Premium SSD (P15) | ~$38 | Included (1,100 IOPS) | Need P30 for 5,000 IOPS: ~$145 |
| Premium SSD v2 | ~$26 | ~$16 extra | Independent IOPS scaling |
| Ultra Disk | ~$82 | ~$16 extra | Overkill for most workloads |

**Premium SSD v2 wins** for PostgreSQL — you pay for exactly the IOPS and throughput you need without over-provisioning capacity.

### Reserved Instances

For dedicated PostgreSQL node pools that run 24/7, reserved instances provide 30-60% savings:

```bash
# 1-year reserved instance for D4s_v3
# Pay-as-you-go: ~$240/mo → Reserved: ~$150/mo (37% savings)
# 3-year reserved: ~$100/mo (58% savings)
```

### Spot Instances — When They Can and Can't Be Used

- ✅ **Read replicas for non-critical analytics** — if the replica is evicted, queries fail gracefully
- ✅ **Development/test clusters** — acceptable downtime
- ❌ **Primary instance** — eviction = database outage
- ❌ **Synchronous replicas** — eviction breaks synchronous replication
- ❌ **Any instance where data loss is unacceptable** — Spot VMs can be evicted with 30s notice

### Cost Comparison: Self-Hosted vs Azure Database for PostgreSQL

| Aspect | Self-Hosted (AKS + CNPG) | Azure DB for PostgreSQL (Flexible Server) |
|---|---|---|
| **4 vCPU, 16 GB, 256 GB storage** | ~$290/mo (D4s_v3 + Premium SSD v2) | ~$410/mo (GP_Standard_D4s_v3) |
| **With 1-year RI** | ~$190/mo | ~$260/mo |
| **3-node HA cluster** | ~$870/mo (3 × D4s_v3) | ~$820/mo (HA with zone redundancy) |
| **Operational overhead** | Higher (you manage the operator) | Lower (Microsoft manages) |
| **Flexibility** | Full (any extension, any version) | Limited extensions list |
| **Backup cost** | Storage account cost only | Included (up to 2× DB size) |

**Break-even point:** Self-hosting becomes cost-effective when you're running **3+ PostgreSQL instances** or need **large instances (8+ vCPUs)**. Below that, the operational overhead of managing the operator often outweighs the cost savings.

---

## 12. Complete Reference Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Azure Resource Group                                │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                     AKS Cluster (3 Availability Zones)                 │  │
│  │                                                                        │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐    │  │
│  │  │  System Pool     │  │  System Pool     │  │  System Pool     │    │  │
│  │  │  (Zone 1)        │  │  (Zone 2)        │  │  (Zone 3)        │    │  │
│  │  │  D2s_v3          │  │  D2s_v3          │  │  D2s_v3          │    │  │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │    │  │
│  │  │  │ CoreDNS    │  │  │  │ CNPG       │  │  │  │ Prometheus │  │    │  │
│  │  │  │ kube-proxy │  │  │  │ Operator   │  │  │  │ Grafana    │  │    │  │
│  │  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │    │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘    │  │
│  │                                                                        │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐    │  │
│  │  │  Postgres Pool   │  │  Postgres Pool   │  │  Postgres Pool   │    │  │
│  │  │  (Zone 1)        │  │  (Zone 2)        │  │  (Zone 3)        │    │  │
│  │  │  D4s_v3          │  │  D4s_v3          │  │  D4s_v3          │    │  │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │    │  │
│  │  │  │ PG Primary │  │  │  │ PG Replica │  │  │  │ PG Replica │  │    │  │
│  │  │  │ (RW)       │  │  │  │ (RO)       │  │  │  │ (RO)       │  │    │  │
│  │  │  │ PgBouncer  │  │  │  │ PgBouncer  │  │  │  │ PgBouncer  │  │    │  │
│  │  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │    │  │
│  │  │  [Premium SSD v2]│  │  [Premium SSD v2]│  │  [Premium SSD v2]│    │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                    │
│                                          │ WAL Archive + Base Backups         │
│                                          ▼                                    │
│  ┌─────────────────────────┐    ┌──────────────────────┐                    │
│  │  User-Assigned          │    │  Azure Blob Storage  │                    │
│  │  Managed Identity       │───▶│  (Standard ZRS)      │                    │
│  │  (Workload Identity)    │    │  /backups             │                    │
│  └─────────────────────────┘    └──────────────────────┘                    │
│                                          │                                    │
│                                          │ Object Replication (DR)            │
│                                          ▼                                    │
│                                  ┌──────────────────────┐                    │
│                                  │  Azure Blob Storage  │                    │
│                                  │  (Secondary Region)  │                    │
│                                  └──────────────────────┘                    │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │  Monitoring:  Azure Managed Grafana                       │               │
│  │               Azure Monitor Workspace                     │               │
│  │               Log Analytics Workspace                     │               │
│  └──────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AKS Cluster Setup (Azure CLI)

```bash
# Variables
export RESOURCE_GROUP="rg-cnpg-production"
export LOCATION="australiaeast"
export CLUSTER_NAME="aks-cnpg-production"
export K8S_VERSION="1.32"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster with system node pool
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --kubernetes-version $K8S_VERSION \
  --generate-ssh-keys \
  --enable-managed-identity \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --nodepool-name systempool \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 4 \
  --enable-azure-monitor-metrics \
  --tier standard

# Add dedicated PostgreSQL node pool
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name postgres \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --zones 1 2 3 \
  --labels workload=postgres \
  --node-taints workload=postgres:NoSchedule \
  --max-pods 30 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5

# Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

### Complete Production YAML Manifests

**1. Namespace:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-database
  labels:
    name: cnpg-database
```

**2. StorageClass:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-v2-postgresql
parameters:
  cachingMode: None
  skuName: PremiumV2_LRS
  DiskIOPSReadWrite: "5000"
  DiskMBpsReadWrite: "200"
provisioner: disk.csi.azure.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**3. CloudNativePG Cluster:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-production
  namespace: cnpg-database
  annotations:
    alpha.cnpg.io/failoverQuorum: "true"
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4

  inheritedMetadata:
    labels:
      azure.workload.identity/use: "true"

  smartShutdownTimeout: 30

  probes:
    startup:
      type: streaming
      maximumLag: 32Mi
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 120
    readiness:
      type: streaming
      maximumLag: 0
      periodSeconds: 10
      failureThreshold: 6

  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          cnpg.io/cluster: pg-production

  affinity:
    nodeSelector:
      workload: postgres
    tolerations:
      - key: "workload"
        operator: "Equal"
        value: "postgres"
        effect: "NoSchedule"

  resources:
    requests:
      memory: "8Gi"
      cpu: 2
    limits:
      memory: "8Gi"
      cpu: 2

  bootstrap:
    initdb:
      database: appdb
      owner: app
      secret:
        name: db-user-pass
      dataChecksums: true

  storage:
    storageClass: premium-ssd-v2-postgresql
    size: 200Gi

  postgresql:
    synchronous:
      method: any
      number: 1
    parameters:
      wal_compression: lz4
      max_wal_size: 6GB
      max_slot_wal_keep_size: 10GB
      checkpoint_timeout: 15min
      checkpoint_completion_target: "0.9"
      checkpoint_flush_after: 2MB
      wal_writer_flush_after: 2MB
      min_wal_size: 2GB
      shared_buffers: 4GB
      effective_cache_size: 12GB
      work_mem: 62MB
      maintenance_work_mem: 1GB
      autovacuum_vacuum_cost_limit: "2400"
      random_page_cost: "1.1"
      effective_io_concurrency: "64"
      maintenance_io_concurrency: "64"
      max_connections: "300"
      log_checkpoints: "on"
      log_lock_waits: "on"
      log_min_duration_statement: "1000"
      log_statement: ddl
      log_temp_files: "1024"
      log_autovacuum_min_duration: 1s
      pg_stat_statements.max: "10000"
      pg_stat_statements.track: all
      hot_standby_feedback: "on"
    pg_hba:
      - hostssl all all all scram-sha-256

  serviceAccountTemplate:
    metadata:
      annotations:
        azure.workload.identity/client-id: "<YOUR_IDENTITY_CLIENT_ID>"
      labels:
        azure.workload.identity/use: "true"

  backup:
    barmanObjectStore:
      destinationPath: "https://<STORAGE_ACCOUNT>.blob.core.windows.net/backups"
      azureCredentials:
        inheritFromAzureAD: true
    retentionPolicy: "30d"
```

**4. Scheduled Backup:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: pg-production-daily-backup
  namespace: cnpg-database
spec:
  schedule: "0 0 2 * * *"
  backupOwnerReference: self
  cluster:
    name: pg-production
  method: barmanObjectStore
  immediate: true
```

**5. PgBouncer Pooler:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pg-production-pooler-rw
  namespace: cnpg-database
spec:
  cluster:
    name: pg-production
  instances: 2
  type: rw
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "1000"
      default_pool_size: "25"
---
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pg-production-pooler-ro
  namespace: cnpg-database
spec:
  cluster:
    name: pg-production
  instances: 2
  type: ro
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "2000"
      default_pool_size: "50"
```

**6. Network Policy:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pg-production-network-policy
  namespace: cnpg-database
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: pg-production
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: app-namespace
      ports:
        - protocol: TCP
          port: 5432
    - from:
        - podSelector:
            matchLabels:
              cnpg.io/cluster: pg-production
      ports:
        - protocol: TCP
          port: 5432
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9187
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - podSelector:
            matchLabels:
              cnpg.io/cluster: pg-production
      ports:
        - protocol: TCP
          port: 5432
    - ports:
        - protocol: TCP
          port: 443
```

**7. PodMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pg-production
  namespace: cnpg-database
  labels:
    cnpg.io/cluster: pg-production
spec:
  selector:
    matchLabels:
      cnpg.io/cluster: pg-production
  podMetricsEndpoints:
    - port: metrics
```

### Helm Values for CloudNativePG Operator

```yaml
# values-cnpg-operator.yaml
replicaCount: 1

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"

monitoring:
  podMonitorEnabled: true
  grafanaDashboard:
    create: true

config:
  CLUSTERS_ROLLOUT_DELAY: "120"
  STANDBY_TCP_USER_TIMEOUT: "10"
```

```bash
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace \
  -f values-cnpg-operator.yaml
```

---

## 13. Summary & Recommendations

### When to Self-Host vs Use Managed

| Use Managed (Azure DB for PostgreSQL) | Use Self-Hosted (AKS + CNPG) |
|---|---|
| Small to medium workloads | Large-scale or multiple instances |
| Standard PostgreSQL usage | Custom extensions (TimescaleDB, pgvector, etc.) |
| Small team without K8s expertise | Team experienced with Kubernetes |
| Minimal operational overhead desired | GitOps-driven infrastructure |
| Single-cloud Azure commitment | Multi-cloud / hybrid requirements |
| Compliance met by Azure managed | Specific compliance requiring dedicated infra |

### Key Takeaways

1. **CloudNativePG is the operator to use in 2026.** It's the most Kubernetes-native, has official Microsoft backing via AKS docs, and requires no external dependencies.

2. **Storage is your most critical decision.** Use Premium SSD v2 with independent IOPS/throughput configuration. Always set `reclaimPolicy: Retain`. Use `WaitForFirstConsumer` volume binding for zone-aware scheduling.

3. **Run 3 instances across 3 availability zones** for production. Use synchronous replication with `number: 1` for zero-RPO.

4. **Use Azure Workload Identity for backups** — no storage keys, no SAS tokens, no secrets to rotate. Barman Cloud + Azure Blob Storage is battle-tested.

5. **Dedicated node pools matter.** Isolate PostgreSQL on nodes with the right VM SKU, taints, and labels. Don't let random workloads compete for CPU and memory.

6. **Connection pooling isn't optional.** Deploy PgBouncer in transaction mode between your applications and PostgreSQL. Your database will thank you.

7. **Monitor everything.** Set up PodMonitors, Grafana dashboards, and alerting rules from day one. WAL archiving failures and replication lag are the two most critical alerts.

8. **Test your restores.** A backup you haven't tested isn't a backup. Periodically create a cluster from backup using PITR and validate your data.

### Essential Links

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/current/)
- [Microsoft Learn: Deploy HA PostgreSQL on AKS](https://learn.microsoft.com/azure/aks/deploy-postgresql-ha)
- [Microsoft Learn: Create PostgreSQL HA Infrastructure on AKS](https://learn.microsoft.com/azure/aks/create-postgresql-ha)
- [Microsoft Learn: PostgreSQL HA Overview on AKS](https://learn.microsoft.com/azure/aks/postgresql-ha-overview)
- [Microsoft Learn: AKS Stateful Workloads](https://learn.microsoft.com/azure/aks/stateful-workloads-overview)
- [Microsoft Learn: Azure Premium SSD v2 on AKS](https://learn.microsoft.com/azure/aks/use-premium-v2-disks)
- [Microsoft Learn: AKS Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-deploy-cluster)
- [Barman Documentation](https://pgbarman.org/documentation/)
- [CloudNativePG GitHub](https://github.com/cloudnative-pg/cloudnative-pg)
- [CNPG Grafana Dashboard](https://grafana.com/grafana/dashboards/20417-cloudnativepg/)
- [CloudNativePG Helm Charts](https://cloudnative-pg.github.io/charts/)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

---

*This guide reflects the state of CloudNativePG, AKS, and the broader ecosystem as of March 2026. The PostgreSQL-on-Kubernetes landscape moves fast — always check the official docs for the latest recommendations.*
