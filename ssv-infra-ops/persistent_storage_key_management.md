# SSV Infra/Ops - Persistent Storage and Key Management

This document covers where SSV node state and key material live in Kubernetes, and the durability/security trade-offs.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Operator key flow

Main path in chart:

1. `ExternalSecret` is created (`charts/ssv-node/templates/external-secrets.yaml`).
2. It reads from Vault path `.Values.vaultSecretPath` and property `{{ fullname }}`.
3. It writes into K8s Secret named `{{ fullname }}`.
4. Pod consumes key via env var:
   - `OPERATOR_KEY` in main container when signer disabled,
   - `PRIVATE_KEY` in `ssv-signer` sidecar when signer enabled.

Security implications:

- Key is not stored in chart repo or ConfigMap.
- Key does exist as a Kubernetes Secret object in cluster memory/etcd path (depending on cluster configuration).
- Any principal with Secret read access in namespace can exfiltrate key material.

## Environment key-path patterns in GitOps

Observed in `gitops-stage` / `gitops-production`:

- Stage nodes: `vaultSecretPath: stage/ssv-nodes-operators-private-keys`
- Production OVH mainnet nodes/exporters: `vaultSecretPath: production/mainnet/ssv-nodes`
- Production OVH hoodi nodes/exporters: `vaultSecretPath: production/hoodi/ssv-nodes`
- Production AWS sepolia exporter: `vaultSecretPath: production/sepolia/ssv-nodes`

Observed cluster secret store names:

- `stage-vault-kv-secret`
- `production-ovh-vault-kv-secret`
- `production-vault-kv-secret`

## Persistent volume layout

Primary data mount:

- Volume name `storage` mounted at `/data`
- Defined in `charts/ssv-node/templates/statefulset.yml`

Persistence modes:

1. `persistence.enabled=false`
- Uses `emptyDir` (ephemeral; erased on pod recreation)

2. `persistence.existingClaim` set
- Reuses provided PVC

3. Default managed PVC
- Uses StatefulSet `volumeClaimTemplates` with size/class from values

Config file mount:

- `share.yaml` comes from ConfigMap and is mounted as `/data/share.yaml` subPath.

## Durability model

What survives pod restart:

- PVC-backed `/data` content (DB/state files)
- ExternalSecret-managed Secret (if Secret/ESO remains healthy)

What does not survive pod recreation with `emptyDir`:

- Node DB and local state in `/data`

What requires restart to apply:

- Env-var key rotations (Kubernetes Secret updates do not hot-reload env vars in running containers).

## Backup and recovery risks

1. Key-path coupling to release fullname
- ExternalSecret `property` key equals release fullname.
- Renaming release can break key lookup if Vault structure is not updated.

2. Ephemeral storage misconfiguration
- If persistence disabled accidentally, node state is wiped on recreation.

3. Secret availability as startup dependency
- If ExternalSecret or secret store is unavailable, pod startup can fail due missing key env refs.

4. Partial backup strategy
- PVC backups alone are not enough if operator key source (Vault/Secret) is not recoverable.

5. Sensitive non-operator keys committed in Git values
- In `gitops-stage/environments/stage/ssv-node-exporter/nodes/*.yaml`, some nodes set `NETWORK_PRIVATE_KEY` directly in `envs`.
- This key is distinct from `OPERATOR_KEY`, but still sensitive (P2P identity/security impact).
- Treat these files as secrets exposure risk and migrate to secret-based delivery.

## Recommended backup model

1. Treat key source of truth as Vault (not Kubernetes Secret).
2. Back up:
- Vault secret path for each node fullname key,
- PVC snapshots for `/data`,
- deployed Helm values overrides (network, DB path, ports, signer toggles).
 - cluster-secret-store configuration references used by those apps.
3. Test restore with a fresh namespace/release name mapping.

## Recovery playbooks

### Pod/pod-node restart with healthy PVC

- Expected: DB and local state recover from `/data`; node resumes without full bootstrap.

### Lost pod + lost PVC

- Expected: full state rebuild/re-sync required.
- Key must still be recoverable from Vault/ExternalSecret chain.

### Secret/Vault outage

- Existing running pod may continue (env already injected), but restarted pod can fail until Secret chain recovers.
