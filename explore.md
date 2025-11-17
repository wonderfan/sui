# Sui — Crate Exploration

This document provides a focused, consolidated exploration of the main data structures, logic flows, and architectures for several core Sui crates. It is intended as a developer-oriented map to help you navigate the implementation in `crates/`.

The `crates/` folder is the heart of the Rust implementation, with subdirectories representing individual crates. Each typically includes a `Cargo.toml` for dependencies, `src/` for source code, and sometimes `benches/` or `tests/`.

Contents
- `crates/sui-types` — canonical runtime types and encodings
- `crates/sui-config` — runtime configuration and operational settings
- `crates/sui-protocol-config` — protocol-level parameters and governance-sensitive configs
- `crates/sui-core` — execution engine, validation, and transaction lifecycle
- `crates/sui-node` — node process wiring: networking, consensus, and services
- `crates/sui-storage` — persistent storage layout, indices, and access patterns
- `crates/sui-network` — transport, RPC, gossip and peer management

---

## `crates/sui-types`

**Purpose**
- **Canonical types & encodings:** `sui-types` defines the canonical Rust types, binary encodings, and small helper primitives used across the Sui node, execution, storage, and client stacks. It focuses on on-chain object representations, transaction envelopes and digests, signatures, checkpoints, events, and light utilities required for deterministic execution and wire compatibility.

**High-level summary**
- This crate intentionally contains definitions (structs, enums, traits, constants) and utilities rather than heavy business logic. It is a direct dependency of most runtime crates (`sui-core`, `sui-storage`, `sui-node`, `sui-rpc`, etc.).
- Serialization: types are encoded for canonical, deterministic on-disk and on-wire formats using BCS and Move-related encodings where appropriate. The crate interoperates with `move-core-types`, `move-binary-format`, and `move-bytecode-utils` for Move value/type interop.
- Cryptography: signatures, multisig primitives, and address/key encodings live here (and the crate depends on `fastcrypto`/`shared-crypto` workspace crates).

**What’s in the crate**
- `base_types` — low-level identifiers & address types (`ObjectID`, `SuiAddress`, `SequenceNumber`, etc.).
- `object` — canonical on-chain object shape and variants (`Object`, `MoveObject`, `ObjectRead`, owner variants, content/digests).
- `transaction` — transaction envelopes, `TransactionKind`, `TransactionData`, `TransactionDigest`.
- `effects` — `TransactionEffects` and related result types produced by execution.
- `event` — `SuiEvent`, `EventID` and event serialization for indexing.
- `crypto`, `signature`, `signature_verification`, `multisig`, `multisig_legacy` — signatures, authenticator types, verification helpers and legacy compatibility layers.
- `move_package`, `layout_resolver`, `execution`, `execution_status`, `programmable_transaction_builder` — types that help describe Move packages, transaction execution payloads, and PTB construction.
- `full_checkpoint_content`, `messages_checkpoint`, `messages_consensus` — checkpoint and consensus message types.
- `gas`, `gas_coin`, `coin`, `coin_registry`, `balance` — gas & coin-related primitives and registries.
- `sui_serde`, `proto_value`, `rpc_proto_conversions`, `sui_sdk_types_conversions` — serde helpers and conversion layers for RPC/SDK interop.
- `global_state_hash`, `digests` — canonical hashing utilities used when constructing global state/checkpoint hashes.
- `in_memory_storage`, `storage`, `inner_temporary_store` — lightweight types used in tests and local state manipulation.

**Cargo / features**
- Package: `name = "sui-types"`, `version = "0.1.0"`, `edition = "2024"` (see `Cargo.toml`).
- Many dependencies are workspace members (Move crates, `fastcrypto`, `mysten-network`, `mysten-metrics`, etc.).
- Features: `default = []`, `tracing = ["move-vm-profiler/tracing", "move-vm-test-utils/tracing"]`, `fuzzing = ["move-core-types/fuzzing"]`.
- Benchmarks: `global_state_hash_bench` and `nitro_attestation_bench` are present under `[[bench]]`.

