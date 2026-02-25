# SSV DVT - Duty execution flow

This document follows a duty end-to-end: scheduler -> controller/queue -> runner execution, with operational log and tracing anchors.

## Architecture at a glance

- Scheduler layer: `operator/duties/*`
- Controller and queue layer: `operator/validator/*` and `protocol/v2/ssv/validator/*`
- Runner layer: `protocol/v2/ssv/runner/*`

## Scheduling path

- Scheduler wiring is anchored in `operator/node.go`:
  - `New(...)` constructs the scheduler via `duties.NewScheduler(...)`.
  - `Start(...)` starts it via `n.dutyScheduler.Start(ctx)`.
- Scheduler startup and head subscription:
  - `operator/duties/scheduler.go::Start()`
  - `operator/duties/scheduler.go::listenToHeadEvents()`
  - `operator/duties/scheduler.go::HandleHeadEvent()`
- Handler responsibilities:
  - `operator/duties/attester.go::AttesterHandler` fetches attester duties and executes aggregator duties only (`executeAggregatorDuties`).
  - `operator/duties/proposer.go::ProposerHandler` fetches and executes proposer duties.
  - `operator/duties/sync_committee.go::SyncCommitteeHandler` fetches and executes sync committee contribution duties.
  - `operator/duties/committee.go::CommitteeHandler` executes attester and sync committee duties grouped by committee.
- Dispatch and timing:
  - validator duties: `Scheduler.ExecuteDuties(...)`
  - committee duties: `Scheduler.ExecuteCommitteeDuties(...)`
  - committee duties wait for one-third slot or an early valid head block:
    - `waitOneThirdIntoSlotOrValidBlock(...)`
    - `HandleHeadEvent(...)` with `advanceHeadSlot(...)`

## Execution path after scheduling

- Validator duty path:
  - `operator/validator/controller.go::ExecuteDuty(...)` builds `duty_id` and trace context.
  - `protocol/v2/ssv/validator/duty_executor.go::Validator.ExecuteDuty(...)` pushes an `ExecuteDuty` event into the per-runner queue.
  - `protocol/v2/ssv/validator/validator.go::Validator.ProcessMessage(...)` routes event messages to `OnExecuteDuty(...)`, which calls `StartDuty(...)`.
- Committee duty path:
  - `operator/validator/controller.go::ExecuteCommitteeDuty(...)` builds committee `duty_id` and starts committee processing.
  - `protocol/v2/ssv/validator/committee.go::Committee.StartDuty(...)` prepares runner and queue, then calls runner `StartNewDuty(...)`.
  - `Committee.ProcessMessage(...)` routes consensus and post-consensus messages to committee runner methods.

## Partial signing, quorum, reconstruction

- Base partial-signature handling:
  - pre-consensus processing: `protocol/v2/ssv/runner/runner.go::basePreConsensusMsgProcessing(...)`
  - post-consensus processing: `protocol/v2/ssv/runner/runner.go::basePostConsensusMsgProcessing(...)`
  - first-quorum detection per `(validator_index, signing_root)`: `protocol/v2/ssv/runner/runner.go::basePartialSigMsgProcessing(...)`
- Container and reconstruction:
  - `protocol/v2/ssv/partial_sig_container.go` stores partials keyed by validator and signing root, checks quorum, and reconstructs signatures.
- Fallback path:
  - `protocol/v2/ssv/runner/runner_validations.go::FallBackAndVerifyEachSignature(...)` verifies individual partial signatures and drops bad ones if reconstructed signature validation fails.
- Important architectural distinction:
  - Committee runner has no pre-consensus phase.
  - `protocol/v2/ssv/runner/committee.go::ProcessPreConsensus(...)` returns `no pre consensus phase for committee runner`.

## Role-specific flow

### Proposer runner (`protocol/v2/ssv/runner/proposer.go`)

1. `executeDuty(...)` starts the duty by signing and broadcasting a partial RANDAO signature.
2. `ProcessPreConsensus(...)` waits for RANDAO quorum, reconstructs RANDAO, fetches block, and starts QBFT.
3. `ProcessConsensus(...)` handles QBFT decision and broadcasts post-consensus partial block signature.
4. `ProcessPostConsensus(...)` waits for post-consensus quorum, reconstructs final block signature, and submits block to the beacon node.

### Committee runner (`protocol/v2/ssv/runner/committee.go`)

1. `executeDuty(...)` fetches attestation data, builds beacon vote, and starts QBFT.
2. `ProcessConsensus(...)` handles QBFT decision, signs validator duties, and broadcasts post-consensus partial signatures.
3. `ProcessPostConsensus(...)` reconstructs signatures per root and validator, then submits attestations and sync committee messages.
4. `ProcessPreConsensus(...)` is intentionally unsupported.

## Logs to watch (with levels)

| Component | Level | Message | Notes |
| --- | --- | --- | --- |
| Scheduler | Info | `duty scheduler started` | startup |
| Scheduler | Info | `subscribing to head events` | startup |
| Scheduler | Info | `received head event. Processing...` | emitted for each received head event |
| Scheduler | Debug | `Head event: Block arrived before 1/3 slot` | emitted only when block arrives early |
| Scheduler | Debug | `executing validator duty` / `executing committee duty` | hidden in default Info-level production logs |
| Scheduler | Warn | `late duty execution` | slot delay warning |
| Base runner | Debug | `got pre-consensus message` / `got pre-consensus quorum` | pre-consensus-capable runners only |
| Base runner | Debug | `QBFT instance is decided` | consensus decision |
| Base runner | Debug | `got post-consensus message` / `got post-consensus quorum` | post-consensus |
| Proposer runner | Debug | `signed & broadcasted partial RANDAO signature` | emitted by `executeDuty(...)` |
| Proposer runner | Info | `got beacon block proposal` | after pre-consensus quorum and block fetch |
| Proposer runner | Debug | `broadcasted post-consensus partial signature message` | after QBFT decide |
| Proposer runner | Info | `submitting block proposal` / `successfully submitted block proposal` | final submission |
| Committee runner | Debug | `signed attestation data` / `reconstructed partial signature` | post-consensus per-validator work |
| Committee runner | Info | `successfully submitted attestations` / `successfully submitted sync committee` | final submission |
| Committee runner | Info | `finished duty processing (100% success|partial success)` | completion |

## `duty_id` filtering

- Key log fields: `duty_id`, `runner_role`, `beacon_role` (see `observability/log/fields/fields.go`).
- Validator duty ID:
  - builder: `BuildDutyID(epoch, slot, runnerRole, validatorIndex)`
  - format: `{RUNNER_ROLE}-e{epoch}-s{slot}-v{validator_index}`
  - example: `PROPOSER-e20888-s668436-v843156`
- Committee duty ID:
  - builder: `BuildCommitteeDutyID(operatorIDs, epoch, slot)`
  - format: `COMMITTEE-{op1_op2_op3_op4}-e{epoch}-s{slot}`
  - example: `COMMITTEE-1_2_3_4-e20897-s668706`

## Trace debugging

- For trace-level debugging, use the `TraceQL` section in `docs/TRACES.md` (prefer section references over line numbers).
