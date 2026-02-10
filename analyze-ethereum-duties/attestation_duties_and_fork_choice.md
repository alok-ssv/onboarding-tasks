# Attestation duties and Fork Choice interaction

## Attestation structure
The attestation contains the following components:

- aggregation_bits: a bitlist of validators where the position maps to the validator index in their committee; the value (0/1) indicates whether the validator signed the data (i.e., whether they are active and agree with the block proposer)
- data: details relating to the attestation, as defined below
- signature: a BLS signature that aggregates the signatures of individual validators

The data contains the following information:

- slot: The slot number that the attestation refers to
- index: A number that identifies which committee the validator belongs to in a given slot
- beacon_block_root: Root hash of the block the validator sees at the head of the chain (the result of applying the fork-choice algorithm)
- source: Part of the finality vote indicating what the validators see as the most recent justified block
- target: Part of the finality vote indicating what the validators see as the first block in the current epoch

Once the data is built, the validator can flip the bit in aggregation_bits corresponding to their own validator index from 0 to 1 to show that they participated.

Finally, the validator signs the attestation and broadcasts it to the network.

## Attestation duty
Each active validator is assigned one attestation duty per epoch: a specific slot within the epoch and a committee index for that slot.

The purpose of the attestation is to vote in favor of the validator's view of the chain, in particular the most recent justified block and the first block in the current epoch (known as source and target checkpoints).

Conceptually it can be viewed as carrying 3 distinct signals:

- Head vote (LMD-GHOST vote)
  - Field: beacon_block_root
  - Meaning: “This is the block I believe should be the chain head right now.”

- Target vote (FFG vote, epoch boundary)
  - Field: target (checkpoint in the current epoch)
  - Meaning: “For epoch E, I vote that this checkpoint is the correct checkpoint.”

- Source vote (FFG vote, justified checkpoint)
  - Field: source (checkpoint I think is currently justified)
  - Meaning: “I’m extending finality from this last-justified checkpoint.”

This is why people say attestations drive both fork-choice (head) and finality (source/target).

## Aggregation
There is a substantial overhead associated with passing this data around the network for every validator. Therefore, the attestations from individual validators are aggregated within subnets before being broadcast more widely. This includes aggregating signatures together so that an attestation that gets broadcast includes the consensus data and a single signature formed by combining the signatures of all the validators that agree with that data. This can be checked using aggregation_bits because this provides the index of each validator in their committee (whose ID is provided in data) which can be used to query individual signatures.

In each epoch 16 validators in each subnet are selected to be the aggregators. The aggregators collect all the attestations they hear about over the gossip network that have equivalent data to their own. The sender of each matching attestation is recorded in the aggregation_bits. The aggregators then broadcast the attestation aggregate to the wider network.

When a validator is selected to be a block proposer they package aggregate attestations from the subnets up to the latest slot in the new block.

## Validator assignments

A validator can get committee assignments for a given epoch using the following helper via `get_committee_assignment(state, epoch, validator_index)` where `epoch <= next_epoch`.

```
def get_committee_assignment(
    state: BeaconState, epoch: Epoch, validator_index: ValidatorIndex
) -> Optional[Tuple[Sequence[ValidatorIndex], CommitteeIndex, Slot]]:
    """
    Return the committee assignment in the ``epoch`` for ``validator_index``.
    ``assignment`` returned is a tuple of the following form:
        * ``assignment[0]`` is the list of validators in the committee
        * ``assignment[1]`` is the index to which the committee is assigned
        * ``assignment[2]`` is the slot at which the committee is assigned
    Return None if no assignment.
    """
    next_epoch = Epoch(get_current_epoch(state) + 1)
    assert epoch <= next_epoch

    start_slot = compute_start_slot_at_epoch(epoch)
    committee_count_per_slot = get_committee_count_per_slot(state, epoch)
    for slot in range(start_slot, start_slot + SLOTS_PER_EPOCH):
        for index in range(committee_count_per_slot):
            committee = get_beacon_committee(state, Slot(slot), CommitteeIndex(index))
            if validator_index in committee:
                return committee, CommitteeIndex(index), Slot(slot)
    return None
```
get_committee_assignment should be called at the start of each epoch to get the assignment for the next epoch (current_epoch + 1). A validator should plan for future assignments by noting their assigned attestation slot and joining the committee index attestation subnet related to their committee assignment.

