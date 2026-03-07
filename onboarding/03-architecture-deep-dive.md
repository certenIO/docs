# Architecture Deep Dive

This document explains the internal architecture of the Certen Protocol, covering system layers, trust boundaries, the proof cycle, governance model, and intent protocol.

## System Layers

The Certen Protocol is organized into four layers, each with distinct responsibilities:

```
+-----------------------------------------------------------------------+
|  USER LAYER                                                            |
|  Browser-based interaction, key management, transaction approval       |
|  [Web App]  [Key Vault Extension]                                      |
+-----------------------------------------------------------------------+
        |  HTTP, window.certen API, Firestore realtime
        v
+-----------------------------------------------------------------------+
|  API LAYER                                                             |
|  Transaction construction, chain abstraction, proof retrieval          |
|  [API Bridge :8085]  [Proofs Service :8080]  [Pending Service]         |
+-----------------------------------------------------------------------+
        |  Accumulate API, PostgreSQL, Firestore
        v
+-----------------------------------------------------------------------+
|  PROTOCOL LAYER                                                        |
|  BFT consensus, proof generation, audit verification                   |
|  [Validators x7 (CometBFT)]  [Miners (LXR P2P)]                      |
+-----------------------------------------------------------------------+
        |  RPC, on-chain transactions
        v
+-----------------------------------------------------------------------+
|  BLOCKCHAIN LAYER                                                      |
|  State persistence, smart contract verification                        |
|  [Accumulate]  [Ethereum/EVM]  [Solana]  [Cosmos]  [Others]           |
+-----------------------------------------------------------------------+
```

## Trust Model and Security Boundaries

### Boundary 1: Key Vault (User Trust Domain)

Private keys never leave the Chrome extension. The Web App cannot access key material directly; it can only request signatures via the `window.certen` message protocol. Keys are encrypted at rest with AES-256-GCM, derived via PBKDF2-SHA512 with 600K iterations.

**Trust assumption**: The user's browser environment is not compromised.

### Boundary 2: Validator BFT Consensus (Protocol Trust Domain)

The validator network operates under BFT assumptions: the system is correct as long as fewer than 1/3 of validators are Byzantine. Proof bundles require 2/3+ validator signatures (BLS aggregate) before anchoring. A single malicious validator cannot forge proofs.

**Trust assumption**: At least 5 of 7 validators are honest.

### Boundary 3: Smart Contracts (On-Chain Trust Domain)

Smart contracts on destination chains verify proofs cryptographically. `CertenAnchorV3` verifies Merkle proofs against anchored state roots. `BLSZKVerifier` verifies Groth16 ZK-SNARK proofs of BLS aggregate signatures. The contracts are pausable and access-controlled.

**Trust assumption**: The destination blockchain consensus is sound.

### Boundary 4: Miner Audit (Economic Trust Domain)

Miners independently verify validator proofs using LXR proof-of-work. They publish attestations via P2P gossip. Invalid proofs are detectable and reportable, creating economic disincentives for validator misbehavior.

**Trust assumption**: Sufficient mining participation to detect invalid proofs.

## The 9-Phase Proof Cycle

Validators execute a repeating proof cycle that builds a chain of cryptographic evidence from individual account state all the way to an on-chain anchor:

```
Phase:  L1 --> L2 --> L3 --> L4 --> G0 --> G1 --> G2 --> BLS --> Anchor
        |      |      |      |      |      |      |      |       |
        v      v      v      v      v      v      v      v       v
      Account BPT   Root   DN    Key    Key   Auth   Aggregate  On-chain
      Merkle  Root  Anchor Anchor Page   Book  Chain  Signature  Commit
```

### Merkle Proof Phases (L1-L4)

These phases prove that a specific account state is included in the global Accumulate state:

**L1 - Account Merkle**: Proves that a specific account's state hash is included in its partition's account tree. The validator queries the Accumulate API for the account and constructs a Merkle inclusion proof against the partition's account root.

**L2 - BPT (Binary Patricia Trie)**: Proves that the partition's account root is included in the partition's BPT root hash. This links individual accounts to the partition-level state commitment.

