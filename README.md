# Xavier's Homelab

<div align="center">
  <img src="https://github.com/user-attachments/assets/00d1bbd1-30f7-4ab0-a4c1-97fd6e2f75be" width="128" height="128" alt="Homelab Architecture">
</div>

**A Kubernetes platform built to learn by operating it.**

Welcome! I'm Xavier, and this repository holds the configuration for my personal homelab. I started it when a new role required more Kubernetes experience. Building and maintaining my own cluster gave me a place to work through deployment failures, networking, storage, and upgrades firsthand.

Today, I run a four-node RKE2 cluster on Proxmox, with Argo CD managing applications from Git. This is an ongoing engineering project: a place to practice platform engineering, GitOps, and DevSecOps through real operational work.

## 🚀 Is Kubernetes Overkill for a Homelab?

Some may argue that Kubernetes is excessive for a homelab, but I see it as an opportunity:

- **Real-World Experience:** Many enterprises run Kubernetes in production. Why not gain hands-on experience at home?
- **Scalability & Flexibility:** Even in a homelab, container orchestration simplifies deployments, updates, and scaling.

While K8s has a learning curve, I believe the benefits outweigh the complexity, making it a valuable addition to my homelab, and in my career toolkit.

## 🔧 Hardware

My homelab runs on a mix of physical and virtual machines. Here’s a breakdown of my setup:

| Component                 | Processor          | Memory       | Storage       |
|---------------------------|--------------------|-------------|--------------|
| **Beelink EQ13 Pro Mini PC** | Intel Core i9 12900HK | 16GB | 512GB  |
| **Beelink GTi12 Mini PC**  | Intel Alder Lake- N200 | 64GB   | 1TB  |

## How it fits together

![Homelab architecture diagram](assets/homelab-platform.svg)

The diagram follows a change from Git to a running workload. **GitHub** holds the desired configuration: which **Helm** charts to deploy, their versions, and the settings for each service. **Argo CD** watches that configuration and uses [**ApplicationSets**](kubernetes-deployments/core/argocd/values.yaml) to discover applications from the repository's directory structure. It renders their **Helm** charts and reconciles the resulting resources with the cluster, with automated sync, pruning, and self-healing enabled. **Nexus** provides proxies for upstream **Helm** repositories, giving chart dependencies a central access point within the lab.

[**Renovate**](renovate.json) feeds into this workflow by opening pull requests for **Helm** dependency updates. I keep most updates behind a manual merge so I can review the change before deployment; only **Renovate**'s own chart is eligible for automerge. Once a change reaches Git, **Argo CD** handles deployment. This separates the decision to accept an update from the mechanics of applying it, while keeping the configuration history reviewable.

The workloads run on four **RKE2** virtual machines hosted on **Proxmox**: one control-plane node and three workers, all running **Rocky Linux** with **containerd**. [**Cilium**](kubernetes-deployments/core/cilium) provides networking between workloads, and **Hubble** makes those network flows visible. For incoming application traffic, [**MetalLB**](kubernetes-deployments/core/metallb) provides a load-balancer address on the local network and [**Traefik**](kubernetes-deployments/core/traefik) routes requests to the appropriate service. **ExternalDNS** maintains service DNS records in **Pi-hole**, while [**cert-manager**](kubernetes-deployments/core/cert-manager) obtains TLS certificates using **Let's Encrypt** and **Cloudflare** DNS validation.

Secrets follow a separate path from application configuration. **Infisical** holds the external secret values, and [**External Secrets Operator**](kubernetes-deployments/core/external-secrets/templates/secret-store.yaml) synchronizes the required values into **Kubernetes Secrets** for workloads to consume. This lets me keep references to credentials in Git without putting those credential values into application manifests. The external-services column shows these supporting integrations; it does not imply that **Pi-hole**, **Infisical**, or **Cloudflare** run inside this cluster.

[**Longhorn**](kubernetes-deployments/storage/longhorn) provides persistent storage for stateful services, including **Nexus**, **Grafana**, and [**VictoriaMetrics**](kubernetes-deployments/observability/victoria-metrics-k8s-stack). The inspected volumes each had three replicas, providing storage redundancy across the cluster. **VictoriaMetrics** collects metrics and **Grafana** presents them in dashboards, complementing **Hubble**'s network visibility. [**Gatus**](kubernetes-deployments/observability/gatus) provides a starting point for endpoint checks and Slack notifications. Together, these components let me follow a deployment beyond its manifests and inspect how the resulting workloads, storage, and network behave.

[Editable Excalidraw diagram](assets/homelab-architecture.excalidraw) · [Full-size image](assets/homelab-platform.svg)

To edit, download the `.excalidraw` file and open it in [Excalidraw](https://excalidraw.com). After editing, save the source and export an SVG to update the README image.

## Repository guide

```text
kubernetes-deployments/
├── core/             # GitOps, networking, DNS, certificates, secrets, dashboard
├── applications/     # Nexus and Renovate
├── observability/    # VictoriaMetrics stack, metrics-server, Gatus
└── storage/          # Longhorn
renovate.json         # Helm dependency update and merge policies
docs/                 # Verified cluster overview and operational notes
```

Each application directory contains a `Chart.yaml` declaring chart dependencies and a `values.yaml` defining local configuration. Some also include `templates/` for resources such as ExternalSecrets, issuers, or ingress configuration.

To follow a concrete example, start with [Nexus](kubernetes-deployments/applications/nexus): its chart, values, and secret template show how an application combines package management, persistent storage, ingress, and external credentials.

## Scope and tradeoffs

This is a personal learning environment with real workloads. A single control-plane node keeps the footprint manageable, but it is a control-plane failure point. Longhorn replication improves storage resilience; it does not establish a tested backup and recovery process. Several platform services also run as single replicas.

The repository primarily manages Kubernetes workloads. It is not a complete, one-command rebuild of the Proxmox hosts, cluster bootstrap, and external services. Those boundaries are part of the [architecture notes](docs/cluster-overview.md).

## Let's connect

I'm happy to discuss the design decisions, troubleshooting, and lessons behind this lab.

[LinkedIn](https://www.linkedin.com/in/xalexander/) · [Email](mailto:xalexander51@gmail.com) · [Repository discussions via issues](https://github.com/xalxnder/homelab/issues)
