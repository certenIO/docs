# Independent Validator

The Independent Validator is the core consensus engine of the Certen Protocol, responsible for BFT consensus, 9-phase cryptographic proof generation, and anchoring Accumulate state to 13+ destination blockchains.

## Documentation Structure

| Document | Purpose | Target Audience |
|----------|---------|-----------------|
| [Architecture](architecture.md) | System design, package structure, database schema | All developers, architects |
| [Consensus & Proofs](consensus-and-proofs.md) | CometBFT consensus, 9-phase proof cycle, ZK proofs | Core developers, cryptographers |
| [Setup & Deployment](setup-deployment.md) | Installation, configuration, production deployment | DevOps, system administrators |
| [Development Guide](development-guide.md) | Code organization, testing, extending | Contributors, maintainers |

## Package Structure

| Package | Path | Description |
|---------|------|-------------|
| `main` | `main.go` | Entry point, CometBFT ABCI application bootstrap |
| `config` | `pkg/config/` | Configuration loading from environment variables |
| `database` | `pkg/database/` | PostgreSQL connection, migrations, repositories |
| `consensus` | `pkg/consensus/` | CometBFT ABCI handlers, block lifecycle |
| `proof` | `pkg/proof/` | 9-phase proof cycle orchestration |
| `anchor` | `pkg/anchor/` | On-chain anchoring to destination chains |
| `batch` | `pkg/batch/` | Proof batch management and scheduling |
| `execution` | `pkg/execution/` | Intent detection, validation, execution |
| `verification` | `pkg/verification/` | Proof verification logic |
| `attestation` | `pkg/attestation/` | Validator attestation handling |
| `intent` | `pkg/intent/` | Intent parsing, 4-blob data structure |
| `crypto` | `pkg/crypto/` | BLS12-381, Ed25519, key management |
| `chain` | `pkg/chain/` | Chain abstraction interface |
| `chain/strategy` | `pkg/chain/strategy/` | Per-chain implementations (13 chains) |
| `merkle` | `pkg/merkle/` | Merkle tree construction and proof generation |
| `server` | `pkg/server/` | HTTP API server (health, metrics) |
| `firestore` | `pkg/firestore/` | Firestore client for state sync |

## Quick Reference

### CLI Commands

```bash
# Start the validator node
go run ./main.go

# Run with specific config
CERTEN_CONFIG=/path/to/config go run ./main.go

# Run database migrations
go run ./cmd/migrate up
go run ./cmd/migrate down
go run ./cmd/migrate status
```

### Key Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ACCUMULATE_API` | `http://localhost:26660/v3` | Accumulate node endpoint |
| `DATABASE_URL` | `postgres://localhost/certen` | PostgreSQL connection string |
| `PROOF_BATCH_MODE` | `hybrid` | Proof scheduling: `on_demand`, `on_cadence`, `hybrid` |
| `PROOF_CYCLE_INTERVAL` | `30s` | Time between proof cycles |
| `COMETBFT_RPC` | `http://localhost:26657` | CometBFT RPC endpoint |
| `COMETBFT_P2P_PORT` | `26656` | CometBFT P2P listen port |
| `VALIDATOR_BLS_KEY` | (required) | Path to BLS12-381 private key |
| `ANCHOR_CONTRACT` | (required) | CertenAnchorV3 contract address |

See [Setup & Deployment](setup-deployment.md) for the full configuration reference.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check |
| `/status` | GET | Validator status and sync state |
| `/metrics` | GET | Prometheus-compatible metrics |
| `/proof/latest` | GET | Latest proof batch |
| `/proof/{batchId}` | GET | Specific proof batch by ID |

### Supported Chains

| Chain | Type | Strategy Implementation |
|-------|------|------------------------|
| Ethereum | EVM | `pkg/chain/strategy/ethereum.go` |
| Arbitrum | EVM | `pkg/chain/strategy/arbitrum.go` |
| Optimism | EVM | `pkg/chain/strategy/optimism.go` |
| Base | EVM | `pkg/chain/strategy/base.go` |
| Polygon | EVM | `pkg/chain/strategy/polygon.go` |
| Avalanche | EVM | `pkg/chain/strategy/avalanche.go` |
| BSC | EVM | `pkg/chain/strategy/bsc.go` |
| Solana | SVM | `pkg/chain/strategy/solana.go` |
| Cosmos | CosmWasm | `pkg/chain/strategy/cosmos.go` |
| Aptos | Move | `pkg/chain/strategy/aptos.go` |
| Sui | Move | `pkg/chain/strategy/sui.go` |
| NEAR | NEAR | `pkg/chain/strategy/near.go` |
| TRON | TVM | `pkg/chain/strategy/tron.go` |
