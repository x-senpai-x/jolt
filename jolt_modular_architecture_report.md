# Jolt Modular Architecture: Prover & Verifier End-State Report

This document details the architectural walkthrough of the proving and verification phases under the new modular Jolt stack. This stack explicitly splits responsibilities across specialized crates (`jolt-prover`, `jolt-verifier`, `jolt-witness`, `jolt-backends`, `jolt-transcript`) to separate protocol orchestration from hardware-specific computation and state management.

---

## 1. Architectural North Star & Differences vs `jolt-core`

### Legacy `jolt-core`
In `jolt-core`, proving is tightly coupled. The same crate owns the trace decoding, witness polynomial allocation, multilinear algebraic evaluation (sumcheck inner loops), protocol orchestration (choosing which sumcheck to do next), transcript management, and zero-knowledge (BlindFold) integration. This monolithic structure makes it difficult to substitute backends (e.g., adding GPU acceleration) or alter protocol layouts without invasive codebase changes.

### Modular `jolt-prover` / `jolt-verifier` Architecture
The new architecture is driven by a separation of concerns:

- **`jolt-verifier`**: The single source of truth for protocol semantics and verifier-visible proof structures. It defines the exact stage outputs (`StageNOutput`), challenge derivation sequence, and final opening logic.
- **`jolt-prover`**: A handwritten, protocol-level orchestrator. It does **not** do heavy computation. It builds the proof by asking `jolt-witness` for data views and `jolt-backends` for mathematical results.
- **`jolt-witness`**: Reusable infrastructure that manages polynomials. It provides data-access semantics (oracles, views, dense/sparse/streaming streams) directly from guest trace artifacts.
- **`jolt-backends` (e.g. `jolt_backends::cpu`)**: Hardware-specific kernels. They consume typed slot-keyed requests from the prover and return mathematical results (commitments, sumcheck messages, opening hints) completely agnostic of the protocol meaning.
- **`jolt-claims` & `jolt-lookup-tables`**: Extract the math. `jolt-lookup-tables` solely owns instruction lookup queries and MLE table evaluations.

**Key Difference:** In the modular stack, you can swap the compute backend (e.g., moving to CUDA) by implementing the `jolt-backends` traits, while `jolt-prover` guarantees the proof's shape, commitment sequencing, and transcript operations remain perfectly identical.

---

## 2. BlindFold ZK Integration (Transparent vs ZK Paths)

Zero-knowledge is introduced through the **BlindFold** protocol, which compiles the sumcheck proofs into a single, succinct, randomized R1CS (proved via Nova folding and Spartan) rather than composing full SNARKs.

### The ZK Feature Gate
The codebase operates via compile-time capabilities. Features (`zk`, `field-inline`) dictate the compiled code:

- **Transparent Mode**:
  - Proofs contain cleartext round polynomial coefficients (`SumcheckProof::Clear`).
  - Output claims are verified via explicit equality checks.
  - PCS evaluation is bound natively into the transcript.
  - Prover uses methods like `prove_stageN` and `bind_opening_inputs`.

- **ZK Mode (BlindFold)**:
  - The feature gate `cfg(feature = "zk")` is activated.
  - Round coefficients are hidden behind Pedersen commitments. The proof contains `BlindFoldProof` and `CommittedSumcheckProof`s.
  - Instead of computing an output claim explicitly, the prover delegates the checks to an `InputClaimConstraint` and `OutputClaimConstraint` inside the BlindFold R1CS.
  - The verifier reconstructs a `VerifierR1CS` matrix from the `StageConfig`s and `BakedPublicInputs` (which incorporate Fiat-Shamir challenges). The Nova fold + Spartan sumcheck proves the verifier matrix is satisfied.
  - In `Stage 8`, rather than directly evaluating the joint polynomial `W(ry)`, the ZK path uses `ZkOpeningScheme::open_zk`, returning a hidden evaluation output/blinding.

**What is Affected?**
- **Transcript Synchronization**: `input_claim()` and output claims are *not* appended to the transcript in ZK mode. Instead, they are constrained within BlindFold. The verifier's `CheckedInputs` handles the split behavior properly (ensuring standard modes reconstruct the exact same public data expected by the transcript).
- **Prover APIs**: `jolt-prover` explicitly separates `prove_stageN` (clear) and `prove_stageN_zk` (committed). At the end of `prove_zk_stages`, `jolt-prover` generates the BlindFold layout and folds the error rows.

---

## 3. PCS Interfacing & Modularity

The prover interacts with the Polynomial Commitment Scheme (PCS) strictly through traits defined in `jolt-openings`. Concrete implementations (like `DoryScheme` from `jolt-dory`) are treated as implementation details.

### How We Interface
`jolt-prover` depends on traits such as:
- `CommitmentScheme`
- `StreamingCommitment`
- `AdditivelyHomomorphic`
- `ZkOpeningScheme`

