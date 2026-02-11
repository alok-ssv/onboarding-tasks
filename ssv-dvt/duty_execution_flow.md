
## Scheduling path

  - Node wiring: scheduler is created and started in operator/node.go:116, operator/node.go:162.
  - Scheduler startup + head subscription: operator/duties/scheduler.go:182, operator/duties/scheduler.go:332.
  - Duty handlers run per role:
      - Attester fetch + aggregator execute: operator/duties/attester.go:76.
      - Proposer fetch + execute: operator/duties/proposer.go:59.
      - Sync committee fetch + execute: operator/duties/sync_committee.go:75.
      - Committee duties (attester + sync committee execution): operator/duties/committee.go:48.
  - Actual dispatch:
      - Validator duties: operator/duties/scheduler.go:414.
      - Committee duties: operator/duties/scheduler.go:464.
  - Committee duties intentionally wait until 1/3 slot or early head block: operator/duties/scheduler.go:567 and early-head handling in operator/duties/scheduler.go:332.


## Execution path after scheduling
  - Validator duty: operator/validator/controller.go:607.
  - Committee duty: operator/validator/controller.go:644.
  - Validator duty is pushed as ExecuteDuty event into per-runner queue: protocol/v2/ssv/validator/duty_executor.go:21.
  - Message loop routes by message type (consensus vs partial sig vs event): protocol/v2/ssv/validator/validator.go:175.


## Partial signing, quorum, reconstruction

  - Common partial-sig handling:
      - Pre-consensus processing: protocol/v2/ssv/runner/runner.go:232.
      - Post-consensus processing: protocol/v2/ssv/runner/runner.go:323.
      - Partials are inserted + quorum checked per (validator, signing_root): protocol/v2/ssv/runner/runner.go:360.
  - Container + reconstruction:
      - Store partials: protocol/v2/ssv/partial_sig_container.go:37.
      - Quorum check: protocol/v2/ssv/partial_sig_container.go:143.
      - Reconstruct BLS signature: protocol/v2/ssv/partial_sig_container.go:112.
  - If reconstructed signature fails verification, fallback verifies individual partials and drops bad ones: protocol/v2/ssv/runner/runner_validations.go:44.


## Role-specific flow

  - Proposer flow:
      - Pre-consensus: partial RANDAO, quorum, reconstruct, fetch block: protocol/v2/ssv/runner/proposer.go:113.
      - Consensus on block value: protocol/v2/ssv/runner/proposer.go:233.
      - Post-consensus: partial block sig quorum, reconstruct, submit block to BN: protocol/v2/ssv/runner/proposer.go:333.
      - Initial duty trigger (broadcast partial RANDAO): protocol/v2/ssv/runner/proposer.go:473.
  - Committee flow (attester + sync committee):
      - Duty starts by fetching attestation data and deciding beacon vote: protocol/v2/ssv/runner/committee.go:1028.
      - After consensus, signs post-consensus partials for validator duties: protocol/v2/ssv/runner/committee.go:259.
      - On quorum, reconstruct per root/validator, build objects, submit attestations/sync msgs to BN: protocol/v2/ssv/runner/committee.go:527.


## Logs to watch
  Filter by duty_id (added in scheduler/controller contexts) and runner_role/beacon_role.

  - Field names are defined in observability/log/fields/fields.go.
  - Scheduler logs:
      - duty scheduler started
      - subscribing to head events
      - Head event: Block arrived before 1/3 slot
      - executing validator duty / executing committee duty
      - late duty execution
  - Base runner state logs:
      - got pre-consensus message
      - got pre-consensus quorum
      - QBFT instance is decided
      - got post-consensus message
      - got post-consensus quorum
  - Proposer runner logs:
      - signed & broadcasted partial RANDAO signature
      - got beacon block proposal
      - broadcasted post-consensus partial signature message
      - submitting block proposal
      - successfully submitted block proposal
  - Committee runner logs:
      - signed attestation data
      - reconstructed partial signature
      - successfully submitted attestations
      - successfully submitted sync committee
      - finished duty processing (100% success|partial success)

  For trace-level debugging, docs/TRACES.md:43 gives useful attributes and TraceQL patterns (ssv.validator.duty.id, slot, role, submission events).
