# Welcome to the Certen Protocol

The Certen Protocol is decentralized cross-chain proof infrastructure built on the Accumulate blockchain, enabling trustless verification of digital identities and governance decisions across 13+ blockchain networks without centralized bridges or custodians.

## How to Use This Documentation

This onboarding sequence is designed to take a new developer from zero context to productive contributor. Read the documents in order, or skip to the sections relevant to your role.

### Reading Paths by Role

**Full-Stack / General Developer**:
1. [Welcome & Reading Guide](01-welcome.md) (you are here)
2. [Ecosystem Overview](02-ecosystem-overview.md)
3. [Architecture Deep Dive](03-architecture-deep-dive.md)
4. [Data Flow Walkthrough](04-data-flow-walkthrough.md)
5. [Development Environment Setup](05-dev-environment-setup.md)
6. [Glossary](06-glossary.md)
7. Repository-specific docs (see below)

**Frontend Developer**:
1. [Welcome](01-welcome.md)
2. [Ecosystem Overview](02-ecosystem-overview.md)
3. [Data Flow Walkthrough](04-data-flow-walkthrough.md)
4. [Development Environment Setup](05-dev-environment-setup.md)
5. [Web App Docs](../web-app/README.md) and [Key Vault Docs](../key-vault/README.md)

**Backend / Go Developer**:
1. [Welcome](01-welcome.md)
2. [Ecosystem Overview](02-ecosystem-overview.md)
3. [Architecture Deep Dive](03-architecture-deep-dive.md)
4. [Data Flow Walkthrough](04-data-flow-walkthrough.md)
5. [Development Environment Setup](05-dev-environment-setup.md)
6. [Validator Docs](../validator/README.md) and [Proofs Service Docs](../proofs-service/README.md)

**Smart Contract Developer**:
1. [Welcome](01-welcome.md)
2. [Ecosystem Overview](02-ecosystem-overview.md)
3. [Architecture Deep Dive](03-architecture-deep-dive.md)
4. [Development Environment Setup](05-dev-environment-setup.md)
5. [Contracts Docs](../contracts/README.md)

**DevOps / Infrastructure**:
1. [Welcome](01-welcome.md)
2. [Ecosystem Overview](02-ecosystem-overview.md)
3. [Development Environment Setup](05-dev-environment-setup.md)
4. Each repository's setup/deployment section

## Repository Map

| Repository | Language | Purpose |
|-----------|----------|---------|
| `independant_validator` | Go | BFT consensus, 9-phase proof generation, anchoring to 13 chains |
| `certen-pending-service` | TypeScript | Polls Accumulate for pending multi-sig transactions, syncs to Firestore |
| `api-bridge` | TypeScript | REST API for ADI management, intents, two-phase signing |
| `key-vault-signer` | TypeScript/React | Chrome extension for secure key storage and signing |
| `proofs_service` | Go | Proof artifact storage, retrieval, verification API + explorer UI |
| `certen-web-app` | TypeScript/React | Primary user interface for ADI management and proof visualization |
| `certen-contracts` | Solidity/Rust/Move | On-chain anchoring, BLS ZK verification, account abstraction |
| `independant_miner` | Go | P2P audit node verifying validator proofs with LXR proof-of-work |

## Documentation Index

### Onboarding Sequence
- [Ecosystem Overview](02-ecosystem-overview.md)
- [Architecture Deep Dive](03-architecture-deep-dive.md)
- [Data Flow Walkthrough](04-data-flow-walkthrough.md)
- [Development Environment Setup](05-dev-environment-setup.md)
- [Glossary](06-glossary.md)

### Repository Documentation
- [Validator](../validator/README.md) | [Architecture](../validator/architecture.md) | [Consensus & Proofs](../validator/consensus-and-proofs.md) | [Setup](../validator/setup-deployment.md) | [Development](../validator/development-guide.md)
- [API Bridge](../api-bridge/README.md) | [Architecture](../api-bridge/architecture.md)
- [Web App](../web-app/README.md) | [Architecture](../web-app/architecture.md)
- [Key Vault](../key-vault/README.md) | [Architecture](../key-vault/architecture.md)
- [Pending Service](../pending-service/README.md)
- [Proofs Service](../proofs-service/README.md) | [Architecture](../proofs-service/architecture.md)
- [Contracts](../contracts/README.md) | [Architecture](../contracts/architecture.md)
- [Miner](../miner/README.md) | [Architecture](../miner/architecture.md) | [Algorithms](../miner/algorithms.md) | [Networking](../miner/networking.md) | [Setup](../miner/setup-deployment.md) | [Development](../miner/development-guide.md)
