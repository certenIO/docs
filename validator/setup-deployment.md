# Validator Setup and Deployment

This document covers installing, configuring, and deploying the Independent Validator.

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16+ GB |
| Storage | 100 GB SSD | 500 GB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

The gnark ZK proof generation is CPU-intensive. More cores significantly reduce proof cycle time.

## Software Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Go | 1.24+ | Build and run the validator |
| PostgreSQL | 15+ | Proof and state storage |
| Docker | 24+ | CometBFT peers, containerized deployment |
| Docker Compose | 2.20+ | Multi-validator testnet setup |
| Git | 2.40+ | Source code |

## Configuration Reference

Configuration is loaded from environment variables. Copy `.env.example` to `.env` and customize.

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `ACCUMULATE_API` | `http://localhost:26660/v3` | Accumulate node API endpoint |
| `ACCUMULATE_API_V2` | `http://localhost:26660/v2` | Accumulate v2 API (legacy queries) |
| `DATABASE_URL` | `postgres://localhost:5432/certen_validator` | PostgreSQL connection string |
| `DATABASE_MAX_CONNS` | `20` | Maximum database connections |
| `LOG_LEVEL` | `info` | Log verbosity: debug, info, warn, error |

### CometBFT Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `COMETBFT_HOME` | `~/.cometbft` | CometBFT data directory |
| `COMETBFT_RPC` | `http://localhost:26657` | CometBFT RPC endpoint |
| `COMETBFT_P2P_PORT` | `26656` | P2P listen port |
| `COMETBFT_RPC_PORT` | `26657` | RPC listen port |
| `COMETBFT_PROXY_APP` | `tcp://localhost:26658` | ABCI application address |
| `COMETBFT_SEEDS` | (empty) | Seed node addresses |
| `COMETBFT_PERSISTENT_PEERS` | (empty) | Persistent peer addresses |

### Proof Cycle Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `PROOF_BATCH_MODE` | `hybrid` | `on_demand`, `on_cadence`, `hybrid` |
| `PROOF_CYCLE_INTERVAL` | `30s` | Duration between proof cycles |
| `PROOF_MAX_INTENTS_PER_BATCH` | `100` | Maximum intents per proof batch |
| `PROOF_TIMEOUT` | `60s` | Timeout for a single proof cycle |
| `ZK_PROOF_WORKERS` | `4` | Parallel gnark proof generation workers |

### Validator Identity

| Variable | Default | Description |
|----------|---------|-------------|
| `VALIDATOR_BLS_KEY` | (required) | Path to BLS12-381 private key file |
| `VALIDATOR_ED25519_KEY` | (required) | Path to Ed25519 private key file |
| `VALIDATOR_MONIKER` | `validator` | Human-readable validator name |
| `VALIDATOR_INDEX` | `0` | Validator index in the set (0-6) |

### Chain RPC Endpoints

| Variable | Default | Description |
|----------|---------|-------------|
| `ETHEREUM_RPC` | (required) | Ethereum mainnet/testnet RPC |
| `ARBITRUM_RPC` | (required) | Arbitrum RPC |
| `OPTIMISM_RPC` | (required) | Optimism RPC |
| `BASE_RPC` | (required) | Base RPC |
| `POLYGON_RPC` | (required) | Polygon RPC |
| `AVALANCHE_RPC` | (optional) | Avalanche RPC |
| `BSC_RPC` | (optional) | BSC RPC |
| `SOLANA_RPC` | (optional) | Solana RPC |
| `COSMOS_RPC` | (optional) | Cosmos gRPC endpoint |
| `APTOS_RPC` | (optional) | Aptos REST API |
| `SUI_RPC` | (optional) | Sui JSON-RPC |
| `NEAR_RPC` | (optional) | NEAR JSON-RPC |
| `TRON_RPC` | (optional) | TRON HTTP API |

### Contract Addresses

| Variable | Default | Description |
|----------|---------|-------------|
| `ANCHOR_CONTRACT_ETHEREUM` | (required) | CertenAnchorV3 on Ethereum |
| `ANCHOR_CONTRACT_ARBITRUM` | (required) | CertenAnchorV3 on Arbitrum |
| `ANCHOR_CONTRACT_OPTIMISM` | (required) | CertenAnchorV3 on Optimism |
| `ANCHOR_CONTRACT_BASE` | (required) | CertenAnchorV3 on Base |
| `ANCHOR_CONTRACT_POLYGON` | (required) | CertenAnchorV3 on Polygon |
| `BLS_VERIFIER_ADDRESS` | (required) | BLSZKVerifier contract address |
| `ACCOUNT_FACTORY_ADDRESS` | (required) | CertenAccountFactory address |

### Anchoring Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `ANCHOR_GAS_LIMIT` | `300000` | Gas limit for anchor transactions |
| `ANCHOR_GAS_PRICE_MULTIPLIER` | `1.2` | Gas price multiplier for priority |
| `ANCHOR_CONFIRMATION_BLOCKS` | `3` | Blocks to wait for confirmation |
| `ANCHOR_RETRY_COUNT` | `3` | Retries for failed anchor transactions |
| `ANCHOR_RETRY_DELAY` | `10s` | Delay between retries |

