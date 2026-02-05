# Certen Protocol

Decentralized cross-chain proof infrastructure built on Accumulate, enabling trustless verification of digital identities and governance decisions across blockchain networks.

## Overview

The Certen Protocol is a multi-layer system that bridges the Accumulate blockchain to external chains (Ethereum, Solana, Cosmos, and others) through cryptographic proofs and BFT consensus. It enables Accumulate Digital Identifiers (ADIs) to authorize transactions on any supported chain without requiring centralized bridges or custodians.

The protocol operates through a network of independent validators that generate Merkle proofs and BLS aggregate signatures, which are then anchored to destination chains via smart contracts. Independent miners audit these proofs using memory-hard proof-of-work to ensure validator honesty.

### Core Capabilities

1. **State Anchoring**: Accumulate state roots committed on-chain with cryptographic proofs
2. **BLS Verification**: Validator consensus verified via ZK-SNARK proofs of aggregate signatures
3. **Cross-Chain Identity**: ADIs linked to accounts on any supported blockchain
4. **Multi-Signature Governance**: Hierarchical key structures for organizational control
5. **Proof-of-Work Auditing**: Decentralized verification of validator behavior

## Architecture

```
+------------------------------------------------------------------+
|                        User Applications                          |
|           (Web App, Mobile, External Integrations)                |
+------------------------------------------------------------------+
                                  |
                                  v
+------------------------------------------------------------------+
|                        Frontend Layer                             |
+------------------------------------------------------------------+
|  +------------------+    +------------------+    +---------------+ |
|  |   Web App        |    |   Key Vault      |    |   Pending     | |
|  |   (React)        |<-->|   (Extension)    |<-->|   Service     | |
|  +------------------+    +------------------+    +---------------+ |
+------------------------------------------------------------------+
                                  |
                                  v
+------------------------------------------------------------------+
|                        API Layer                                  |
+------------------------------------------------------------------+
|  +------------------+    +------------------+                      |
|  |   API Bridge     |    |   Proofs         |                      |
|  |   (Express)      |<-->|   Service        |                      |
|  +------------------+    +------------------+                      |
+------------------------------------------------------------------+
                                  |
                                  v
+------------------------------------------------------------------+
|                        Protocol Layer                             |
+------------------------------------------------------------------+
|  +------------------+    +------------------+    +---------------+ |
|  |   Validators     |    |   Miners         |    |   Smart       | |
|  |   (BFT/CometBFT) |<-->|   (LXR PoW)      |<-->|   Contracts   | |
|  +------------------+    +------------------+    +---------------+ |
+------------------------------------------------------------------+
                                  |
                                  v
+------------------------------------------------------------------+
|                        Blockchain Layer                           |
+------------------------------------------------------------------+
|  +------------------+    +------------------+    +---------------+ |
|  |   Accumulate     |    |   Ethereum       |    |   Other       | |
|  |   Network        |    |   (EVM Chains)   |    |   Chains      | |
|  +------------------+    +------------------+    +---------------+ |
+------------------------------------------------------------------+
```

## Components

### Core Infrastructure

| Component | Repository | Language | Description |
|-----------|------------|----------|-------------|
| **Independent Validator** | `independant_validator` | Go | BFT consensus node generating proofs and anchoring to Ethereum |
| **Independent Miner** | `independant_miner` | Go | P2P audit node verifying validator proofs with LXR proof-of-work |
| **Smart Contracts** | `certen-contracts` | Solidity | EVM anchor contracts with BLS verification and account abstraction |
| **Network Setup** | `certen-network-set-up` | Shell/Go | Bootstrap scripts for deploying validator networks |

### Application Layer

| Component | Repository | Language | Description |
|-----------|------------|----------|-------------|
| **Web Application** | `certen-web-app` | TypeScript | React SPA for ADI management, multi-sig coordination, proof exploration |
| **Key Vault** | `key-vault-signer` | TypeScript | Chrome extension for secure key storage and transaction signing |
| **Pending Service** | `certen-pending-service` | TypeScript | Background service discovering multi-sig transactions requiring signatures |

### Integration Layer

