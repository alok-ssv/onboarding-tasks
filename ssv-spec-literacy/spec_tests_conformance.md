# SSV Spec Literacy - Spec Tests and Conformance

This document explains how SSV spec tests are generated, executed, and used to validate protocol conformance and interoperability.

## Scope

- Execution of JSON-defined spec tests generated from `ssv-spec`
- Validation of SSV protocol transitions/message flows independent from CL/EL business logic
- Verification of duty orchestration, sequencing, and signature aggregation behavior
- Deterministic input/output checks for core protocol components
- Conformance checks for independent implementations

## Outcome

- Ability to prove an SSV node implementation conforms to the protocol spec using deterministic, reusable vectors.

## What is being tested

### `types` spectests

- Encoding/decoding and validation correctness for protocol objects:
  - `SSVMessage`, `SignedSSVMessage`, `PartialSignatureMessages`, consensus payload types.
- Structure-size bounds and SSZ limits:
  - `ssv-spec/types/spectest/tests/maxmsgsize/test.go`.
- Canonical validation rules:
  - signer uniqueness, field consistency, message shape.

### `qbft` spectests

- Message processing and controller behavior:
  - proposal/prepare/commit/round-change,
  - late/future/decided message handling,
  - round-robin proposer behavior,
  - timeout behavior.
- Deterministic post-state and output-message assertions:
  - `ssv-spec/qbft/spectest/tests/msg_processing_spectest.go`
  - `ssv-spec/qbft/spectest/tests/controller_spectest.go`.

### `ssv` spectests

- Runner and committee duty execution behavior around QBFT:
  - pre/consensus/post flow,
  - partial-signature quorum and reconstruction,
  - committee and validator runner orchestration.
- Deterministic checks include:
  - expected error codes,
  - output partial-signature messages,
  - beacon broadcast roots,
  - post-duty state roots.

## JSON vector generation pipeline (`ssv-spec`)

- Generators:
  - `ssv-spec/qbft/spectest/generate/main.go`
  - `ssv-spec/ssv/spectest/generate/main.go`
  - `ssv-spec/types/spectest/generate/main.go`
- Each generator writes:
  - `tests.json` (full vector set),
  - per-test JSON files,
  - `state_comparison/` snapshots for deterministic post-state validation.
- Entry points:
  - `go generate ./...` from `ssv-spec` root.
  - `make generate-jsons` (`ssv-spec/Makefile`).

## Running tests in the spec repository

From `../ssv-spec`:

```bash
go generate ./...
go test ./types/spectest ./qbft/spectest ./ssv/spectest
```

Notes:

- `run_test.go` in each spectest package loads generated `generate/tests.json` and re-runs vectors through typed test structs.
- This validates that generated vectors and in-repo reference implementation stay consistent.

## Running conformance tests in `ssv` implementation

From `../ssv`:

```bash
go test ./protocol/v2/qbft/spectest ./protocol/v2/ssv/spectest
```

Or run the repo target:

```bash
make spec-test
```

Implementation mapping entry points:

- `protocol/v2/qbft/spectest/qbft_mapping_test.go::TestQBFTMapping`
- `protocol/v2/ssv/spectest/ssv_mapping_test.go::TestSSVMapping`

JSON loading bridge:

- `ibft/storage/testutils.go::GenerateSpecTestJSON`
  - resolves spec module path,
  - builds generator binary,
  - executes it,
  - reads `tests.json`.

## Determinism model (why vectors are useful)

The spectest format validates deterministic protocol behavior with fixed input/output:

- Input:
  - duty objects, incoming message sequence, initial runner/controller state.
- Output:
  - emitted SSV messages (including signer/root expectations),
  - reconstructed/beacon-submission roots,
  - expected error codes.
- Post-state:
  - root snapshots (`PostRoot`, `PostDutyRunnerStateRoot`, etc.) compared against expected state-comparison fixtures.

This gives reproducible protocol assertions across implementations without requiring live Ethereum CL/EL systems.

## Conformance and interoperability

- JSON vectors are implementation-agnostic artifacts.
- Independent SSV implementations can run the same vectors and compare:
  - acceptance/rejection decisions,
  - emitted outputs,
  - final state roots.
- Matching these outcomes is the practical interoperability signal.

## Important caveats

- `GenerateSpecTestJSON` uses the `github.com/ssvlabs/ssv-spec` module resolved from `go.mod`.
- If you want to validate against local `../ssv-spec` changes, add a temporary replace in `../ssv/go.mod`:

```go
replace github.com/ssvlabs/ssv-spec => ../ssv-spec
```

- Some implementation-side assertions are intentionally relaxed pending upstream alignment (for example, comments in `protocol/v2/ssv/spectest/msg_processing_type.go` referencing `ssv-spec` issue alignment).

## Recommended verification workflow

1. Generate fresh vectors in `ssv-spec`.
2. Run `ssv-spec` spectests to confirm vector integrity.
3. Run `ssv` mapping spectests to validate implementation conformance.
4. If mismatches occur, inspect:
   - message semantic checks,
   - quorum/threshold conditions,
   - state transition side effects,
   - output message ordering/contents.