**Stage 0 (Commitments)**: The prover maps `jolt-witness` streams into `CommitmentRequest`s. The backend processes these streams (potentially using coarse-grained, optimized CPU streams or GPU acceleration) and returns `CommittedPolynomialOutput` containing `PCS::Output` and `PCS::OpeningHint`.

**Stage 8 (Final Opening)**: The prover constructs a structured, typed `OpeningRequest` (containing `RamInc`, `RdInc`, `Ra`s, etc.). The PCS traits (`PCS::combine`, `PCS::open_poly`, `PCS::open_zk_poly`) natively handle combining the commitments using the transcript-derived Gamma powers. `jolt-prover` retains the hints and evaluations without needing to peek into the inner struct of the Dory proof.

### Ease of Replacing PCS
Because the entire backend math is isolated behind generic traits in `jolt-openings` and `jolt-backends::CommitmentBackend`, replacing Dory with another PCS (e.g., HyperKZG, Brakedown) is highly modular:
1. Implement the `CommitmentScheme` and `ZkOpeningScheme` traits for the new scheme.
2. Provide the type bound (`PCS = NewScheme`) when invoking `jolt_prover::prove`.
3. The prover will naturally route `NewScheme::Output` and `NewScheme::OpeningHint` throughout the stages. No manual transcript adjustment or stage rewriting is required as `jolt-prover` solely orchestrates the bounds.

---

## 4. Transcript: Spongefish Integration

Jolt transcripts use a `state || round_counter` domain separation paradigm to securely transition interactive protocols into non-interactive Fiat-Shamir proofs.

Currently, the `jolt-transcript` crate natively supports:
- `Blake2bTranscript` (Default)
- `KeccakTranscript` (EVM compatible)
- `PoseidonTranscript` (BN254 specific, for SNARK wrapping)

**Spongefish Integration**:
The `spongefish` dependency (recently bumped to `0.7.0` on the `main` branch) represents a lean, secure cryptographic sponge function tailored for succinct proof transcripts. As `jolt-prover` is merged back into `main`, the `spongefish` transcript will implement the `jolt_transcript::Transcript` trait (i.e. `append_bytes`, `challenge`, `challenge_vector`).

Because `jolt-prover` operates entirely over the `T: Transcript` generic bound, the upgrade is completely seamless for the prover. The prover code (e.g., `absorb_stage0_transcript`) simply calls `transcript.append(&Label(...))` or `append_payload_label`. It remains agnostic to the underlying hash permutation (Blake2b vs Spongefish).

---

## 5. Prover and Verifier Stages (Walkthrough)

The Jolt protocol is broken into sequential sumcheck stages (1 through 8). In the modular stack, every stage is a distinct function (e.g., `prove_stageN` and `verify_stageN`) acting over purely typed inputs and outputs.

### Stages 1 to 7: The Sumcheck Pipeline
- **Stage 1 (Spartan Outer)**: Evaluates R1CS constraints over the execution trace.
- **Stage 2 (Batch Sumcheck)**: Processes RAM/Register Read-Write checking, Instruction Lookups, and Product Uniskips.
- **Stage 3 to 7 (Claim Reductions)**: Sequentially reduce the complex polynomial relationships (Hamming weights, booleanity, bytecode evaluations, advice mappings) down to single multi-linear evaluations over the trace domain.

**Prover flow for a given Stage N:**
1. The prover takes `StageNProverInput` (derived from previous stages).
2. Uses `jolt-witness` to lazily construct polynomial streams.
3. Issues a `SumcheckBackendRequest` to `jolt-backends::cpu`.
4. Receives back a math-only result.
5. `jolt-prover` builds the verifier-owned protocol data (`StageNOutput`) and private state data.
6. The `StageNOutput` is committed to the transcript.

### Stage 8: The Grand RLC & PCS Evaluation
Stage 8 brings the entire system together. It maps all the reduced evaluation claims into a final `OpeningRequest`.
1. `jolt-prover/src/stages/stage8/prove.rs` calls `derive_stage8_structure_and_gamma`. This determines the strict ordering of claims (e.g. `RamInc`, `RdInc`, `Ra`, `Advice`) expected by the verifier.
2. It hashes the claims to squeeze `Gamma` challenges.
3. It creates a joint evaluation polynomial (RLC) by combining witness rows multiplied by `Gamma`.
4. It calls `PCS::open_poly` (or `open_zk_poly`) with the combined hints generated in Stage 0.
5. `jolt-verifier/src/stages/stage8/verify.rs` receives the combined commitment, derives the same `Gamma`s, constructs the combined verification commitment, and runs `PCS::verify`.

This precise alignment guarantees that the prover cannot inject un-modeled polynomials, as the verifier strictly demands the exact layout (`require_commitment_layout`).
