# Ecosystem Overview

The Certen Protocol bridges the Accumulate blockchain to external chains through cryptographic proofs and BFT consensus, enabling Accumulate Digital Identifiers (ADIs) to authorize transactions on any supported chain without centralized bridges.

## The Problem

Blockchain identities are chain-specific. An Ethereum address cannot authorize a Solana transaction. Organizations managing assets across multiple chains need separate key management for each, creating security fragmentation and operational complexity.

## The Solution

Certen creates a unified identity layer using Accumulate's ADI system. A single ADI with hierarchical governance (key books, key pages, multi-sig thresholds) can control accounts on 13+ blockchains. Validators generate cryptographic proofs that Accumulate authorized a cross-chain action, and smart contracts on destination chains verify these proofs trustlessly.

## Architecture

```
+------------------------------------------------------------------+
|                        User Applications                          |
|           (Web App, Mobile, External Integrations)                |
+------------------------------------------------------------------+
                                  | HTTP, WebSocket
                                  v
+------------------------------------------------------------------+
|                        Frontend Layer                             |
+------------------------------------------------------------------+
|  +------------------+  window   +------------------+   Firestore  |
|  |   Web App        |<-------->|   Key Vault      |   realtime   |
|  |   (React SPA)    |  .certen |   (Chrome Ext)   |   listener   |
|  +------------------+          +------------------+              |
|          |                           |                            |
|          | Firestore                 | signs tx hashes            |
|          v                           v                            |
|  +------------------+                                             |
|  |   Pending Service |  (polls Accumulate -> syncs to Firestore) |
|  +------------------+                                             |
+------------------------------------------------------------------+
                                  | HTTP
                                  v
+------------------------------------------------------------------+
|                        API / Service Layer                        |
+------------------------------------------------------------------+
|  +------------------+   HTTP    +------------------+              |
|  |   API Bridge     |<-------->|   Proofs Service  |              |
|  |   (Express :8085)|          |   (Go :8080)      |              |
|  +------------------+          +------------------+              |
+------------------------------------------------------------------+
                                  | Accumulate API, P2P
                                  v
+------------------------------------------------------------------+
|                        Protocol Layer                             |
+------------------------------------------------------------------+
|  +------------------+  CometBFT +------------------+  on-chain   |
|  |   Validators x7  |<-------->|   Smart Contracts |<------->    |
|  |   (BFT/CometBFT) |  P2P     |   (EVM, Solana)  |  verify     |
|  +------------------+          +------------------+              |
|  +------------------+                                             |
|  |   Miners (LXR)   |  (audit validator proofs via P2P)          |
|  +------------------+                                             |
+------------------------------------------------------------------+
                                  | RPC, REST
                                  v
+------------------------------------------------------------------+
|                        Blockchain Layer                           |
+------------------------------------------------------------------+
|  Accumulate | Ethereum | Arbitrum | Optimism | Base | Polygon    |
|  Solana | Cosmos | Aptos | Sui | NEAR | TRON | TON              |
+------------------------------------------------------------------+
```

## Component Descriptions

### Independent Validator (`independant_validator` - Go)

The core consensus engine. A network of 7 validators runs CometBFT-based BFT consensus to agree on Accumulate state and generate cryptographic proofs. Each validator executes a 9-phase proof cycle: building Merkle proofs of account state (L1-L4), governance proofs (G0-G2), aggregating BLS signatures, and anchoring the final proof bundle to destination chains via smart contracts. The validator manages connections to 13 target blockchains through a chain strategy pattern and stores proof artifacts in PostgreSQL.

### Web Application (`certen-web-app` - TypeScript/React)

The primary user interface for the Certen Protocol. Built with React 18, Vite, and Material-UI, it provides ADI management (creation, key rotation, governance configuration), multi-signature transaction coordination, intent creation for cross-chain operations, and proof visualization through an explorer view. Integrates with Firebase for authentication, Firestore for real-time data sync, and communicates with Key Vault via the `window.certen` browser API.

### API Bridge (`api-bridge` - TypeScript/Express)

The HTTP gateway between frontend applications and the Accumulate network. Exposes 30+ REST endpoints for ADI operations, key management, credit purchases, intent submission, and multi-chain account deployment. Implements the two-phase signing protocol where transactions are prepared server-side, signed client-side in Key Vault, then submitted. Manages chain-specific account deployment across 7 chain families (EVM, Solana, Aptos, Sui, NEAR, TRON, TON).

