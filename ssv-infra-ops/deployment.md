# SSV Infra/Ops - Deployment

This document describes how SSV node deployments are packaged and promoted using `charts`, `gitops-stage`, and `gitops-production`.

## Read this if

- You are changing chart/appset versions and need safe rollout/rollback flow.
- You need to know where stage and production deployment logic diverge.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Deployment process
- Repo responsibilities
- Promotion and rollback model

## Outcome

- Clear understanding of how SSV node infra changes move from chart edits to running workloads.

## Repositories in scope

Validated repositories:

1. `ssvlabs/charts`
2. `ssvlabs/gitops-stage`
3. `ssvlabs/gitops-production` (this is the production repo name; teams may call it `gitops-prod`)

## Deployment topology (GitOps)

### App-of-apps chain

Stage:

- Cluster app: `gitops-stage/clusters/stage/45-environment.yaml`
- Points to `path: environments/stage` in `gitops-stage`

Production:

- OVH cluster app: `gitops-production/clusters/ovh/environment.yaml`
- AWS cluster app: `gitops-production/clusters/aws/environment.yaml`
- These point to `environments/ovh` and `environments/aws` respectively.

### SSV node entry points

Stage:

- `environments/stage/ssv-nodes-set.yaml` (ApplicationSet, dynamic generation plugin)
- `environments/stage/ssv-node-exporter.yaml` -> `environments/stage/ssv-node-exporter/applicationset.yaml`

Production:

- OVH mainnet nodes: `environments/ovh/mainnet/mainnet-ssv-nodes-set.yaml`
- OVH hoodi nodes: `environments/ovh/hoodi/ssv-nodes-set.yaml`
- AWS hoodi arm64 nodes: `environments/aws/hoodi/ssv-nodes-set-arm64.yaml` (currently empty elements list)
- Exporters are separate Applications, e.g.:
  - `environments/ovh/mainnet/ssv-node-exporter-1.yaml`
  - `environments/ovh/mainnet/ssv-node-exporter-2.yaml`
  - `environments/ovh/hoodi/ssv-node-exporter.yaml`
  - `environments/aws/sepolia/ssv-node-exporter.yaml`

## What `ssvlabs/charts` defines

### Packaging and publish flow

- CI lint/template checks: `.github/workflows/helm-check.yaml`
- Push-to-main publish trigger: `.github/workflows/main.yaml`
- Publish performed via Argo workflow: `.argo/publish.yaml`

### Important workflow behavior

- Chart version bump is automated by CI process.
- Repo guidance (`AGENTS.md`, `CLAUDE.md`) says not to manually edit `Chart.yaml` version.

## End-to-end deployment model

1. Chart changes merge to `charts` main.
2. Charts CI publishes chart updates to chartmuseum (`https://chartmuseum.ops.ssvlabsinternal.com`).
3. GitOps repos pin chart version and override values in Argo `Application`/`ApplicationSet`.
4. ArgoCD auto-sync (`prune: true`, `selfHeal: true`) reconciles to cluster.

## Stage vs production rollout characteristics

### Stage (`ssv-nodes-set`)

- Uses plugin generator (`argo-deployment-generator`) with:
  - `githubRepo: ssvlabs/ssv`
  - `defaultBranch: stage`
  - requeue every 30s
- Chart version pinned in template (currently `0.2.95`).
- Image tag comes from generator output (`{{ .tag }}`), so stage can advance quickly.

### Production (`ssv-nodes-set`)

- Uses explicit static lists of node instances in YAML.
- Chart version pinned per appset (`0.2.93` in sampled files).
- Image tags are explicit per node group.
- This is slower but more controlled than stage generator flow.

## Environment-specific inputs that must be managed outside chart defaults

Examples from real GitOps manifests:

- `network` (`hoodi-stage`, `hoodi`, `mainnet`, `sepolia`)
- `vaultSecretPath` and `clusterSecretStoreName`
- `consensusAddress` / `executionAddress` (often multi-endpoint semicolon lists)
- `p2pPort` and `p2pService.externalTrafficPolicy`
- `resources`, `persistence`, `storageClassName`
- exporter toggles (`FULLNODE`, `EXPORTER`, `EXPORTER_MODE`)
- TLS signer wiring (web3signer integration in mainnet OVH set)

## Safe rollout checklist

1. Chart validation
```bash
helm lint charts/ssv-node
helm template <release> charts/ssv-node --namespace <ns>
```

2. GitOps diff
- Open Argo app diff for changed `Application`/`ApplicationSet`.
- Verify generated child apps for ApplicationSets before sync.

3. Pre-deploy checks
- Secret store path exists for target release fullname.
- PVC class and size available in target cluster.
- P2P NodePort availability and firewall rules validated.

4. Post-deploy checks
```bash
kubectl -n <ns> get sts,po,svc,ep | grep <release>
kubectl -n <ns> logs <pod> --tail=200
```

## Rollback model

Preferred:

- Revert the GitOps manifest commit (chart version/tag/values) and let Argo self-heal to previous desired state.

Emergency:

- Disable auto-sync for the affected app and manually sync to known-good revision in Argo UI/CLI.

Critical notes:

- Rollback safety depends on PVC compatibility and key-source continuity.
- Stage and production are intentionally on different chart versions in current manifests; do not assume parity.

## Observed operational risks in GitOps manifests

1. Sensitive env in Git
- Stage exporter node files include `NETWORK_PRIVATE_KEY` in plaintext values.
- Even if this is not the operator key, it is still sensitive infrastructure key material.

2. Version drift
- Stage and production pin different `ssv-node` chart revisions (`0.2.95` vs `0.2.93` in sampled files).
- Recovery runbooks must be environment-specific.

3. Auto-sync blast radius
- Most apps use automated prune + self-heal.
- A bad commit can propagate immediately without manual gate unless branch protections/checks are strict.

4. Stale manifest references
- Some AWS exporter files are explicitly marked "no longer used" and point operators to OVH paths.
- Recovery/patch work should follow the active path annotations, not filename intuition.
