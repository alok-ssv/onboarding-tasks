# SSV QBFT - Leader Election and Round Changes

This document focuses on how QBFT maintains liveness when the current leader is slow, faulty, or unreachable.

## Scope

- Leader selection logic
- Timeout behavior
- Round-change triggers and proposal justification

## Leader election in SSV QBFT

- Leader selection is deterministic round-robin:
  - `protocol/v2/qbft/round_robin_proposer.go::RoundRobinProposer(...)`.
- Inputs are current `height` (instance) and `round`.
- Rotation properties:
  - within one height, leader advances with each round;
  - across heights, first-round leader is offset by `height % committee_size`.
- Proposal sender must match computed leader:
  - `protocol/v2/qbft/instance/proposal.go::isValidProposal(...)`
  - network-side validation also checks this in `message/validation/consensus_validation.go::validateQBFTLogic(...)`.

## Timeout model

- Round timer is configured per runner role in `protocol/v2/qbft/roundtimer/timer.go`.
- For proposer-like roles:
  - rounds `<= 8`: quick timeout (`2s`),
  - rounds `> 8`: slow timeout (`2m`).
- For committee/aggregation roles:
  - timeout is aligned to slot start plus role-specific base (`1/3` or `2/3` slot) and round-based increments.
- Instance start and round transitions always arm/reset timer with `TimeoutForRound(height, round)`.

## Round-change triggers

1. Timeout trigger
- `protocol/v2/qbft/controller/timer.go::Controller.OnTimeout(...)` calls `protocol/v2/qbft/instance/timeout.go::Instance.UponRoundTimeout(...)`.
- Instance creates and broadcasts `ROUND-CHANGE` for `round+1`, then bumps local round and resets timer.
- Stale timeout events are ignored (`timeoutData.Round < instance.State.Round`).

2. Partial-quorum trigger (`f+1`)
- `protocol/v2/qbft/instance/round_change.go::hasReceivedPartialQuorum(...)` checks `HasPartialQuorum`.
- If `f+1` round-change messages for future rounds exist, node jumps to minimum observed future round and broadcasts its own round-change (`uponChangeRoundPartialQuorum(...)`).
- This is the fast catch-up path when peers already advanced.

3. Full round-change quorum trigger (`2f+1`)
- When a round accumulates full round-change quorum, only the leader for that round may propose.
- Logic:
  - `hasReceivedProposalJustificationForLeadingRound(...)`
  - `isProposalJustificationForLeadingRound(...)`.

## Proposal justification for non-first rounds

For rounds `> 1`, proposal validity requires round-change justification:

- A quorum (`2f+1`) of round-change messages for the same target round.
- If no message indicates "prepared", leader may propose start value.
- If prepared evidence exists, leader must propose the highest prepared value:
  - choose highest `DataRound` via `highestPrepared(...)`,
  - require prepare quorum for that prepared round/root,
  - ensure proposed value root matches that prepared root.
- Implemented in `protocol/v2/qbft/instance/proposal.go::isProposalJustification(...)`.

## Liveness walkthrough: leader failure

1. Round `r` leader fails to send proposal.
2. Timer for round `r` expires -> all honest nodes emit `ROUND-CHANGE(r+1)`.
3. Nodes that receive `f+1` future round-change messages can catch up early (partial quorum path).
4. Once a leader receives `2f+1` valid round-change messages for a round, it broadcasts justified proposal.
5. Network resumes normal proposal -> prepare -> commit flow and decides.

## Guardrails and bounds

- QBFT instance processing halts at configured cut-off round (`Instance.CanProcessMessages()` with `config.GetCutOffRound()`).
- Consensus message validation also enforces role-specific max rounds (`message/validation/consensus_validation.go::maxRound(...)`).

