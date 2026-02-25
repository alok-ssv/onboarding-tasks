# SSV DVT - Validator key splitting and operator roles

## Key shares, quorum, and what is fixed in SSV

SSV is not a free-choice `(n, t)` system in production. It follows the IBFT/QBFT fault model:

- Committee size: `n = 3f + 1`
- Fault tolerance parameter: `f = (n - 1) / 3`
- Signing quorum (threshold): `t = 2f + 1 = n - f`
- Partial quorum (used in round-change/liveness logic): `f + 1`

In current SSV implementation, valid committee sizes are constrained to:

- `n ∈ {4, 7, 10, 13}`

Key handling model:

- The validator private key is split into operator shares using Shamir Secret Sharing (SSS) / threshold BLS mechanisms.
- SSV also supports DKG-based flows (no trusted dealer): no single operator learns the full secret key.
- Operators hold shares; the full validator secret key is not required online on operator nodes.
- Operationally, if a full key exists before splitting, it can be stored offline (cold) or deleted per policy after split.

Signing model:

- Operators do not independently sign arbitrary payloads on demand.
- They first run IBFT/QBFT consensus on what must be signed.
- After consensus, operators produce partial BLS signatures, and a collector/aggregator reconstructs a final BLS signature.
- The aggregator combines partial signatures (not key shares), so this process reconstructs a signature, not the validator private key.

## Operator responsibilities

- Run the operator node software and maintain healthy p2p connectivity.
- Participate in setup/resharing flows (including DKG-based flows where applicable).
- Participate in consensus rounds (proposal/prepare/commit/round-change behavior).
- Produce correct partial signatures from local shares only after the consensus decision for the duty.
- Broadcast partial signatures and/or collect them for signature reconstruction.
- Protect share material (HSM or hardened key store preferred), rotate credentials, patch quickly.
- Provide monitoring, telemetry, and alerting for uptime, latency, consensus progress, and signature flow health.
- Participate in operator set changes and resharing when cluster membership changes.

## Fault tolerance guarantees (separate the threat models)

Think in distinct dimensions:

1. Protocol safety (consensus safety)
- Under the IBFT/QBFT assumptions (up to `f` Byzantine), at most one value can be decided for an instance.
- This is the primary protection against equivocation at the protocol layer.

2. Key compromise / unauthorized threshold signing
- Requires `t = 2f + 1` shares to be exfiltrated/colluding.
- This is the key-compromise threshold, not the liveness-stall threshold.

3. Liveness (ability to keep producing signatures)
- Progress requires quorum `2f + 1`.
- Up to `f` unavailable/malicious operators can be tolerated for progress.
- `f + 1` Byzantine/offline can stall progress even though they still cannot reconstruct the key.

Concrete example (`n=4`):

- `f=1`, `t=3`
- Key compromise threshold: `3` shares/operators
- Liveness stall threshold: `2` Byzantine/offline operators

## Trust boundaries — who trusts whom

- Staker/deposit owner -> cluster:
  assumes fewer than `t` share compromises for key security and no more than `f` Byzantine for expected liveness/safety guarantees.
- Operators <-> operators:
  do not need mutual trust for key custody; trust is placed in protocol correctness, signatures, and verifiable outcomes.
- Aggregator/collector:
  untrusted for key material; can hurt liveness by withholding partials, but cannot derive validator private key from partial signatures alone.
- On-chain contract and Ethereum slashing:
  contract enforces registered membership/public key state; slashing is an economic/on-chain backstop, not the primary equivocation prevention mechanism.

## Common failure scenarios and mitigations

- More than `f` operators unavailable -> quorum (`2f+1`) not reached -> liveness loss.
  Mitigation: independent operators, geo/provider diversity, redundancy, strong on-call.

- `f + 1` Byzantine operators withhold/with invalid behavior -> consensus/signing stalls.
  Mitigation: fast detection, performance SLAs, operator replacement/reshare playbooks.

- `t` share compromises (`2f+1`) -> key compromise / unauthorized threshold signing.
  Mitigation: strict key custody, HSM-backed storage, least privilege, regular audits.

- Collector/aggregator bottleneck -> delayed or failed reconstruction/submission.
  Mitigation: redundant collection paths, robust gossip, fallback collectors.

- Network partition -> only one partition can reach quorum under protocol assumptions; other partitions stall.
  Mitigation: resilient networking, partition detection, conservative failover policies.

- Operator set change without proper resharing -> stale trust assumptions.
  Mitigation: explicit resharing/DKG procedure for every membership change.

## Practical reference table (supported committee sizes)

| Committee size `n` | Faults tolerated `f` | Quorum / threshold `t = 2f+1` | Max unavailable for progress (`f`) | Byzantine needed to stall (`f+1`) |
| --- | --- | --- | --- | --- |
| 4 | 1 | 3 | 1 | 2 |
| 7 | 2 | 5 | 2 | 3 |
| 10 | 3 | 7 | 3 | 4 |
| 13 | 4 | 9 | 4 | 5 |

Recommended operator selection remains the same:

- prioritize operational independence (infra, cloud, legal domain),
- prefer hardened key custody (HSM where feasible),
- and enforce clear replacement/incident response procedures.
