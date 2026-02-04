# Stateful Workloads on AKS: A Comprehensive Guide to Architecture Patterns and Considerations

*Published: 4 February 2026*

Should you run your databases on Kubernetes? What exactly counts as a "stateful workload"? And if your app runs on AKS but uses Azure SQL Database as a backend, is that still considered hosting stateful workloads?

This guide answers these questions and provides a comprehensive framework for thinking about stateful workloads on Azure Kubernetes Service (AKS).

---

## TL;DR - Quick Decision Framework

| Your Scenario | Is It Stateful on AKS? | Recommendation |
|---------------|------------------------|----------------|
| App on AKS + Azure SQL/PostgreSQL (PaaS) | **No** — data lives in managed PaaS | ✅ Best practice for most workloads |
| App on AKS + Redis (PaaS) for caching | **No** — ephemeral cache in managed service | ✅ Great for session state |
| App on AKS + PostgreSQL running in AKS pods | **Yes** — data lives in PVCs on AKS | ⚠️ Consider carefully |
| App on AKS + Kafka/RabbitMQ in AKS | **Yes** — message queues with persistent state | ⚠️ Valid for specific use cases |
| App on AKS + self-managed MongoDB in pods | **Yes** — full stateful workload | ⚠️ Higher operational burden |

---

## What is a Stateful Workload?

A **stateful workload** is an application that uses persistent data storage to preserve state across restarts, rescheduling, or failures. The key characteristic: *the data must survive the container lifecycle*.

### Examples of Stateful Workloads

| Category | Examples |
|----------|----------|
| **Databases** | PostgreSQL, MySQL, MongoDB, SQL Server, Cassandra |
| **Message Queues** | Kafka, RabbitMQ, Redis (persistent mode), NATS JetStream |
| **Search Engines** | Elasticsearch, OpenSearch, Solr |
| **File/Object Storage** | MinIO, Ceph |
| **Caches (persistent)** | Redis with AOF/RDB persistence |
| **Blockchain/Ledger** | Hyperledger, blockchain nodes |

### Stateless vs Stateful

| Aspect | Stateless | Stateful |
|--------|-----------|----------|
| **Data persistence** | None needed | Critical |
| **Pod identity** | Interchangeable | Matters (hostname, network ID) |
| **Scaling** | Trivial | Complex (replication, sharding) |
| **Recovery** | Restart anywhere | Must restore data first |
| **Kubernetes resource** | Deployment | StatefulSet |

---

## The Hybrid Pattern: AKS + Azure PaaS Backends

**Here's the key insight:** If your application runs on AKS but stores data in Azure managed services (like Azure SQL, PostgreSQL Flexible Server, Cosmos DB, or Azure Cache for Redis), **you are NOT running stateful workloads on AKS**.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    AKS Cluster                          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│  │  │  API     │  │  Worker  │  │  Worker  │  ← Stateless  │    │
│  │  │  Pod     │  │  Pod     │  │  Pod     │    Pods       │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘               │    │
│  └───────┼─────────────┼─────────────┼─────────────────────┘    │
│          │             │             │                           │
│          └─────────────┼─────────────┘                           │
│                        │ Private Endpoint                        │
│                        ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │               Azure Managed Services                     │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ PostgreSQL   │  │ Redis Cache  │  │ Cosmos DB    │   │    │
│  │  │ Flexible Svr │  │ (Managed)    │  │              │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  │                    ← Stateful PaaS                       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Pattern is Recommended

| Benefit | Explanation |
|---------|-------------|
| **Separation of concerns** | AKS handles compute; Azure handles data durability |
| **Managed HA** | Azure PaaS services provide built-in zone-redundancy, failover |
| **Managed backups** | Point-in-time restore, geo-redundant backups included |
| **Managed patching** | No need to patch database engines yourself |
| **SLA** | Clear SLA from Azure (e.g., 99.99% for PostgreSQL) |
| **Scaling independence** | Scale compute (AKS) and data (PaaS) separately |

### When to Use This Pattern

- ✅ Web applications and APIs
- ✅ Microservices architectures
- ✅ Applications migrating from VMs
- ✅ Teams without deep Kubernetes/DBA expertise
- ✅ Workloads requiring strong SLAs

---

## True Stateful Workloads on AKS

