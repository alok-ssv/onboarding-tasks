# SSV QBFT - Basics and Message Flow

This document explains how a single SSV duty reaches QBFT finality.

## Scope

- Proposal (`pre-prepare` equivalent), prepare, commit phases
- Quorum requirements under `n >= 3f+1`
- Consensus instance lifecycle from start to decided

## Quorum model used by SSV

- Fault tolerance is represented as `f` (`CommitteeMember.FaultyNodes`): `ssv-spec/types/operator.go::CommitteeMember`.
- Full quorum is `2f+1`: `ssv-spec/types/operator.go::HasQuorum(...)`.
- Partial quorum is `f+1`: `ssv-spec/types/operator.go::HasPartialQuorum(...)`.
- QBFT helper functions consume those thresholds: `ssv-spec/qbft/messages.go::HasQuorum(...)` and `HasPartialQuorum(...)`.
- IBFT/QBFT assumption is `n >= 3f+1` (see `ssv/ibft/IBFT.md`).
- Current implementation targets committee sizes `4, 7, 10, 13` (`ssv-spec/qbft/README.md`).

## Consensus instance lifecycle

1. Instance start
- Runners start QBFT via `protocol/v2/ssv/runner/runner.go::decide(...)`.
- This calls `protocol/v2/qbft/controller/controller.go::StartNewInstance(...)`.
- A new instance is created and started in `protocol/v2/qbft/instance/instance.go::Start(...)`.
- Round-1 timeout is armed immediately with `TimeoutForRound(height, FirstRound)`.

2. Proposal phase (`pre-prepare` in IBFT terms)
- Leader is computed deterministically (`ProposerForRound` -> `RoundRobinProposer`).
- If local operator is leader, `Start(...)` creates and broadcasts a proposal.
- Recipients validate proposal in `protocol/v2/qbft/instance/proposal.go::isValidProposal(...)`:
  - correct proposer for round,
  - valid signatures and signer in committee,
  - `Hash(fullData) == Root`,
  - round-change/prepare justifications for rounds `> 1`.
- On acceptance, `uponProposal(...)` stores proposal and broadcasts prepare.

3. Prepare phase
- `protocol/v2/qbft/instance/prepare.go::uponPrepare(...)` stores first prepare per signer/round.
- When `2f+1` unique prepare signers are present:
  - instance records `LastPreparedRound` and `LastPreparedValue`,
  - broadcasts commit (`CreateCommit(...)` + `Broadcast(...)`).

4. Commit phase
- `protocol/v2/qbft/instance/commit.go::UponCommit(...)` stores first commit per signer/round.
- On commit quorum (`2f+1` unique signers for same round/root), it:
  - aggregates commit signatures (`aggregateCommitMsgs(...)`),
  - returns `decided = true` and the decided value.

5. Decided propagation
- Controller receives decided result in `protocol/v2/qbft/controller/controller.go::UponExistingInstanceMsg(...)`.
- It broadcasts the aggregated decided commit via `protocol/v2/qbft/controller/controller.go::broadcastDecided(...)`.
- Runners mark QBFT completion in `protocol/v2/ssv/runner/runner.go::baseConsensusMsgProcessing(...)` and move to post-consensus signing/submission.

## What "finality" means in this flow

- A duty is final at QBFT layer when there is a commit message with quorum signer set (`2f+1`) for one root/value.
- Decided-message validation enforces:
  - commit message type,
  - valid signatures,
  - `Hash(fullData) == Root`.
  - See `protocol/v2/qbft/controller/decided.go::ValidateDecided(...)`.

## State and dedupe mechanics

- Per-instance containers track proposal/prepare/commit/round-change messages:
  - `State.ProposeContainer`
  - `State.PrepareContainer`
  - `State.CommitContainer`
  - `State.RoundChangeContainer`
  - Defined in `ssv-spec/qbft/state.go`.
- For proposal/prepare/commit/round-change processing, first message per signer+round is kept (`ssv-spec/qbft/message_container.go::AddFirstMsgForSignerAndRound(...)`), preventing duplicate-trigger loops.
- Controller stores only recent instances (`protocol/v2/qbft/controller/types.go::InstanceContainer`).

