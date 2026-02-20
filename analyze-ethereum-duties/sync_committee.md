# Sync committee

## Selection and rotation

Committee size
- 512 validators
- Fixed size

Rotation period
- 256 epochs ≈ 27 hours (256 × 6.4 minutes)

Selection is:
- Random
- Stake-weighted (via number of validators)
- Derived from RANDAO randomness
- Determined one period in advance

Process (conceptual):
1. Take active validator set
2. Shuffle using RANDAO
3. Pick first 512 → sync committee

Once selected:

You are on duty for every slot in the entire period.

## Duty

For each slot, a sync committee validator must sign the current beacon chain head root

Specifically:

- Sign a SyncCommitteeMessage

  Includes:
  - Slot
  - Beacon block root

No execution payload

No transactions

No fork choice

## Signature aggregation and verification
- Each sync committee member produces one BLS signature over (slot, block_root) using the sync committee domain.

- Designated aggregator is selected. This creates the SCC (SyncCommitteeContribution).
  - Collects signatures
  - Aggregates signature into one BLS signature
  - Builds bitmap of who signed
  - Publishes the object on a dedicated gossip topic

    `/eth2/beacon_sync_committee/{subnet_id}/ssz_snappy`

    - There are multiple sync committee subnets
    - Each validator is subscribed to exactly one subnet
    - Aggregators broadcast on the subnet they belong to

- Proposer's beacon node is subscribed to all the sync committee subnets and gets the aggregated contributions through gossip.
- Beacon node verifies and selects the best one (highest participation)
- Includes it in the block body


### What if no aggregator appears?

Outcomes:
  - Slot proceeds without sync aggregate
  - Light clients skip update for that slot
  - No slashing
  - Small reward loss
  - Redundancy makes this very unlikely. (16/512)


## SSV sync committee interaction

### Spec

```
Sync Committee Duty Full Cycle:
-> Wait to slot 1/3
   -> Received new beacon chain duty
      -> Check can start a new consensus instance
         -> Come to consensus on duty + sync message
            -> Broadcast and collect partial signature to reconstruct signature
               -> Reconstruct signature, broadcast to BN

Sync Committee Aggregator Duty Full Cycle:
-> Received new beacon chain duty
   -> Check can start a new consensus instance
      -> Locally get sync subcommittee indexes for slot
         -> Partial sign contribution proofs (for each subcommittee) and wait for other signatures
            -> wait to slot 2/3
               -> Come to consensus on duty + contribution (for each subcommittee)
                  -> Broadcast and collect partial signature to reconstruct signature
                     -> Reconstruct signature, broadcast to BN
```

### Implementation

#### Sync committee

1. Duty lifecycle driver
  operator/duties/sync_committee.go (SyncCommitteeHandler)

  - Runs every slot via the handler ticker.
  - Maintains two intent flags:
      - fetchCurrentPeriod
  - It fetches duties for the current period and prefetches the next period 1.5 epochs (48 slots) ahead of a period boundary.

  - HandleInitialDuties(...)
    Sets fetchCurrentPeriod=true and possibly fetchNextPeriod=true, then calls processFetching(...).
  - HandleDuties(...)
    Every slot:
      1. processExecution(...)
      2. processFetching(...)
      3. Toggles fetchNextPeriod when near period boundary.
      4. Resets old period duties at end of period.
  2. Fetch duties from Beacon node
  operator/duties/sync_committee.go → fetchAndProcessDuties(...)

  - Picks the correct epoch for the target period:
  - Builds the eligible validator index list:
  - Calls Beacon node:
  Beacon API implementation:

  - beacon/goclient/sync_committee.go
    SyncCommitteeDuties(...) → Beacon API call, returns []*SyncCommitteeDuty.


  - Builds a “self” index set:
  - Writes into duties.Set(period, storeDuties).
  - Calculates sync committee subnet subscriptions:
    calculateSubscriptions(endEpoch, duties)
  - Submits to Beacon node in a goroutine:
      - h.beaconNode.SubmitSyncCommitteeSubscriptions(...)

  This is required so the Beacon node will gossip sync committee messages on the correct subnets.

  5. Execution scheduling
  processExecution(...) in operator/duties/sync_committee.go

  - duties := h.duties.CommitteePeriodDuties(period)
  - Filters by slot alignment + validator participation (shouldExecute(...)).
      - Slot = current slot (so scheduler triggers it)
  - Calls:
    → this goes through the same scheduler/validator queue path as proposer duties.
  6. Duty execution and signing

      - operator/duties/scheduler.go
      - protocol/v2/ssv/validator/duty_executor.go
  - Actual sync committee signing and submission happens in:
      - protocol/v2/ssv/runner/committee.go


  - For sync committee duties, it builds altair.SyncCommitteeMessage objects.
      - r.beacon.SubmitSyncMessages(ctx, syncCommitteeMessages)

  Beacon client path:

    SubmitSyncMessages(...) → SubmitSyncCommitteeMessages API.