**L3 - Root Anchor**: Proves that the partition's BPT root is anchored into the Directory Network. Each Accumulate partition periodically submits its root to the DN.

**L4 - DN Anchor**: Proves that the DN's state root (which includes all partition anchors) is final and agreed upon by the Accumulate network. This is the top-level commitment that represents the entire Accumulate state.

### Governance Proof Phases (G0-G2)

These phases prove the governance structure that authorizes an action:

**G0 - Key Page**: Proves the state of a specific key page, including which public keys it contains and the signature threshold (e.g., 2-of-3).

**G1 - Key Book**: Proves the structure of the key book containing the key page, including how many pages it has and their relationships.

**G2 - Authority**: Proves the full authority chain from the ADI to the key book to the key page, including any delegation relationships to other ADIs.

### Aggregation and Anchoring

**BLS - Aggregate Signature**: Each validator signs the proof bundle with its BLS12-381 private key. Individual signatures are aggregated into a single BLS aggregate signature. A gnark circuit generates a Groth16 ZK-SNARK proof that verifies the aggregate signature, enabling efficient on-chain verification (~220K gas instead of millions).

**Anchor - On-Chain Commit**: The final proof bundle (Merkle roots, governance proofs, ZK proof of BLS aggregate signature) is submitted to the `CertenAnchorV3` smart contract on each destination chain. The contract stores the Merkle root and emits events for indexing.

## ADI Governance Model

Accumulate Digital Identifiers support hierarchical governance through key books, key pages, and delegation:

```
acc://organization.acme                    (ADI - Root Identity)
|
+-- /book0                                 (Key Book - Primary Authority)
|   |
|   +-- /1                                 (Key Page - Threshold: 2-of-3)
|   |   +-- key: ed25519:0x1234...         (Public Key - Member A)
|   |   +-- key: ed25519:0x5678...         (Public Key - Member B)
|   |   +-- key: secp256k1:0x9abc...       (Public Key - Member C)
|   |
|   +-- /2                                 (Key Page - Admin: 1-of-1)
|       +-- delegate: acc://admin.acme/book0
|
+-- /book1                                 (Key Book - Secondary Authority)
|   +-- /1                                 (Key Page - Threshold: 1-of-1)
|       +-- key: ed25519:0xdef0...         (Public Key - Service Account)
|
+-- /tokens                                (Token Account)
+-- /data                                  (Data Account - Intent Storage)
+-- /intents                               (Data Account - Cross-Chain Intents)
```

### Governance Rules

1. **Threshold signatures**: A key page with threshold M-of-N requires M signatures from its N keys to authorize an operation
2. **Delegation**: A key page can delegate authority to another ADI's key book, enabling organizational hierarchies
3. **Priority**: Lower-numbered key pages have higher authority; page /1 can modify page /2 but not vice versa
4. **Key rotation**: Keys can be added, removed, or replaced without changing the ADI's identity or its cross-chain accounts

## Intent Protocol

An **intent** is a structured declaration that an ADI wants to perform a cross-chain action. Intents are stored as data entries on Accumulate and detected by validators during the proof cycle.

### Intent Data Structure (4 Blobs)

```
Intent {
  Blob 0 - Metadata:
    version:        uint8       // Protocol version
    action_type:    uint8       // 0=transfer, 1=contract_call, 2=deploy, 3=governance
    target_chain:   string      // e.g., "ethereum", "solana", "arbitrum"
    timestamp:      uint64      // Creation timestamp

  Blob 1 - Cross-Chain Parameters:
    destination:    bytes       // Target address on destination chain
    value:          uint256     // Native token amount (wei, lamports, etc.)
    calldata:       bytes       // Contract call data (ABI-encoded for EVM)
    gas_limit:      uint64      // Maximum gas/compute units

  Blob 2 - Governance:
    authority:      string      // ADI authority URL (e.g., acc://org.acme/book0)
    required_sigs:  uint8       // Number of required signatures
    signers:        []bytes     // Public keys of required signers
    threshold:      uint8       // Signature threshold

  Blob 3 - Replay Protection:
    nonce:          uint64      // Unique per-ADI sequence number
    expiry:         uint64      // Unix timestamp after which intent is invalid
    chain_nonce:    uint64      // Per-chain sequence for ordering
    intent_hash:    bytes32     // SHA-256 hash of blobs 0-2
}
```