**Notable constants & helpers (from `src/lib.rs`)**
- Built-in package addresses and object IDs: constants for Move stdlib, Sui framework, Sui system, Bridge, Deepbook, and several well-known Sui object IDs (e.g., `SUI_SYSTEM_STATE_OBJECT_ID`, `SUI_CLOCK_OBJECT_ID`, etc.).
- Parsing helpers: `parse_sui_address`, `parse_sui_module_id`, `parse_sui_fq_name`, `parse_sui_struct_tag`, `parse_sui_type_tag` — wrappers around `move-core-types` parsers that resolve the crate's named addresses.
- `resolve_address` — maps short names (`std`, `sui`, `sui_system`, `deepbook`, `bridge`) to the canonical `AccountAddress` constants.
- Move-type helpers: `MoveTypeTagTrait` and `MoveTypeTagTraitGeneric` — small traits used to obtain `TypeTag`s for Rust types; utilities `is_primitive`, `is_object`, `is_object_vector` detect object/primitive types from bytecode signatures and ability sets.

**Key types and responsibilities**
- Object model: `Object` (and `MoveObject`) is the canonical persisted unit with fields for id, version/sequence number, digest, owner, and content. Objects model Move values and track previous transaction digests.
- Transactions & effects: `Transaction`, `TransactionData`, `TransactionKind` (including programmable transactions), `TransactionEffects` (status, written objects, gas used, events).
- Signatures & auth: lightweight signer/authenticator types used by RPC and authority layers; support for single signatures, threshold multisig and legacy multisig variants.
- Checkpoints & consensus messages: checkpoint content, accumulator/roots, and message shapes used by the checkpointing and consensus layers.
- Events: `SuiEvent` carries BCS-encoded bytes and parsed metadata for indexing and RPC consumption.

**Serialization & deterministic semantics**
- BCS is used widely for deterministic binary encodings; Move type tags and Move-related encodings are used for VM interop. Canonical hashing/digests are produced from BCS-serialized payloads and used in digests/global state roots.

**Tests & developer utilities**
- The crate contains unit tests (see `src/unit_tests/`) and utility modules used by tests and benches (`utils`, `in_memory_storage`, `test_checkpoint_data_builder`).

**How other crates use it**
- `sui-core`, `sui-storage`, `sui-node`, RPC and indexer crates import these canonical types for execution, persistence and wire formats. Any change to types or serialization must preserve wire/on-disk compatibility or be accompanied by migration code.

---

## `crates/sui-config`

Purpose
- Expose typed, validated configuration for node processes and service components: storage backends, network transports, telemetry, RPC endpoints, and runtime flags.

Major components and fields
- Node identity: keypairs, validator metadata, and network addresses.
- Storage config: DB paths, column-family options, cache sizes, and compaction flags.
- Network config: transport selection (TCP/QUIC), listen addresses, connection limits, TLS/certificates.
- Observability: logging level, Prometheus exporter config, tracing layers.

Typical flow
- At process startup the node loads a config file (TOML/JSON) and applies environment overrides. The typed config is validated and used to instantiate subcomponents (network, storage, executor).

Operational guidance
- Keep secrets (private keys, credentials) out of the repository; prefer environment injection or secret managers.
- Provide reasonable defaults for local dev and more explicit values for production/testnet deployments.

---

## `crates/sui-protocol-config`

Purpose
- Contain protocol-level parameters that affect core invariants: gas schedule, epoch length, quorum sizes, checkpoint frequency, and other economic/consensus knobs.

Key elements
- Gas schedule & cost tables: values used by the execution layer to compute gas usage.
- Consensus/timeout parameters: leader timeouts, round durations, and quorum thresholds.
- Checkpointing config: checkpoint intervals and certificate thresholds.

How changes are applied
- Protocol parameters are governance-sensitive. In production these values are typically changed via governance/on-chain mechanisms or carefully coordinated node updates.