When **data lives inside the AKS cluster** (in Persistent Volumes), you're running stateful workloads on AKS. This is a valid architecture pattern, but comes with additional considerations.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    AKS Cluster                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │  │
│  │  │  API     │  │  App     │  │  App     │  ← Stateless    │  │
│  │  │  Pod     │  │  Pod     │  │  Pod     │                 │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                 │  │
│  │       │             │             │                        │  │
│  │       └─────────────┼─────────────┘                        │  │
│  │                     │                                      │  │
│  │                     ▼                                      │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │           StatefulSet (PostgreSQL)                   │ │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │ │  │
│  │  │  │ postgres │  │ postgres │  │ postgres │            │ │  │
│  │  │  │  -0      │  │  -1      │  │  -2      │            │ │  │
│  │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘            │ │  │
│  │  │       │             │             │                   │ │  │
│  │  │  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐            │ │  │
│  │  │  │  PVC-0   │  │  PVC-1   │  │  PVC-2   │ ← Stateful │ │  │
│  │  │  │ 100 GiB  │  │ 100 GiB  │  │ 100 GiB  │            │ │  │
│  │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘            │ │  │
│  │  └───────┼─────────────┼─────────────┼──────────────────┘ │  │
│  └──────────┼─────────────┼─────────────┼────────────────────┘  │
│             │             │             │                        │
│             ▼             ▼             ▼                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Azure Managed Disks (Premium SSD)              │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### When to Run Stateful Workloads on AKS

| Use Case | Why It Makes Sense |
|----------|-------------------|
| **Multi-cloud portability** | Same Helm chart works on AKS, EKS, GKE, on-prem |
| **Specialized databases** | e.g., CockroachDB, TiDB, ClickHouse not available as Azure PaaS |
| **Edge/disconnected scenarios** | Data must be local, can't reach cloud PaaS |
| **Cost optimization** | For dev/test, running PostgreSQL in AKS can be cheaper |
| **Unified platform** | Single Kubernetes-native deployment model for everything |
| **CNCF ecosystem** | Want to use operators like CloudNativePG, Zalando, Strimzi |

### When to Avoid Stateful Workloads on AKS

| Scenario | Why Azure PaaS is Better |
|----------|-------------------------|
| **Need guaranteed SLA** | Azure manages HA, failover, and provides contractual SLA |
| **Limited DBA expertise** | PaaS handles patching, backups, tuning |
| **Compliance requirements** | Easier to demonstrate compliance with managed services |
| **Predictable costs** | PaaS pricing is clearer than compute + storage + ops time |
| **Just need a database** | Don't add Kubernetes complexity for a simple data store |

---

## Storage Options for Stateful Workloads on AKS

Choosing the right storage is critical for stateful workloads. Here's what Azure offers:

| Data Type | Recommended Storage | When to Use |
|-----------|-------------------|-------------|
| **Database/transactional** | Azure Managed Disks (Premium SSD, Premium SSD v2, Ultra Disk) | Dedicated, high-performance block storage with IOPS guarantees |
| **Shared file data (high perf)** | Azure NetApp Files or Azure Files Premium | Submillisecond latency, ReadWriteMany access |
| **Shared configuration** | Azure Files Standard | Config files, shared app state |
| **Unstructured data** | Azure Blob Storage (via BlobFuse or NFS) | Media files, documents, logs |
| **Ephemeral/temporary** | Ephemeral NVMe disks | Caching, scratch space, temporary processing |

### Storage Classes in AKS

AKS provides built-in storage classes:

```yaml
# Premium SSD - recommended for databases
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Key considerations:**
- Use `WaitForFirstConsumer` for zone-aware provisioning
- Set `reclaimPolicy: Retain` for production databases
- Enable `allowVolumeExpansion: true` for growing workloads

---

## Kubernetes Primitives for Stateful Workloads

### StatefulSets

StatefulSets are the Kubernetes primitive for stateful applications:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-premium
      resources:
        requests:
          storage: 100Gi
```

**StatefulSet guarantees:**
- **Stable, unique network identities:** `postgres-0`, `postgres-1`, `postgres-2`
- **Stable, persistent storage:** Each pod gets its own PVC
- **Ordered deployment/scaling:** Pods created and deleted in order
- **Ordered rolling updates:** Updates happen one pod at a time

