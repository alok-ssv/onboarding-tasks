# Onboarding Tasks

This repository contains research and technical documentation for understanding Ethereum validator duties and SSV protocol internals, including DVT behavior, QBFT consensus, and specification-to-implementation alignment.

## Version Context

- Source snapshot and commit anchors: [REPO_CONTEXT.md](REPO_CONTEXT.md)

## Suggested Onboarding Order (SSV)

1. [Validator Key Splitting](ssv-dvt/validator_key_splitting.md)
2. [Duty Execution Flow](ssv-dvt/duty_execution_flow.md)
3. [Consensus Interaction](ssv-dvt/consensus_interaction.md)
4. [QBFT Basics and Message Flow](ssv-qbft/qbft_basics_message_flow.md)
5. [QBFT Leader Election and Round Changes](ssv-qbft/qbft_leader_election_round_changes.md)
6. [QBFT Failure and Adversarial Scenarios](ssv-qbft/qbft_failure_adversarial_scenarios.md)
7. [Spec Structure and Core Concepts](ssv-spec-literacy/spec_structure_core_concepts.md)
8. [Spec to Implementation Matching](ssv-spec-literacy/spec_to_implementation_matching.md)
9. [Spec Tests and Conformance](ssv-spec-literacy/spec_tests_conformance.md)
10. [P2P Fundamentals](ssv-p2p/p2p_fundamentals.md)
11. [Gossip Topics and Subnets](ssv-p2p/gossip_topics_subnets.md)
12. [Message Validation at the Network Boundary](ssv-p2p/message_validation.md)
13. [Network Failure Modes](ssv-p2p/network_failure_modes.md)
14. [Node Architecture in Kubernetes](ssv-infra-ops/node_architecture_kubernetes.md)
15. [Persistent Storage and Key Management](ssv-infra-ops/persistent_storage_key_management.md)
16. [Deployment](ssv-infra-ops/deployment.md)
17. [Debugging and Failure Recovery](ssv-infra-ops/debugging_failure_recovery.md)

## Table of Contents

### Ethereum Duties Analysis

- [Aggregation Duties](analyze-ethereum-duties/aggregation_duties.md)
- [Attestation Duties and Fork Choice](analyze-ethereum-duties/attestation_duties_and_fork_choice.md)
- [Block Proposal](analyze-ethereum-duties/block_proposal.md)
- [Sync Committee](analyze-ethereum-duties/sync_committee.md)

### SSV DVT

- [Consensus Interaction](ssv-dvt/consensus_interaction.md)
- [Duty Execution Flow](ssv-dvt/duty_execution_flow.md)
- [Validator Key Splitting](ssv-dvt/validator_key_splitting.md)

### SSV QBFT

- [QBFT Basics and Message Flow](ssv-qbft/qbft_basics_message_flow.md)
- [QBFT Failure and Adversarial Scenarios](ssv-qbft/qbft_failure_adversarial_scenarios.md)
- [QBFT Leader Election and Round Changes](ssv-qbft/qbft_leader_election_round_changes.md)

### SSV Specification Literacy

- [Spec Structure and Core Concepts](ssv-spec-literacy/spec_structure_core_concepts.md)
- [Spec Tests and Conformance](ssv-spec-literacy/spec_tests_conformance.md)
- [Spec to Implementation Matching](ssv-spec-literacy/spec_to_implementation_matching.md)

### SSV P2P Networking

- [P2P Fundamentals](ssv-p2p/p2p_fundamentals.md)
- [Gossip Topics and Subnets](ssv-p2p/gossip_topics_subnets.md)
- [Network Failure Modes](ssv-p2p/network_failure_modes.md)
- [Message Validation at the Network Boundary](ssv-p2p/message_validation.md)

### SSV Infrastructure and Operations

- [Node Architecture in Kubernetes](ssv-infra-ops/node_architecture_kubernetes.md)
- [Persistent Storage and Key Management](ssv-infra-ops/persistent_storage_key_management.md)
- [Debugging and Failure Recovery](ssv-infra-ops/debugging_failure_recovery.md)
- [Deployment](ssv-infra-ops/deployment.md)