Specifically a validator should:

- Call `_, committee_index, _ = get_committee_assignment(state, next_epoch, validator_index)` when checking for next epoch assignments.
- Calculate the committees per slot for the next epoch: `committees_per_slot = get_committee_count_per_slot(state, next_epoch)`
- Calculate the subnet index: `subnet_id = compute_subnet_for_attestation(committees_per_slot, slot, committee_index)`
- Find peers of the pubsub topic `beacon_attestation_{subnet_id}`.
- If an insufficient number of current peers are subscribed to the topic, the validator must discover new peers on this topic. Via the discovery protocol, find peers with an ENR containing the attnets entry such that ENR["attnets"][subnet_id] == True. Then validate that the peers are still persisted on the desired topic by requesting GetMetaData and checking the resulting attnets field.
- If the validator is assigned to be an aggregator for the slot (see is_aggregator()), then subscribe to the topic.

Note: If the validator is not assigned to be an aggregator, the validator only needs sufficient number of peers on the topic to be able to publish messages. The validator does not need to subscribe and listen to all messages on the topic.

## Rewards
Validators are rewarded for submitting attestations. The attestation reward depends on the participation flags (source, target and head), the base reward and the participation rate.

Each of the participation flags can be either true or false, depending on the submitted attestation and its inclusion delay.

The best scenario occurs when all three flags are true, in which case a validator would earn (per correct flag):

reward += base reward * flag weight * flag attesting rate / 64

The flag attesting rate is measured using the sum of effective balances of all attesting validators for the given flag compared the total active effective balance.

- Base reward
  The base reward is calculated according to the number of attesting validators and their effective staked ether balances:

  `base reward = validator effective balance x 2^6 / SQRT(Effective balance of all active validators)`

- Inclusion delay
  At the time when the validators voted on the head of the chain (block n), block n+1 was not proposed yet. Therefore attestations naturally get included one block later so all attestations who voted on block n being the chain head got included in block n+1 and, the inclusion delay is 1. If the inclusion delay doubles to two slots, the attestation reward halves, because to calculate the attestation reward the base reward is multiplied by the reciprocal of the inclusion delay.


## Ethereum’s consensus (“Gasper” - LMD-GHOST + Casper-FFG).

- How LMD-GHOST uses attestations

  Fork choice keeps each validator’s latest message (their most recent attestation) and weights branches accordingly (“Latest Message Driven”).
  Practically: if your attestation arrives late, your weight may apply to a head that the rest of the network has already moved past → you’ll often be counted as a non-head vote.

- How FFG uses source/target

  FFG looks at votes linking source → target checkpoints. If enough stake votes, the target becomes justified, and with consecutive justification it becomes finalized (in normal flow).


### Key operational implication

A common failure mode is late block observation/validation → your node computes head too early (or with incomplete info), so beacon_block_root differs from the eventual canonical head. This doesn’t usually invalidate the attestation, but it makes it “incorrect” in the head sense.

## Slashing conditions related to attestations

Casper-FFG attester slashing boils down to two forbidden patterns between two signed AttestationData from the same validator:

- Double vote: two different attestations for the same target epoch
- Surround vote: one vote’s (source epoch, target epoch) “surrounds” another

The Phase0 spec expresses this directly in is_slashable_attestation_data.
```
def is_slashable_attestation_data(data_1: AttestationData, data_2: AttestationData) -> bool:
    """
    Check if ``data_1`` and ``data_2`` are slashable according to Casper FFG rules.
    """
    return (
        # Double vote
        (data_1 != data_2 and data_1.target.epoch == data_2.target.epoch)
        or
        # Surround vote
        (data_1.source.epoch < data_2.source.epoch and data_2.target.epoch < data_1.target.epoch)
    )
```

Two very practical corollaries:

- Re-broadcasting the exact same attestation is fine (slashing requires data_1 != data_2 for double-vote).
- Most real-world attester slashings come from running the same validator key in two places, or restoring from backup without consistent slashing protection DB.

### Scenarios

- “Missed” attestation
  Meaning: your duty did not make it on-chain at all (or too late to matter for rewards).
  Typical root causes:
  - validator never produced it (VC issue, duty scheduling, clock skew)
  - produced but didn’t reach peers (networking)
  - reached peers but wasn’t aggregated/included

