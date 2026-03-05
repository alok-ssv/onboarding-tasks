# SSV Infra/Ops - Node Architecture in Kubernetes

This document describes how SSV nodes are deployed and run in Kubernetes using `ssvlabs/charts`, `ssvlabs/gitops-stage`, and `ssvlabs/gitops-production` (often referred to as `gitops-prod`).

## Read this if

- You are onboarding to production node lifecycle in Kubernetes.
- You need to map Argo apps to runtime StatefulSets/services.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Pods and containers
- Configs
- Services and networking
- Startup and shutdown behavior

## Outcome

- Clear mental model of how an SSV node is created, configured, scheduled, and exposed in cluster environments.

## Control plane topology (GitOps)

Stage chain:

- `gitops-stage/clusters/stage/45-environment.yaml` points ArgoCD to `environments/stage`.
- `environments/stage/ssv-nodes-set.yaml` manages validator node apps via plugin generator.
- `environments/stage/ssv-node-exporter/applicationset.yaml` manages exporter node apps via git files generator.

Production chain:

- `gitops-production/clusters/ovh/environment.yaml` points to `environments/ovh`.
- `gitops-production/clusters/aws/environment.yaml` points to `environments/aws`.
- Node ApplicationSets are explicit lists (`environments/ovh/mainnet/mainnet-ssv-nodes-set.yaml`, `environments/ovh/hoodi/ssv-nodes-set.yaml`, `environments/aws/sepolia/ssv-nodes-set.yaml`).
- Exporters are mostly standalone Argo Applications (`environments/ovh/mainnet/ssv-node-exporter-1.yaml`, `environments/ovh/mainnet/ssv-node-exporter-2.yaml`, `environments/ovh/hoodi/ssv-node-exporter.yaml`).

## Runtime workload shape (Helm chart)

Primary deployment object:

- `charts/ssv-node/templates/statefulset.yml` (StatefulSet, typically `replicas: 1` in GitOps values)

Primary container:

- `ssv-node` image from Helm values
- Starts through chart command wiring (default command path still driven by `values.yaml`)

Optional sidecars:

- `ssv-signer` sidecar when signer mode is enabled
- `ssv-pulse` sidecar when pulse telemetry is enabled

Optional init container:

- `init-nodeport` when `useExternalIPFromNode.enabled=true` (enabled in most stage/prod node definitions)

## Config and key wiring

ConfigMap path:

- `charts/ssv-node/templates/configmap.yaml` renders `share.yaml` from `values.config`
- Mounted as `/data/share.yaml` via subPath

Operator key path:

- `charts/ssv-node/templates/external-secrets.yaml` creates `ExternalSecret`
- Secret source configured per environment in GitOps (`stage/ssv-nodes-operators-private-keys`, `production/mainnet/ssv-nodes`, `production/hoodi/ssv-nodes`, `production/sepolia/ssv-nodes`)
- Secret store names vary by cluster (`stage-vault-kv-secret`, `production-ovh-vault-kv-secret`, `production-vault-kv-secret`)

Signer TLS path:

- Production mainnet OVH appset enables TLS files for web3signer in `mainnet-ssv-nodes-set.yaml`.

## Scheduling and placement model

Observed patterns in GitOps values:

- Single replica per node app (horizontal scaling is done by adding more named apps, not by scaling one StatefulSet).
- Production mainnet and hoodi appsets use anti-affinity/topology spread to separate nodes across hosts.
- Some environments pin zones or node types via node affinity/tolerations (for example AWS sepolia and arm64 definitions).

## Service and networking model

From chart and GitOps overrides:

- P2P uses NodePort (TCP+UDP) on per-node `p2pPort`.
- Many production apps set `p2pService.externalTrafficPolicy: Local`.
- `useExternalIPFromNode.enabled` is commonly enabled to publish reachable addresses.
- Exporter workloads may also expose ws/api ports and ingress in stage (for example stage exporter nodes).
- Headless service is still used for StatefulSet identity (`service-headless.yaml`).

## Startup and shutdown behavior

Startup sequence:

1. Git commit updates desired state in GitOps repo.
2. ArgoCD reconciles Application/ApplicationSet (`automated.prune=true`, `selfHeal=true` in sampled files).
3. Helm renders StatefulSet/Service/ExternalSecret resources.
4. Pod starts, init container may inject host IP, then node/signer/pulse containers start.

Shutdown and replacement:

- Deletions or spec changes in Git are applied automatically by Argo.
- Chart does not define liveness/readiness/startup probes for `ssv-node` StatefulSet.
- Chart does not define preStop/postStart hooks for `ssv-node`.

Operational implication:

- Lifecycle is strongly GitOps-driven; drift is corrected automatically.
- Process-level hangs may not self-recover quickly without probe-based restarts.

## Architectural risks to track

1. Automated sync blast radius
- With prune + self-heal enabled broadly, a bad Git commit can roll out fast.

2. Single-replica per app assumptions
- Most apps are one replica and rely on multi-node fleet composition for redundancy.

3. Sensitive network key material in stage exporter manifests
- Some stage exporter node files include plaintext `NETWORK_PRIVATE_KEY`.

4. Environment drift
- Stage and production pin different chart versions and image tags; runbooks must be environment-specific.
