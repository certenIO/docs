# Validator Development Guide

This document covers the code organization, build system, testing strategy, and common development tasks for the Independent Validator.

## Repository Layout

```
independant_validator/
├── main.go                        # Entry point, ABCI app bootstrap
├── go.mod / go.sum                # Go module dependencies
├── .env.example                   # Configuration template
├── docker-compose.yml             # 7-validator testnet setup
├── Dockerfile                     # Production container image
├── Makefile                       # Build, test, lint targets
├── scripts/
│   ├── init-testnet.sh            # Generate testnet configuration
│   └── migrate.sh                 # Database migration helper
├── cmd/
│   └── migrate/                   # Migration CLI tool
│       └── main.go
├── pkg/
│   ├── config/                    # Environment-based configuration
│   │   └── config.go
│   ├── database/
│   │   ├── database.go            # Connection pool, factory
│   │   ├── migrations/            # SQL migration files (001-007)
│   │   ├── batch_repo.go          # Batch CRUD operations
│   │   ├── proof_repo.go          # Proof CRUD operations
│   │   ├── anchor_repo.go         # Anchor record CRUD
│   │   ├── attestation_repo.go    # Attestation CRUD
│   │   └── intent_repo.go         # Intent tracking CRUD
│   ├── consensus/
│   │   ├── app.go                 # ABCI Application implementation
│   │   ├── prepare.go             # PrepareProposal handler
│   │   ├── process.go             # ProcessProposal handler
│   │   └── finalize.go            # FinalizeBlock handler
│   ├── proof/
│   │   ├── engine.go              # Proof cycle orchestrator
│   │   ├── phases.go              # L1-L4 Merkle proof phases
│   │   ├── governance.go          # G0-G2 governance phases
│   │   └── bls.go                 # BLS aggregation + ZK proof
│   ├── anchor/
│   │   ├── manager.go             # Anchor submission orchestrator
│   │   └── tracker.go             # Confirmation tracking
│   ├── batch/
│   │   ├── scheduler.go           # Batch scheduling logic
│   │   └── manager.go             # Batch lifecycle management
│   ├── execution/
│   │   ├── scanner.go             # Intent detection on Accumulate
│   │   └── executor.go            # Intent validation and queueing
│   ├── verification/
│   │   └── verifier.go            # Proof verification logic
│   ├── attestation/
│   │   └── handler.go             # Attestation collection
│   ├── intent/
│   │   ├── parser.go              # 4-blob intent parsing
│   │   └── validator.go           # Intent format validation
│   ├── crypto/
│   │   ├── bls.go                 # BLS12-381 operations
│   │   ├── ed25519.go             # Ed25519 signing
│   │   ├── circuit.go             # gnark Groth16 circuit
│   │   └── keys.go                # Key loading, generation
│   ├── chain/
│   │   ├── strategy.go            # ChainStrategy interface
│   │   ├── registry.go            # Chain registration
│   │   └── strategy/
│   │       ├── evm_base.go        # Shared EVM logic
│   │       ├── ethereum.go        # Ethereum-specific
│   │       ├── arbitrum.go
│   │       ├── optimism.go
│   │       ├── base.go
│   │       ├── polygon.go
│   │       ├── avalanche.go
│   │       ├── bsc.go
│   │       ├── solana.go
│   │       ├── cosmos.go
│   │       ├── aptos.go
│   │       ├── sui.go
│   │       ├── near.go
│   │       └── tron.go
│   ├── merkle/
│   │   ├── tree.go                # Merkle tree construction
│   │   └── proof.go               # Inclusion proof generation
│   ├── server/
│   │   ├── server.go              # HTTP server setup
│   │   ├── health.go              # Health endpoint
│   │   └── metrics.go             # Prometheus metrics
│   └── firestore/
│       └── client.go              # Firestore sync client
└── test/
    ├── integration/               # Integration tests
    └── fixtures/                  # Test data files
```

## Build, Test, and Lint

```bash
# Build
go build -o bin/validator ./main.go
# or
make build

# Run tests
go test ./...
# or
make test

# Run tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Lint
golangci-lint run
# or
make lint

# Format
gofumpt -w .
# or
make fmt

# Run all checks (format, lint, test)
make check
```

## Common Development Tasks

### Adding a New Proof Phase

If the proof cycle needs an additional phase (e.g., a new governance layer):

1. Define the phase type and output structure in `pkg/proof/`:

```go
// pkg/proof/phases.go
type NewPhaseProof struct {
    Data      []byte
    ProofPath [][]byte
    Root      []byte
}
```

2. Implement the phase execution in the proof engine:

```go
// pkg/proof/engine.go
func (e *ProofEngine) executeNewPhase(ctx context.Context, batch *Batch) (*NewPhaseProof, error) {
    // Query Accumulate for required data
    // Construct Merkle proof
    // Return proof structure
}
```

3. Add the phase to the proof cycle sequence in `engine.go`
4. Update the proof bundle format in `pkg/proof/` to include the new phase
5. Add database storage for the new proof type (new migration + repository method)
6. Update verification logic in `pkg/verification/verifier.go`
7. Add tests for the new phase

### Adding Support for a New Target Chain

1. Create a new strategy file in `pkg/chain/strategy/`:

```go
// pkg/chain/strategy/newchain.go
type NewChainStrategy struct {
    rpcClient *rpc.Client
    config    ChainConfig
}

func NewNewChainStrategy(cfg ChainConfig) (*NewChainStrategy, error) {
    // Initialize RPC client
}

func (s *NewChainStrategy) Name() string { return "newchain" }
func (s *NewChainStrategy) ChainID() uint64 { return 12345 }

func (s *NewChainStrategy) SubmitAnchor(ctx context.Context, data AnchorData) (TxHash, error) {
    // Construct chain-specific anchor transaction
    // Submit via RPC
    // Return transaction hash
}

func (s *NewChainStrategy) DeployAccount(ctx context.Context, adi string, salt []byte) (Address, error) {
    // Deploy ADI-linked account on the new chain
}

func (s *NewChainStrategy) VerifyProof(ctx context.Context, root, proof, leaf []byte) (bool, error) {
    // Call on-chain verification contract
}

func (s *NewChainStrategy) GetBalance(ctx context.Context, address string) (*big.Int, error) {
    // Query native token balance
}
```

2. Register the strategy in `pkg/chain/registry.go`:

```go
if cfg.EnableNewChain {
    strategy, err := NewNewChainStrategy(cfg.NewChainConfig)
    registry.Register("newchain", strategy)
}
```

3. Add configuration variables to `pkg/config/config.go`:
   - `NEWCHAIN_RPC`: RPC endpoint
   - `ANCHOR_CONTRACT_NEWCHAIN`: Contract address
   - `ENABLE_NEWCHAIN`: Feature flag

4. Deploy the `CertenAnchor` contract on the new chain (if EVM-compatible, reuse existing Solidity; otherwise implement equivalent in the chain's native language)

5. Add tests: unit tests for the strategy, integration test with a local chain node

### Database Migration Workflow

```bash
# Create a new migration
# Manually create pkg/database/migrations/008_description.up.sql
# and pkg/database/migrations/008_description.down.sql

# Apply the migration
go run ./cmd/migrate up

# Verify
go run ./cmd/migrate status

# Rollback if needed
go run ./cmd/migrate down
```

Migration files should be idempotent where possible (use `IF NOT EXISTS`, `IF EXISTS`).

## Testing Strategy

### Unit Tests

Each package has `_test.go` files alongside the source. Use interfaces and mocks for external dependencies:

```bash
# Run a specific package's tests
go test ./pkg/proof/...

# Run with verbose output
go test -v ./pkg/consensus/...

# Run a specific test
go test -run TestProofEngineL1 ./pkg/proof/...
```

### Integration Tests

Located in `test/integration/`. Require running PostgreSQL and optionally CometBFT:

```bash
# Run integration tests (requires DATABASE_URL)
go test -tags=integration ./test/integration/...
```

### Consensus Simulation

Test the full consensus flow with multiple validator instances:

```bash
# Start a 4-validator testnet
./scripts/init-testnet.sh 4
docker-compose up -d

# Run consensus simulation tests
go test -tags=consensus ./test/integration/consensus_test.go
```

### Test Coverage Targets

| Package | Target | Rationale |
|---------|--------|-----------|
| `pkg/proof/` | >90% | Critical proof generation logic |
| `pkg/consensus/` | >85% | ABCI state machine correctness |
| `pkg/crypto/` | >95% | Cryptographic operations must be correct |
| `pkg/chain/strategy/` | >80% | Chain interaction reliability |
| `pkg/database/` | >80% | Data integrity |
| `pkg/intent/` | >90% | Intent parsing correctness |

## Code Style

- **Go version**: 1.24+ required
- **Formatter**: `gofumpt` (stricter than `gofmt`)
- **Linter**: `golangci-lint` with the project's `.golangci.yml` configuration
- **Error handling**: Always wrap errors with context: `fmt.Errorf("phase L1 failed: %w", err)`
- **Context**: Pass `context.Context` as the first parameter to all functions that do I/O
- **Logging**: Use structured logging with `slog` package
