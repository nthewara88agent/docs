---
title: "NGINX Ingress Controller Discontinuation in AKS: Complete Migration Guide to Modern Gateway Options"
date: 2026-03-09
description: "The definitive guide to migrating from NGINX Ingress Controller in AKS. Covers the retirement timeline, Gateway API, Application Gateway for Containers, Istio service mesh, and hands-on migration walkthroughs with complete code examples."
tags: ["aks", "kubernetes", "nginx", "gateway-api", "istio", "application-gateway", "azure", "migration"]
---

# NGINX Ingress Controller Discontinuation in AKS: Complete Migration Guide to Modern Gateway Options

The Kubernetes ingress landscape is undergoing its most significant shift since the Ingress API was first introduced. The [Kubernetes SIG Network and Security Response Committee announced](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/) the retirement of the **Ingress-NGINX controller** — the most widely deployed community ingress controller in the Kubernetes ecosystem. Best-effort maintenance continues until **March 2026**, after which the project will receive no further releases, bug fixes, or security updates.

For Azure Kubernetes Service (AKS) customers, this has immediate implications. If you're running the **Application Routing add-on** (managed NGINX), a **self-managed NGINX Ingress Controller**, or even the legacy **HTTP Application Routing add-on**, you need a migration plan. This guide provides one.

## Table of Contents