- “Late” attestation
  Meaning: it got included, but with high inclusion delay, reducing the value of the vote(s).
  A useful decomposition (from Lighthouse’s analysis framing):
  - observed_delay: how late you observed the block you’re meant to attest to
  - attestable_delay: time left before the attestation deadline; if it’s too high, you miss the window

- “Incorrect” attestation (usually means “non-head”, sometimes “bad source/target”)
  There are three ways to be “incorrect,” matching the three votes:
  - Non-head vote: beacon_block_root not equal to eventual canonical head for that slot. Most often caused by late/slow block propagation or validation, so your local fork choice head lags.
  - Bad target vote: voting for the wrong epoch checkpoint. This is rarer in normal operation, but can happen under reorgs or if your node is seriously out of sync.
  - Bad source vote: source not matching the correct justified checkpoint. Typically indicates you’re on the wrong justification view (often due to being partitioned / very lagged). Clients have historically differed in where they reject these (fork-choice vs block inclusion), but modern behavior aims to prevent bad votes from being included.

#### Reward-timing intuition (why “late” hurts differently per vote)

Some references break down that each of the three votes effectively has its own “timeliness” window (head is most sensitive, target least). For example, Besu’s public docs summarize reward eligibility conditions across source/target/head windows.
(Exact constants vary by fork and reward rules, but the key takeaway stands: head correctness is the first thing you lose when you’re late.)


## References
- https://ethereum.github.io/consensus-specs/specs/phase0/beacon-chain/
- https://ethereum.org/developers/docs/consensus-mechanisms/pos/attestations
- https://ethereum.github.io/consensus-specs/specs/phase0/validator/

## SSV Attestation interaction

### Spec

```
Attestation Duty Full Cycle:
-> Wait to slot 1/3
   -> Received new beacon chain duty
      -> Check can start a new consensus instance
         -> Come to consensus on duty + attestation data
            -> Broadcast and collect partial signature to reconstruct signature
               -> Reconstruct signature, broadcast to BN

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
  1. Duty scheduling + grouping by committee

  - The duty scheduler ticks each slot, gathers attester duties from the duty stores, and groups them by committee into a spectypes.CommitteeDuty.
      - operator/duties/committee.go builds committee duties in processExecution and buildCommitteeDuties, mapping attester duties → committee duty.
      - operator/duties/scheduler.go calls ExecuteCommitteeDuties, waits ~1/3 into the slot, then triggers execution.

  2. Controller starts the committee duty

  - The controller finds the committee instance and starts the duty, wiring the runner + queue consumer.
      - operator/validator/controller.go in ExecuteCommitteeDuty calls cm.StartDuty and cm.ConsumeQueue.

  3. Committee prepares runner + duty

  - The committee filters out validator duties it can’t run (no share), collects attesting validators, creates the runner, and starts it.
      - protocol/v2/ssv/validator/committee.go in prepareDuty, prepareDutyAndRunner, StartDuty.
      - The CommitteeDutyGuard is injected for exclusivity and validity checks.

  4. CommitteeRunner executes duty: fetch attestation data + run QBFT

  - executeDuty fetches attestation data from the beacon node (slot→attestation data), builds a BeaconVote, then runs QBFT consensus (BaseRunner.decide).
      - protocol/v2/ssv/runner/committee.go in executeDuty.
      - Beacon calls: protocol/v2/blockchain/beacon/client.go and beacon/goclient/attest.go.

  5. Post‑consensus: sign attester duties + broadcast partial sigs

  - Each validator duty is processed (worker pool). Attester duties call signAttesterDuty, which:
      - Checks doppelganger gating (attester only)
      - Builds AttestationData
      - Signs via signBeaconObject
  - A post‑consensus partial signature message is broadcast to the network.
      - protocol/v2/ssv/runner/committee.go in signAttesterDuty and the block around “broadcasted post‑consensus partial signature message”.
      - Doppelganger gating is called via r.doppelgangerHandler.CanSign.

  - When quorum is reached, the runner reconstructs the BLS signature, inserts it into the attestation, and submits to the beacon node.
      - protocol/v2/ssv/runner/committee.go in ProcessPostConsensus.
      - Submission is via r.beacon.SubmitAttestations.
      - Once submitted, it records the duty completion and finalizes.
