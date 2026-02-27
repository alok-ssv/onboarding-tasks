# SSV Spec Literacy - Spec-to-Implementation Matching

This document maps `ssv-spec` definitions to `ssv` implementation entry points so protocol reviews can be done against the real source of truth.

## Scope

- Message formats
- State machines
- Consensus transitions

## Outcome

- Ability to verify implementation behavior against spec expectations with concrete symbol-level anchors.

## Version pinning comes first

- The implementation depends on `github.com/ssvlabs/ssv-spec` directly (`../ssv/go.mod`).
- Current pinned module in the ssv repository is `github.com/ssvlabs/ssv-spec v1.2.2`.
- Practical implication:
  - If you compare local `../ssv-spec` checkout to `../ssv` code, confirm they are the same version/commit or add a `replace` in `../ssv/go.mod` during review.

## Message formats: spec -> implementation

| Protocol contract | Spec anchor | Implementation anchor | Match to verify |
| --- | --- | --- | --- |
| Message routing ID | `ssv-spec/types/messages.go::MessageID`, `NewMsgID` | Used via `spectypes.MessageID` across runner/validator/validation layers | Role/domain/duty-executor routing is derived from same type and accessors (`GetRoleType`, `GetDutyExecutorID`). |
| Envelope message | `ssv-spec/types/messages.go::SSVMessage` | Consumed in `message/validation` and `protocol/v2/ssv/queue` paths | Msg type/id/data semantics match spec encoding and validation expectations. |
| Signed envelope | `ssv-spec/types/messages.go::SignedSSVMessage` | Consensus and partial-sig processing consume `*spectypes.SignedSSVMessage` directly | Signer/signature cardinality rules are inherited from spec object validation. |
| Partial signatures | `ssv-spec/types/partial_sig_message.go::PartialSignatureMessages` | `protocol/v2/ssv/runner/*::ProcessPreConsensus/ProcessPostConsensus`, `protocol/v2/ssv/partial_sig_container.go` | Pre/post-consensus signature flow is built on spec message schema and roots. |
| QBFT message object | `ssv-spec/qbft/messages.go::Message` | Decoded via `specqbft.DecodeMessage(...)` in controller/validation | MsgType/Height/Round/Identifier/Root semantics remain spec-defined. |

## QBFT state machine: spec -> implementation

| Transition | Spec anchor | Implementation anchor | What to check |
| --- | --- | --- | --- |
| Start instance | `ssv-spec/qbft/controller.go::StartNewInstance`, `qbft/instance.go::Start` | `protocol/v2/qbft/controller/controller.go::StartNewInstance`, `protocol/v2/qbft/instance/instance.go::Start` | Same start invariants: value check, height checks, round-1 start, leader proposal broadcast. |
| Process message dispatch | `ssv-spec/qbft/instance.go::ProcessMsg` | `protocol/v2/qbft/instance/instance.go::ProcessMsg` | Proposal/prepare/commit/round-change dispatch parity. |
| Proposal rules | `ssv-spec/qbft/proposal.go::isValidProposal`, `uponProposal` | `protocol/v2/qbft/instance/proposal.go::isValidProposal`, `uponProposal` | Proposer identity, justification checks, root/fullData consistency. |
| Prepare quorum -> commit | `ssv-spec/qbft/prepare.go::uponPrepare` | `protocol/v2/qbft/instance/prepare.go::uponPrepare` | `2f+1` prepare quorum drives commit broadcast in same round/value. |
| Commit quorum -> decided | `ssv-spec/qbft/commit.go::UponCommit` | `protocol/v2/qbft/instance/commit.go::UponCommit` | Decided only on quorum, aggregate commit signers, carry decided full data. |
| Round change | `ssv-spec/qbft/round_change.go::uponRoundChange` | `protocol/v2/qbft/instance/round_change.go::uponRoundChange` | Partial quorum -> broadcast RC, quorum -> next-round proposal path. |

## Runner state machine: spec -> implementation

| Runner behavior | Spec anchor | Implementation anchor | What to check |
| --- | --- | --- | --- |
| Shared runner phases | `ssv-spec/ssv/runner.go::basePreConsensusMsgProcessing`, `baseConsensusMsgProcessing`, `basePostConsensusMsgProcessing` | `protocol/v2/ssv/runner/runner.go::basePreConsensusMsgProcessing`, `baseConsensusMsgProcessing`, `basePostConsensusMsgProcessing` | Same three-phase model with duty-scoped state and quorum-driven transitions. |
| Runner state container | `ssv-spec/ssv/runner_state.go::State` | `protocol/v2/ssv/runner/runner_state.go::State` | Holds pre/post sig containers, running instance, decided value, duty completion state. |
| Partial sig container semantics | `ssv-spec/ssv/partial_sig_container.go` | `protocol/v2/ssv/partial_sig_container.go` | Same validator/root/signer keying and quorum checks; impl adds thread safety. |
| Committee runner pre-consensus | `ssv-spec/ssv/committee_runner.go::ProcessPreConsensus` | `protocol/v2/ssv/runner/committee.go::ProcessPreConsensus` | Explicitly no pre-consensus phase for committee runner in both. |

## Consensus-transition matching checklist

Use this when reviewing a behavior change in `../ssv`:

1. Identify the transition stage in spec:
   - proposal / prepare / commit / round-change
   - or runner phase pre / consensus / post.
2. Locate the corresponding implementation function in `protocol/v2`.
3. Compare three invariants:
   - message validity checks,
   - quorum threshold condition (`HasQuorum` / `HasPartialQuorum`),
   - state mutation + emitted message(s).
4. Confirm post-state determinism via spectests (see third doc).

## Useful alignment tooling

- Automated structural comparison exists in `../ssv/scripts/spec-alignment/`:
  - `../ssv/scripts/spec-alignment/differ.sh`
  - `../ssv/scripts/spec-alignment/differ.config.yaml`
- Make target: `make spec-alignment-diff` (from `../ssv` root).
- This helps catch drift between spec package APIs and implementation equivalents.

## Expected implementation-only differences (not protocol drift)

- Context/logging/instrumentation wrappers in `protocol/v2` code.
- Retryable error wrappers for async queue-driven processing.
- Concurrency guards (locks/thread-safe containers) in production paths.
- Additional message admission checks in `message/validation/*` before runner/QBFT processing.

These differences are operational hardening; they should not change protocol semantics defined by `ssv-spec`.