### Kubernetes Operators

For production stateful workloads, use **Kubernetes Operators** instead of raw StatefulSets:

| Workload | Recommended Operators |
|----------|----------------------|
| **PostgreSQL** | CloudNativePG, Zalando Postgres Operator, Crunchy PGO |
| **MySQL** | MySQL Operator, Percona Operator |
| **MongoDB** | MongoDB Community Operator, Percona Operator |
| **Redis** | Redis Operator, Spotahome Redis Operator |
| **Kafka** | Strimzi, Confluent Operator |
| **Elasticsearch** | ECK (Elastic Cloud on Kubernetes) |

**Why operators?**
- Automated failover and recovery
- Backup/restore workflows
- Rolling upgrades
- Connection pooling
- Monitoring integration

---

## Comparison: Stateful on AKS vs Azure PaaS

| Aspect | Stateful on AKS | Azure PaaS Backend |
|--------|-----------------|-------------------|
| **Management** | You manage (with operators) | Microsoft manages |
| **HA/Failover** | Operator-dependent | Built-in, zone-redundant |
| **Backups** | You configure (Velero, pgBackRest) | Automated, point-in-time |
| **Patching** | You schedule | Automatic |
| **Scaling** | Horizontal via replicas | Vertical tier change |
| **SLA** | Based on your design | Up to 99.99% |
| **Portability** | Multi-cloud | Azure-only |
| **Cost model** | Compute + storage + ops time | Predictable per-tier |
| **Expertise required** | High (K8s + DBA) | Low |

---

## The KATE Stack for Stateful Workloads

Microsoft recommends the **KATE stack** as a foundation for stateful workloads on AKS:

| Component | Purpose |
|-----------|---------|
| **K**ubernetes (AKS) | Container orchestration |
| **A**rgoCD | GitOps continuous delivery |
| **T**erraform | Infrastructure as code |
| **E**xternal Secrets Operator | Sync secrets from Azure Key Vault |

### Additional Components for Stateful Workloads

| Component | Purpose |
|-----------|---------|
| **Persistent Volume subsystem** | Storage abstraction (PV, PVC, StorageClass) |
| **Azure Key Vault** | Secure storage for connection strings, passwords |
| **Secrets Store CSI Driver** | Mount Key Vault secrets into pods |
| **Kubernetes Operators** | Lifecycle management for databases |
| **Velero** | Backup and disaster recovery |

---

## Best Practices for Stateful Workloads on AKS

### 1. Use Dynamic Provisioning

Avoid statically creating PersistentVolumes. Let Kubernetes provision them dynamically:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 100Gi
```

### 2. Set Appropriate Reclaim Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `Delete` | PV deleted when PVC is deleted | Dev/test environments |
| `Retain` | PV retained when PVC is deleted | Production databases |

### 3. Implement Backup and Recovery

Use [Velero](https://velero.io/) or database-native tools:

```bash
# Install Velero with Azure plugin
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.7.0 \
  --bucket $BLOB_CONTAINER \
  --secret-file ./credentials-velero \
  --backup-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_ID
```

### 4. Use Pod Disruption Budgets

Prevent accidental disruption of stateful pods:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
```

### 5. Configure Resource Requests and Limits

Stateful workloads need guaranteed resources:

```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"
```

### 6. Use Node Affinity for Data Locality

Pin stateful pods to specific node pools:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: workload-type
          operator: In
          values:
          - stateful
```

### 7. Monitor Storage Performance

Track key metrics:
- IOPS consumed vs provisioned
- Latency (read/write)
- Throughput
- Volume utilization

Use Azure Monitor or Prometheus with Grafana.

---

## Decision Flowchart

```
                    ┌─────────────────────────────┐
                    │  Need persistent data for   │
                    │     your AKS workload?      │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │  Is there an Azure PaaS     │
                    │  service for your data?     │
                    │  (SQL, PostgreSQL, Redis,   │
                    │   Cosmos DB, etc.)          │
                    └─────────────┬───────────────┘
                                  │
               ┌──────────────────┴──────────────────┐
               │ YES                              NO │
               ▼                                     ▼
    ┌──────────────────────┐          ┌──────────────────────┐
    │  Do you need         │          │  Run stateful        │
    │  multi-cloud         │          │  workload on AKS     │
    │  portability?        │          │  with operators      │
    └──────────┬───────────┘          └──────────────────────┘
               │
      ┌────────┴────────┐
      │ YES         NO  │
      ▼                 ▼
