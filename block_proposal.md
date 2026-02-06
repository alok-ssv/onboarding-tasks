# Block proposal

## Selection
Only one proposer is choosen per slot to propose a block. Proposer assignments for all 32 slots of epoch E are known at the start of epoch E.

For epoch E:
- Randomness used comes from epoch E-1. Specifically; the RANDAO mix at the end of E-1
- Proposer assignments for all 32 slots of epoch E are known at the start of epoch E

1) Protocol-level

   At any slot S, the canonical proposer is computed from the beacon state deterministically via the spec function:
   `get_beacon_proposer_index(state)`
   Under the hood, it uses:
   - a seed derived from RANDAO mixes and the epoch (via get_seed(...))
   - a validator shuffling and a weighted filter (compute_proposer_index(...)) that uses effective balances as the acceptance criterion

3) The usual flow (most clients)

   A. Once per epoch (or on reorg), validator client asks the beacon node for proposer duties

      - It calls the standard Beacon API endpoint: `GET /eth/v1/validator/duties/proposer/{epoch}`

        This returns the mapping: slot → validator_index for that epoch.
        Important nuance: duties are “typically stable within an epoch” but may change after a reorg, so safe clients either:
      - re-fetch on head changes, or
      - validate the “duty dependent root” concept when applicable (implementation-specific), but the key point is: reorgs can change duties.

   B. Validator client runs a slot ticker

      Every slot boundary it increments “current slot”. Prysm’s documentation describes this model: a ticker runs each slot, and if the current slot matches an assigned role, it proposes/attests.

   C. When current slot == one of “my proposer slots”, it starts proposing

      At that moment the validator client knows: “I’m selected for this slot.”
      Not because it got a special message from the network—because it precomputed / fetched duties and the ticker hit that slot.

## Slot deadlines

| Time in slot | What should happen |
| -------------|--------------------|
| 0–2s	| Receive head of chain |
| 2–4s | Build block |
| ≤4s |	Broadcast block (ideal) |
| >4–6s	| Attestations degrade |
| ~12s	| Slot ends → block useless |

Late blocks = fewer attestations = lower rewards

Very late = effectively missed slot

## Block contents
A proposer creates two tightly coupled things:

  A) Execution Layer (EL) block
     
     - Transactions (ordered)
     - Fee recipient (your reward address)
     - Gas usage
     - Execution receipts
     - Updated execution state root

This is built by your execution client (e.g. Geth, Nethermind).

  B) Consensus Layer (CL) block (Beacon Block)
     
     Wraps the EL block and adds:
     - Slot number
     - Proposer index
     - Parent root
     - Randao reveal
     - Attestations
     - Slashings (if any)
     - Sync committee data (if applicable)


### State transition responsibility

As proposer, you are responsible for ensuring:
- All transactions execute correctly
- State roots match
- No invalid gas usage
- No malformed data
- If any of this is wrong → invalid block → rejected by the network.

## Signing requirements

- Validator signing key
- Same key used for attestations (different domain)
- You sign:
  - The Beacon Block header
  - Including slot, parent root, state root, body root


## Common failure cases

- ❌ Missed slot
  - Node offline
  - Execution client not synced
  - Builder latency
  - Network issues

  Impact
  - Lost proposal reward
  - No slashing
  - Reputation hit (low effectiveness)

- ❌ Late block
  - Slow block building
  - Slow relay (MEV-Boost)
  - Overloaded hardware

  Impact
  - Fewer attestations
  - Lower rewards
  - Still valid, just inefficient

- ❌ Invalid block (danger zone)
  - EL/CL desync
  - Buggy client
  - Corrupt state
  - Bad MEV payload

  Impact
  - Block rejected
  - Lost rewards
  - Possible slashing if double-signed variants exist

- ❌ Double proposal (slashing)
  - Running same validator on two machines
  - Restore from old backup without protection
  - Misconfigured failover

  Impact
  - Slashing
  - Forced exit
  - Loss of stake (can be severe)