#### Sync committee contribution

1. Duty execution kicks off


  - For each subcommittee index, it builds altair.SyncAggregatorSelectionData{ Slot, SubcommitteeIndex } and signs it using threshold signing.
  - It broadcasts PartialSignatureMessages of type ContributionProofs.

  This is the pre‑consensus step.

  3. Pre‑consensus quorum processing

  - In ProcessPreConsensus(...), once 2f+1 partial signatures are collected:
      - It reconstructs the full selection proof signature for each root.
      - Only if aggregator = true, it proceeds to fetch contributions.

  4. Fetch sync committee contributions from Beacon node

  - For each selection proof that passes the aggregator check, it calls:
      - GetSyncCommitteeContribution(...) (Beacon client).
  - That call:
      - Waits 1/3 slot to get block root.
      - Waits 2/3 slot.
      - Fetches SyncCommitteeContribution per subnet.

  5. QBFT consensus on contribution data

  - The contributions are marshaled into ValidatorConsensusData.
  - The runner starts QBFT consensus using r.BaseRunner.decide(...).

  6. Post‑consensus: sign and submit ContributionAndProof

  - After consensus decides:
      - It signs ContributionAndProof using threshold signatures in ProcessConsensus(...).
      - Then in ProcessPostConsensus(...) it reconstructs the final signature and submits:
          - SubmitSignedContributionAndProof(...).


  ##### Why the Pre‑Consensus Step Is Required in SCC

  Because the selection proof needed to determine if the validator is an aggregator is a BLS signature under the validator’s key, and in SSV the validator key is split across operators. No single operator can produce a valid selection proof alone.

  Concrete reasons (from code behavior):

  1. Aggregator eligibility depends on the selection proof signature
      - IsSyncCommitteeAggregator(proof) (Beacon client) hashes the signature to decide aggregator status.
      - This signature must be the real threshold‑reconstructed signature, not just a partial.
  2. Only aggregators should fetch and submit contributions
      - ProcessPreConsensus(...) explicitly checks IsSyncCommitteeAggregator(...) and skips if false.
      - Without pre‑consensus, operators would either:
          - Fetch and submit when they shouldn’t, or
          - Fail to submit because they can’t prove aggregator status.
  3. Contribution data depends on the correct selection proof per subnet
      - The call GetSyncCommitteeContribution(...) takes selection proofs and subnet IDs.
      - Those proofs must match the final reconstructed selection proof signatures.

  So pre‑consensus is the step where the cluster reconstructs selection proofs, decides aggregator status, and collectively chooses the exact contribution data to run consensus on.

  ———

  ## Key Code References

  - Selection proofs and pre‑consensus:
      - protocol/v2/ssv/runner/sync_committee_contribution.go in executeDuty(...) and ProcessPreConsensus(...)
      - expectedPreConsensusRootsAndDomain(...) uses DomainSyncCommitteeSelectionProof
  - Aggregator check:
      - beacon/goclient/sync_committee_contribution.go → IsSyncCommitteeAggregator(...)
  - Fetch contributions:
      - beacon/goclient/sync_committee_contribution.go → GetSyncCommitteeContribution(...)
  - Submit signed contributions:
      - beacon/goclient/sync_committee_contribution.go → SubmitSignedContributionAndProof(...)