### Key Vault (`key-vault-signer` - TypeScript/React)

A Chrome Manifest V3 extension that stores cryptographic keys securely in the browser. Keys are encrypted at rest with AES-256-GCM using PBKDF2-SHA512 key derivation (600K iterations). Supports Ed25519, secp256k1, and BLS12-381 key types with HD wallet derivation (BIP-39, BIP-44/SLIP-0010). Generates addresses for 9 chain types. Private keys never leave the extension; the Web App sends transaction hashes for signing via the `window.certen` message protocol.

### Pending Service (`certen-pending-service` - TypeScript)

A background polling service that discovers multi-signature transactions on Accumulate that require additional signatures. Runs a 4-phase discovery algorithm: resolves signing paths for user keys, processes pending accounts, scans signature chains, and deduplicates results. Syncs discovered pending actions to Firestore so the Web App can display them to signers in real-time.

### Proofs Service (`proofs_service` - Go)

Stores, indexes, and serves proof artifacts generated by validators. Provides 20+ REST endpoints for querying proofs, proof bundles, anchor records, attestations, and intent status. Includes a React/Vite/Ant Design frontend (Proof Explorer) for browsing and verifying proofs visually. Uses PostgreSQL for storage with a repository pattern.

### Smart Contracts (`certen-contracts` - Solidity/Rust/Move)

On-chain components deployed to destination blockchains. The core EVM contracts include: `CertenAnchorV3` (stores Accumulate state roots with Merkle verification), `BLSZKVerifier` (verifies Groth16 ZK-SNARK proofs of BLS aggregate signatures at ~220K gas), `CertenAccountV2/V3` (ERC-4337 account abstraction linked to ADI governance), and `CertenAccountFactory` (deterministic CREATE2 deployment). Solana (Anchor), CosmWasm, NEAR, and Move platforms are in various stages of development.

### Independent Miner (`independant_miner` - Go)

Audits validator behavior using LXR memory-hard proof-of-work. Miners fetch proof bundles from validators, verify their correctness, and publish attestations via a LibP2P P2P network. Provides economic security by ensuring validators cannot publish invalid proofs without detection.

## Inter-Repository Dependencies

| Source | Target | Protocol | Purpose |
|--------|--------|----------|---------|
| Web App | API Bridge | HTTP (port 8085) | ADI operations, intent submission |
| Web App | Key Vault | `window.certen` message API | Transaction signing |
| Web App | Firestore | Firebase SDK | Real-time pending actions, user data |
| Key Vault | (internal) | Web Crypto API | Key storage, signing |
| API Bridge | Accumulate | HTTP (v2/v3 API) | Transaction construction, submission |
| API Bridge | Target Chains | RPC | Account deployment, balance queries |
| Pending Service | Accumulate | HTTP (v2/v3 API) | Pending transaction discovery |
| Pending Service | Firestore | firebase-admin | Sync pending actions |
| Validators | Accumulate | HTTP (v2/v3 API) | State queries, proof data |
| Validators | Validators | CometBFT P2P (26656) | BFT consensus |
| Validators | Proofs Service | HTTP (port 8080) | Proof artifact storage |
| Validators | Target Chains | RPC | Anchor transaction submission |
| Validators | Smart Contracts | On-chain | Proof anchoring, verification |
| Miners | Validators | HTTP | Proof bundle fetching |
| Miners | Miners | LibP2P (4001) | Audit attestation gossip |

## Technology Summary

| Repository | Language | Runtime | Database | Key Libraries |
|-----------|----------|---------|----------|---------------|
| `independant_validator` | Go 1.24 | Native binary | PostgreSQL | CometBFT, gnark, go-ethereum |
| `certen-pending-service` | TypeScript | Node.js 18+ | Firestore | firebase-admin, axios |
| `api-bridge` | TypeScript | Node.js 18+ | None (stateless) | Express, ethers, chain SDKs |
| `key-vault-signer` | TypeScript/React | Chrome Extension | IndexedDB/localStorage | Web Crypto, @noble/curves, Webpack |
| `proofs_service` | Go 1.21 | Native binary | PostgreSQL | net/http, database/sql |
| `certen-web-app` | TypeScript/React | Browser (Vite) | Firestore | React 18, MUI, Firebase |
| `certen-contracts` | Solidity/Rust/Move | EVM/SVM/CosmWasm | On-chain state | Foundry, Anchor, OpenZeppelin |
| `independant_miner` | Go 1.23 | Native binary | None | LibP2P, LXR |
