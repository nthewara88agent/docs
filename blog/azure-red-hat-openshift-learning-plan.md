# Azure Red Hat OpenShift (ARO) Learning Plan: From Zero to Production

*Published: 4 February 2026*

So you need to learn Azure Red Hat OpenShift (ARO)? This structured 6-week learning plan takes you from fundamentals to production-ready operations, with hands-on labs, official documentation links, and practical milestones.

---

## What is ARO?

**Azure Red Hat OpenShift (ARO)** is a fully managed OpenShift service jointly engineered by Microsoft and Red Hat. It combines Kubernetes with OpenShift's developer tools, CI/CD pipelines, monitoring, and enterprise security features — all managed for you.

**Key facts:**
- **SLA:** 99.95% availability
- **Jointly supported** by Microsoft and Red Hat
- **No VMs to manage** — control plane, infrastructure, and nodes are patched/updated automatically
- **Runs in your Azure subscription** — included in your Azure bill

---

## Learning Path Overview

| Week | Focus | Outcome |
|------|-------|---------|
| 1 | Foundations | Understand ARO architecture, networking concepts |
| 2 | Hands-On Deployment | Create your first cluster, deploy an app |
| 3 | Networking Deep Dive | Private clusters, network policies, egress |
| 4 | Security & Identity | Entra ID SSO, RBAC, encryption |
| 5 | Operations (Day 2) | Monitoring, backup, scaling |
| 6+ | Advanced Topics | Terraform/Bicep, GitOps, Service Mesh |

---

## Phase 1: Foundations (Week 1)

*Understand the basics before touching anything.*

### Day 1-2: OpenShift & Kubernetes Fundamentals

