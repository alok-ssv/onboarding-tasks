# SSV QBFT - Failure and Adversarial Scenarios

This document describes how SSV QBFT behaves under Byzantine input, delays, and partial outages, and what is guaranteed vs what can stall.

## Scope

- Byzantine behavior and equivocation attempts
- Message delays, reordering, and drops
- Partial operator outages and partitions

## Assumptions and guarantees

- Safety/liveness analysis uses the standard `n >= 3f+1` model.
- Quorum to decide is `2f+1`; partial round-change acceleration is `f+1`.
- Under these assumptions:
  - safety: at most one value can be decided per instance,
  - liveness: progress requires enough responsive operators and eventual network delivery.

## Scenario analysis

| Scenario | What code does | Safety impact | Liveness impact |
| --- | --- | --- | --- |
| Byzantine proposer sends wrong or unjustified proposal | Proposal is rejected unless leader/signature/root/justifications are valid (`protocol/v2/qbft/instance/proposal.go::isValidProposal(...)`, `isProposalJustification(...)`) | Preserved | May delay current round |
| Signer equivocates in same round (e.g., two different proposals) | Validation tracks per-signer message history and rejects conflicting repeats (`message/validation/consensus_validation.go::validateQBFTLogic(...)`) | Preserved | Can cause retries/timeouts |
| Duplicate messages / replay spam | Per-signer first message per round is retained (`ssv-spec/qbft/message_container.go::AddFirstMsgForSignerAndRound(...)`) | Preserved | Minimal direct impact; can add queue pressure |
| Invalid signatures or unknown signers | Rejected by signature and committee checks (`message/validation/consensus_validation.go`, `protocol/v2/qbft/instance/*VerifySignature(...)`) | Preserved | No progress from invalid msgs |
| Leader silent/offline | Timeout and round-change path triggers (`protocol/v2/qbft/instance/timeout.go`, `protocol/v2/qbft/instance/round_change.go`) | Preserved | Progress continues if enough honest nodes |
| Message delay/out-of-order arrival | Retryable errors are replayed by queue consumers (`protocol/v2/ssv/validator/queue_validator.go`, `protocol/v2/ssv/validator/committee_queue.go`) | Preserved | Usually recovers; latency increases |
| Future messages arrive before local instance starts | Controller marks them retryable (`protocol/v2/qbft/controller/error.go::ErrFutureConsensusMsg`, `ErrInstanceNotFound`) | Preserved | Recovers once instance exists |
| Network partition | Only partition with `2f+1` can decide; others cannot reach quorum | Preserved | Minority partition stalls |
| `f+1` offline/Byzantine operators | Quorum cannot be reached | Preserved | Liveness lost (stall) |
| Queue saturation (queue full) | Messages can be dropped (`TryPush` failure logs in validator/committee queues) | Usually preserved | Can hurt liveness and increase missed duties |

## Important behaviors for delayed/faulty networks

1. Late decided commits can still finalize
- Aggregated commit messages (multiple signers) are handled via `protocol/v2/qbft/controller/decided.go::UponDecided(...)`.
- If valid and quorum-signed, they can mark instance decided even when local node was behind.

2. Retries are bounded, not infinite
- Retry delay is fixed at `25ms`.
- Max attempts are approximately `SlotDuration / 25ms`.
- See retry logic in:
  - `protocol/v2/ssv/validator/queue_validator.go`
  - `protocol/v2/ssv/validator/committee_queue.go`.

3. Timeout events are queue-driven
- RoundTimer callback enqueues timeout events (`protocol/v2/ssv/validator/timer.go`).
- If queue is full, timeout event can be dropped; this is a liveness risk, not a direct safety break.

## Equivocation and conflicting decisions: why safety holds

- Decision requires `2f+1` commit signers for one root (`protocol/v2/qbft/instance/commit.go::commitQuorumForRoundRoot(...)`).
- In `n=3f+1`, any two `2f+1` quorums intersect in at least `f+1` operators.
- With at most `f` Byzantine, two conflicting commit quorums cannot both be formed by honest protocol execution.
- Decided-message validation also checks commit type, signatures, and root/fullData consistency (`protocol/v2/qbft/controller/decided.go::ValidateDecided(...)`).

## Evidence in tests

- SSV runs mapping tests against spec-generated QBFT scenarios:
  - `protocol/v2/qbft/spectest/qbft_mapping_test.go`.
- Included scenario families cover:
  - happy flows for 4/7/10/13 committees,
  - duplicate signers/messages,
  - future/past round handling,
  - `f+1` speed-up round-change cases,
  - timeout transitions and round-change justification cases.