┌───────────────┐  ┌───────────────────────┐
│ Run stateful  │  │  Use Azure PaaS       │
│ on AKS with   │  │  (Recommended)        │
│ operators     │  │                       │
└───────────────┘  │  • Built-in HA        │
                   │  • Managed backups    │
                   │  • Clear SLA          │
                   │  • Less operational   │
                   │    burden             │
                   └───────────────────────┘
```

---

## Real-World Architecture Examples

### Example 1: E-commerce Application (PaaS Backend)

**Stateless on AKS + Stateful in PaaS**

```
AKS Cluster:
├── Frontend (React) - Deployment
├── API Gateway - Deployment
├── Order Service - Deployment
├── Inventory Service - Deployment
└── Notification Service - Deployment

Azure PaaS:
├── Azure SQL Database (orders, inventory)
├── Azure Cache for Redis (sessions, caching)
├── Azure Service Bus (async messaging)
└── Azure Blob Storage (product images)
```

**Result:** Zero stateful workloads on AKS. All data managed by Azure.

### Example 2: Data Platform (Stateful on AKS)

**Specialized databases that don't exist as PaaS**

```
AKS Cluster:
├── API Layer - Deployment
├── ClickHouse (analytics) - StatefulSet + Operator
├── Apache Kafka (streaming) - Strimzi Operator
├── Redis Cluster (caching) - Redis Operator
└── MinIO (S3-compatible storage) - StatefulSet

Azure:
├── Azure Managed Disks (Premium SSD v2 for databases)
├── Azure NetApp Files (shared storage for analytics)
└── Azure Key Vault (secrets)
```

**Result:** Stateful workloads on AKS, managed with operators.

### Example 3: Hybrid Approach

**Some PaaS, some on-cluster**

```
AKS Cluster:
├── Application Services - Deployments (stateless)
├── Apache Kafka - Strimzi Operator (stateful on AKS)
└── Redis (ephemeral cache) - Deployment with emptyDir

Azure PaaS:
├── Azure PostgreSQL Flexible Server (primary database)
├── Azure Cosmos DB (global distribution needed)
└── Azure Blob Storage (files)
```

**Result:** Primary data in PaaS; specialized streaming in AKS.

---

## Cost Considerations

| Approach | Cost Factors |
|----------|--------------|
| **AKS + PaaS Backend** | PaaS service tiers + AKS compute (no storage for data) |
| **Stateful on AKS** | AKS compute + Azure Disks + backup storage + operational time |

**Hidden costs of stateful on AKS:**
- Engineering time for setup and maintenance
- Incident response and troubleshooting
- Backup testing and DR drills
- Upgrade planning and execution

**When stateful on AKS can be cheaper:**
- Dev/test environments (stop clusters when not in use)
- Specialized databases with complex PaaS pricing
- Very high IOPS requirements (NVMe local disks)

---

## Conclusion

**The question isn't "can I run stateful workloads on AKS?" — it's "should I?"**

For most applications:
1. **Use Azure PaaS services** (SQL, PostgreSQL, Cosmos DB, Redis) for persistent data
2. **Keep AKS stateless** — let it do what it does best: orchestrate containers
3. **Connect via Private Endpoints** for security

For specialized scenarios:
1. **Run stateful on AKS** when you need multi-cloud portability, specialized databases, or unified Kubernetes operations
2. **Use operators** (CloudNativePG, Strimzi, etc.) — don't manage StatefulSets manually
3. **Invest in backup/recovery** — Velero, database-native tools, regular testing

The hybrid pattern (stateless AKS + stateful PaaS) gives you the best of both worlds: Kubernetes flexibility for compute, Azure reliability for data.

---

## References

- [Stateful workloads in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/stateful-workloads-overview)
- [Storage options for applications in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- [Best practices for storage and backups in AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-storage)
- [Storage considerations for AKS (Cloud Adoption Framework)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/aks/storage)
- [Choose an Azure container service](https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service)

---

*This blog post was compiled from official Microsoft Learn documentation and Azure architecture guidance.*