Notes
- Treat this configuration as part of the protocol surface area — changes need coordination across validators and tooling to avoid forks.

---

## `crates/sui-core`

**Purpose**
- Implements the Sui authority runtime: transaction ingestion, validation, scheduling, Move VM execution, and application of write-sets to authoritative state.

**High-level summary**
- `sui-core` contains the core server-side logic an authority node needs to accept, order, execute, and commit transactions. It bridges consensus inputs to state changes by providing Authority state, consensus adapters, transaction orchestration, scheduler/executor, checkpointing, and storage wiring.
- The crate is intentionally modular: distinct components handle consensus integration, scheduling, execution caching, verification, and checkpointing so other crates (notably `sui-node`) can compose them.

**What’s in the crate**
- `authority`, `authority_aggregator`, `authority_client`, `authority_server` — Authority state and RPC/validator-facing server pieces.
- `consensus_adapter`, `consensus_handler`, `consensus_manager`, `consensus_validator` — Adapters and glue to the consensus layer.
- `execution_scheduler`, `execution_cache`, `execution_driver`, `transaction_orchestrator`, `transaction_driver` — Scheduling and execution orchestration for parallel, deterministic execution.
- `checkpoints`, `db_checkpoint_handler`, `mock_checkpoint_builder` — Checkpoint construction, verification, and helpers for testing.
- `quorum_driver` — High-level API to submit transactions and wait for finality.
- `storage`, `rpc_index`, `jsonrpc_index`, `global_state_hasher` — Storage integration, indices and global-state hashing used by checkpointing.
- `signature_verifier`, `module_cache_metrics`, `metrics` — Verification and observability helpers.
- `runtime`, `validator_tx_finalizer`, `authority_aggregator` — Runtime glue to finalize transactions and manage per-epoch authority components.

**Cargo / features**
- Package: `name = "sui-core"`, `version = "0.1.0"`, `edition = "2024"` (see `crates/sui-core/Cargo.toml`).
- Heavy workspace dependency usage: Move toolchain crates, `fastcrypto`, `mysten-network`, `typed-store`, `tokio`, `tracing`, and many internal Sui crates.
- Benchmarks and examples: `verified_cert_cache_bench`, `batch_verification_bench`, and `generate-format` example are declared.

**Notable helpers & structure (from `src/lib.rs`)**
- The crate exposes a wide set of modules for authority lifecycle management and testing utilities under `src/` (see the module list in `lib.rs`).
- Unit test harnesses live under `src/unit_tests/` and many test helpers (e.g., `mock_checkpoint_builder`, `mock_consensus`) are included for integration tests.

**Key types and responsibilities**
- Authority state: `AuthorityState` and `AuthorityStore` encapsulate persistent state, object application, and per-epoch metadata.
- Transaction flow: `TransactionOrchestrator`, `TransactionDriver`, and `ExecutionScheduler` are responsible for ordering, scheduling non-conflicting transactions, driving execution, and collecting `TransactionEffects`.
- Consensus integration: `ConsensusAdapter` and `ConsensusHandler` accept ordered inputs, adapt them for local execution, and connect checkpoint submission to consensus.
- Checkpointing: modules build and verify checkpoints, compute global state hashes (`global_state_hasher`), and coordinate checkpoint submission.

**Tests & developer utilities**
- Extensive unit and integration test helpers under `src/unit_tests`, plus mock components to run local consensus/authority tests.
- Example and bench targets help profile verification and batching behavior.

**How other crates use it**
- `sui-node` composes `sui-core` to run an authority process (consensus client, executor, checkpoint services). `sui-storage` is used by `sui-core` for persistence, and `sui-json-rpc` and indexers read the artifacts produced by `sui-core`.

---

## `crates/sui-node`

**Purpose**
- Process wiring and orchestration for a full Sui node: configuration, networking, consensus client, authority runtime, RPC servers, background tasks and observability.

**High-level summary**
- `sui-node` is the binary-level composition layer that instantiates `sui-core` components, network services (`sui-network`), JSON-RPC/gRPC endpoints, checkpointing, and housekeeping (DB checkpoints, admin APIs, telemetry). It focuses on lifecycle management and operational concerns rather than execution logic.

