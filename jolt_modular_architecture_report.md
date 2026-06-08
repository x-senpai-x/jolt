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

## 3. PCS Interfacing & Modularity: An In-Depth Analysis

The new Jolt architecture achieves cryptographic modularity by completely abstracting the Polynomial Commitment Scheme (PCS) behind a hierarchy of traits found in the `jolt-openings` crate. The core goal is that `jolt-prover` orchestrates the *protocol schedule* (which polynomial to commit to, which to open, and when), while the PCS itself defines the *algebraic implementation* (how to compress vectors, how to stream, how to achieve zero-knowledge).

This strict separation means replacing the PCS (e.g., swapping Dory for HyperKZG or Brakedown) requires zero changes to the complex Jolt sumcheck protocol paths.

### The Trait Hierarchy (`jolt-openings`)

To support everything from hardware acceleration (via `jolt-backends`) to Zero-Knowledge, the PCS abstractions are divided into highly specialized traits. Here is the in-depth breakdown of every trait involved, its usage, and its necessity:

#### 1. `CommitmentScheme`
**Necessity:** This is the foundational trait. It defines the base parameters of the commitment scheme (the Field, the Proof type, the Prover/Verifier Setup) and the required mathematical operations (commit, open, verify).
**Usage:**
- **Types:** It defines associated types such as `Output` (the commitment itself) and `OpeningHint`. The `OpeningHint` is crucial for performance: when a prover commits to a polynomial, the PCS can return auxiliary data (like intermediate tree nodes or row-commitments in Dory) that are saved and passed directly to the `open` function later, preventing redundant computation.
- **Methods:** `commit()` transforms a polynomial into an `Output` and `OpeningHint`. `open()` evaluates the polynomial at a challenge point `r`, producing a `Proof`. `verify()` is used by the verifier to check the proof against the evaluation and `Output`.

#### 2. `AdditivelyHomomorphic`
**Necessity:** Jolt's final Stage 8 opening is heavily optimized. Instead of opening dozens of polynomials (RAM, Registers, Advice) individually, the protocol groups them together using a Random Linear Combination (RLC). This trait guarantees the PCS can algebraically combine commitments without reconstructing the source polynomials.
**Usage:**
- **Methods:** Provides `combine(commitments, scalars)` and `combine_hints()`.
- In `jolt-prover` Stage 8, once the Fiat-Shamir `Gamma` challenges are squeezed, the prover calls `PCS::combine` to fold `N` commitments into a single `JointCommitment`. It simultaneously folds the `OpeningHints` to build the joint opening proof instantly.

#### 3. `StreamingCommitment`
**Necessity:** Jolt traces can be gigabytes in size. Materializing a massive, dense vector in memory simply to commit to it would cause Out-Of-Memory (OOM) panics. `StreamingCommitment` allows the backend to feed data to the PCS in chunks, keeping the memory footprint minimal.
**Usage:**
- **Methods:** `begin()`, `feed()`, `feed_zeros()`, `feed_u64()`, `process_one_hot_chunk()`, and `finish()`.
- **Integration:** The `jolt-backends` (like the CPU backend) uses this heavily in Stage 0. The witness provider yields a stream of elements (dense, sparse zeros, or one-hot index chunks). The CPU backend feeds these sequentially into a `PartialCommitment` state machine. For sparse matrices (like RA tables), `process_one_hot_chunk` bypasses dense memory materialization entirely by committing only to the activated indices.

#### 4. `ZkOpeningScheme`
**Necessity:** Standard PCS openings reveal the underlying evaluation $f(r)$. In Zero-Knowledge mode (BlindFold), the prover must convince the verifier that the joint polynomial evaluates correctly without exposing the actual evaluation value.
**Usage:**
- **Types:** Introduces `HidingCommitment` (typically a Pedersen commitment, e.g., `Bn254G1`) and `Blind` (the randomness scalar).
- **Methods:** `commit_zk()` overrides the standard commit with hiding parameters. `open_zk()` creates an opening proof that is bound to the `HidingCommitment` rather than the cleartext field evaluation.
- `jolt-prover` dynamically switches to these methods if `cfg(feature = "zk")` is enabled, natively integrating the PCS's ZK capability with BlindFold's R1CS logic without knowing how the PCS achieves ZK natively.

#### 5. `ZkStreamingCommitment`
**Necessity:** An intersection trait to ensure that even when ZK mode is enabled, the prover does not lose the OOM-preventing streaming capabilities.
**Usage:**
- **Methods:** `finish_zk_with_hint()`, `finish_zk_one_hot_column_major_chunks()`.
- Allows `jolt-backends` to finalize a streamed trace chunk while still properly injecting the blinding factors necessary for `open_zk()` later.

### Backend Orchestration (`jolt-backends` and `jolt-dory`)

**The Backend Abstraction:**
`jolt-prover` does not call `PCS::commit` directly. Instead, `jolt-prover` constructs a `CommitmentRequest` representing the necessary protocol view (e.g. "I need the RamInc polynomial stream"). It passes this request to the generic `CommitmentBackend` trait. The concrete backend (e.g. `jolt_backends::cpu::CpuBackend`) then reads the `jolt-witness` streams, optimizes the chunking, and calls `PCS::feed` on the `StreamingCommitment`.

**The Implementation (`jolt-dory`):**
If you look inside `crates/jolt-dory/src`, the `DoryScheme` implements all the aforementioned traits:
- In `scheme.rs`, it maps `CommitmentScheme` types to `DoryProof`, `DoryProverSetup`, and `DoryHint`.
- It implements `AdditivelyHomomorphic` by performing elliptic curve group additions (using `rayon` for parallelization).
- In `streaming.rs`, it implements `StreamingCommitment` where `PartialCommitment` holds a vector of row-commitments that are aggregated hierarchically on `finish()`.

### Summary of Ease of Replacement
Replacing the PCS is fully decoupled from Jolt's intricate trace processing and sumchecks:
1. **Develop a new crate** (e.g., `jolt-hyperkzg`).
2. **Implement the 5 traits** from `jolt-openings` on your target struct (e.g., `HyperKzgScheme`).
3. **Inject via Generics**: Start the prover using `jolt_prover::prove::<HyperKzgScheme, ...>`.
4. `jolt-prover` routes the math, `jolt-backends` manages the streaming limits, and `jolt-verifier` expects the new `HyperKzgScheme::Proof` in the payload. No sumcheck or transcript code requires modification.

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
