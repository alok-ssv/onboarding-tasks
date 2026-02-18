
## Duty retrieval from Consensus Client

  - Scheduler is initialized in operator/node.go:116 with a beacon-facing interface in operator/duties/scheduler.go:53.
  - It starts by subscribing to head events and fetching initial duties in operator/duties/scheduler.go:185 and operator/duties/scheduler.go:201.
  - Attestation duties come from AttesterDuties in beacon/goclient/attest.go:35, consumed by operator/duties/attester.go:280.
  - Proposal duties come from ProposerDuties in beacon/goclient/proposer.go:31, consumed by operator/duties/proposer.go:210.
  - Sync committee duties come from SyncCommitteeDuties in beacon/goclient/sync_committee.go:15, consumed by operator/duties/sync_committee.go:236.
  - Aggregator duties are derived from attester duties (not fetched as a separate CL duty API), see operator/duties/attester.go:210 and operator/duties/attester.go:236.
  - Duties are cached in duty stores (operator/duties/dutystore/duties.go:99, operator/duties/dutystore/sync_committee.go:63).

## Slot/epoch scheduling and re-scheduling

  - The scheduler runs role handlers (attester/proposer/sync/committee) from operator/duties/scheduler.go:155.
  - Each handler has epoch/period lifecycle logic and reset/reload paths:
      - Attester: operator/duties/attester.go:124, operator/duties/attester.go:149
      - Proposer: operator/duties/proposer.go:100
      - Sync committee: operator/duties/sync_committee.go:101, operator/duties/sync_committee.go:118
  - Validator index changes trigger duty refresh through indicesChange (operator/validator/controller.go:946, operator/validator/task_executor.go:63).
  - Scheduled duties are handed to validator execution via operator/validator/controller.go:625 and then queued/processed by SSV validator logic (protocol/v2/ssv/validator/duty_executor.go:21).

## Reorg / fork-choice / head-change handling

  - Beacon head events are streamed by beacon/goclient/events.go:194.
  - Scheduler compares duty-dependent roots and emits ReorgEvent when current or previous roots changed in operator/duties/scheduler.go:349 and operator/duties/scheduler.go:376.
  - Handlers consume reorg signals and invalidate/reset affected epoch/period duties before refetching.
  - This avoids acting on stale duty assignments after fork-choice movement.

## Alignment with slot boundaries

  - Slot math is driven by beacon network config in networkconfig/beacon.go:39 and slot ticker setup in cli/operator/node.go:492.
  - Scheduler advances head-slot timing with:
      - a fallback at ~1/3 slot (operator/duties/scheduler.go:303)
      - an early-block fast path (operator/duties/scheduler.go:394)
  - Committee execution is gated until headSlot >= dutySlot (operator/duties/scheduler.go:565).
  - Runner pulls attestation data near execution time (protocol/v2/ssv/runner/committee.go:1041) to stay protocol-timed.

## Consensus-rule enforcement vs SSV orchestration

  - Orchestration layer: scheduler + handlers + controller decide when to run duties.
  - Consensus enforcement layer: beacon object APIs + value/message validation decide what is valid:
      - Beacon client contract: protocol/v2/blockchain/beacon/client.go:17
      - Duty value checks: protocol/v2/ssv/value_check.go:41
      - Consensus message checks: message/validation/consensus_validation.go:46, message/validation/common_checks.go:38
  - This separation keeps Ethereum consensus correctness anchored to CL/rule validators, while SSV focuses on distributed execution/liveness.

## Why this prevents misses/duplicates under failures

  - Duplicate/stale suppression: CommitteeDutyGuard in protocol/v2/ssv/validator/committee_guard.go:27.
  - Submission dedupe and completion tracking: protocol/v2/ssv/runner/committee.go:879 and protocol/v2/ssv/runner/committee.go:824.
  - Retry model for transient timing/order issues: protocol/v2/ssv/validator/queue_validator.go:230.
  - Inference from code: this design gives lockstep timing with beacon slots plus fault-tolerant distributed execution, while reorg-triggered invalidation/reload reduces stale or duplicated duty execution risk.
