# SSV Infra/Ops - Debugging and Failure Recovery

This document provides infra-level debugging and recovery guidance for SSV nodes running under ArgoCD GitOps in Kubernetes.

## Read this if

- You are on-call for SSV nodes in Kubernetes and need triage/runbook flow.
- You need fast diagnosis paths for rollout, scheduling, secret, and network failures.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Pod restarts
- Resource exhaustion
- Network disruptions

## Outcome

- Ability to diagnose failures quickly and apply low-risk recovery actions with GitOps awareness.

## Validation source and limitation

Sources used:

- `ssvlabs/charts` (`charts/ssv-node/*`)
- `ssvlabs/gitops-stage`
- `ssvlabs/gitops-production` (`gitops-prod`)

Runtime limitation:

- Live cluster failure simulation was not executed from this environment because cluster auth requires `tsh login` and `tsh kube login`.
- Scenarios below are based on manifests and operational runbook reasoning.

## Baseline triage checklist

1. ArgoCD app and appset health (first check)
```bash
kubectl -n argocd get applications,applicationsets | rg -i "ssv|exporter"
kubectl -n argocd describe applicationset <appset-name>
kubectl -n argocd describe application <app-name>
```

2. Pod health and restart signals
```bash
kubectl -n <ns> get pods -l app.kubernetes.io/name=ssv-node -o wide
kubectl -n <ns> describe pod <pod>
kubectl -n <ns> logs <pod> --previous
```

3. StatefulSet, services, and endpoints
```bash
kubectl -n <ns> get sts,svc,endpoints | rg <release>
kubectl -n <ns> get events --sort-by=.metadata.creationTimestamp | tail -n 80
```

4. External secret chain
```bash
kubectl -n <ns> get externalsecret,secret | rg <release>
kubectl -n <ns> describe externalsecret <release>
```

## Scenario 1: Desired app is missing or stale (GitOps generation failure)

### Symptoms

- Node app not created at all, or manifest changes do not appear in cluster.

### High-probability causes

- Stage plugin generator issues in `environments/stage/ssv-nodes-set.yaml` (`argo-deployment-generator`).
- Stage exporter files generator mismatch in `environments/stage/ssv-node-exporter/applicationset.yaml`.
- Wrong environment path/branch in cluster environment app.

### Recovery actions

1. Check ApplicationSet controller events and conditions.
2. Validate generator inputs (ranges, list elements, files path patterns).
3. Confirm cluster environment app points to the intended repo path (`clusters/stage/45-environment.yaml` for stage, `clusters/{ovh,aws}/environment.yaml` for prod).

## Scenario 2: Pod restarts (CrashLoopBackOff/OOMKill)

### Symptoms

- Restart count climbs, pod oscillates between Running and CrashLoopBackOff.

### High-probability causes

- Bad value override rendered from GitOps (`envs`, `extraEnvs`, `config`).
- Missing key material from ExternalSecret/Vault chain.
- Memory pressure (`OOMKilled`) against configured limits.

### Recovery actions

1. Verify rendered app values in Argo match intent.
2. Verify ExternalSecret status and resulting Secret keys.
3. Validate PVC mount and DB path behavior.
4. Restart only after root cause is fixed:
```bash
kubectl -n <ns> rollout restart statefulset/<release>
```

### Expected behavior

- PVC enabled: DB/state under `/data` should persist restart.
- Ephemeral mode (`emptyDir`): state loss on pod recreation.

## Scenario 3: Secret chain outage or misconfiguration

### Symptoms

- New pods fail to start with missing key env refs.
- ExternalSecret conditions show sync errors.

### Common causes

- Wrong `vaultSecretPath` for environment.
- Wrong `clusterSecretStoreName` for cluster.
- Vault backend/ESO availability incident.

### Recovery actions

1. Confirm environment-specific path/store pair in GitOps values.
2. Check ESO controller logs and `ExternalSecret` status.
3. Restore Vault access first; then recycle affected pods.

## Scenario 4: Scheduling failure or poor placement

### Symptoms

- Pods Pending with affinity, storage class, or resource errors.

### Causes seen in manifests

- Zone/node affinity constraints (for example sepolia and arm64 definitions).
- Topology spread and anti-affinity requirements (production OVH mainnet/hoodi).
- StorageClass mismatch (`local-path`, `local-storage`, `gp3-non-encrypted`) by cluster.

### Recovery actions

1. Inspect pod scheduling events in `describe pod`.
2. Confirm storage class exists in target cluster.
3. Reduce hard placement constraints only if operationally safe.

## Scenario 5: Network degradation or partition symptoms

### Symptoms

- Peer count drops, consensus lag, missed duties, exporter lag.

### Relevant configuration

- P2P NodePort TCP+UDP on per-node `p2pPort`.
- Often `externalTrafficPolicy: Local` in production node/exporter manifests.
- `useExternalIPFromNode.enabled: true` in many environments.

### Recovery actions

1. Verify service and endpoint health for each affected node.
2. Validate firewall/NACL/SG paths for node p2p ports.
3. Check for duplicated or colliding port allocations in GitOps lists.

## Scenario 6: Bad rollout due to automated sync

### Symptoms

- Widespread config drift or breakage immediately after merge.

### Why this happens

- Most SSV apps are `automated.prune=true` and `selfHeal=true`.

### Recovery actions

1. Revert the Git commit in the source GitOps repo.
2. Confirm Argo reconciles back to healthy revision.
3. For emergency containment, disable auto-sync on affected app, then sync known-good revision.

## Operational risks to keep visible

1. No probe-based restart logic in chart StatefulSet.
- Stuck process may remain Running without self-recovery.

2. Sensitive network keys present in stage exporter node YAML.
- `NETWORK_PRIVATE_KEY` appears in plaintext in some stage files.

3. Stale/unused manifests can mislead responders.
- Example: `gitops-production/environments/aws/mainnet/ssv-node-exporter-{1,2}.yaml` are marked as no longer used.

## Suggested failure drills (staging)

1. Generator drill
- Break and restore one ApplicationSet input to test alerting and recovery.

2. Secret drill
- Rotate one Vault key path and verify restart procedure.

3. Network drill
- Block one node p2p port and measure peer/duty recovery after unblock.

4. Rollback drill
- Intentionally deploy a bad value in stage, then revert through GitOps and measure time to steady state.
