## Duty retrieval from Consensus Client

- Scheduler is wired in `operator/node.go::New(...)` and started in `operator/node.go::Start(...)`.
- Beacon-facing duty APIs are abstracted in `operator/duties/scheduler.go::BeaconNode` and consumed by scheduler handlers.
- Startup path:
  - `operator/duties/scheduler.go::Start(...)` subscribes to head events (`listenToHeadEvents`) and calls each handler's `HandleInitialDuties(...)`.
- Duty sources:
  - Attester duties: `beacon/goclient/attest.go::AttesterDuties(...)` -> consumed in `operator/duties/attester.go::fetchAndProcessDuties(...)`.
  - Proposer duties: `beacon/goclient/proposer.go::ProposerDuties(...)` -> consumed in `operator/duties/proposer.go::fetchAndProcessDuties(...)`.
  - Sync committee duties: `beacon/goclient/sync_committee.go::SyncCommitteeDuties(...)` -> consumed in `operator/duties/sync_committee.go::fetchAndProcessDuties(...)`.
- Aggregator duties are derived from attester duties (not fetched from a separate CL duty endpoint) in `operator/duties/attester.go::executeAggregatorDuties(...)`.
  - This is explicitly post-Alan fork behavior (see the function comment).
- Duties are cached in:
  - `operator/duties/dutystore/duties.go::Duties.Set(...)`
  - `operator/duties/dutystore/sync_committee.go::SyncCommitteeDuties.Set(...)`

## Slot/epoch scheduling and re-scheduling

- `operator/duties/scheduler.go::NewScheduler(...)` registers handlers; `Start(...)` actually runs them (including `HandleInitialDuties` and goroutine launch for `HandleDuties`).
- Handlers implement lifecycle reset/refetch logic:
  - attester: `operator/duties/attester.go::HandleDuties(...)`
  - proposer: `operator/duties/proposer.go::HandleDuties(...)`
  - sync committee: `operator/duties/sync_committee.go::HandleDuties(...)`
- Validator index changes trigger refresh via `operator/validator/controller.go::reportIndicesChange(...)` and are emitted from task execution paths such as `operator/validator/task_executor.go::ReactivateCluster(...)`.
- Execution dispatch:
  - `operator/validator/controller.go::ExecuteDuty(...)`
  - then queue/event handoff in `protocol/v2/ssv/validator/duty_executor.go::Validator.ExecuteDuty(...)`

## Reorg / fork-choice / head-change handling

- Head event subscription and streaming entry points:
  - `beacon/goclient/events.go::SubscribeToHeadEvents(...)`
  - `beacon/goclient/events.go::startEventListener(...)`
- Scheduler reorg checks happen in `operator/duties/scheduler.go::HandleHeadEvent(...)` with three distinct comparisons:
  1. Epoch transition: old `currentDutyDependentRoot` vs new `PreviousDutyDependentRoot`.
  2. Same-epoch previous-root change: old vs new `PreviousDutyDependentRoot`.
  3. Same-epoch current-root change: old vs new `CurrentDutyDependentRoot`.
- Each check emits `ReorgEvent` with the corresponding `Previous`/`Current` flags.
- Handlers consume reorg events and reset/refetch affected duty windows to avoid stale assignments.

## Alignment with slot boundaries

- Slot math is driven by `networkconfig/beacon.go` (`SlotStartTime`, `Estimated*`, `IntervalDuration` where `intervalsPerSlot=3`) and slot ticker setup in `cli/operator/node.go` (`slotTickerProvider`).
- Scheduler head advancement has dual triggers:
  - time-based one-third-slot fallback in `operator/duties/scheduler.go::SlotTicker(...)`
  - early-head fast path in `operator/duties/scheduler.go::HandleHeadEvent(...)`
- Committee execution is not a simple static gate; it blocks in `operator/duties/scheduler.go::waitOneThirdIntoSlotOrValidBlock(...)` on a condition variable and unblocks when either trigger advances `headSlot`.
- Runner pulls attestation data near execution time in `protocol/v2/ssv/runner/committee.go::executeDuty(...)`.

## Consensus-rule enforcement vs SSV orchestration

- Orchestration layer: scheduler + handlers + controller decide when duties run.
- Consensus enforcement layer: beacon object APIs and validation logic decide what is valid.
  - Beacon client contract: `protocol/v2/blockchain/beacon/client.go` (interfaces).
  - Duty value checks: `protocol/v2/ssv/value_check.go::(*voteChecker).CheckValue(...)`.
  - Consensus message checks: `message/validation/consensus_validation.go::validateConsensusMessage(...)` and shared checks in `message/validation/common_checks.go`.
- This separation keeps Ethereum consensus correctness anchored to CL/rule validators while SSV focuses on distributed execution and liveness.

## Why this prevents misses/duplicates under failures

- Duplicate/stale suppression and execution safety:
  - `protocol/v2/ssv/validator/committee_guard.go::StartDuty(...)` enforces exclusive per-validator duty progression.
  - `ValidDuty(...)` rejects stale/outdated duties.
  - `StopValidator(...)` prevents execution for stopped validators.
- Submission dedupe and completion tracking:
  - `protocol/v2/ssv/runner/committee.go::HasSubmitted(...)`
  - `RecordSubmission(...)`
  - `HasSubmittedAllValidatorDuties(...)`
- Retry model for transient ordering/timing failures:
  - `protocol/v2/ssv/validator/queue_validator.go` replays retryable errors with fixed `25ms` delay and bounded attempts `SlotDuration / 25ms` (not infinite retries).
- Inference from code: lockstep slot timing + bounded retries + reorg-triggered invalidation/refetch significantly reduce stale and duplicate execution under normal fault conditions.