**What’s in the crate**
- `lib.rs` and `main.rs` — node entrypoint, `SuiNode` struct and `start` lifecycle functions.
- `admin` — admin & debugging endpoints.
- `handle` — `SuiNodeHandle` and programmatic control surfaces.
- `metrics` — node-level metrics registration and collection.
- HTTP & RPC wiring (JSON-RPC, gRPC) plus integration with `sui-core`'s authority components.

**Cargo / features**
- Package: `name = "sui-node"`, `version.workspace = true`, `edition = "2024"` (see `crates/sui-node/Cargo.toml`).
- Depends on `anemo`, `axum`, `tokio`, `sui-core`, `sui-network`, `sui-storage`, `sui-config`, `sui-json-rpc` and many telemetry/metrics crates.
- Feature flags: `jemalloc` is enabled by default in the crate features (can be toggled).

**Notable constants & helpers (from `src/lib.rs`)**
- `DEFAULT_GRPC_CONNECT_TIMEOUT` — default connection timeout used by gRPC clients.
- `SuiNode` struct encapsulates validator components and P2P components, and exposes `start` helper to bootstrap the node with a `NodeConfig`.
- The node wires: consensus adapter, checkpoint executor, randomness, discovery, state-sync, and RPC servers.

**Key responsibilities**
- Lifecycle: initialize DBs, load genesis/state, start consensus client (or connect to consensus), start RPC servers, and run background tasks (checkpoint submission, state-sync).
- Orchestration: construct `AuthorityAggregator`, `ConsensusAdapter`, `CheckpointStore`, and service components; handle graceful shutdown and epoch transitions.
- Observability: configure metrics, tracing, admin endpoints and runtime diagnostics.

**Tests & developer utilities**
- `main.rs` provides the normal binary entry; `lib.rs` exposes an API to programmatically start/stop nodes for integration tests and simulator environments.
- Simulation-specific wiring (cfg(msim)) enables testing under the Sui simulator with mock JWK injectors and other test hooks.

**How other crates use it**
- Operators and integration tests run `sui-node` to start a full node process. `sui-node` composes `sui-core`, `sui-network`, `sui-storage` and `sui-json-rpc` to present a single runnable artifact.

---

## `crates/sui-storage`

**Purpose**
- Durable storage primitives and helpers used by the node and indexers: file/blob helpers, object/package stores, key-value interfaces, and checkpoint artifact management.

**High-level summary**
- `sui-storage` provides both low-level file/blob utilities and higher-level key-value/object store abstractions used to persist checkpoints, packages, objects, and transaction-related artifacts. It includes helpers for checksums, compression, and streaming large checkpoint blobs.

**What’s in the crate**
- `blob` — file/blob iterator and helpers for reading archived checkpoint blobs.
- `object_store`, `package_object_cache` — object-level caches and on-disk object helpers.
- `key_value_store`, `http_key_value_store` — pluggable key-value store interfaces and HTTP-backed store adapters.
- `write_path_pending_tx_log`, `mutex_table`, `sharded_lru` — write-path and caching primitives.

**Cargo / features**
- Package: `name = "sui-storage"`, `version = "0.1.0"`, `edition = "2024"` (see `crates/sui-storage/Cargo.toml`).
- Depends on `object_store`, `typed-store`, `bcs`, `zstd`, `tokio`, and `sui-types` among others.

**Notable constants & helpers (from `src/lib.rs`)**
- `SHA3_BYTES` — constant for SHA3-256 digest size (32 bytes).
- Checksum helpers: `compute_sha3_checksum`, `compute_sha3_checksum_for_bytes`, `compute_sha3_checksum_for_file`.
- Compression helpers: `FileCompression` enum with `zstd` compress/decompress utilities and `compress`/`decompress` helper functions; generic `compress`/`read` utilities for blob formats.
- Checkpoint verification helpers: `verify_checkpoint`, `verify_checkpoint_with_committee`, and `verify_checkpoint_range` which validate checkpoint summaries against committee signatures.

