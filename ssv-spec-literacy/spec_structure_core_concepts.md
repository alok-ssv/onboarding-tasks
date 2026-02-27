# SSV Spec Literacy - Structure and Core Concepts

This document is a navigation map for `ssv-spec` so you can quickly find the protocol source of truth and use the right abstraction level when reading or reviewing code.

## Scope

- Repository layout
- Terminology and abstractions
- Separation of concerns between protocol layers

## Outcome

- Comfortably navigate `ssv-spec` and cite the right source file for any protocol question.

## Repository layout (what each top-level package owns)

- `README.md`
  - High-level protocol model, message model, runner model, and spec-test overview.
- `types/`
  - Canonical protocol data model and validation primitives.
  - Defines message envelopes, duty types, runner roles, consensus payload types, committee and threshold helpers, and SSZ limits.
- `qbft/`
  - QBFT consensus state machine and message processing rules.
  - Owns proposal/prepare/commit/round-change transitions and controller/instance lifecycle.
- `ssv/`
  - Duty execution logic around QBFT:
  - pre-consensus partial signatures, consensus triggering, post-consensus aggregation/submission flow.
- `p2p/`
  - Networking spec and wire/protocol expectations (transport, topics, validation policy, scoring, forks/config).
- `*/spectest/`
  - JSON-vector test generation and JSON-based runners for deterministic spec conformance.

## Core terminology and abstractions

### Message and routing model

- `types.MessageID` is the routing key (`domain + role + duty executor id`): `ssv-spec/types/messages.go::MessageID`.
- `types.NewMsgID(...)` builds deterministic IDs used across layers: `ssv-spec/types/messages.go::NewMsgID`.
- `types.SSVMessage` is the unsigned envelope (`MsgType`, `MsgID`, `Data`): `ssv-spec/types/messages.go::SSVMessage`.
- `types.SignedSSVMessage` is the signed transport object used on the wire: `ssv-spec/types/messages.go::SignedSSVMessage`.

### Duty and role model

- Beacon roles (`ATTESTER`, `AGGREGATOR`, `PROPOSER`, etc.): `ssv-spec/types/beacon_types.go::BeaconRole`.
- Runner roles (`COMMITTEE_RUNNER`, `PROPOSER_RUNNER`, etc.): `ssv-spec/types/runner_role.go::RunnerRole`.
- Duty interfaces and concrete forms:
  - `types.Duty`
  - `types.ValidatorDuty`
  - `types.CommitteeDuty`
  - `ssv-spec/types/beacon_types.go`.
- Role mapping entry point: `ssv-spec/types/beacon_types.go::MapDutyToRunnerRole`.

### Threshold and quorum model

- Committee fault parameter lives in `CommitteeMember.FaultyNodes`: `ssv-spec/types/operator.go::CommitteeMember`.
- Full quorum is `2f+1`: `ssv-spec/types/operator.go::GetQuorum`, `HasQuorum`.
- Partial quorum is `f+1`: `ssv-spec/types/operator.go::HasPartialQuorum`.
- QBFT helpers use unique signers over messages: `ssv-spec/qbft/messages.go::HasQuorum`, `HasPartialQuorum`.
- IBFT/QBFT assumption is `n >= 3f+1` (see `ssv-spec/p2p/SPEC.md` and `ssv-spec/qbft/README.md`).

### Consensus object model

- QBFT message type/fields (proposal/prepare/commit/round-change, root, justifications):
  - `ssv-spec/qbft/messages.go::Message`.
- QBFT instance state:
  - `ssv-spec/qbft/state.go::State`.
- Per-round message containers and dedupe:
  - `ssv-spec/qbft/message_container.go::MsgContainer`.

### Runner lifecycle model

- Runner interface and phase entry points:
  - `StartNewDuty`, `ProcessPreConsensus`, `ProcessConsensus`, `ProcessPostConsensus`
  - `ssv-spec/ssv/runner.go::Runner`.
- Shared base flow:
  - `ssv-spec/ssv/runner.go::basePreConsensusMsgProcessing`
  - `baseConsensusMsgProcessing`
  - `basePostConsensusMsgProcessing`.
- Runner execution state:
  - `ssv-spec/ssv/runner_state.go::State`.

## Separation of concerns (how to avoid mixing layers)

- `types` answers:
  - "What is the canonical wire/data structure?"
  - "What fields are valid and how are objects encoded?"
- `qbft` answers:
  - "When does an instance move rounds?"
  - "When is a value decided?"
  - "What is a valid consensus message at each phase?"
- `ssv` answers:
  - "When does a duty start QBFT?"
  - "How are partial signatures collected/reconstructed before and after consensus?"
  - "When is a duty finished?"
- `p2p` answers:
  - "How are these messages transported, validated in pubsub, and scored in the network?"

## Suggested reading order

1. `ssv-spec/types/messages.go`
2. `ssv-spec/types/beacon_types.go`
3. `ssv-spec/types/operator.go`
4. `ssv-spec/qbft/messages.go`
5. `ssv-spec/qbft/state.go`
6. `ssv-spec/qbft/controller.go`
7. `ssv-spec/qbft/instance.go`
8. `ssv-spec/ssv/runner.go`
9. A concrete runner (`ssv-spec/ssv/proposer.go` or `ssv-spec/ssv/committee_runner.go`)
10. `ssv-spec/p2p/SPEC.md`

## Quick navigation checklist

- Message format question -> start in `types/messages.go`.
- Duty/role mapping question -> start in `types/beacon_types.go`.
- Quorum threshold question -> start in `types/operator.go` then `qbft/messages.go`.
- Consensus transition question -> start in `qbft/instance.go` and phase files (`proposal.go`, `prepare.go`, `commit.go`, `round_change.go`).
- "Who signs what and when?" question -> start in `ssv/runner.go` plus a concrete runner file.