- **If new to Kubernetes:** Complete [Kubernetes basics tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/) (2-3 hours)
- **OpenShift concepts:** Read [What is OpenShift?](https://www.redhat.com/en/technologies/cloud-computing/openshift)
- Understand the key difference: **OpenShift = Kubernetes + developer tools + security + CI/CD**

### Day 3-4: ARO Architecture & Concepts

**Required reading:**
- [What is Azure Red Hat OpenShift?](https://learn.microsoft.com/en-us/azure/openshift/intro-openshift)
- [Network concepts for ARO](https://learn.microsoft.com/en-us/azure/openshift/concepts-networking)
- [Responsibility matrix](https://learn.microsoft.com/en-us/azure/openshift/responsibility-matrix) — understand what Microsoft/Red Hat manages vs what you manage

**Key networking components:**

| Component | Purpose |
|-----------|---------|
| `aro-pls` | Private Link for Microsoft/Red Hat SRE cluster management |
| `aro-internal` | Internal load balancer for API server and service traffic |
| `aro` | Public load balancer for ingress/egress traffic |
| `aro-nsg` | Network security group controlling traffic flow |

### Day 5: Microsoft Learn Module

- Complete: [Introduction to Red Hat on Azure](https://learn.microsoft.com/en-us/training/modules/introduction-to-red-hat-azure/) (30-45 min)

---

## Phase 2: Hands-On Deployment (Week 2)

*Deploy your first ARO cluster.*

### Prerequisites

Before you start:
1. Azure subscription with **Contributor** + **User Access Administrator** roles
2. [Red Hat account](https://www.redhat.com/wapps/ugc/register.html) (free)
3. Download your [pull secret](https://cloud.redhat.com/openshift/install/azure/aro-provisioned) from Red Hat
4. Azure CLI installed locally

### Day 1-2: Deploy ARO Cluster via CLI

Follow: [Quickstart: Create ARO cluster with Azure CLI](https://learn.microsoft.com/en-us/azure/openshift/create-cluster)

**Key commands:**

```bash
# Set variables
LOCATION=australiaeast
RESOURCEGROUP=aro-rg
CLUSTER=my-aro-cluster
VIRTUALNETWORK=aro-vnet

# Check available OpenShift versions
az aro get-versions --location $LOCATION --output table

# Create resource group
az group create --name $RESOURCEGROUP --location $LOCATION

# Create virtual network with subnets
az network vnet create \
  --resource-group $RESOURCEGROUP \
  --name $VIRTUALNETWORK \
  --address-prefixes 10.0.0.0/22

az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VIRTUALNETWORK \
  --name master-subnet \
  --address-prefixes 10.0.0.0/23

az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VIRTUALNETWORK \
  --name worker-subnet \
  --address-prefixes 10.0.2.0/23

# Create ARO cluster (takes ~35-45 minutes)
az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --vnet $VIRTUALNETWORK \
  --master-subnet master-subnet \
  --worker-subnet worker-subnet \
  --pull-secret @pull-secret.txt

# Get cluster credentials
az aro list-credentials \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER

# Get console URL
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query consoleProfile.url -o tsv
```

### Day 3: Connect & Explore

- Follow: [Connect to ARO cluster](https://learn.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster)
- Access the OpenShift web console
- Install the `oc` CLI (OpenShift client)
- Run `oc login` and explore the cluster

### Day 4-5: Deploy Your First App

- Deploy a sample application via the web console
- Expose it via an OpenShift Route
- Understand **Projects** (OpenShift's enhanced namespaces)

---

## Phase 3: Networking Deep Dive (Week 3)

*ARO networking is critical for enterprise deployments.*

### Day 1-2: Private Clusters

- Read: [Create a private ARO cluster](https://learn.microsoft.com/en-us/azure/openshift/howto-create-private-cluster-4x)
- Understand private API endpoint vs private ingress
- **Lab:** Deploy a private cluster

### Day 3-4: Network Policies & Segmentation

- Read: [Networking policies](https://learn.microsoft.com/en-us/azure/openshift/concepts-networking#networking-policies)
- Create `NetworkPolicy` objects to restrict pod-to-pod traffic
- Understand OpenShift SDN vs OVN-Kubernetes networking

### Day 5: Egress & Firewall Integration

- Configure egress IP addresses
- Integrate with Azure Firewall for outbound traffic control
- Understand SNAT port limitations (1,024 ports per node)

---

## Phase 4: Security & Identity (Week 4)

*Enterprise security configuration.*

### Day 1-2: Microsoft Entra ID Integration

- Follow: [Configure Azure AD authentication for ARO](https://learn.microsoft.com/en-us/azure/openshift/configure-azure-ad-ui)
- Set up SSO with Entra ID
- Map Azure AD groups to OpenShift RBAC roles

### Day 3: Security Baseline

- Read: [ARO Security Baseline](https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/azure-red-hat-openshift-aro-security-baseline)
- Understand network security controls
- Review identity management requirements

### Day 4-5: Data Protection

- Follow: [Encrypt PVCs with customer-managed keys](https://learn.microsoft.com/en-us/azure/openshift/howto-encrypt-data-disks)
- Configure etcd encryption
- Set up Key Vault CSI driver for secrets management

---

## Phase 5: Operations & Day 2 (Week 5)

*Running ARO in production.*

### Day 1-2: Monitoring & Logging

- Explore built-in OpenShift monitoring (Prometheus/Grafana stack)
- Configure Azure Monitor integration
- Set up log forwarding to Azure Log Analytics or external SIEM

### Day 3: Backup & Disaster Recovery

- Read: [Backup and restore with OADP](https://cloud.redhat.com/experts/aro/backup-restore/)
- Configure Velero for cluster backup
- Practice backup and restore workflow

### Day 4: Updates & Maintenance

- Understand OpenShift version lifecycle
- Read: [Update ARO cluster certificates](https://learn.microsoft.com/en-us/azure/openshift/howto-update-certificates)
- Review maintenance windows and patching process

### Day 5: Scaling

- Scale worker nodes up and down
- Understand MachineSets and MachineConfigs
- Configure the cluster autoscaler

---

## Phase 6: Advanced Topics (Week 6+)

*Production patterns and integrations.*

### Infrastructure as Code

- Read: [Deploy ARO with ARM/Bicep](https://learn.microsoft.com/en-us/azure/openshift/quickstart-openshift-arm-bicep-template)
- Read: [Deploy ARO with Terraform](https://learn.microsoft.com/en-us/azure/templates/microsoft.redhatopenshift/openshiftclusters?pivots=deployment-language-terraform)
- Build reusable deployment templates for your organization

### Azure Arc Integration

- Connect ARO to Azure Arc-enabled Kubernetes
- Apply Azure Policy to ARO clusters for governance
- Use GitOps with Flux for configuration management

### Service Mesh

- Deploy Red Hat OpenShift Service Mesh (Istio-based)
- Configure mTLS between services
- Set up traffic management and observability

### CI/CD Pipelines

- Set up OpenShift Pipelines (Tekton)
- Integrate with GitHub Actions or Azure DevOps
- Configure OpenShift GitOps (ArgoCD) for declarative deployments

---

## Key Resources

### Official Documentation
| Resource | Link |
|----------|------|
| Azure Red Hat OpenShift docs | [learn.microsoft.com/azure/openshift](https://learn.microsoft.com/en-us/azure/openshift/) |
| Red Hat OpenShift docs | [docs.openshift.com](https://docs.openshift.com/container-platform/latest/welcome/index.html) |
| ARO Landing Zone Accelerator | [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/azure-red-hat-openshift/landing-zone-accelerator) |

### Hands-On Labs
| Resource | Link |
|----------|------|
| Red Hat Interactive Learning | [developers.redhat.com/learn](https://developers.redhat.com/learn) |
| ARO Workshop | [microsoft.github.io/aroworkshop](https://microsoft.github.io/aroworkshop/) |

### Community
| Resource | Link |
|----------|------|
| ARO GitHub repo | [github.com/Azure/OpenShift](https://github.com/Azure/OpenShift) |
| Red Hat Customer Portal | [access.redhat.com](https://access.redhat.com/) |

### Certifications (Optional)
- **Red Hat Certified Specialist in OpenShift Administration (EX280)**
- **Red Hat Certified Engineer (RHCE) with OpenShift**

---

## Cost Tips

ARO costs include:
1. **Cluster fee:** ~$0.35/hour for control plane
2. **Worker nodes:** VM costs (D4s_v3 or larger recommended)
3. **Networking:** Load balancer, egress data transfer

**To minimize costs while learning:**
- Use the smallest viable worker node size (D4s_v3)
- Delete the cluster when not in use: `az aro delete --resource-group $RESOURCEGROUP --name $CLUSTER`
- Consider Azure Dev/Test subscription pricing
- Use Azure Cost Management to set budget alerts

---

## Milestones Checklist

| Week | Milestone | How to Validate |
|------|-----------|-----------------|
| 1 | Understand ARO architecture | Can explain networking components to someone |
| 2 | Deploy first cluster | Running cluster with a deployed sample app |
| 3 | Configure private networking | Private cluster with egress controls working |
| 4 | Integrate Entra ID | SSO working, RBAC roles configured |
| 5 | Production readiness | Monitoring, backup, and scaling configured |
| 6+ | Advanced operations | IaC deployments, GitOps, or service mesh working |

---

## Conclusion

Learning ARO is a journey through Kubernetes, OpenShift, Azure networking, and enterprise operations. This plan gives you a structured path from zero knowledge to production-ready skills in about 6 weeks.

**Start with Week 1** — get your Red Hat account, download your pull secret, and work through the foundational reading. By Week 2, you'll have your first cluster running.

Good luck! 🚀

---

## References

- [What is Azure Red Hat OpenShift?](https://learn.microsoft.com/en-us/azure/openshift/intro-openshift)
- [Network concepts for ARO](https://learn.microsoft.com/en-us/azure/openshift/concepts-networking)
- [ARO Security Baseline](https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/azure-red-hat-openshift-aro-security-baseline)
- [Responsibility Matrix](https://learn.microsoft.com/en-us/azure/openshift/responsibility-matrix)
- [ARO Quickstart](https://learn.microsoft.com/en-us/azure/openshift/create-cluster)

---

*This learning plan was compiled from official Microsoft Learn and Red Hat documentation.*
