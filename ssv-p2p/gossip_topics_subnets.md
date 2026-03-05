# SSV P2P - Gossip Topics and Subnets

This document explains how SSV maps duties and consensus traffic onto gossip topics and how messages propagate across operators.

## Read this if

- You need to trace a message from runner output to pubsub topic.
- You need to confirm which subnet a given committee/duty uses.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Duty topics
- Consensus message topics
- Validation hooks

## Outcome

- Ability to trace how protocol messages move across operators over pubsub.

## Topic model used by the implementation

- SSV uses 128 subnets: `network/commons/subnets.go::SubnetsCount`.
- Topic namespace is `ssv.v2.<subnet>` via:
  - `GetTopicFullName(...)`,
  - `SubnetTopicID(...)`.
- Committee to subnet mapping is:
  - `CommitteeSubnet(committeeID) = committeeID mod 128`,
  - topic selection via `CommitteeTopicID(...)`.

### Important practical point

- The network does not have separate "duty topic" and "consensus topic" types.
- Both consensus (`SSVConsensusMsgType`) and partial-signature (`SSVPartialSignatureMsgType`) traffic for a duty are broadcast on the same committee subnet topic.

## How broadcast topic is chosen

Broadcast path is `network/p2p/p2p_pubsub.go::Broadcast(...)`:

- For `RoleCommittee`: topic is derived directly from message `CommitteeID`.
- For other roles (proposer/aggregator/sync contribution/etc.):
  - load validator share from registry storage,
  - derive its committee ID,
  - broadcast to that committee subnet topic.

This means validator-runner duties still propagate on committee subnet gossip.

## Subscription lifecycle

### Persistent and dynamic subscriptions

- Fixed startup subnets come from config and are subscribed in `subscribeToFixedSubnets()`.
- Dynamic committee subscriptions are handled by:
  - `Subscribe(validatorPK)`,
  - `subscribeCommittee(...)`,
  - `Unsubscribe(...)`.
- Effective current subnets are recomputed in `SubscribedSubnets()` and refreshed by `UpdateSubnets()`.

### Subscription filtering

- Pubsub is created with a dynamic subscription whitelist:
  - `topics/pubsub.go::NewPubSub(...)` + `WithSubscriptionFilter(...)`.
- Topic controller updates whitelist on subscribe/unsubscribe:
  - `topics/controller.go::Subscribe(...)`,
  - `Unsubscribe(...)`.

## Propagation path

1. Sender encodes `SignedSSVMessage`.
2. Sender publishes through `topics/controller.go::Broadcast(...)`.
3. Gossipsub forwards through mesh peers (`go-libp2p-pubsub`).
4. Receiver validates through topic validator:
   - `setupTopicValidator(...)`,
   - validator function from `message/validation`.
5. Accepted messages are passed to the router:
   - `p2p_pubsub.go::handlePubsubMessages(...)`,
   - `msgRouter.Route(...)`.

## Validation and dedupe hooks inside pubsub

- Gossipsub options in `topics/pubsub.go::NewPubSub(...)` include:
  - `WithSeenMessagesTTL(...)`,
  - `WithValidateQueueSize(...)`,
  - `WithValidateThrottle(...)`,
  - `WithMaxMessageSize(...)`,
  - `WithMessageIdFn(...)` (custom message ID),
  - `WithPeerScore(...)` thresholds/params.

- Message ID logic is in `topics/msg_id.go`:
  - message ID is content-derived (`xxhash` over encoded message),
  - peers per message ID are tracked for resolver/observability.

## Scoring impact on propagation

- Topic/peer scoring is configured in:
  - `topics/params/topic_score.go`,
  - `topics/params/peer_score.go`,
  - `topics/params/gossipsub.go`.
- Invalid message deliveries (P4) and behavior penalties can reduce score below gossip/publish thresholds, reducing that peer's propagation influence over time.

## End-to-end trace (operator view)

1. Duty message created in runner/controller layer.
2. Network `Broadcast(msgID, signedMsg)` chooses committee subnet topic.
3. Topic controller publishes to `ssv.v2.<subnet>`.
4. Receiving peers run topic validator and assign `Accept/Ignore/Reject`.
5. If accepted, decoded message is routed to validator/committee queues for processing.

## Critical snippet

```go
func (n *p2pNetwork) Broadcast(msgID spectypes.MessageID, msg *spectypes.SignedSSVMessage) error {
    ...
    if msg.SSVMessage.MsgID.GetRoleType() == spectypes.RoleCommittee {
        topics = commons.CommitteeTopicID(spectypes.CommitteeID(msg.SSVMessage.MsgID.GetDutyExecutorID()[16:]))
    } else {
        val, exists := n.nodeStorage.ValidatorStore().Validator(msg.SSVMessage.MsgID.GetDutyExecutorID())
        if !exists {
            return fmt.Errorf("could not find share for validator %s", hex.EncodeToString(msg.SSVMessage.MsgID.GetDutyExecutorID()))
        }
        topics = commons.CommitteeTopicID(val.CommitteeID())
    }
    ...
}
```

Source: `../ssv/network/p2p/p2p_pubsub.go`.