### Firestore Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `FIRESTORE_PROJECT_ID` | (optional) | GCP project ID for Firestore |
| `GOOGLE_APPLICATION_CREDENTIALS` | (optional) | Path to service account JSON |
| `FIRESTORE_ENABLED` | `false` | Enable Firestore state sync |

### Feature Flags

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_SOLANA` | `false` | Enable Solana chain strategy |
| `ENABLE_COSMOS` | `false` | Enable Cosmos chain strategy |
| `ENABLE_MOVE_CHAINS` | `false` | Enable Aptos/Sui strategies |
| `ENABLE_NEAR` | `false` | Enable NEAR chain strategy |
| `ENABLE_TRON` | `false` | Enable TRON chain strategy |
| `ENABLE_METRICS` | `true` | Enable Prometheus metrics endpoint |

## Database Setup

### Create Database

```bash
createdb certen_validator
```

### Run Migrations

```bash
cd ~/certen/independant_validator
go run ./cmd/migrate up
```

### Verify Schema

```bash
go run ./cmd/migrate status
```

### Rollback

```bash
go run ./cmd/migrate down    # rolls back one migration
go run ./cmd/migrate down 3  # rolls back 3 migrations
```

## Development Setup: Single Node

For local development without a full validator network:

```bash
cd ~/certen/independant_validator
cp .env.example .env

# Edit .env with minimal configuration:
# - DATABASE_URL pointing to local PostgreSQL
# - ACCUMULATE_API pointing to Kermit testnet or local devnet
# - At least one chain RPC (e.g., ETHEREUM_RPC for Sepolia)

# Run migrations
go run ./cmd/migrate up

# Start the validator
go run ./main.go
```

## Development Setup: 7-Validator Testnet (Docker Compose)

The `docker-compose.yml` in the repository sets up a complete 7-validator CometBFT network:

```bash
cd ~/certen/independant_validator

# Generate validator keys and genesis
./scripts/init-testnet.sh 7

# Start all validators with PostgreSQL
docker-compose up -d

# View logs
docker-compose logs -f validator-1

# Stop network
docker-compose down

# Stop and remove all data
docker-compose down -v
```

The `init-testnet.sh` script:
1. Generates 7 sets of CometBFT + BLS keys
2. Creates a `genesis.json` with all validators
3. Configures peer addresses for each node
4. Writes `.env` files for each validator
5. Creates PostgreSQL databases (one per validator)

## Production Deployment

### Systemd Service

Create `/etc/systemd/system/certen-validator.service`:

```ini
[Unit]
Description=Certen Independent Validator
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=certen
Group=certen
WorkingDirectory=/opt/certen/validator
EnvironmentFile=/opt/certen/validator/.env
ExecStart=/opt/certen/validator/bin/validator
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Build binary
cd ~/certen/independant_validator
go build -o /opt/certen/validator/bin/validator ./main.go

# Enable and start
sudo systemctl enable certen-validator
sudo systemctl start certen-validator

# Check status
sudo systemctl status certen-validator
sudo journalctl -u certen-validator -f
```

### Monitoring

The validator exposes Prometheus metrics on the `/metrics` endpoint:

| Metric | Type | Description |
|--------|------|-------------|
| `certen_proof_cycle_duration_seconds` | Histogram | Time per proof cycle |
| `certen_proof_cycles_total` | Counter | Total proof cycles completed |
| `certen_anchor_submissions_total` | Counter | Anchor transactions submitted (by chain) |
| `certen_anchor_failures_total` | Counter | Failed anchor transactions (by chain) |
| `certen_consensus_rounds_total` | Counter | CometBFT consensus rounds |
| `certen_active_intents` | Gauge | Currently processing intents |
| `certen_database_connections` | Gauge | Active database connections |

### Backup

**Database**: Use PostgreSQL pg_dump for regular backups:

```bash
pg_dump certen_validator > backup_$(date +%Y%m%d).sql
```

**Keys**: Back up BLS and Ed25519 keys securely. Loss of keys requires re-joining the validator set.

**CometBFT state**: Back up `~/.cometbft/data/` for fast recovery without full re-sync.

## Ports and Firewall

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 26656 | TCP | Inbound/Outbound | CometBFT P2P (required) |
| 26657 | TCP | Inbound | CometBFT RPC (restrict to trusted IPs) |
| 8080 | TCP | Inbound | Validator API (restrict to monitoring) |
| 5432 | TCP | Localhost only | PostgreSQL |

```bash
# Example firewall rules (ufw)
sudo ufw allow 26656/tcp    # CometBFT P2P - open to all
sudo ufw allow from 10.0.0.0/8 to any port 26657  # RPC - internal only
sudo ufw allow from 10.0.0.0/8 to any port 8080   # API - internal only
```
