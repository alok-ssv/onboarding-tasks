# SSV P2P - Fundamentals

This document explains how SSV nodes discover peers, handshake, and maintain connections in the libp2p network.

## Read this if

- You need a mental model of peer lifecycle before diving into topic propagation.
- You are debugging peer churn, failed handshakes, or weak connectivity.

## Version context

- Snapshot reference: [`REPO_CONTEXT.md`](../REPO_CONTEXT.md)

## Scope

- Peer discovery in the SSV network
- Handshakes
- Connection lifecycle

## Outcome

- Understanding how peers find each other and keep stable connectivity for duty execution.

## Network boot sequence

1. P2P network setup starts in `network/p2p/p2p_setup.go::Setup(...)`.
2. Host is created in `SetupHost(...)` with:
   - libp2p identity from the network key,
   - noise security and TCP transport (`network/p2p/config.go::Libp2pOptions(...)`),
   - connection gater (`network/peers/connections/conn_gater.go`),
   - backoff connector for retries.
3. Services are wired in strict order in `SetupServices(...)`:
   - stream controller,
   - peer services (index/identify/handshaker/conn handler),
   - pubsub,
   - discovery.
4. Runtime starts in `network/p2p/p2p.go::Start(...)`:
   - starts discovery loop,
   - starts trimming/reporting loops,
   - subscribes fixed subnets.

## Peer discovery

### Discovery modes

- `network/discovery/service.go::NewService(...)` chooses:
  - `discv5` when `DiscV5Opts` is present,
  - local mDNS when running in local mode.

### Discv5 bootstrap and filtering

- Main loop: `network/discovery/dv5_service.go::Bootstrap(...)`.
- Candidate nodes are filtered by:
  - same domain/network (`checkPeer(...)` + ENR domain entry),
  - non-zero subnets,
  - shared subnet requirement (`sharedSubnetsFilter(1)`),
  - not already connected/discovered/recently trimmed,
  - not marked bad (`badNodeFilter()`).

### Candidate selection and connection attempts

- Discovered peers are buffered in `p2pNetwork.discoveredPeersPool`.
- `network/p2p/p2p_discovery.go::startDiscovery(...)` periodically scores candidates and proposes the best ones based on subnet usefulness plus retry backoff.
- Actual dialing is done through the libp2p backoff connector (`getConnector()` / `BackoffConnector.Connect(...)`), with trusted peers injected first.

## Handshake

### Protocol and payload

- NodeInfo stream protocol is registered in `network/p2p/p2p_setup.go::setupPeerServices(...)` via `peers.NodeInfoProtocol`.
- Handler/initiator logic is in `network/peers/connections/handshaker.go`:
  - inbound stream path: `Handler()`,
  - outbound request path: `Handshake(...)`.
- `sealedNodeRecord()` updates and signs local `records.NodeInfo` before sending.

### Handshake checks

- Filters are applied in `applyFilters(...)`, currently:
  - `NetworkIDFilter(...)` (same network/domain),
  - `BadPeerFilter(...)`.
- Peer subnets are updated from NodeInfo metadata in `updateNodeSubnets(...)`.

## Connection lifecycle

### Admission controls (pre and post secure)

- `network/peers/connections/conn_gater.go` enforces:
  - bad-peer blocking,
  - inbound limit guard,
  - IP rate limiting for inbound accepts,
  - rejection of recently trimmed peers during secure phase.

### Connection state machine

- `network/peers/connections/conn_handler.go::Handle(...)` drives lifecycle.
- Outbound connection flow:
  - set `StateConnecting`,
  - run active handshake,
  - on success set `StateConnected`.
- Inbound connection flow:
  - wait up to 20s for peer-initiated handshake result,
  - require at least one shared subnet (`sharesEnoughSubnets(...)`),
  - set `StateConnected` on success.
- On disconnect, state becomes `StateDisconnected`.
- Peer states are stored in `network/peers/peer_info.go` (`StateConnecting`, `StateConnected`, `StateDisconnected`).

## Guarantees vs non-guarantees

- Guarantees provided by this layer:
  - only same-domain peers pass handshake filters,
  - malformed/blocked peers can be gated early,
  - peer set is continuously rebalanced and trimmed.
- Non-guarantees:
  - no guarantee of full graph connectivity,
  - no guarantee of low-latency paths to every committee peer,
  - discovery/selection are best-effort and adaptive, not deterministic.
