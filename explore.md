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

Purpose
- Implement the runtime semantics: transaction validation, scheduling, Move VM execution, and application of write-sets to authoritative state.

Major subsystems
- Executor / VM integration: load packages and execute transaction payloads in a deterministic configuration of the Move VM; return effects (write-sets, events), gas usage, and VM status.
- Authority / State manager: apply committed write-sets to the object state, update sequence numbers/versions, and produce new object digests.
- Scheduler: arrange transactions for execution, enforce object-level dependency ordering, and coordinate parallelism while preventing conflicting writes.
- Gas and Fee logic: compute gas, validate payer balance, and handle fee distribution bookkeeping.

Typical transaction lifecycle
1. Ingest: transaction arrives (RPC or from consensus). Parse into typed `Transaction` and validate signatures.
2. Preliminary checks: format, nonce/sequence, basic gas/balance checks, and object existence checks.
3. Scheduling: place transaction into a scheduler that groups non-conflicting transactions for parallel execution.
4. Execution: the Move VM executes transaction; result is a write-set and events.
5. Commit: state manager applies write-set atomically to storage and advances object versions.

Integration points
- `sui-node`/consensus provides ordered inputs (certificates or batches) and `sui-core` applies them.
- `sui-storage` persists the results; indexers and RPC surfaces read the stored state/events.

Determinism
- Execution must be deterministic across validators; canonical encodings and consistent VM configuration are critical.

---

## `crates/sui-node`

Purpose
- Wire together the complete node: configuration, storage, network, consensus client, execution/authority, and operational tooling (metrics, health, admin).

Primary responsibilities
- Process initialization: load config, keys, DBs, and initialize subcomponents.
- Service orchestration: start network listeners, RPC servers, consensus participation, and background tasks.
- Lifecycle management: graceful shutdown, state snapshotting, and recovery from checkpoints.

Common components
- Networking stack from `sui-network` for peer communication.
- Consensus client or adaptor that talks to the consensus crate/implementation.
- Local instance of `sui-core` to validate and apply transactions when acting as an authority.

Startup and recovery
- Nodes typically resume from the latest persisted checkpoint and re-sync with peers if lagging. The node must ensure idempotent application of committed write-sets.

Operational concerns
- Observability: logs, metrics, and trace correlation are configured at startup; admin endpoints allow live health checks and debugging.
- Resource management: ensure DB caches, thread pools, and network limits are tuned for the deployment.

---

## `crates/sui-storage`

Purpose
- Provide durable, high-performance storage for objects, transactions, events, checkpoints, and indices needed by RPC/indexing surfaces.

Core components
- Object Store: maps `(ObjectID, Version)` to an `Object` payload (Move state, metadata, owner).
- Transaction / Effects Store: stores transactions and their execution effects, events, and gas usage.
- Checkpoint Store: stores checkpoint metadata, certificates, and associated state digests.

Storage layout
- Typically backed by an LSM-store (RocksDB) using column families to separate objects, indices, and metadata for efficient targeted reads and compactions.

Access patterns
- Read-heavy workloads (RPC, indexers): requires fast point-lookup and range scan semantics.
- Write bursts: applying an epoch or checkpoint can cause large batches of writes; batching and WAL ensure durability.

Maintenance
- Compaction and GC policies are important for long-term storage costs; checkpoint archiving is used to keep recent data fast while cold data can be archived.

---

## `crates/sui-network`

Purpose
- Provide networking primitives (transport, RPC handlers, gossip) used by nodes and services for transaction propagation, state sync, and consensus messaging.

Major responsibilities
- Transport layer: connection lifecycle, multiplexing, TLS/authentication, and optional QUIC/TCP abstraction.
- RPC endpoints: request/response handlers used by peers and clients for transaction submission, object queries, and state sync.
- Gossip/PubSub: broadcast mechanisms for propagating transactions and other ephemeral messages.

Common flows
- Transaction propagation: clients submit to a node → node gossips to validators and peers using pubsub/gossip protocols.
- State sync: lagging node requests checkpoint artifacts or object ranges from peers and replays to catch up.

Operational concerns
- Security: authenticated channels and replay protection; rate limiting and backpressure protect against abuse.
- Performance: batching, prioritized message queues, and connection pooling are used to maximize throughput.

---

## Cross-cutting notes & how to use this file

- When exploring implementations, start with `crates/sui-types` to understand canonical formats used by other crates.
- Follow runtime wiring from `crates/sui-node` to see how configuration, networking, consensus, and `sui-core` are composed.
- For storage designs and troubleshooting, examine `crates/sui-storage` and DB column-family layouts.