| Component | Repository | Language | Description |
|-----------|------------|----------|-------------|
| **API Bridge** | `api-bridge` | TypeScript | REST API for Accumulate operations and two-phase signing |
| **Proofs Service** | `proofs_service` | Go | Proof storage, retrieval, and verification API |

## Key Concepts

### Proof Cycle

The validator network generates proofs in a 9-phase cycle:

| Phase | Name | Description |
|-------|------|-------------|
| L1 | Account Merkle | Account state inclusion in partition |
| L2 | BPT | Binary Patricia Trie root proof |
| L3 | Root Anchor | Partition root to Directory Network |
| L4 | DN Anchor | Directory Network consensus anchor |
| G0 | Key Page | Key page state for governance |
| G1 | Key Book | Key book structure and thresholds |
| G2 | Authority | Authority chain and delegation |
| BLS | Aggregation | BLS12-381 aggregate signature |
| Anchor | Ethereum | On-chain anchor with ZK verification |

### ADI Governance

Accumulate Digital Identifiers support hierarchical governance:

```
acc://organization.acme              (ADI - Root Identity)
├── /book                            (Key Book - Authority Container)
│   ├── /1                           (Key Page - Threshold: 2-of-3)
│   │   ├── key: 0x1234...           (ED25519 Public Key)
│   │   ├── key: 0x5678...           (ED25519 Public Key)
│   │   └── key: 0x9abc...           (ED25519 Public Key)
│   └── /2                           (Key Page - Admin: 1-of-1)
│       └── delegate: acc://admin.acme/book
├── /tokens                          (Token Account)
└── /data                            (Data Account - Intent Storage)
```

### Two-Phase Signing

The protocol supports external signing through Key Vault:

1. **Prepare**: API Bridge constructs transaction and returns hash
2. **Sign**: Key Vault signs hash with user's private key
3. **Submit**: API Bridge submits transaction with external signature

This ensures private keys never leave the browser extension.

## Quick Start

### Prerequisites

- Go 1.21+ (for validators, miners, proofs service)
- Node.js 18+ (for web app, API bridge, services)
- Docker (optional, for containerized deployment)
- Foundry (for smart contract development)

### Development Setup

```bash
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

# Start local validator network (4 validators)
cd certen-network-set-up
./scripts/init-network.sh
docker-compose up -d

# Start API Bridge
cd ../api-bridge
cp .env.example .env
npm install && npm run build && npm start

# Start Web App
cd ../certen-web-app
cp .env.example .env
npm install && npm run dev
```

### Network Environments

| Environment | Accumulate | Ethereum | Use Case |
|-------------|------------|----------|----------|
| DevNet | localhost:26660 | Anvil | Local development |
| Kermit | 206.191.154.164 | Sepolia | Testing |
| Mainnet | mainnet.accumulatenetwork.io | Ethereum | Production |

## Documentation

### Component Documentation

Each repository contains its own detailed README:

- [Independent Validator](https://github.com/certenIO/independant_validator/blob/main/README.md) - BFT consensus and proof generation
- [Independent Miner](https://github.com/certenIO/independant_miner/blob/main/README.md) - LXR proof-of-work auditing
- [Smart Contracts](https://github.com/certenIO/certen-contracts/blob/main/README.md) - EVM contract deployment
- [Web Application](https://github.com/certenIO/certen-web-app/blob/main/README.md) - User interface
- [Key Vault](https://github.com/certenIO/key-vault-signer/blob/main/README.md) - Browser extension
- [API Bridge](https://github.com/certenIO/api-bridge/blob/main/README.md) - REST API
- [Pending Service](https://github.com/certenIO/certen-pending-service/blob/main/README.md) - Multi-sig discovery
- [Proofs Service](https://github.com/certenIO/proofs_service/blob/main/README.md) - Proof storage

### Detailed Documentation

- [Miner Architecture](miner/architecture.md) - System design and data flow
- [Miner Algorithms](miner/algorithms.md) - LXR proof-of-work details
- [Miner Networking](miner/networking.md) - LibP2P protocols
- [Miner Deployment](miner/setup-deployment.md) - Production deployment
- [Miner Development](miner/development-guide.md) - Contributing guide

## Security

### Cryptographic Primitives

| Algorithm | Use Case | Implementation |
|-----------|----------|----------------|
| ED25519 | Transaction signatures | tweetnacl, noble-ed25519 |
| BLS12-381 | Validator consensus | Groth16 ZK-SNARK |
| AES-256-GCM | Key Vault encryption | Web Crypto API |
| PBKDF2-SHA512 | Key derivation | 600K iterations (OWASP 2023) |
| SHA-256 | Merkle trees, hashing | Go crypto, Web Crypto |
| LXR | Miner proof-of-work | Memory-hard (1GB table) |

### Security Model

- **Private Keys**: Never leave Key Vault extension, encrypted at rest
- **Transport**: All communications over TLS/HTTPS
- **Signatures**: ED25519 for transactions, BLS for validator consensus
- **Smart Contracts**: Pausable pattern, access control, replay protection
- **Validators**: BFT consensus (2/3+ threshold), slashing for misbehavior

### Audit Status

Smart contracts are pending formal audit. Security measures implemented:

- CRITICAL-001: Immutable merkleRoot binding all commitments
- CRITICAL-002: Merkle root cannot be modified after creation
- MEDIUM-001: Pausable pattern for emergency stops
- HIGH-001: Real Merkle verification for governance proofs

## Configuration Reference

### Validator Configuration

```yaml
# ~/.certen/validator/config.yaml
accumulate_api: http://localhost:26660/v3
ethereum_rpc: https://sepolia.infura.io/v3/...
proof_batch_mode: hybrid        # on_demand, on_cadence, hybrid
proof_cycle_interval: 30s
anchor_contract: 0x...
validator_private_key: /path/to/key
```

### Miner Configuration

```yaml
# ~/.certen/miner/config.yaml
validator_url: https://validator.certen.io
audit_interval: 5s
listen_addrs:
  - /ip4/0.0.0.0/tcp/4001
  - /ip4/0.0.0.0/udp/4001/quic-v1
lxr:
  table_bits: 30    # 1GB lookup table
  loops: 5
  passes: 6
```

### API Bridge Configuration

```env
# api-bridge/.env
PORT=8085
ACCUM_ENDPOINT=http://206.191.154.164/v3
ACCUM_PUBLIC_KEY=...
ACCUM_PRIV_KEY=...
ETHEREUM_URL=https://sepolia.infura.io/v3/...
ANCHOR_CONTRACT_ADDRESS=0x...
```

## Deployment

### Docker Compose (Development)

```yaml
version: '3.8'
services:
  validator-1:
    image: certen/validator:latest
    ports:
      - "26657:26657"
    volumes:
      - ./config:/root/.certen/validator
    environment:
      - ACCUMULATE_API=http://accumulate:26660/v3

  api-bridge:
    image: certen/api-bridge:latest
    ports:
      - "8085:8085"
    env_file: .env

  web-app:
    image: certen/web-app:latest
    ports:
      - "3000:3000"
    environment:
      - VITE_API_BASE_URL=http://localhost:8085
```

### Production Deployment

See individual repository READMEs for production deployment guides including:

- Systemd service configuration
- Cloud Run / GCP deployment
- Kubernetes manifests
- Monitoring and alerting setup

## Contributing

### Development Workflow

1. Fork the relevant repository
2. Create a feature branch from `main`
3. Implement changes with tests
4. Submit pull request with detailed description
5. Address code review feedback
6. Maintainer merges after approval

### Code Standards

- **Go**: gofmt, golangci-lint, test coverage >80%
- **TypeScript**: ESLint, Prettier, type coverage
- **Solidity**: Foundry fmt, Slither analysis

### Communication

- GitHub Issues: Bug reports, feature requests
- Pull Requests: Code contributions
- Discussions: Architecture decisions, roadmap

## Roadmap

### Current (v1.0)

- EVM chain support (Ethereum, Arbitrum, Optimism, Base)
- ED25519/secp256k1/BLS key support
- Single and multi-leg transaction intents
- On-demand and batched proof generation

### Planned (v2.0)

- Solana program deployment
- CosmWasm contract support
- Move (Aptos/Sui) module support
- TON and NEAR integration
- Mobile wallet support

## License

MIT License

Copyright 2025 Certen Protocol. All rights reserved.

Individual components may have specific licensing - see each repository for details.
