# Aggregation duties

## Aggregator selection per slot

- Committee-based

  Every validator that is assigned to attest in a slot is also (potentially) eligible to be an aggregator for that same committee/slot. The spec describes that some validators are selected to locally aggregate attestations with similar attestation_data for the assigned slot.

- Selection proofs + modulo check

  Each attester produces a selection proof (a BLS signature over slot/selection-domain) and runs a deterministic check to decide “am I an aggregator?”. This is designed so:

  - It’s locally computable (no leader election)
  - A small subset becomes aggregators per committee (for redundancy)
  - It’s unpredictable until the selection proof exists

### Math

1) How many validators attest in a slot?

   Let N = number of active validators.
   Each validator attests once per epoch (32 slots), so per slot you get:
   attesters per slot ≈ 𝑁/32
​
   As per today's figures: Attesters per slot ≈ 969,100 / 32 ≈ 30,285


2) How many committees per slot?

   committees_per_slot=max(1,min(64,⌊32⋅128N​⌋))
   Constants:
   - MAX_COMMITTEES_PER_SLOT = 64
   - TARGET_COMMITTEE_SIZE = 128
   - SLOTS_PER_EPOCH = 32
  
   Capped to 64. According to today's figures, committees_per_slot ≈ 236, so it will be 64 most of the time.

3) Average committee size (how many attestations per committee)

   Total attesters per slot divided by committees per slot:
   committee size ≈ (𝑁/32)/committees_per_slot
	​
   If committees_per_slot = 64:

   committee size ≈ 𝑁/2048
   With N ≈ 969,100:
   committee size ≈ 969,100 / 2048 ≈ 473 validators per committee

4) How many aggregators per slot?

   Per committee, the protocol targets:
   `TARGET_AGGREGATORS_PER_COMMITTEE = 16`

   So expected aggregators per slot:
   aggregators per slot ≈ 16 × committees_per_slot

   If committees_per_slot = 64:
   Expected aggregators per slot ≈ 16 × 64 = 1,024 aggregators

5) How many messages does the proposer “have to look for”?

   There are two relevant gossip streams:
   A) Individual attestations (subnet gossip)

      Count ≈ N/32 per slot (e.g., ~30k)
      But proposers do not need to individually “hunt” these; aggregators do that work.

   B) Aggregated attestations (global gossip)

      Aggregators publish AggregateAndProof messages (global topic), and proposers include aggregates in blocks.

      - Expected aggregate producers per slot ≈ 16 × committees_per_slot (e.g., ~1,024)
      - But the proposer only needs ≈ 1 aggregate per committee to get full coverage: ~64 aggregates included (when 64 committees/slot)


      Messages proposer may receive: up to ~1,000 aggregate proofs for that slot (often fewer in practice).
      Messages proposer needs to select from: “best per committee” → ~64 winners.

## Collection of individual validator attestations within a committee

- Where individual attestations go

  Each attester gossips its individual attestation on an attestation subnet corresponding to its committee. This is why we have subnets: to avoid flooding the whole network with every single attestation.

  Aggregators for that committee subscribe to the same subnet and collect individual attestations they hear with the same attestation_data as their own.

- Practical rule: “same data or ignore”

  An aggregator only aggregates attestations that match:
  - same slot
  - same beacon_block_root (vote for head)
  - same source/target checkpoints (FFG)
  - same committee/index structure

  If attestation_data differs, it must not be merged (you’d create an invalid aggregate).


## Construction of aggregated attestations

An aggregate attestation is basically:

  - the common attestation_data
  - aggregation_bits: a bitlist saying which committee positions are included
  - one aggregated BLS signature

Aggregation bits correctness
For each attestation received:
  - determine the validator’s position in the committee
  - set that bit in aggregation_bits

Rules you must maintain:
  - No duplicate signers (don’t set the same bit twice)
  - Bits correspond to the correct committee ordering
  - Bits only set for signers whose signature you actually include

This is a huge part of “slashing safety” at the network level: the bits bind the aggregate to a specific signer set, and verifiers use it to reconstruct which public keys should validate the aggregate signature.

## BLS signature aggregation and validation
Aggregator’s job
  - verify each incoming attestation’s signature (or batch verify)
  - aggregate signatures (BLS aggregate)
  - output a single aggregate signature

Verifier’s job (what the network checks)
  Nodes verify:
  - attestation_data is valid for that slot/committee
  - aggregation_bits length matches committee size
  - aggregated signature is valid for the set of pubkeys indicated by the bits, under the correct domain (attestation domain)

This ensures:
  - fork-choice correctness (votes are real, not invented)
  - the aggregate cannot be “massaged” without breaking signature verification