- [The Deprecation Timeline](#the-deprecation-timeline)
- [Understanding the Current State](#understanding-the-current-state)
- [The Gateway API Standard](#the-gateway-api-standard)
- [Option 1: Application Gateway for Containers](#option-1-application-gateway-for-containers-alb-controller)
- [Option 2: Istio-based Service Mesh Add-on](#option-2-istio-based-service-mesh-add-on)
- [Option 3: Managed NGINX (Application Routing Add-on)](#option-3-managed-nginx-application-routing-add-on)
- [Option 4: Third-Party Solutions](#option-4-third-party-solutions)
- [Migration Strategy Framework](#migration-strategy-framework)
- [Hands-On: NGINX → Application Gateway for Containers](#hands-on-migration-nginx-ingress--application-gateway-for-containers)
- [Hands-On: NGINX → Istio Gateway](#hands-on-migration-nginx-ingress--istio-gateway)
- [Monitoring & Observability Post-Migration](#monitoring--observability-post-migration)
- [Summary & Recommendations](#summary--recommendations)

---

## The Deprecation Timeline

Understanding the timeline is critical for planning. Here's the sequence of events:

| Date | Event |
|------|-------|
| **2023** | HTTP Application Routing add-on deprecated in favour of Application Routing add-on |
| **March 2025** | HTTP Application Routing add-on fully retired (no longer available) |
| **November 2025** | Kubernetes SIG Network announces Ingress-NGINX retirement |
| **December 2025** | [Microsoft acknowledges impact on AKS](https://github.com/Azure/AKS/issues/5516), commits to support through November 2026 |
| **March 2026** | Upstream Ingress-NGINX project: end of maintenance, no new releases |
| **November 2026** | Microsoft ends critical security patching for Application Routing add-on NGINX resources |
| **Beyond 2026** | Gateway API becomes the standard for L7 traffic management in Kubernetes |

**The bottom line:** You have until **November 2026** for managed NGINX on AKS. That's not a lot of time for production migrations. Start planning now.

---

## Understanding the Current State

Before choosing a migration target, you need to understand what you're migrating *from*. There are three distinct NGINX-related ingress options that have existed in AKS:

### HTTP Application Routing (Legacy — Already Retired)

The original HTTP Application Routing add-on (`addon-http-application-routing`) was a convenience feature that deployed an NGINX ingress controller and an external DNS controller into your cluster. It was designed for **development and testing only** and was never recommended for production:

- No SSL/TLS support out of the box
- No Azure DNS zone integration
- No Azure Key Vault integration
- Limited configuration options
- **Retired as of March 2025** — if you're still running this, migrate immediately

### Application Routing Add-on (Managed NGINX — Current)

The [Application Routing add-on](https://learn.microsoft.com/azure/aks/app-routing) (`app-routing-addon`) is the current managed NGINX ingress for AKS. It uses the `webapprouting.kubernetes.azure.com` IngressClass and deploys into the `app-routing-system` namespace.

**Key features:**
- Managed NGINX Ingress controller based on the [Kubernetes NGINX Ingress controller](https://kubernetes.github.io/ingress-nginx/)
- Integration with [Azure DNS](https://learn.microsoft.com/azure/dns/dns-overview) for public and private zone management
- SSL termination with certificates stored in [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/general/overview)
- Up to five Azure DNS zones per cluster
- Supported on clusters with [managed identity](https://learn.microsoft.com/azure/aks/use-managed-identity)

**Current status:** Fully supported with critical security patches through **November 2026**. AKS is evolving this add-on toward Gateway API alignment — you don't necessarily need to switch products, but you do need to plan for the API migration.

### Self-Managed NGINX Ingress Controller

Many teams deploy NGINX Ingress via Helm or manifest directly, bypassing AKS add-ons entirely. This includes both the community `kubernetes/ingress-nginx` and the NGINX Inc (`nginxinc/kubernetes-ingress`) variants.

**If you're running self-managed community NGINX:** After March 2026, you're on unsupported software with no security patches. You have several options:
- Migrate to the [Application Routing add-on](https://learn.microsoft.com/azure/aks/app-routing) for managed NGINX with support through November 2026
- Migrate directly to [Application Gateway for Containers](https://learn.microsoft.com/azure/application-gateway/for-containers/overview) (Gateway API)
- Migrate to the [Istio service mesh add-on](https://learn.microsoft.com/azure/aks/istio-about) if you need mesh capabilities
- Switch to NGINX Inc's commercial offering (F5 NGINX Ingress Controller), which continues development independently

### Key Differences Between the Three

| Feature | HTTP App Routing | App Routing Add-on | Self-Managed NGINX |
|---------|-----------------|--------------------|--------------------|
| **Status** | Retired | Supported → Nov 2026 | Upstream EOL Mar 2026 |
| **IngressClass** | `addon-http-application-routing` | `webapprouting.kubernetes.azure.com` | `nginx` (configurable) |
| **Azure DNS Integration** | External DNS (basic) | Native (up to 5 zones) | Manual |
| **Key Vault TLS** | No | Yes | Manual with CSI driver |
| **Microsoft Support** | None | Yes | No |
| **Production Ready** | No | Yes | Depends on config |
| **ConfigMap Editing** | Yes | No (blocked) | Yes |

---

## The Gateway API Standard

The [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) is the successor to the Ingress API. Understanding it is essential because every major migration path now leads here.

### What is Gateway API?

Gateway API is a collection of Kubernetes API resources that model networking in Kubernetes. It graduated to **GA (v1.0)** in October 2023 and is now the **official CNCF standard** for Layer 7 routing. Unlike the Ingress API, which was designed as a lowest-common-denominator spec, Gateway API was designed from the ground up with:

- **Role-oriented design:** Infrastructure providers, cluster operators, and application developers each get their own API resources
- **Expressiveness:** Native support for header-based routing, traffic splitting, URL rewrites, request/response manipulation, and more — without annotations
- **Extensibility:** Extension points built into the spec rather than relying on provider-specific annotations
- **Portability:** A conformance test suite ensures implementations behave consistently

### How It Differs from Ingress API

| Aspect | Ingress API | Gateway API |
|--------|------------|-------------|
| **Role separation** | Single resource, one role | GatewayClass → Gateway → Routes (infra/ops/dev) |
| **Traffic splitting** | Annotations (non-portable) | Native `backendRefs` with weights |
| **Header routing** | Annotations | Native `HTTPRouteMatch` |
| **URL rewrite** | `nginx.ingress.kubernetes.io/rewrite-target` | Native `HTTPURLRewriteFilter` |
| **TLS** | `tls` section on Ingress | TLS on Gateway listener, certificates via Secret refs |
| **Multi-tenancy** | Limited | Cross-namespace routing with ReferenceGrant |
| **Request modification** | Annotations | `RequestHeaderModifier`, `ResponseHeaderModifier` filters |

### Core Resources Explained

**GatewayClass** — Defines a class of Gateways, similar to IngressClass. Typically provided by the infrastructure (e.g., `azure-alb-external` for Application Gateway for Containers, `istio` for Istio):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: azure-alb-external
spec:
  controllerName: alb.networking.azure.io/alb-controller
```

**Gateway** — A deployment of infrastructure that binds to a GatewayClass. Defines listeners (ports, protocols, TLS), similar to a load balancer configuration:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: test-infra
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
```

**HTTPRoute** — Defines how HTTP traffic should be routed to backend services. This is where application developers work:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: test-infra
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

### Why Gateway API is the Future

1. **Ingress-NGINX is retiring** — the most popular Ingress implementation is going away
2. **CNCF graduation** — Gateway API is an official graduated project
3. **Vendor adoption** — Azure, AWS, GCP, Istio, Envoy, Traefik, Kong all implement it
4. **No more annotation soup** — features are first-class API fields, not provider-specific hacks
5. **Better security model** — role-based separation with explicit cross-namespace grants

---

## Option 1: Application Gateway for Containers (ALB Controller)

[Application Gateway for Containers](https://learn.microsoft.com/azure/application-gateway/for-containers/overview) (AGC) is Microsoft's next-generation L7 load balancer for AKS. It's the spiritual successor to Application Gateway Ingress Controller (AGIC) but built from the ground up for containers and Gateway API.

### Architecture

AGC is an **Azure-managed data plane** running outside your cluster, controlled by the **ALB Controller** running inside your cluster. This is fundamentally different from in-cluster ingress controllers:

```
┌─────────────────────────────────────┐
│           Azure Cloud               │
│  ┌─────────────────────────────┐    │
│  │  Application Gateway for    │    │
│  │  Containers (Data Plane)    │    │
│  │  ┌──────────┐ ┌──────────┐ │    │
│  │  │ Frontend │ │ Frontend │ │    │
│  │  └─────┬────┘ └────┬─────┘ │    │
│  │        │            │       │    │
│  │  ┌─────┴────────────┴─────┐ │    │
│  │  │    Association         │ │    │
│  │  │  (Subnet Delegation)   │ │    │
│  │  └───────────┬────────────┘ │    │
│  └──────────────┼──────────────┘    │
│                 │                    │
│  ┌──────────────┼──────────────┐    │
│  │  AKS Cluster │              │    │
│  │  ┌───────────┴────────────┐ │    │
│  │  │    ALB Controller      │ │    │
│  │  │  (Control Plane)       │ │    │
│  │  └───────────┬────────────┘ │    │
│  │  ┌───────────┴────────────┐ │    │
│  │  │  Pods / Services       │ │    │
│  │  └────────────────────────┘ │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

### Key Features

- **Gateway API support:** GatewayClass, Gateway, HTTPRoute, ReferenceGrant
- **Ingress API support:** Also works with traditional Ingress resources (migration bridge)
- **Traffic splitting:** Native weighted backend references for canary/blue-green deployments
- **URL rewrites:** Path and header-based routing with rewrites
- **TLS termination and SSL offloading**
- **Mutual TLS (mTLS):** Both frontend and backend mTLS supported
- **Web Application Firewall (WAF):** Azure WAF integration with DRS 2.1 ruleset
- **Service mesh integration:** Works with Istio for end-to-end mTLS
- **Near real-time convergence:** Configuration updates propagate in seconds, not minutes
- **Azure-native observability:** Integrated with Azure Monitor metrics and diagnostics

### When to Choose AGC

✅ You want a fully managed, Azure-native L7 load balancer
✅ You need WAF capabilities
✅ You're adopting Gateway API as your standard
✅ You need advanced traffic management (splitting, rewrites, header manipulation)
✅ You want the data plane outside your cluster (no pod resource consumption)
✅ You need mTLS between the gateway and your backends

### Deployment Walkthrough

AGC supports two deployment strategies: **ALB managed** (controller creates Azure resources) and **Bring Your Own** (you pre-create Azure resources).

#### Prerequisites

```bash
# Set variables
AKS_NAME='my-aks-cluster'
RESOURCE_GROUP='my-resource-group'

# Register required providers
az provider register --namespace Microsoft.ServiceNetworking
az provider register --namespace Microsoft.ContainerService

# Enable the ALB Controller add-on
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --enable-alb-controller
```

#### Create the Association Subnet

```bash
# Get AKS VNet info
MC_RESOURCE_GROUP=$(az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP \
  --query "nodeResourceGroup" -o tsv)
CLUSTER_SUBNET_ID=$(az vmss list --resource-group $MC_RESOURCE_GROUP \
  --query '[0].virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].ipConfigurations[0].subnet.id' -o tsv)
read -d '' VNET_NAME VNET_RESOURCE_GROUP VNET_ID <<< \
  $(az network vnet show --ids $CLUSTER_SUBNET_ID --query '[name, resourceGroup, id]' -o tsv)

# Create subnet with delegation (minimum /24)
az network vnet subnet create \
  --resource-group $VNET_RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name subnet-alb \
  --address-prefixes '10.225.0.0/24' \
  --delegations 'Microsoft.ServiceNetworking/trafficControllers'

ALB_SUBNET_ID=$(az network vnet subnet show \
  --name subnet-alb \
  --resource-group $VNET_RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --query '[id]' --output tsv)
```

#### Configure RBAC for ALB Controller

```bash
IDENTITY_RESOURCE_NAME='azure-alb-identity'
mcResourceGroupId=$(az group show --name $MC_RESOURCE_GROUP --query id -otsv)
principalId=$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query principalId -otsv)

# AppGW for Containers Configuration Manager role
az role assignment create \
  --assignee-object-id $principalId \
  --assignee-principal-type ServicePrincipal \
  --scope $mcResourceGroupId \
  --role "fbc52c3f-28ad-4303-a892-8a056630b8f1"

# Network Contributor for subnet join
az role assignment create \
  --assignee-object-id $principalId \
  --assignee-principal-type ServicePrincipal \
  --scope $ALB_SUBNET_ID \
  --role "4d97b98b-1d4f-4787-a291-c67834d212e7"
```

#### Deploy with ALB Managed Strategy

```yaml
# ApplicationLoadBalancer resource — tells ALB Controller to create AGC
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: my-agc
  namespace: alb-infra
spec:
  associations:
  - <ALB_SUBNET_ID>
---
# Gateway using AGC
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: alb-infra
  annotations:
    alb.networking.azure.io/alb-id: <will be auto-populated>
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: alb-infra
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: my-service
      port: 80
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Fully managed Azure service | Requires dedicated /24 subnet |
| Gateway API + Ingress API support | More complex initial setup than in-cluster NGINX |
| WAF integration | Only ports 80 and 443 on listeners |
| Data plane outside cluster | Provisioning takes 5-6 minutes |
| Near real-time config updates | Learning curve for Gateway API concepts |
| mTLS support (frontend + backend) | Cost (~$120-$150+/month minimum) |

### Pricing

AGC billing has four meters ([full pricing details](https://learn.microsoft.com/azure/application-gateway/for-containers/understanding-pricing)):

| Meter | Price (East US 2) | With WAF |
|-------|-------------------|----------|
| AGC resource-hour | $0.017/hr | $0.031/hr |
| Frontend-hour | $0.01/hr | $0.018/hr |
| Association-hour | $0.12/hr | $0.216/hr |
| Capacity Unit-hour | $0.008/hr | $0.014/hr |

**Example monthly cost (simple deployment):** 1 AGC + 1 frontend + 1 association + 5 CUs ≈ **$136.51/month**

---

## Option 2: Istio-based Service Mesh Add-on

The [Istio-based service mesh add-on for AKS](https://learn.microsoft.com/azure/aks/istio-about) provides a fully managed Istio experience, including ingress gateway capabilities. If you need service mesh features (mTLS between services, traffic management, observability), Istio's ingress gateway can replace NGINX.

### Architecture

Istio on AKS deploys:
- **Istiod control plane** — managed by Microsoft, handles configuration, certificate management, and service discovery
- **Envoy sidecar proxies** — injected into application pods for mesh traffic
- **Ingress gateways** — dedicated Envoy proxies that handle north-south traffic (external/internal)

```
┌──────────────────────────────────────────┐
│              AKS Cluster                 │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │  aks-istio-ingress namespace     │    │
│  │  ┌────────────────────────────┐  │    │
│  │  │ Istio Ingress Gateway      │  │    │
│  │  │ (Envoy)                    │  │    │
│  │  │ aks-istio-ingressgateway-  │  │    │
│  │  │ external (LoadBalancer)    │  │    │
│  │  └──────────┬─────────────────┘  │    │
│  └─────────────┼────────────────────┘    │
│                │                          │
│  ┌─────────────┼────────────────────┐    │
│  │  istiod (managed control plane)  │    │
│  └─────────────┼────────────────────┘    │
│                │                          │
│  ┌─────────────┼────────────────────┐    │
│  │  Application namespace           │    │
│  │  ┌──────┐  │  ┌──────┐          │    │
│  │  │Pod+  │◄─┘  │Pod+  │          │    │
│  │  │Envoy │◄───►│Envoy │          │    │
│  │  │sidecar│    │sidecar│          │    │
│  │  └──────┘     └──────┘          │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

### Gateway API Support in Istio on AKS

As of **revision asm-1-26** and later, the Istio add-on supports [Gateway API for ingress traffic management](https://learn.microsoft.com/azure/aks/istio-gateway-api) (preview). This uses the `istio` GatewayClass and requires the [Managed Gateway API](https://learn.microsoft.com/azure/aks/managed-gateway-api) feature to be enabled on your cluster.

### When to Choose Istio

✅ You need service mesh capabilities (mTLS, traffic management, observability between services)
✅ You already use or plan to use Istio
✅ You want end-to-end encryption (mutual TLS) across all service-to-service communication
✅ You need advanced traffic management (fault injection, retries, circuit breaking)
✅ You want fine-grained access control (AuthorizationPolicy)

### Deployment Walkthrough

#### Enable the Istio Add-on

```bash
RESOURCE_GROUP='my-resource-group'
CLUSTER='my-aks-cluster'

# Enable Istio add-on
az aks mesh enable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER

# Enable external ingress gateway
az aks mesh enable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER \
  --ingress-gateway-type external
```

#### Using Istio Classic API (Gateway + VirtualService)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-vs
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - my-gateway
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  - route:
    - destination:
        host: frontend-service
        port:
          number: 80
```

#### Using Gateway API (Preview, asm-1-26+)

```bash
# Enable managed Gateway API on the cluster
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER \
  --enable-gateway-api
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
  - backendRefs:
    - name: frontend-service
      port: 80
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Full service mesh capabilities | Higher resource overhead (Envoy sidecars) |
| mTLS across all services automatically | More complex to operate and troubleshoot |
| Microsoft-managed control plane | Gateway API support still in preview |
| No extra cost for the add-on | No ambient mesh support yet |
| Rich traffic management (retries, fault injection, circuit breaking) | No Windows container support |
| Integrated with Azure Monitor + Grafana | Blocked customizations (ProxyConfig, WasmPlugin, etc.) |

### Pricing

The Istio service mesh add-on itself is **free** — there's no additional charge beyond the standard AKS compute costs. However, Envoy sidecars consume CPU and memory in each pod, so factor in the resource overhead (typically 50-100MB RAM and 0.1 CPU per sidecar).

### Key Limitations

- No **ambient mesh** (sidecar-less) support yet (on roadmap)
- No **multi-cluster** mesh support
- No **Windows container** support
- **Gateway API for Istio ingress** is preview (requires asm-1-26+)
- Certain CRDs blocked: `ProxyConfig`, `WorkloadEntry`, `WorkloadGroup`, `IstioOperator`, `WasmPlugin`

---

## Option 3: Managed NGINX (Application Routing Add-on)

If you're already using the Application Routing add-on, you don't need to panic. Microsoft has committed to supporting it through November 2026, and the add-on is evolving toward Gateway API alignment.

### Current Status

From the [official AKS documentation](https://learn.microsoft.com/azure/aks/app-routing):

> *AKS is aligning with upstream Kubernetes by moving to Gateway API as the long-term standard for ingress and L7 traffic management. We recommend you start planning your migration path based on your current setup.*

The Application Routing add-on team is working on [Gateway API support within the add-on itself](https://github.com/Azure/AKS/issues/5515). This means you may be able to migrate from Ingress resources to Gateway API resources without changing your underlying ingress product.

### When It Still Makes Sense

- You're already on the Application Routing add-on and your setup is simple (basic path/host routing)
- You need a managed NGINX solution with Azure DNS and Key Vault integration **today**
- You're migrating from self-managed NGINX and need a supported bridge while planning your Gateway API migration
- Your migration timeline extends into late 2026

### Migrating from Self-Managed NGINX to Managed NGINX

If you're running self-managed community NGINX, migrating to the Application Routing add-on buys you time:

```bash
# Enable the add-on on your existing cluster
az aks approuting enable \
  --resource-group <ResourceGroupName> \
  --name <ClusterName>
```

Then update your Ingress resources to use the managed IngressClass:

```yaml
# Before (self-managed)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80

# After (managed)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Limitations

- Up to **5 Azure DNS zones** per cluster
- Editing the ingress-nginx **ConfigMap in `app-routing-system`** is not supported
- Certain **snippet annotations are blocked** (`load_module`, `lua_package`, `_by_lua`, `location`, `root`, `proxy_pass`, etc.)
- Must use managed identity clusters
- All global/private Azure DNS zones must be in the same resource group (per zone type)
- **End of life: November 2026** — this is a bridge, not a destination

---

## Option 4: Third-Party Solutions

If Azure-native solutions don't fit your needs, several third-party ingress controllers support Gateway API and continue active development.

### Traefik

[Traefik](https://traefik.io/) is a popular cloud-native ingress controller with full Gateway API support since v3.0.

- **Gateway API:** Full implementation including TCPRoute, TLSRoute, GRPCRoute
- **Middleware:** Rich request/response transformation pipeline
- **Auto-discovery:** Native Kubernetes service discovery
- **Dashboard:** Built-in monitoring UI
- **Let's Encrypt:** Automatic certificate management
- **Best for:** Teams that want a feature-rich, self-managed ingress with strong Gateway API support

### Kong

[Kong Gateway](https://konghq.com/) provides both open-source and enterprise ingress controllers.

- **Gateway API:** Full implementation via Kong Ingress Controller (KIC)
- **Plugins:** Extensive plugin ecosystem (rate limiting, auth, transformations)
- **API Gateway:** Full API management platform (enterprise)
- **Best for:** Teams that need API gateway functionality (rate limiting, API key management, developer portal)

### Envoy Gateway

[Envoy Gateway](https://gateway.envoyproxy.io/) is the official Gateway API implementation by the Envoy project.

- **Gateway API:** Reference implementation, always up-to-date with spec
- **Envoy Proxy:** Built on the same proxy used by Istio and many service meshes
- **Extensibility:** Envoy's filter chain model
- **CNCF project:** Community-driven, vendor-neutral
- **Best for:** Teams that want a pure Gateway API implementation without vendor lock-in

### HAProxy

[HAProxy Kubernetes Ingress Controller](https://www.haproxy.com/documentation/kubernetes-ingress/) offers high-performance L7 load balancing.

- **Gateway API:** Supported in recent versions
- **Performance:** Known for extremely high throughput and low latency
- **Enterprise features:** Advanced health checking, connection pooling
- **Best for:** Extremely high-performance requirements

### Quick Comparison

| Solution | Gateway API | Managed on Azure | Free Tier | Commercial Support |
|----------|------------|-----------------|-----------|-------------------|
| Traefik | ✅ Full | No (self-managed) | OSS | Traefik Enterprise |
| Kong | ✅ Full | No (self-managed) | OSS | Kong Enterprise |
| Envoy Gateway | ✅ Full | No (self-managed) | OSS | Tetrate, Solo.io |
| HAProxy | ✅ Partial | No (self-managed) | Community | HAProxy Enterprise |

> **My recommendation:** Unless you have specific requirements that only a third-party solution meets, stick with Azure-native options (AGC or Istio) for better integration and support. If you do go third-party, Envoy Gateway or Traefik are the strongest Gateway API options.

---

## Migration Strategy Framework

### Assessment Checklist

Before migrating, audit your current NGINX setup:

**1. Inventory your Ingress resources:**
```bash
# List all Ingress resources across all namespaces
kubectl get ingress --all-namespaces -o wide

# Export all Ingress manifests for review
kubectl get ingress --all-namespaces -o yaml > all-ingresses.yaml
```

**2. Catalog annotations in use:**
```bash
# Find all unique NGINX annotations
kubectl get ingress --all-namespaces -o json | \
  jq -r '.items[].metadata.annotations // {} | keys[]' | \
  sort -u | grep nginx
```

**3. Check for ConfigMap customisations:**
```bash
kubectl get configmap -n ingress-nginx -o yaml
# or for app-routing:
kubectl get configmap -n app-routing-system -o yaml
```

**4. Review TLS configuration:**
```bash
kubectl get ingress --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.tls) | .metadata.namespace + "/" + .metadata.name'
```

**5. Check for custom snippets (high risk!):**
```bash
kubectl get ingress --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.annotations["nginx.ingress.kubernetes.io/server-snippet"] or .metadata.annotations["nginx.ingress.kubernetes.io/configuration-snippet"]) | .metadata.namespace + "/" + .metadata.name'
```

### Decision Tree

```
Start Here: What's your current setup?
│
├── Self-managed community NGINX (ingress-nginx)
│   ├── Need service mesh features? ──► Istio Service Mesh Add-on
│   ├── Need WAF? ──► Application Gateway for Containers
│   ├── Simple routing only? ──► App Routing Add-on (bridge) → Gateway API
│   └── Complex annotations/snippets? ──► AGC or evaluate third-party
│
├── Application Routing Add-on (managed NGINX)
│   ├── Simple setup? ──► Wait for Gateway API support in add-on
│   ├── Need WAF or advanced features now? ──► AGC
│   └── Need mesh? ──► Istio + AGC or Istio Gateway
│
├── HTTP Application Routing (legacy)
│   └── Migrate IMMEDIATELY ──► App Routing Add-on or AGC
│
└── NGINX Inc commercial (F5)
    └── Continues independently — evaluate renewal vs migration
```

### Step-by-Step Migration Approach

1. **Assess** — Inventory all Ingress resources, annotations, ConfigMap customisations, and TLS configs
2. **Choose target** — Use the decision tree above to select your migration target
3. **Deploy target alongside NGINX** — Run both in parallel; never cut over in one step
4. **Migrate one service at a time** — Start with the simplest, lowest-risk service
5. **Test thoroughly** — Validate routing, TLS, headers, rewrites, and performance
6. **Cut over DNS/traffic** — Update DNS or traffic split to point to the new gateway
7. **Monitor** — Watch error rates, latency, and throughput for at least 48 hours
8. **Decommission NGINX** — Remove old Ingress resources and controller only after validation

### Blue-Green / Canary Migration Pattern

The safest approach is a weighted traffic split during migration:

```
                    ┌──────────────────┐
                    │   DNS / Azure    │
                    │   Traffic Mgr    │
                    │   or Front Door  │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │                  │
              90% weight          10% weight
                    │                  │
           ┌────────┴───────┐  ┌──────┴────────┐
           │  NGINX Ingress │  │  New Gateway   │
           │  (existing)    │  │  (AGC/Istio)   │
           └────────┬───────┘  └──────┬─────────┘
                    │                  │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │   Backend Pods   │
                    └──────────────────┘
```

Use [Azure Traffic Manager](https://learn.microsoft.com/azure/traffic-manager/traffic-manager-overview) or [Azure Front Door](https://learn.microsoft.com/azure/frontdoor/front-door-overview) to implement weighted routing between old and new gateways. Gradually shift traffic: 10% → 25% → 50% → 100%.

### Rollback Strategy

Always maintain rollback capability:

1. **Keep NGINX running** until the new gateway is fully validated in production
2. **Don't delete Ingress resources** — just remove them from active DNS
3. **Maintain NGINX controller deployment** with a lower resource allocation
4. **Document the rollback procedure** — DNS change or Traffic Manager weight adjustment
5. **Set a decommission date** at least 2 weeks after 100% cutover

---

## Hands-On Migration: NGINX Ingress → Application Gateway for Containers

This section provides complete, copy-pasteable examples for migrating common NGINX patterns to AGC using Gateway API.

### Annotation Mapping Table

| NGINX Annotation | Gateway API Equivalent |
|------------------|----------------------|
| `nginx.ingress.kubernetes.io/rewrite-target: /` | `HTTPURLRewriteFilter` on HTTPRoute |
| `nginx.ingress.kubernetes.io/ssl-redirect: "true"` | HTTPS listener on Gateway + HTTPRoute redirect filter |
| `nginx.ingress.kubernetes.io/use-regex: "true"` | `path.type: RegularExpression` on HTTPRoute match |
| `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` | AGC-specific RoutePolicy or backend setting |
| `nginx.ingress.kubernetes.io/cors-*` | `RequestHeaderModifier` / `ResponseHeaderModifier` filter |
| `nginx.ingress.kubernetes.io/affinity: "cookie"` | Session affinity via AGC RoutePolicy |
| `nginx.ingress.kubernetes.io/canary*` | Weighted `backendRefs` on HTTPRoute |
| `nginx.ingress.kubernetes.io/auth-tls-*` | `FrontendTLSPolicy` CRD (AGC-specific) |
| `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` | `BackendTLSPolicy` CRD (AGC-specific) |
| `nginx.ingress.kubernetes.io/whitelist-source-range` | AGC RoutePolicy or Azure NSG |

### Example 1: Basic Host + Path Routing

**Before (NGINX Ingress):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**After (Gateway API with AGC):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: production
  annotations:
    alb.networking.azure.io/alb-namespace: alb-infra
    alb.networking.azure.io/alb-name: my-agc
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: myapp.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: myapp-tls
    allowedRoutes:
      namespaces:
        from: Same
  - name: http-redirect
    port: 80
    protocol: HTTP
    hostname: myapp.example.com
---
# Redirect HTTP to HTTPS
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
  namespace: production
spec:
  parentRefs:
  - name: my-gateway
    sectionName: http-redirect
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
---
# API route with URL rewrite
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: production
spec:
  parentRefs:
  - name: my-gateway
    sectionName: https
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: api-service
      port: 8080
---
# Frontend catch-all route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: production
spec:
  parentRefs:
  - name: my-gateway
    sectionName: https
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

### Example 2: Canary Deployment (Traffic Splitting)

**Before (NGINX Ingress — requires two Ingress resources with canary annotations):**

```yaml
# Primary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-primary
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-v1
            port:
              number: 80
---
# Canary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-v2
            port:
              number: 80
```

**After (Gateway API — native weighted routing):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-canary
  namespace: production
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - myapp.example.com
  rules:
  - backendRefs:
    - name: my-app-v1
      port: 80
      weight: 90
    - name: my-app-v2
      port: 80
      weight: 10
```

*Much cleaner.* One resource instead of two, with native weight support.

### Example 3: Header-Based Routing

**Before (NGINX — custom annotations/snippets):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      if ($http_x_version = "v2") {
        set $proxy_upstream_name "production-my-app-v2-80";
      }
```

**After (Gateway API — native header matching):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-routing
  namespace: production
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - headers:
      - name: x-version
        value: v2
    backendRefs:
    - name: my-app-v2
      port: 80
  - backendRefs:
    - name: my-app-v1
      port: 80
```

### Common Gotchas

1. **Subnet sizing:** AGC requires a dedicated subnet with at least **250 available IP addresses** (`/24` or larger). Plan this in your VNet design.

2. **Port restrictions:** AGC Gateway listeners only support ports **80 and 443**. If you're using custom ports, you'll need to adjust.

3. **Provisioning time:** AGC resources take **5-6 minutes** to provision. Don't panic during initial deployment.

4. **NGINX regex paths don't translate directly:** NGINX uses PCRE regex; Gateway API uses RE2 syntax. Test your regex patterns.

5. **ConfigMap tweaks:** If you relied on NGINX ConfigMap customisations (proxy-buffer-size, large-client-header-buffers, etc.), you'll need to find AGC equivalents or accept defaults.

6. **Snippet annotations:** If you use `server-snippet` or `configuration-snippet`, these have no Gateway API equivalent. You'll need to refactor this logic using native HTTPRoute features or backend logic.

7. **Session affinity:** NGINX cookie-based affinity requires AGC-specific `RoutePolicy` CRDs rather than Gateway API standard resources.

---

## Hands-On Migration: NGINX Ingress → Istio Gateway

If you're adopting Istio as a service mesh, its ingress gateway provides a natural replacement for NGINX.

### Prerequisites

```bash
# Enable the Istio add-on
az aks mesh enable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER

# Enable external ingress gateway
az aks mesh enable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER \
  --ingress-gateway-type external

# Verify the gateway service
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```

### Before/After: Basic Service Exposure

**Before (NGINX Ingress):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

**After (Istio Gateway + VirtualService):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: myapp-tls  # Kubernetes Secret
    hosts:
    - "myapp.example.com"
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true
    hosts:
    - "myapp.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - my-gateway
  http:
  - route:
    - destination:
        host: my-service
        port:
          number: 80
```

**After (Istio with Gateway API — preview, asm-1-26+):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: myapp.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: myapp-tls
  - name: http
    port: 80
    protocol: HTTP
    hostname: myapp.example.com
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
  - name: my-gateway
    sectionName: https
  rules:
  - backendRefs:
    - name: my-service
      port: 80
---
# HTTP to HTTPS redirect
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
spec:
  parentRefs:
  - name: my-gateway
    sectionName: http
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

### Configuring mTLS (Mesh-Wide)

One of Istio's biggest advantages is automatic mTLS between all services in the mesh:

```yaml
# Enable STRICT mTLS for the entire mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

For namespace-level mTLS:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Authorization Policies

Control which services can communicate:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/frontend-service"
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

### Common Gotchas

1. **Sidecar injection:** Ensure your application namespace has the Istio injection label:
   ```bash
   kubectl label namespace production istio.io/rev=asm-1-26
   ```

2. **Service port naming:** Istio requires service ports to follow the `<protocol>[-<suffix>]` naming convention (e.g., `http-web`, `grpc-api`). Without this, Istio treats traffic as TCP.

3. **Gateway API is preview:** The Istio Gateway API integration on AKS requires asm-1-26+ and is still in preview. Use Istio's native Gateway/VirtualService for production today.

4. **Resource overhead:** Each sidecar adds ~50-100MB RAM and ~0.1 CPU. Factor this into node sizing.

5. **Startup order:** Sidecars must be ready before the application container. Use `holdApplicationUntilProxyStarts` in MeshConfig if you see startup race conditions.

---

## Monitoring & Observability Post-Migration

Switching your ingress layer is high-risk. Robust monitoring is non-negotiable.

### Key Metrics to Watch

| Metric | What to Watch | Alert Threshold |
|--------|--------------|-----------------|
| **Request success rate** | 2xx/3xx vs 4xx/5xx ratio | < 99.5% over 5 min |
| **P50/P95/P99 latency** | Compare against NGINX baseline | > 2x baseline |
| **Connection errors** | TCP resets, timeouts | Any increase |
| **TLS handshake failures** | Certificate issues | Any occurrence |
| **Backend health** | Healthy vs unhealthy targets | Any unhealthy |
| **Throughput** | Requests per second | Unexpected drops |

### Azure Monitor Integration

#### For Application Gateway for Containers

AGC emits metrics to Azure Monitor automatically:

```bash
# View AGC metrics
az monitor metrics list \
  --resource <AGC_RESOURCE_ID> \
  --metric "TotalRequests,FailedRequests,ResponseStatus,HealthyHostCount,UnhealthyHostCount" \
  --interval PT1M
```

Enable diagnostic logs:

```bash
az monitor diagnostic-settings create \
  --name agc-diagnostics \
  --resource <AGC_RESOURCE_ID> \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace <LOG_ANALYTICS_WORKSPACE_ID>
```

#### For Istio

The Istio add-on integrates with [Azure Monitor managed service for Prometheus](https://learn.microsoft.com/azure/azure-monitor/essentials/prometheus-metrics-overview) and [Azure Managed Grafana](https://learn.microsoft.com/azure/managed-grafana/overview):

```bash
# Enable Prometheus metrics collection
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER \
  --enable-azure-monitor-metrics

# Key Prometheus metrics for Istio
# istio_requests_total - Total request count
# istio_request_duration_milliseconds - Request latency
# istio_tcp_connections_opened_total - TCP connections
# envoy_server_memory_allocated - Sidecar memory usage
```

### Troubleshooting Common Issues

**502 Bad Gateway after migration:**
- Check backend pod health and readiness probes
- Verify service port names match (especially for Istio)
- Check if the new gateway can reach your pods (network policies, NSGs)

**TLS certificate errors:**
- Verify secrets are in the correct namespace
- Check certificate chain completeness (intermediate certs)
- For AGC: ensure the TLS secret is referenced correctly in the Gateway listener

**Increased latency:**
- For AGC: expected during initial provisioning; should stabilise within minutes
- For Istio: sidecar proxy adds ~1-2ms per hop; check if sidecars are resource-constrained
- Compare with NGINX baseline — some difference is normal

**Routes not matching:**
- Gateway API matching is case-sensitive and strict
- PathPrefix `/api` matches `/api`, `/api/`, `/api/v1` but NOT `/apikeys`
- Check HTTPRoute status: `kubectl get httproute <name> -o yaml` — look for `Accepted` and `ResolvedRefs` conditions

**Traffic not flowing:**
- Verify the Gateway has an address assigned: `kubectl get gateway <name> -o yaml`
- Check the Gateway status shows `Programmed: True`
- Verify `allowedRoutes` on the Gateway listener matches your HTTPRoute's namespace

---

## Summary & Recommendations

### Quick Comparison

| Feature | AGC | Istio Add-on | App Routing (NGINX) | Third-Party |
|---------|-----|-------------|---------------------|-------------|
| **Gateway API** | ✅ GA | ✅ Preview | 🔜 Planned | ✅ Varies |
| **Ingress API** | ✅ | ❌ (own CRDs) | ✅ | ✅ |
| **Managed by Azure** | ✅ | ✅ (control plane) | ✅ | ❌ |
| **WAF** | ✅ | ❌ | ❌ | Varies |
| **mTLS (frontend)** | ✅ | ✅ | ❌ | Varies |
| **mTLS (service mesh)** | ❌ (needs Istio) | ✅ | ❌ | Varies |
| **Traffic splitting** | ✅ Native | ✅ Native | ❌ Annotation hack | Varies |
| **Data plane location** | Azure-managed (external) | In-cluster (sidecars) | In-cluster | In-cluster |
| **Base cost** | ~$120+/month | Free (compute overhead) | Free | Free (OSS) |
| **Long-term viability** | ✅ Strategic | ✅ Strategic | ⚠️ EOL Nov 2026 | ✅ |
| **Complexity** | Medium | High | Low | Medium-High |

### Recommended Path by Scenario

**🟢 "I just need simple HTTP/HTTPS routing"**
→ **Application Gateway for Containers.** It's the Azure-native answer, supports both Ingress (for migration) and Gateway API (for the future), and has a clear roadmap.

**🟢 "I need a service mesh (mTLS everywhere, advanced traffic policies)"**
→ **Istio service mesh add-on.** Use Istio's ingress gateway now, migrate to Gateway API when it goes GA. Optionally pair with AGC for WAF and L7 load balancing at the edge.

**🟡 "I'm on the App Routing add-on and things are working fine"**
→ **Stay put for now**, but start planning. Wait for Gateway API support in the add-on. Begin learning Gateway API concepts. Have a migration plan ready for Q3 2026.

**🟡 "I'm on self-managed community NGINX"**
→ **Migrate to the App Routing add-on immediately** for support coverage, then plan your Gateway API migration to AGC or Istio. Don't wait until March 2026 when upstream goes EOL.

**🔴 "I'm on the legacy HTTP Application Routing add-on"**
→ **Migrate immediately.** This is already retired. Move to the App Routing add-on as a quick fix, then plan your longer-term migration.

**🟣 "I need API gateway features (rate limiting, auth, dev portal)"**
→ **Kong or a dedicated API gateway** in front of AGC/Istio. Gateway API handles routing; API management is a separate concern.

### The Long View

The Kubernetes ecosystem is consolidating around Gateway API. Every major cloud provider, every major proxy, and every major service mesh now supports it. NGINX's retirement isn't just about one controller going away — it's the signal that the Ingress API era is ending.

**Start your migration now.** November 2026 sounds far away, but production migrations with proper testing, canary rollouts, and team training take months, not weeks.

---

### Official Documentation Links

- [AKS Application Routing Add-on](https://learn.microsoft.com/azure/aks/app-routing)
- [Application Gateway for Containers Overview](https://learn.microsoft.com/azure/application-gateway/for-containers/overview)
- [AGC Quickstart (ALB Managed)](https://learn.microsoft.com/azure/application-gateway/for-containers/quickstart-create-application-gateway-for-containers-managed-by-alb-controller)
- [AGC Pricing](https://learn.microsoft.com/azure/application-gateway/for-containers/understanding-pricing)
- [Istio Service Mesh Add-on for AKS](https://learn.microsoft.com/azure/aks/istio-about)
- [Istio Ingress Gateway Deployment](https://learn.microsoft.com/azure/aks/istio-deploy-ingress)
- [Istio Gateway API on AKS (Preview)](https://learn.microsoft.com/azure/aks/istio-gateway-api)
- [Migration from OSS Istio to AKS Add-on](https://learn.microsoft.com/azure/aks/migration-from-open-source-istio-to-addon)
- [Kubernetes Gateway API Specification](https://gateway-api.sigs.k8s.io/)
- [Ingress-NGINX Retirement Announcement](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/)
- [AKS GitHub Issue #5516: Retirement Tracking](https://github.com/Azure/AKS/issues/5516)
- [AKS GitHub Issue #5515: Gateway API in App Routing](https://github.com/Azure/AKS/issues/5515)

---

*Published on March 9, 2026. All information is current as of publication date. Check the linked documentation for the latest updates.*