## SSV Block proposal interaction

### Spec

```
Block Proposal Duty Full Cycle:
-> Received new beacon chain duty
   -> Check can start a new consensus instance
      -> Sign partial RANDAO and wait for other signatures
         -> Come to consensus on duty + beacon block
            -> Broadcast and collect partial signature to reconstruct signature
               -> Reconstruct signature, broadcast to BN
```

### Implementation

1. Duty fetch from Beacon node
      - operator/duties/proposer.go fetches proposer duties each epoch via BeaconNode.ProposerDuties(...), stores them, and schedules execution on the duty’s slot.
      - It builds the eligible validator index list from local shares and filters to “self” indices for committee membership, then stores duties in a duty store.
      - operator/duties/beacon_adapter.go wraps the beacon client to cache/batch proposer duty requests (batches of 2048) and serves handler calls from cache.
  2. Scheduling and dispatch to validator
      - operator/duties/scheduler.go executes duties by launching ExecuteDuty(...) in the background for each duty, with slot timing checks.
      - operator/validator/controller.go locates the validator by pubkey and calls v.ExecuteDuty(...).
  3. Validator queues and start duty
      - protocol/v2/ssv/validator/duty_executor.go wraps the duty in an ExecuteDuty event and pushes it to the validator’s queue.
      - protocol/v2/ssv/validator/validator.go handles the event, starts the validator if needed, and starts the duty runner for the role (proposer runner).
  4. Proposer runner: pre‑consensus and block fetch
      - protocol/v2/ssv/runner/proposer.go starts the duty by signing and broadcasting a partial RANDAO signature.
      - Once 2f+1 partial RANDAO sigs are collected, it waits out ProposerDelay, then requests a block from the Beacon node using GetBeaconBlock(...).
      - If the returned block is full, it is converted to a blinded block for QBFT consensus, while caching the full block locally for later submission.
  5. QBFT consensus and post‑consensus signature
      - After QBFT decides on the block data (blinded SSZ), the runner signs the block root and broadcasts partial signatures.
      - When 2f+1 post‑consensus sigs are collected, it reconstructs the final block signature.
  6. Submit block to Beacon node
      - protocol/v2/ssv/runner/proposer.go submits the decided block to the Beacon node via SubmitBeaconBlock(...).
      - If this operator was the decided round leader and cached a full block matching the decided blinded root, it submits the full block (with blobs) instead of the blinded one.

  Beacon Node Interaction Details

  - beacon/goclient/proposer.go implements:
      - ProposerDuties(...) → /eth/v1/validator/duties/proposer/{epoch}
      - GetBeaconBlock(...) → uses Proposal API; if multiple clients are configured, it races them and chooses the best proposal.
      - SubmitBeaconBlock(...) → submits blinded or full blocks depending on block.Blinded.
  - If external builder/MEV is enabled, the beacon node returns blinded blocks and SSV submits blinded proposals (/eth/v1/beacon/blinded_blocks). See docs/EXTERNAL_BUILDERS.md.
  - ProposerDelay is an operator‑configurable wait to improve MEV outcomes, with safety limits. See docs/MEV_CONSIDERATIONS.md.

  Key Files

  - Duty fetch and execution: operator/duties/proposer.go
  - Beacon duty caching: operator/duties/beacon_adapter.go
  - Scheduling: operator/duties/scheduler.go
  - Validator dispatch: operator/validator/controller.go
  - Validator duty handling: protocol/v2/ssv/validator/duty_executor.go, protocol/v2/ssv/validator/validator.go
  - Proposer duty logic: protocol/v2/ssv/runner/proposer.go
  - Beacon node API calls: beacon/goclient/proposer.go
  - MEV/blinded behavior: docs/EXTERNAL_BUILDERS.md, docs/MEV_CONSIDERATIONS.md