### Intent Lifecycle

```
1. CREATE     User creates intent via Web App -> API Bridge -> Accumulate data entry
2. DETECT     Validator detects new intent during proof cycle scan
3. VALIDATE   Validator verifies governance (signatures meet threshold)
4. PROVE      9-phase proof cycle generates cryptographic evidence
5. ANCHOR     Proof bundle anchored to destination chain smart contract
6. EXECUTE    CertenAccount contract on destination chain executes the intent
7. CONFIRM    Proofs Service records execution status
```

## Multi-Chain Account Abstraction

Certen maps ADIs to accounts on destination chains using deterministic addressing:

```
ADI: acc://organization.acme
    |
    +-- Ethereum:  0x7a3b... (CertenAccountV3 via CREATE2)
    +-- Arbitrum:  0x7a3b... (same address, same factory + salt)
    +-- Optimism:  0x7a3b... (same address across all EVM chains)
    +-- Base:      0x7a3b...
    +-- Polygon:   0x7a3b...
    +-- Solana:    Cert...   (PDA derived from ADI hash)
    +-- Cosmos:    certen1... (contract instantiation)
    +-- Aptos:     0x8f2c... (Move module account)
    +-- Sui:       0x9d4e... (Move object)
    +-- NEAR:      org-acme.certen.near (named account)
    +-- TRON:      T...     (sponsored deployment)
    +-- TON:       EQ...    (sponsored deployment)
```

### EVM Account Architecture

On EVM chains, each ADI maps to a `CertenAccountV3` smart contract deployed via `CertenAccountFactory` using CREATE2. The account:

1. Implements ERC-4337 for account abstraction (gasless transactions via paymasters)
2. Validates intents against the anchored Accumulate governance state
3. Executes arbitrary calls (transfers, contract interactions) when governance requirements are met
4. Uses the same deterministic address across all EVM chains (same factory address + same salt = same account address)

## Life of a Cross-Chain Intent (End-to-End)

```
User                Web App        Key Vault      API Bridge     Accumulate
 |                    |               |               |              |
 |  create intent     |               |               |              |
 |------ click ------>|               |               |              |
 |                    |-- prepare tx ----------------->|              |
 |                    |               |               |-- build ----->|
 |                    |               |               |<-- hash ------|
 |                    |<-- tx hash -------------------|              |
 |                    |-- sign req -->|               |              |
 |  approve           |               |               |              |
 |------ click ------>|               |               |              |
 |                    |               |-- signature -->|              |
 |                    |               |               |-- submit ---->|
 |                    |               |               |<-- confirmed -|
 |                    |<-- success -------------------|              |
 |                    |               |               |              |

Validators (x7)     CometBFT       Smart Contract   Proofs Service
 |                    |               |               |
 |-- detect intent -->|               |               |
 |-- propose block -->|               |               |
 |                    |-- consensus ->|               |
 |-- 9-phase proof -->|               |               |
 |-- BLS aggregate -->|               |               |
 |-- ZK proof ------->|               |               |
 |                    |               |               |
 |-- anchor tx ---------------------->|               |
 |                    |               |-- verify ZK -->|
 |                    |               |-- store root ->|
 |                    |               |               |
 |-- store artifacts -------------------------------->|
 |                    |               |               |-- index ----->
 |                    |               |               |
 |                    |               |               |
 |-- execute intent ----------------->|               |
 |                    |               |-- call target  |
 |                    |               |   contract     |
 |                    |               |<-- result      |
 |                    |               |               |
```

The entire flow from user intent creation to cross-chain execution is trustless: the user signs only in Key Vault, validators reach BFT consensus, proofs are verified on-chain by smart contracts, and execution is atomic on the destination chain.
