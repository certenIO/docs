# Validator Architecture

The Independent Validator is a Go application that integrates with CometBFT for BFT consensus and manages the complete proof generation pipeline from Accumulate state queries through on-chain anchoring.

## High-Level Architecture

```
+-----------------------------------------------------------------------+
|                        Independent Validator                           |
+-----------------------------------------------------------------------+
|                                                                        |
|  +------------------+     +-------------------+     +----------------+ |
|  |   CometBFT ABCI  |     |   Proof Engine    |     |   HTTP Server  | |
|  |   (consensus)     |---->|   (9-phase cycle)  |     |   (API)        | |
|  +------------------+     +-------------------+     +----------------+ |
|         |                        |                         |           |
|         v                        v                         v           |
|  +------------------+     +-------------------+     +----------------+ |
|  |   Block Handler  |     |   Chain Strategy  |     |   Metrics      | |
|  |   (propose,      |     |   (13 chains)     |     |   (Prometheus) | |
|  |    prevote,       |     +-------------------+     +----------------+ |
|  |    precommit,     |            |                                     |
|  |    commit)        |            v                                     |
|  +------------------+     +-------------------+                        |
|         |                 |   Anchor Manager  |                        |
|         v                 |   (on-chain tx)   |                        |
|  +------------------+     +-------------------+                        |
|  |   State Manager  |            |                                     |
|  |   (ABCI state)   |            v                                     |
|  +------------------+     +-------------------+                        |
|         |                 |   Smart Contracts |                        |
|         v                 |   (CertenAnchorV3)|                        |
|  +------------------+     +-------------------+                        |
|  |   PostgreSQL     |                                                  |
|  |   (proof store)  |                                                  |
|  +------------------+                                                  |
|         |                                                              |
|         v                                                              |
|  +------------------+     +-------------------+                        |
|  |   Accumulate API |     |   Firestore       |                        |
|  |   (state queries)|     |   (state sync)    |                        |
|  +------------------+     +-------------------+                        |
+-----------------------------------------------------------------------+
```

## Package Deep Dive

### `pkg/config/`

Loads configuration from environment variables. Defines the `Config` struct with 80+ parameters organized by category: database, Accumulate API, CometBFT, proof cycle, chain RPCs, contract addresses, and feature flags.

### `pkg/database/`

PostgreSQL database layer using `database/sql` with the `pgx` driver.

**Migrations** (`pkg/database/migrations/`): 7 migration files that create and evolve the schema:

| Migration | Purpose |
|-----------|---------|
| 001 | Create `batches` table (proof batch metadata) |
| 002 | Create `proofs` table (individual phase proofs) |
| 003 | Create `anchors` table (on-chain anchor records) |
| 004 | Create `attestations` table (validator attestations) |
| 005 | Create `intents` table (cross-chain intent tracking) |
| 006 | Add indexes for batch lookups, chain filtering |
| 007 | Add `status` and `error` columns for failed proofs |

**Repositories**: Each table has a corresponding repository with CRUD operations and query methods. A `RepositoryFactory` creates all repositories from a shared database connection.

### `pkg/consensus/`

CometBFT ABCI (Application BlockChain Interface) integration. Implements the four core ABCI methods:

- **`CheckTx`**: Validates incoming transactions before mempool inclusion
- **`PrepareProposal`**: Block proposer selects which proof operations to include
- **`ProcessProposal`**: Non-proposer validators verify the proposed block
- **`FinalizeBlock`**: All validators execute the block and update state

The consensus package manages the state machine transitions that drive the proof cycle.

### `pkg/proof/`

Orchestrates the 9-phase proof cycle. The `ProofEngine` coordinates phase execution:

1. Queries Accumulate API for account state (L1)
2. Fetches BPT roots from partition data (L2)
3. Retrieves root anchors from DN (L3-L4)
4. Queries governance state for key pages, books, authority (G0-G2)
5. Collects BLS signatures from all validators
6. Generates ZK proof via gnark (BLS phase)
7. Constructs and submits anchor transactions (Anchor phase)

See [Consensus & Proofs](consensus-and-proofs.md) for detailed phase documentation.

### `pkg/anchor/`

Manages on-chain anchoring to destination blockchains. The `AnchorManager` coordinates with chain strategies to submit anchor transactions.

Key responsibilities:
- Construct anchor transaction data (Merkle root, ZK proof, public inputs)
- Submit transactions via chain-specific strategies
- Track transaction confirmation and finality
- Handle retries for failed anchoring attempts
- Record anchor results in the database

### `pkg/batch/`

Proof batch lifecycle management. Supports three scheduling modes:

