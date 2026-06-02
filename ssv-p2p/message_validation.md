# SSV P2P - Message Validation at the Network Boundary

This document explains how SSV validates incoming protocol messages before they are allowed to propagate and enter runner/consensus processing.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

- Understanding how P2P validation protects consensus safety/liveness by filtering invalid traffic early.

## Where validation is attached

1. Validator is instantiated in node wiring:
   - `cli/operator/node.go` creates `validation.New(...)`.
2. P2P pubsub is configured with this validator:
   - `network/p2p/p2p_setup.go::setupPubsub(...)`.
3. Topic validators are registered per subscription:
   - `network/topics/controller.go::setupTopicValidator(...)`,
   - `RegisterTopicValidator(..., msgValidator.ValidatorForTopic(...))`.

## Validation pipeline

Top-level entrypoint:

- `message/validation/validation.go::Validate(...)`

Flow:

1. `handlePubsubMessage(...)`
2. `validatePubSubMessage(...)` (raw pubsub envelope checks)
3. `decodeSignedSSVMessage(...)`
4. `validateSignedSSVMessage(...)`
5. `validateSSVMessage(...)`
6. committee/topic checks (`committeeChecks(...)`)
7. role-specific body validation:
   - `validateConsensusMessage(...)`, or
   - `validatePartialSignatureMessage(...)`
8. return `Accept`, `Ignore`, or `Reject`.

## Result semantics: Accept vs Ignore vs Reject

- Mapping is handled by `handleValidationError(...)` in `errors.go`.
- `Reject` errors return `pubsub.ValidationReject` and contribute to negative peer scoring signals.
- Non-reject validation errors return `pubsub.ValidationIgnore`.
- Successful validation returns `pubsub.ValidationAccept`.

This distinction is critical: not every invalid message is punished equally.

## Stateless checks (fast boundary checks)

### Pubsub envelope

- `pubsub_validation.go::validatePubSubMessage(...)` checks:
  - non-empty data,
  - max encoded size (`MaxEncodedMsgSize`).

### SignedSSV envelope

- `signed_ssv_message.go::validateSignedSSVMessage(...)` checks:
  - non-nil message,
  - signer/signature counts exist and match,
  - RSA signature byte length,
  - sorted unique non-zero signers,
  - signer existence in operator registry.

### SSV message envelope

- `validateSSVMessage(...)` checks:
  - payload non-empty and size bounded,
  - allowed message type,
  - domain matches local network domain,
  - role is one of supported runner roles.

## Contextual/topic checks (committee boundary)

- `validation.go::committeeChecks(...)` enforces:
  - signers belong to expected committee,
  - message arrived on the correct committee subnet topic.

This prevents cross-topic pollution and wrong-committee forwarding.

## Consensus message validation

Entrypoint:

- `consensus_validation.go::validateConsensusMessage(...)`

Checks include:

- decodable QBFT body,
- valid QBFT message type,
- non-zero round,
- identifier consistency (`QBFT.Identifier == SSV.MsgID`),
- fullData hash consistency with root (when fullData is present),
- proposal-only and round-change-only justification shape rules,
- role eligibility (e.g. roles without consensus are rejected),
- role round limits (`maxRound(...)`),
- proposer/leader check for proposal messages (`roundRobinProposer(...)`),
- duplicate/equivocation controls via per-signer state and seen message types,
- timing bounds (slot/round spread and duty-time windows),
- operator signature verification for all included signers,
- state update (`updateConsensusState(...)`) only after successful checks.

## Partial-signature message validation

Entrypoint:

- `partial_validation.go::validatePartialSignatureMessage(...)`

Checks include:

- decodable partial-signature body,
- exactly one signer at signed envelope level,
- no fullData in partial signature envelope,
- valid partial signature type and role/type compatibility,
- non-empty partial signature list,
- signer consistency for each partial message item,
- validator index checks (for validator-duty roles),
- slot/duty timing checks,
- per-duty message count limits and anti-dup limits,
- signature verification on envelope signer,
- state update (`updatePartialSignatureState(...)`) after successful checks.

## Timing, slot, and round protection

- Slot freshness and lateness windows:
  - `common_checks.go::validateSlotTime(...)`,
  - `messageEarliness(...)`,
  - `messageLateness(...)`.
- Round plausibility:
  - `consensus_validation.go::roundBelongsToAllowedSpread(...)`.
- Duty assignment checks against local duty store:
  - `common_checks.go::validateBeaconDuty(...)`.

These checks prevent stale/future/out-of-context messages from entering consensus execution paths.

## Stateful anti-equivocation and anti-replay tracking

Validator state structures:

- `consensus_state.go` (`ValidatorState`, `OperatorState`),
- `signer_state.go` (`SignerState`),
- `seen_msg_types.go` (`SeenMsgTypes` bitset).

Used to enforce:

- one message of each type per signer per round where applicable,
- no repeated decided signer-sets in same duty context,
- no backwards slot/round regressions per signer state.

## What this boundary does and does not guarantee

- It guarantees strong message-admission checks before routing to runner queues.
- It does not guarantee that accepted messages will be processed in time:
  - downstream queues can still saturate and drop (`validator/queue_*`, `validator/timer.go`).
- It is a correctness gate, not a delivery guarantee.

## Critical snippets

### Validation entrypoint and result mapping

```go
func (mv *messageValidator) Validate(ctx context.Context, peerID peer.ID, pmsg *pubsub.Message) pubsub.ValidationResult {
    decodedMessage, err := mv.handlePubsubMessage(pmsg, time.Now())
    if err != nil {
        return mv.handleValidationError(ctx, peerID, decodedMessage, err)
    }
    pmsg.ValidatorData = decodedMessage
    return mv.handleValidationSuccess(ctx, decodedMessage)
}
```

Source: `../ssv/message/validation/validation.go`.

### Non-reject errors are ignored; reject errors are penalized

```go
func (mv *messageValidator) handleValidationError(...) pubsub.ValidationResult {
    var valErr Error
    if !errors.As(err, &valErr) {
        return pubsub.ValidationIgnore
    }
    if !valErr.Reject() {
        return pubsub.ValidationIgnore
    }
    return pubsub.ValidationReject
}
```

Source: `../ssv/message/validation/errors.go`.

## How to verify quickly

1. Run message validation tests in `../ssv/message/validation`.
2. Inject malformed messages and confirm `Ignore` vs `Reject` behavior matches `errors.go`.
3. Confirm rejected traffic appears in peer-score related telemetry.
