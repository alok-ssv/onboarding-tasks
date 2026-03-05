# SSV P2P - Network Failure Modes

This document analyzes how adverse network conditions affect SSV consensus liveness and validator performance.

## Read this if

- You need incident triage expectations under partitions/latency/loss.
- You are correlating networking symptoms with QBFT liveness outcomes.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Partitions
- High latency
- Message duplication or loss

## Outcome

- Understanding how network faults map to QBFT liveness risk and operational behavior.

## Failure model summary

| Condition | Primary impact | Safety | Liveness |
| --- | --- | --- | --- |
| Network partition | Some operators cannot exchange enough messages | Preserved | Can stall |
| High latency / jitter | Late proposal/prepare/commit and timeout churn | Preserved | Degraded |
| Duplicate/replayed messages | Extra validation/load | Preserved | Usually minor |
| Queue saturation / local drops | Valid messages/timeouts can be dropped locally | Preserved | Degraded/stalled |

## 1. Partitions

### What happens

- QBFT still requires quorum (`2f+1`) to decide.
- If a partition has fewer than `2f+1` reachable operators, it cannot finalize that duty instance.
- Minority partitions stall until connectivity recovers.

### Why

- Round timers keep pushing round changes:
  - `protocol/v2/qbft/roundtimer/timer.go::TimeoutForRound(...)`,
  - `protocol/v2/qbft/instance/timeout.go::UponRoundTimeout(...)`.
- But round change still needs enough valid messages to progress.

### Practical effect

- Duty misses increase when partition duration exceeds slot-time budget.
- Safety remains intact, but liveness is lost for affected instances.

## 2. High latency and jitter

### What happens

- Messages arrive after local expected round/slot windows.
- Nodes trigger additional round changes and may reject far-out-of-window messages.

### Validation boundaries involved

- Round spread checks: `message/validation/consensus_validation.go::roundBelongsToAllowedSpread(...)`.
- Slot freshness checks: `message/validation/common_checks.go::validateSlotTime(...)`.
- Role-dependent timeout schedule:
  - committee/aggregator start from slot fractions,
  - proposer uses quick/slow timeout progression.
  - See `protocol/v2/qbft/roundtimer/timer.go::RoundTimeout(...)`.

### Practical effect

- Consensus may eventually complete, but with extra rounds and lower headroom before duty deadlines.

## 3. Message duplication

### What happens

- Duplicate gossip is expected in mesh networks.
- Replay/equivocal patterns are filtered by validation state tracking.

### Controls

- Gossipsub seen-cache: `topics/pubsub.go::WithSeenMessagesTTL(...)`.
- Message-type dedupe per signer/round: `message/validation/seen_msg_types.go`.
- Decided signer-set replay tracking via bitmasks:
  - `message/validation/consensus_validation.go::validateQBFTLogic(...)`,
  - `message/validation/quorum.go`.

### Practical effect

- Usually extra CPU only; direct liveness impact is small unless flooding causes local queue pressure.

## 4. Message loss and local drops

### Sources

- Real network loss.
- Local queue backpressure:
  - validator queue full: `protocol/v2/ssv/validator/queue_validator.go::EnqueueMessage(...)`,
  - committee queue full: `protocol/v2/ssv/validator/committee_queue.go`,
  - timeout event drops when queue full: `protocol/v2/ssv/validator/timer.go`.

### Retry behavior

- Retryable processing errors are replayed, but bounded:
  - fixed retry delay `25ms`,
  - attempts bounded by `SlotDuration / 25ms`.
  - See `queue_validator.go` and `committee_queue.go`.

### Practical effect

- Not all transient failures recover.
- Once bounded retries expire or queue keeps dropping, duty liveness is at risk.

## 5. Discovery/peer-set churn effects

- Discovery and peer trimming are continuous:
  - peer proposal/selection: `network/p2p/p2p_discovery.go::startDiscovery(...)`,
  - periodic trimming: `network/p2p/p2p.go::peersTrimming(...)`.
- Recently trimmed peers are temporarily blocked by connection gater:
  - `network/peers/connections/conn_gater.go::InterceptSecured(...)`.

If churn is high, committee paths may become unstable, increasing round changes and tail latency.

## Safety and liveness takeaway

- Safety is primarily preserved by QBFT quorum/intersection and strict validation.
- Liveness depends on enough timely connectivity, bounded local queues, and completion within slot-round timing windows.
- P2P is best-effort; consensus liveness is probabilistic under stress, not guaranteed.
