# Development Environment Setup

This guide covers setting up a local development environment for the Certen Protocol.

## Prerequisites

Install the following before proceeding:

| Tool | Version | Used By | Install |
|------|---------|---------|---------|
| Go | 1.24+ | Validator, Proofs Service | [go.dev/dl](https://go.dev/dl/) |
| Node.js | 18+ | Web App, API Bridge, Pending Service, Key Vault | [nodejs.org](https://nodejs.org/) |
| npm | 9+ | All TypeScript repos | Bundled with Node.js |
| Docker | 24+ | Validator network, databases | [docker.com](https://www.docker.com/) |
| Docker Compose | 2.20+ | Multi-container setups | Bundled with Docker Desktop |
| Foundry | Latest | Smart contracts (EVM) | [getfoundry.sh](https://getfoundry.sh/) |
| PostgreSQL | 15+ | Validator, Proofs Service | [postgresql.org](https://www.postgresql.org/) |
| Chrome/Chromium | Latest | Key Vault extension | [chrome.google.com](https://www.google.com/chrome/) |
| Firebase CLI | Latest | Web App, Pending Service | `npm install -g firebase-tools` |
| Git | 2.40+ | All repos | [git-scm.com](https://git-scm.com/) |

### Optional

| Tool | Version | Used By |
|------|---------|---------|
| Anchor CLI | 0.29+ | Solana contracts |
| Rust/Cargo | 1.75+ | CosmWasm/NEAR contracts |
| Make | Any | Miner, Validator build scripts |

## Clone All Repositories

```bash
# Create workspace directory
mkdir -p ~/certen && cd ~/certen

# Clone all repositories
git clone https://github.com/certenIO/independant_validator.git
git clone https://github.com/certenIO/independant_miner.git
git clone https://github.com/certenIO/certen-web-app.git
git clone https://github.com/certenIO/key-vault-signer.git
git clone https://github.com/certenIO/api-bridge.git
git clone https://github.com/certenIO/certen-pending-service.git
git clone https://github.com/certenIO/proofs_service.git
git clone https://github.com/certenIO/certen-contracts.git
git clone https://github.com/certenIO/certen-network-set-up.git
```

## Setup Options

### Option A: Full Local Stack

Run the entire Certen infrastructure locally. Best for integration testing and understanding end-to-end flows.

**1. Start the Validator Network**

```bash
cd ~/certen/certen-network-set-up
./scripts/init-network.sh
docker-compose up -d
```

This starts a 4-validator CometBFT network with PostgreSQL.

**2. Start PostgreSQL (if not using Docker)**

```bash
# Create databases
createdb certen_validator
createdb certen_proofs

# Run migrations
cd ~/certen/independant_validator
go run ./cmd/migrate up

cd ~/certen/proofs_service
go run ./cmd/migrate up
```

**3. Start the Proofs Service**

```bash
cd ~/certen/proofs_service
cp .env.example .env
# Edit .env: set DATABASE_URL, API_PORT=8080
go run ./cmd/proof-service
```

**4. Start the API Bridge**

```bash
cd ~/certen/api-bridge
cp .env.example .env
# Edit .env: set ACCUM_ENDPOINT, keys, chain RPCs
npm install
npm run build
npm start
```

**5. Start Firebase Emulators (for Web App and Pending Service)**

```bash
cd ~/certen/certen-web-app
firebase emulators:start
```

**6. Start the Pending Service**

```bash
cd ~/certen/certen-pending-service
cp .env.example .env
# Edit .env: set Firebase credentials, Accumulate endpoint
npm install
npm run dev
```

**7. Start the Web App**

```bash
cd ~/certen/certen-web-app
cp .env.example .env
# Edit .env: set VITE_API_BASE_URL=http://localhost:8085
npm install
npm run dev
```

**8. Load Key Vault Extension**

```bash
cd ~/certen/key-vault-signer
npm install
npm run build
```

Then in Chrome: navigate to `chrome://extensions/`, enable Developer Mode, click "Load unpacked", select the `dist/` directory.

**9. Start Smart Contract Local Chain (optional)**

```bash
cd ~/certen/certen-contracts/evm
# Start local Anvil chain
anvil

# In a separate terminal, deploy contracts
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast
```

### Option B: Kermit Testnet

Connect to the shared Kermit testnet and run only the service you are developing. This avoids running validators, databases, and blockchain nodes locally.

```bash
# Common testnet configuration
ACCUM_ENDPOINT=http://206.191.154.164/v3
ETHEREUM_RPC=https://sepolia.infura.io/v3/YOUR_KEY
```

Set these in the `.env` file of whichever service you are working on. The Kermit testnet validators, API Bridge, and Proofs Service are already running.

### Option C: Single-Repo Focus

Minimal setup for working on one repository at a time.

| Repository | Minimal Dependencies |
|-----------|---------------------|
| `independant_validator` | Go, PostgreSQL, Docker (for CometBFT peers) |
| `api-bridge` | Node.js, Accumulate endpoint (Kermit or local) |
| `certen-web-app` | Node.js, Firebase CLI (emulators), API Bridge (Kermit or local) |
| `key-vault-signer` | Node.js, Chrome |
| `certen-pending-service` | Node.js, Firebase credentials, Accumulate endpoint |
| `proofs_service` | Go, PostgreSQL |
| `certen-contracts` | Foundry (EVM), Anchor (Solana), Rust (CosmWasm/NEAR) |

## Per-Repository Quick Start

### Validator

```bash
cd ~/certen/independant_validator
cp .env.example .env
# Edit .env with database URL, Accumulate API, chain RPCs
docker-compose up -d  # starts CometBFT peers and PostgreSQL
go run ./main.go
```

See [Validator Setup & Deployment](../validator/setup-deployment.md) for full configuration reference.

### API Bridge

```bash
cd ~/certen/api-bridge
cp .env.example .env
# Edit .env: PORT=8085, ACCUM_ENDPOINT, keys, chain RPCs
npm install
npm run dev
```

See [API Bridge README](../api-bridge/README.md) for endpoint documentation.

### Web App

```bash
cd ~/certen/certen-web-app
cp .env.example .env
# Edit .env: Firebase config, VITE_API_BASE_URL
npm install
npm run dev
# Optionally start Firebase emulators: firebase emulators:start
```

See [Web App README](../web-app/README.md) for page inventory and context providers.

### Key Vault

```bash
cd ~/certen/key-vault-signer
npm install
npm run build
# Load dist/ as unpacked extension in Chrome
```

See [Key Vault README](../key-vault/README.md) for security model details.

### Pending Service

```bash
cd ~/certen/certen-pending-service
cp .env.example .env
# Edit .env: Firebase service account, Accumulate endpoint
npm install
npm run dev
```

See [Pending Service README](../pending-service/README.md) for discovery algorithm.

### Proofs Service

```bash
cd ~/certen/proofs_service
cp .env.example .env
# Edit .env: DATABASE_URL, API_PORT=8080
# Run database migrations
go run ./cmd/migrate up
go run ./cmd/proof-service
```

See [Proofs Service README](../proofs-service/README.md) for API reference.

### Smart Contracts

```bash
cd ~/certen/certen-contracts/evm
forge install
forge build
forge test

# Local deployment
anvil &
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast
```

See [Contracts README](../contracts/README.md) for platform-specific instructions.

## Port Reference

| Service | Default Port | Protocol | Notes |
|---------|-------------|----------|-------|
| Web App (dev server) | 3000 | HTTP | Vite dev server |
| API Bridge | 8085 | HTTP | Express REST API |
| Proofs Service API | 8080 | HTTP | Go REST API |
| Proofs Explorer | 3000 | HTTP | Vite dev server (conflicts with Web App) |
| Validator API | 8080 | HTTP | Health, metrics |
| CometBFT P2P | 26656 | TCP | Validator consensus |
| CometBFT RPC | 26657 | HTTP | Tendermint RPC |
| PostgreSQL | 5432 | TCP | Validator + Proofs Service |
| Firebase Auth Emulator | 9099 | HTTP | Local auth |
| Firebase Firestore Emulator | 8080 | HTTP | Local Firestore (conflicts with Proofs) |
| Firebase Functions Emulator | 5001 | HTTP | Local Cloud Functions |
| Firebase Emulator UI | 4000 | HTTP | Emulator dashboard |
| Anvil (local EVM) | 8545 | HTTP | Local blockchain |
| LibP2P (Miner) | 4001 | TCP/QUIC | Miner P2P network |

**Port conflict notes**: Proofs Service API and Firebase Firestore Emulator both default to 8080. Web App and Proofs Explorer both default to 3000. Adjust ports in `.env` or emulator config if running both simultaneously.

## Verification Steps

After starting services, verify they are running:

```bash
# API Bridge
curl http://localhost:8085/health

# Proofs Service
curl http://localhost:8080/api/v1/health

# CometBFT RPC (validator)
curl http://localhost:26657/status

# Web App
# Open http://localhost:3000 in browser

# Key Vault
# Check chrome://extensions/ for loaded extension

# Firebase Emulators
curl http://localhost:4000
```

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `ECONNREFUSED` on port 8085 | API Bridge not running | Start API Bridge: `npm run dev` in api-bridge/ |
| PostgreSQL connection refused | Database not running | Start PostgreSQL or Docker container |
| `GOOGLE_APPLICATION_CREDENTIALS` not set | Missing Firebase service account | Download service account JSON from Firebase Console |
| Key Vault not detected | Extension not loaded | Load unpacked extension from `dist/` in Chrome |
| CometBFT peers not connecting | Incorrect genesis or peer IDs | Re-run `init-network.sh` to regenerate configs |
| `forge: command not found` | Foundry not installed | Run `curl -L https://foundry.paradigm.xyz | bash && foundryup` |
| Port already in use | Another service on same port | Check `lsof -i :PORT` or `netstat -ano | findstr PORT` (Windows) |
| Go module errors | Wrong Go version | Ensure Go 1.24+ for validator, 1.21+ for proofs service |
| Firebase emulator conflict on 8080 | Proofs Service also uses 8080 | Change Firestore emulator port in `firebase.json` |

## Network Environments

| Environment | Accumulate Endpoint | Ethereum | Use Case |
|-------------|-------------------|----------|----------|
| DevNet | `http://localhost:26660/v3` | Anvil (`localhost:8545`) | Local development |
| Kermit | `http://206.191.154.164/v3` | Sepolia | Shared testing |
| Mainnet | `mainnet.accumulatenetwork.io` | Ethereum Mainnet | Production |