- **`on_demand`**: Generate proofs only when new intents are detected
- **`on_cadence`**: Generate proofs at fixed intervals regardless of new intents
- **`hybrid`**: Combine both approaches (default) - generate on cadence, but also trigger immediately for high-priority intents

### `pkg/execution/`

Intent detection and execution. Scans Accumulate data accounts for new intent entries, validates governance requirements (signature thresholds), and queues intents for proof generation.

### `pkg/verification/`

Proof verification logic used both internally (self-verification after generation) and externally (verifying proofs from other validators during consensus).

### `pkg/attestation/`

Handles validator attestations - signed statements from validators confirming they have verified a proof bundle. Attestations are collected during consensus and stored in the database.

### `pkg/intent/`

Parses the 4-blob intent data structure from Accumulate data entries. Validates intent format, extracts cross-chain parameters, and checks replay protection fields (nonce, expiry).

### `pkg/crypto/`

Cryptographic operations:
- **BLS12-381**: Key generation, signing, signature aggregation
- **Ed25519**: Transaction signing for Accumulate interactions
- **gnark**: Groth16 circuit definition for BLS aggregate signature verification
- **Merkle**: Hash functions for Merkle tree construction

### `pkg/chain/` and `pkg/chain/strategy/`

Chain abstraction layer. Defines a `ChainStrategy` interface that all chain implementations must satisfy:

```go
type ChainStrategy interface {
    Name() string
    ChainID() uint64
    SubmitAnchor(ctx context.Context, data AnchorData) (TxHash, error)
    DeployAccount(ctx context.Context, adi string, salt []byte) (Address, error)
    VerifyProof(ctx context.Context, root []byte, proof []byte, leaf []byte) (bool, error)
    GetBalance(ctx context.Context, address string) (*big.Int, error)
}
```

**EVM chains** (7 implementations) share a common base that uses `go-ethereum` for RPC calls and contract interactions. Each EVM chain overrides chain-specific parameters (chain ID, gas estimation, finality requirements).

**Non-EVM chains** (6 implementations) each have unique SDKs and transaction formats:
- **Solana**: Uses Solana Go SDK, PDA-based addressing
- **Cosmos**: CosmWasm contract execution via gRPC
- **Aptos/Sui**: Move module calls via REST API
- **NEAR**: Rust contract calls via JSON-RPC
- **TRON**: TVM contract calls via HTTP API

### `pkg/merkle/`

Merkle tree construction and proof generation. Builds trees from account state hashes and generates inclusion proofs for specific leaves. Uses SHA-256 as the hash function, consistent with Accumulate's Merkle implementation.

### `pkg/server/`

HTTP server exposing health checks, metrics, and proof query endpoints. Runs alongside CometBFT on a separate port (default 8080).

### `pkg/firestore/`

Firestore client for syncing validator state to the cloud. Used to share proof status and validator health with the Web App's real-time listeners.

## CometBFT ABCI Integration

The validator runs as a CometBFT ABCI application. CometBFT handles peer discovery, consensus rounds, and block propagation. The validator application handles proof generation logic:

```
CometBFT                          Validator ABCI App
   |                                     |
   |-- PrepareProposal (proposer) ------>|
   |                                     |-- check pending intents
   |                                     |-- select proof operations
   |<-- proposed block ------------------|
   |                                     |
   |-- ProcessProposal (all) ----------->|
   |                                     |-- verify proposed operations
   |<-- accept/reject -------------------|
   |                                     |
   |-- FinalizeBlock (all, after commit)->|
   |                                     |-- execute proof phases
   |                                     |-- store results in PostgreSQL
   |                                     |-- trigger anchor if cycle complete
   |<-- state updates -------------------|
```

## Multi-Chain Strategy Pattern

Adding a new chain requires implementing the `ChainStrategy` interface. The strategy is registered in the chain registry during startup, keyed by chain name:

```
Chain Registry
  |
  +-- "ethereum"   -> EthereumStrategy (ChainID: 1)
  +-- "arbitrum"   -> ArbitrumStrategy (ChainID: 42161)
  +-- "optimism"   -> OptimismStrategy (ChainID: 10)
  +-- "base"       -> BaseStrategy     (ChainID: 8453)
  +-- "polygon"    -> PolygonStrategy  (ChainID: 137)
  +-- "avalanche"  -> AvalancheStrategy(ChainID: 43114)
  +-- "bsc"        -> BSCStrategy      (ChainID: 56)
  +-- "solana"     -> SolanaStrategy
  +-- "cosmos"     -> CosmosStrategy
  +-- "aptos"      -> AptosStrategy
  +-- "sui"        -> SuiStrategy
  +-- "near"       -> NEARStrategy
  +-- "tron"       -> TRONStrategy
```

See [Development Guide](development-guide.md) for instructions on adding a new chain.