**Key responsibilities**
- Persisting large artifacts: efficient read/write streaming for checkpoint blobs and package artifacts.
- Object & package persistence: key-value interfaces and object caches used by higher-level services.
- Data integrity: checksum and compression utilities to ensure on-disk integrity and to support archive formats.

**Tests & developer utilities**
- Includes test helpers and dev features for in-memory stores and simulation testing; dev-dependencies include `sui-test-transaction-builder` and `sui-simulator`.

**How other crates use it**
- `sui-core` and `sui-node` use `sui-storage` for persistent storage of objects, transactions, and checkpoint artifacts. Indexers and archival tooling also consume the blob and key-value helpers here.

---

## `crates/sui-network`

**Purpose**
- Networking primitives and transport glue used by Sui nodes and services for P2P RPC, discovery, state-sync, randomness distribution, and validator APIs.

**High-level summary**
- `sui-network` wraps lower-level transport libraries (notably `anemo` / `mysten-network`) and provides Sui-specific protocols: discovery, validator server/client APIs, state sync, randomness distribution, and helper defaults for network tuning.

**What’s in the crate**
- `api` — network API definitions and request/response types.
- `discovery` — peer discovery and trusted-peer management.
- `randomness` — randomness distribution utilities and shims used by consensus/epoch flows.
- `state_sync` — state-sync protocol implementations for fetching checkpoint and object artifacts.
- `validator` — validator-specific server/client bindings and the `ServerBuilder` convenience.

**Cargo / features**
- Package: `name = "sui-network"`, `edition = "2024"` (see `crates/sui-network/Cargo.toml`).
- Depends on `anemo`, `mysten-network`, `tonic`, `tokio`, `shared-crypto`, `sui-types`, and `sui-storage`.
- Build dependencies include `anemo-build` and `tonic-build` for protocol codegen.

**Notable constants & helpers (from `src/lib.rs`)**
- `DEFAULT_CONNECT_TIMEOUT_SEC`, `DEFAULT_REQUEST_TIMEOUT_SEC`, `DEFAULT_HTTP2_KEEPALIVE_SEC` — network timing defaults.
- `default_mysten_network_config()` — returns a `mysten_network::config::Config` instance pre-populated with sensible defaults tuned by the crate.

**Key responsibilities (practical view)**
- Transport & RPC: provide robust, authenticated RPC and streaming transports for validator-to-validator and client-to-validator communication.
- Discovery & state sync: maintain trusted-peer lists and provide the state-sync protocol to fetch checkpoint artifacts and object ranges.
- Validator APIs: provide server implementations (`validator::server::ServerBuilder`) used by `sui-node` to expose validator endpoints.

**Tests & developer utilities**
- Includes test harnesses and dev-dependencies for simulated networking and protocol tests; `build.rs` helps codegen for gRPC/Prost where needed.

**How other crates use it**
- `sui-node` depends on `sui-network` to provide P2P networking, discovery, state-sync and validator RPC servers; `sui-core` and the consensus adapters interact through the network APIs exposed here.

---

## Design Principles & Architecture Patterns

This project follows a set of consistent design principles and architectural patterns across `crates/sui-types`, `crates/sui-core`, `crates/sui-node`, `crates/sui-storage`, and `crates/sui-network`. The list below distills those cross-cutting choices so you can reason about trade-offs, extend the system, and maintain compatibility.

- **Modularity & strong crate boundaries:**
	- **Single responsibility per crate:** each crate focuses on a narrow layer (types, runtime/authority, node wiring, storage, networking). This keeps APIs small and reviewable and enables parallel development.
	- **Public API surface minimization:** crates export a compact set of types and helpers; internal modules and test helpers remain private.

- **Composition over duplication:**
	- Reuse of small, well-defined crates (e.g., `sui-types` for canonical types, `sui-storage` for persistence) avoids duplication and centralizes compatibility guarantees.
	- Higher-level components (like `sui-node`) compose lower-level pieces rather than bake logic into a single binary.