## Timely broadcast within the same slot

Aggregators don’t just gossip an “Attestation”; they gossip an AggregateAndProof (aggregate + proof that the aggregator was eligible).

This is sent on the global gossip topic commonly referred to as beacon_aggregate_and_proof, while individual attestations are on per-subnet topics like beacon_attestation_{subnet_id}. The consensus networking spec explicitly calls out these topics (and notes upgrades like Electra modify them, but the concept remains).

### Why “same slot” matters

Attesters are assigned to slot S, but the aggregate is most valuable if it reaches block proposers fast enough to be included in blocks built shortly after. Practically:
  - late aggregates get fewer inclusions → lower rewards
  - they also reduce the “fresh vote weight” seen by peers, which can affect how quickly the network converges on the head in turbulent conditions


## How this reduces load + preserves safety

### Load reduction
Instead of (say) 128 attestations being gossiped globally, you get:
  - 128 individual attestations gossiped only inside a subnet
  - a handful of aggregates gossiped globally

### Fork-choice correctness

Fork-choice (LMD-GHOST) cares about latest votes. Aggregation preserves each validator’s vote, just compressed. As long as:
  - attestation_data matches
  - bits are correct
  - signatures validate

### Slashing safety

Aggregation itself isn’t a slashable action, but invalid aggregation can:
  - get rejected (wasted effort)
  - harm your rewards
  - in badly designed DVT flows, increase risk of accidentally producing inconsistent messages across operators (this is why DVT stacks pay special attention to aggregation duties)

### “Where does the proposer get it?” (operationally)

The proposer’s beacon node simply receives beacon_aggregate_and_proof gossip like everyone else and includes the best aggregates it has when assembling the next block.

If you’re implementing this via the Beacon API, you’ll commonly see flows involving:
  - retrieving an aggregate to publish
  - publishing aggregate_and_proofs via validator APIs (common across clients)


## SSV aggregator interaction

### Spec

```
Attestation Aggregator Duty Full Cycle:
-> Received new beacon chain duty
   -> Check can start a new consensus instance
      -> Sign partial selection proof and wait for other signatures
         -> Wait to slot 2/3
            -> Come to consensus on duty + aggregated selection proof
               -> Broadcast and collect partial signature to reconstruct signature
                  -> Reconstruct signature, broadcast to BN
```

### Implementation

 1. Scheduling: aggregator duties are driven by the attester handler
  File: operator/duties/attester.go

  - Attester duties are fetched each epoch.
  - On each slot, the handler executes aggregator duties for all attester duties in that slot:
      - executeAggregatorDuties(...)
  - It always schedules the aggregator role for a duty because it can’t know if it is the aggregator until the selection proof is reconstructed.

  Key comment in code:

  > “we won't know if it's needed or not until we rebuild validator signature (done during pre‑consensus step)”

  ———

  2. Pre‑consensus: selection proof signing + aggregator check
  File: protocol/v2/ssv/runner/aggregator.go

  Step A: executeDuty()

  - Signs selection proof: DomainSelectionProof
  - Broadcasts partial sigs (SelectionProofPartialSig)

  Step B: ProcessPreConsensus(...)

  - Reconstructs the full selection proof signature from 2f+1 partials.
  - Uses IsAggregator(...) to determine if this validator is an aggregator for this committee/slot.
      - This is the spec’s hash‑based selection logic.

  If not an aggregator, duty ends immediately.
  
  3. If aggregator: get aggregate attestation from BN
  File: protocol/v2/ssv/runner/aggregator.go, Beacon client in beacon/goclient/aggregator.go

  - In ProcessPreConsensus(...), after passing IsAggregator(...), it calls:
    SubmitAggregateSelectionProof(...) on the Beacon node client.
  - Beacon client waits until 2/3 of the slot (per spec), fetches attestation data, then uses AggregateAttestation to build an AggregateAndProof object (versioned).

  ———

  4. QBFT consensus on AggregateAndProof
  File: protocol/v2/ssv/runner/aggregator.go

  - The AggregateAndProof is marshaled into ValidatorConsensusData.
  - The runner runs QBFT: r.BaseRunner.decide(...)
  - This makes all operators agree on the exact aggregate object before signing.

  ———

  5. Post‑consensus: sign & submit AggregateAndProof
  Files: protocol/v2/ssv/runner/aggregator.go, beacon/goclient/aggregator.go

  - In ProcessConsensus(...): each operator signs the aggregate root (partial sig).
  - In ProcessPostConsensus(...): reconstruct final signature.
  - Submit to BN via:
    SubmitSignedAggregateSelectionProof(...)
    which calls SubmitAggregateAttestations on the Beacon node API.
