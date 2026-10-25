# Cluster overview

This document records a read-only inspection of the live Kubernetes cluster and the repository on **October 24, 2026**. It complements the [welcome README](../README.md) with the evidence behind the architecture summary. No cluster configuration was changed during the inspection, and Kubernetes Secret payloads were not retrieved.

## Compute and orchestration

The cluster has four Ready nodes: one control-plane/etcd node and three workers. All reported RKE2 `v1.36.5+rke2r1`, Rocky Linux 10.2, and containerd `2.3.4-k3s1`.

The nodes expose 12 CPUs in total and approximately 43.5 GiB of memory to Kubernetes. These are guest-reported capacities, not a physical hardware inventory or available workload capacity. Proxmox hosting and mini-PC hardware are described in the repository's existing project documentation; the hypervisor configuration was not independently inspected.

There is one control-plane node, so the cluster does not have a highly available control plane. The System Upgrade Controller is deployed, but no upgrade Plans were present at inspection time.

## GitOps and application inventory

Four ApplicationSets organize deployments by directory. All 14 live Applications reported `Synced` and `Healthy`, with automated sync, pruning, and self-healing enabled.

| ApplicationSet | Applications |
| --- | --- |
| `core-applications` | Argo CD, cert-manager, Cilium, ExternalDNS, External Secrets, Glance, MetalLB, Traefik |
| `applications` | Nexus, Renovate |
| `observability` | Gatus, metrics-server, VictoriaMetrics Kubernetes stack |
| `storage` | Longhorn |

The chart dependencies and local configuration live under [kubernetes-deployments](../kubernetes-deployments). Argo CD's repository configuration includes Nexus proxies for upstream Helm repositories. Git remains the reviewable record of desired application configuration.

The intended change flow is:

1. Edit a chart or its values, or review a Renovate dependency PR.
2. Merge the desired change into the tracked Git branch.
3. Argo CD renders the chart and reconciles the Application.
4. Check application health, workload readiness, and relevant metrics after deployment.

Renovate currently enables only the `helmv3` manager and ignores `values*.yaml`. Its policy therefore covers Helm dependency updates, not every image tag embedded in values. Only the Renovate chart has automerge enabled; other dependency PRs require a manual merge.

## Networking, DNS, and certificates

Cilium supplies cluster networking, with Hubble relay and UI deployed for visibility. MetalLB is configured with an address pool and Layer 2 advertisement. Traefik uses a LoadBalancer Service and routes ingress traffic to workloads.

ExternalDNS is configured to update Pi-hole using an `upsert-only` policy. cert-manager uses Let's Encrypt issuers and Cloudflare DNS-01 challenges. All eight observed Certificate resources reported Ready.

Pi-hole, Infisical, Cloudflare, and the virtualization layer are dependencies outside this Kubernetes workload inventory. The diagram in the README represents logical relationships, not a public exposure map or a full physical network topology.

## Secrets and storage

External Secrets Operator uses an Infisical ClusterSecretStore. All seven observed ExternalSecrets reported `SecretSynced` and Ready, with hourly refresh intervals. They supply credentials or configuration to cert-manager, ExternalDNS, Gatus, Glance, Nexus, Renovate, and Grafana.

Longhorn is the default storage class. Four persistent volume claims were Bound:

| Workload | Requested capacity | Observed volume state |
| --- | --- | --- |
| Nexus | 8 GiB | Attached, healthy, 3 replicas |
| VictoriaMetrics | 5 GiB | Attached, healthy, 3 replicas |
| Grafana | 2 GiB | Attached, healthy, 3 replicas |
| Gatus | 10 MiB | Attached, healthy, 3 replicas |

Replication was verified from live Longhorn volume resources. Backup destinations, restore tests, and recovery objectives were not verified; volume health should not be interpreted as evidence of recoverability.

## Observability

The live metrics stack includes VictoriaMetrics single-node storage, vmagent, vmalert, Alertmanager, Grafana, and kube-state-metrics. Hubble adds network-level visibility. The repository configures Grafana persistence and dashboard discovery.

Gatus is deployed with a sample external HTTP check and Slack alert configuration. This establishes the integration, but does not demonstrate comprehensive application health-check coverage or successful notification delivery. The VictoriaMetrics values disable default rule creation, so the existence of generated rule files in the repository alone is not proof that those rules are active.

## Configuration difference to follow up

The repository's [Renovate values](../kubernetes-deployments/applications/renovate/values.yaml) specify `0 2 * * 0`: Sundays at 2 AM in `America/New_York`. The live CronJob still reported `0 2 * * *`: daily at 2 AM, even though its Argo CD Application reported Synced.

The observed schedule is therefore still daily. The source revision and rendered CronJob should be compared before treating the weekly schedule as deployed. This inspection did not change the CronJob or diagnose the discrepancy further.

## Inspection scope

Evidence came from node and namespace inventories, Argo CD Applications and ApplicationSets, Deployments, StatefulSets, CronJobs, storage classes, PVCs, Longhorn volume status, ExternalSecret status, Certificate status, and upgrade Plan inventory, alongside the checked-in Helm configuration.

This was an architecture and readiness inspection, not a security audit, performance benchmark, disaster recovery exercise, or uptime measurement. Internal addresses, secret values, and private service URLs are intentionally omitted from this document.