- **Deterministic execution & canonical serialization:**
	- Canonical binary formats (BCS) and well-defined hashing/digest functions are used everywhere a cross-node agreement or on-disk reproducibility matters.
	- Move interop respects `move-core-types` encodings and `move-binary-format` semantics so VM behavior is consistent across environments.

- **Object-centric concurrency model:**
	- The runtime centers on objects with IDs, versions, owners, and digests. Scheduling, locking, and parallel execution are object-scoped to maximize concurrency while preserving deterministic outcomes.

- **Authority vs consensus separation (adapter pattern):**
	- `sui-core` exposes authority logic and a `ConsensusAdapter` layer that translates consensus-ordered inputs into local actions. This separation enables the same runtime to be driven by different consensus implementations or test harnesses.

- **Checkpointing & global-state roots:**
	- Checkpoints are first-class: code to assemble, hash, sign, and verify checkpoint artifacts lives in `sui-core` and `sui-types`. Storage and networking crates handle efficient blob transfer and verification.
	- Global-state hashing utilities and accumulator roots provide compact, auditable snapshots used by state-sync and verification.

- **Pluggable persistence & typed stores:**
	- Storage abstractions (key-value, object store, blob formats) are pluggable and wrapped with typed-store helpers where possible to get compile-time schema checks and metrics.
	- Storage focuses on both fast point-reads (RPC/indexers) and efficient batched writes (checkpoints/epoch writes).

- **Network design: protocol layering and defaults:**
	- `sui-network` wraps a lower-level transport stack (`anemo`/`mysten-network`) and provides Sui-specific protocols (discovery, validator RPCs, state sync).
	- Sensible network defaults (timeouts, keepalive, request timeouts) are centralized and reused to ensure consistent behaviors across services.

- **Security & cryptography-first design:**
	- Authentication, signature verification, replay protection, and multisig/legacy-sig compatibility are core concerns implemented in `sui-types` and exercised by `sui-core` and `sui-network`.
	- Secrets and keys are treated as operational concerns (in `sui-config`), and node code avoids embedding private secrets in repos.

- **Observability & operational readiness:**
	- All crates expose metrics, structured tracing, and health/admin surfaces. `sui-node` wires Prometheus metrics and tracing layers by default.
	- Benchmarks, examples, and diagnostic endpoints are included to reason about performance and operational issues.

- **Backpressure, throttling & resource safety:**
	- The runtime includes mechanisms for backpressure (queue monitors, submission throttles), execution caches and limits, and explicit guards to avoid resource exhaustion under load.

- **Test-first design and rich test harnesses:**
	- Mock components (mock consensus, mock checkpoint builders), unit tests under `unit_tests/`, and simulator-specific wiring (`cfg(msim)`) make it straightforward to exercise distributed behaviors locally.
	- Deterministic test utilities (in-memory stores, injected randomness hooks) speed up unit and integration testing.

- **Compatibility, versioning & migration patterns:**
	- Types and wire formats are treated as long-lived compatibility surfaces; changes must be accompanied by migration helpers or version-gating via protocol config.
	- Support for legacy encodings and `*_legacy` modules allow gradual rollouts and backwards compatibility.

- **Developer ergonomics via workspace & codegen:**
	- The monorepo workspace centralizes dependencies and versions to avoid rippling upgrades across crates.
	- Generated code (gRPC/prost, move bindings) is produced via crate-local build scripts to keep source-of-truth IDLs near their consumers.

- **Error handling & clarity of failure modes:**
	- Crates use typed error enums (with `thiserror`) where callers need to react to specific failures, and use `anyhow`/`eyre` in higher-level orchestration for ergonomic propagation of errors.

- **Incremental deployability & safety gates:**
	- Epoch-based configuration, `SupportedProtocolVersions`, and on-chain governance-sensitive knobs are used to coordinate upgrades across validators and clients.
