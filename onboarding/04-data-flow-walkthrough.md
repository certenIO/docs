# Data Flow Walkthrough

This document traces four concrete scenarios end-to-end through the Certen Protocol, showing which repositories participate, what protocols are used, and how data transforms at each step.

## Scenario 1: Creating an ADI

A user creates a new Accumulate Digital Identifier through the Web App.

**Participating repos**: `certen-web-app`, `key-vault-signer`, `api-bridge`
**Protocols**: HTTP, `window.certen` message API, Accumulate v3 API

```
User         Web App           Key Vault          API Bridge         Accumulate
 |              |                  |                   |                  |
 | fill form    |                  |                   |                  |
 |--- click --->|                  |                   |                  |
 |              |                  |                   |                  |
 |              |-- POST /adi/prepare --------------->|                  |
 |              |   {name, keyBook, keyPage, keys}    |                  |
 |              |                  |                   |                  |
 |              |                  |                   |-- build CreateIdentity tx
 |              |                  |                   |-- compute tx hash
 |              |                  |                   |                  |
 |              |<-- {txHash, envelope} --------------|                  |
 |              |                  |                   |                  |
 |              |-- signHash(txHash) ->|               |                  |
 |              |   via window.certen  |               |                  |
 |              |                  |                   |                  |
 | approve      |                  |                   |                  |
 |--- click --->|           popup  |                   |                  |
 |              |                  |                   |                  |
 |              |<-- {signature} --|                   |                  |
 |              |                  |                   |                  |
 |              |-- POST /adi/submit ----------------->|                  |
 |              |   {envelope, signature}              |                  |
 |              |                  |                   |                  |
 |              |                  |                   |-- submit tx ---->|
 |              |                  |                   |   with external  |
 |              |                  |                   |   signature      |
 |              |                  |                   |                  |
 |              |                  |                   |<-- txId, status -|
 |              |                  |                   |                  |
 |              |<-- {success, txId} -----------------|                  |
 |              |                  |                   |                  |
 | see success  |                  |                   |                  |
 |<-------------|                  |                   |                  |
```

**Data transformations**:

1. **Web App**: User form data (ADI name, key types, threshold) packaged into a JSON request
2. **API Bridge (prepare)**: Constructs an Accumulate `CreateIdentity` transaction envelope, computes the transaction hash for signing
3. **Key Vault**: Signs the raw transaction hash with the user's Ed25519 private key, returns the signature bytes
4. **API Bridge (submit)**: Attaches the external signature to the transaction envelope and submits to Accumulate's v3 API
5. **Accumulate**: Validates the transaction, creates the ADI with its key book and key page on-chain

## Scenario 2: Multi-Sig Pending Approval

A 2-of-3 multi-sig ADI has a pending transaction that needs a second signature.

**Participating repos**: `certen-web-app`, `key-vault-signer`, `api-bridge`, `certen-pending-service`
**Protocols**: HTTP, Firestore realtime, `window.certen` message API, Accumulate v2/v3 API

```
Signer A     Web App (A)     API Bridge     Accumulate     Pending Service
 |              |                |              |                |
 | initiate tx  |                |              |                |
 |--- sign ---->|-- prepare ---->|-- build ---->|                |
 |              |<-- hash -------|              |                |
 |              |-- sign (KV) -->|              |                |
 |              |-- submit ----->|-- submit --->|                |
 |              |                |              |                |
 |              |                |              | tx pending     |
 |              |                |              | (1 of 2 sigs)  |
 |              |                |              |                |
 |              |                |              |<-- poll -------|
 |              |                |              |   /v2/query    |
 |              |                |              |                |
 |              |                |              |-- pending ---->|
 |              |                |              |   txs found    |
 |              |                |              |                |
 |              |                |              |   Firestore    |
 |              |                |              |       |        |
 |              |                |              |       v        |

Signer B     Web App (B)     Key Vault (B)   Firestore      Accumulate
 |              |                |              |                |
 |              |<-- realtime listener --------|                |
 |              |   pendingActions update      |                |
 |              |                |              |                |
 | see pending  |                |              |                |
 |<-------------|                |              |                |
 |              |                |              |                |
 | approve      |                |              |                |
 |--- click --->|                |              |                |
 |              |-- signHash --->|              |                |
 |              |                |              |                |
 | confirm      |                |              |                |
 |--- click --->|<-- signature --|              |                |
 |              |                |              |                |
 |              |-- POST /tx/submit ---------->|-- add sig ---->|
 |              |                |              |                |
 |              |                |              | threshold met  |
 |              |                |              | tx executes    |
 |              |                |              |                |
 |              |<-- success ------------------|                |
```

**Data transformations**:

1. **Signer A's Web App**: Initiates transaction via two-phase signing (same as Scenario 1)
2. **Accumulate**: Records transaction with 1-of-2 required signatures, marks as pending
3. **Pending Service**: Polls Accumulate's `/v2/query` endpoint, discovers the pending transaction through its 4-phase discovery algorithm:
   - Phase 1: Resolve signing paths for the user's keys
   - Phase 2: Process pending accounts found in signing paths
   - Phase 3: Scan signature chains for partial signatures
   - Phase 4: Deduplicate and merge results
4. **Firestore**: Pending Service writes to `/users/{uid}/pendingActions/{actionId}` with transaction details, required signers, and current signature count
5. **Signer B's Web App**: Firestore realtime listener triggers UI update showing the pending action
6. **Key Vault (B)**: Signs the same transaction hash with Signer B's private key
7. **Accumulate**: Receives second signature, threshold (2-of-2) met, transaction executes

## Scenario 3: Cross-Chain Intent and Proof Anchoring

A user creates an intent to transfer tokens on Ethereum, and validators generate proofs and anchor them.

**Participating repos**: `certen-web-app`, `key-vault-signer`, `api-bridge`, `independant_validator`, `proofs_service`, `certen-contracts`
**Protocols**: HTTP, CometBFT P2P, Accumulate API, Ethereum RPC, PostgreSQL

```
User        Web App      API Bridge     Accumulate     Validators (x7)
 |             |              |              |              |
 | create      |              |              |              |
 | intent      |              |              |              |
 |-- form ---->|              |              |              |
 |             |-- prepare -->|              |              |
 |             |   intent     |-- write ---->|              |
 |             |              |  data entry  |              |
 |             |-- sign (KV)->|              |              |
 |             |-- submit --->|-- submit --->|              |
 |             |              |              |              |
 |             |              |              | intent stored |
 |             |              |              | in data acct  |
 |             |              |              |              |
 |             |              |              |<-- scan ------|
 |             |              |              |   new intents |
 |             |              |              |              |
 |             |              |              |-- intent ---->|
 |             |              |              |   data        |
 |             |              |              |              |

Validators (x7)           CometBFT          Smart Contract    Proofs Service
 |                           |                   |                |
 |-- validate governance     |                   |                |
 |   (check sigs >= threshold)                   |                |
 |                           |                   |                |
 |-- Phase L1: Account Merkle proof              |                |
 |-- Phase L2: BPT root proof                    |                |
 |-- Phase L3: Root anchor proof                 |                |
 |-- Phase L4: DN anchor proof                   |                |
 |-- Phase G0: Key page state proof              |                |
 |-- Phase G1: Key book structure proof          |                |
 |-- Phase G2: Authority chain proof             |                |
 |                           |                   |                |
 |-- propose block --------->|                   |                |
 |                           |-- prevote ------->|                |
 |                           |-- precommit ----->|                |
 |                           |-- commit -------->|                |
 |                           |                   |                |
 |-- Phase BLS: sign with BLS key               |                |
 |-- aggregate BLS signatures                    |                |
 |-- generate Groth16 ZK proof (gnark)           |                |
 |                           |                   |                |
 |-- Phase Anchor: submit tx ----------------->  |                |
 |   (merkleRoot, zkProof, pubInputs)            |                |
 |                           |                   |                |
 |                           |   CertenAnchorV3: |                |
 |                           |   - verify ZK proof                |
 |                           |   - store merkleRoot               |
 |                           |   - emit AnchorEvent               |
 |                           |                   |                |
 |-- store proof artifacts ------------------------------------->|
 |   (batch, proofs, anchors, attestations)       |               |
 |                           |                   |                |
 |                           |                   |                |
 |-- execute intent via CertenAccount ---------->|                |
 |   (verify Merkle proof against stored root)   |                |
 |   (execute: transfer tokens to destination)   |                |
```

**Data transformations**:

1. **Web App/API Bridge**: Intent structured as 4-blob data entry (metadata, cross-chain params, governance, replay protection), written to ADI's data account on Accumulate
2. **Validators**: Detect new intent, validate governance requirements (signatures meet threshold), begin 9-phase proof cycle
3. **CometBFT**: Validators propose a block containing the proof data, reach consensus through prevote/precommit/commit rounds
4. **BLS Aggregation**: Each validator signs the proof bundle hash with its BLS12-381 key; signatures aggregated into one; gnark generates Groth16 proof of valid aggregation
5. **Smart Contract (Anchor)**: `CertenAnchorV3` receives the Merkle root and ZK proof, verifies the Groth16 proof (~220K gas), stores the Merkle root, emits `AnchorEvent`
6. **Smart Contract (Execute)**: `CertenAccountV3` verifies the intent's Merkle proof against the stored root, then executes the cross-chain action (e.g., ERC-20 transfer)
7. **Proofs Service**: Validator stores all proof artifacts (batch metadata, individual proofs, anchor records, attestations) in PostgreSQL via REST API

## Scenario 4: Proof Verification

A user or auditor verifies a proof through the Proof Explorer.

**Participating repos**: `proofs_service` (frontend + backend), `certen-contracts`
**Protocols**: HTTP, Ethereum RPC

```
User        Proof Explorer     Proofs Service API    PostgreSQL     Smart Contract
 |              |                    |                   |              |
 | search       |                    |                   |              |
 | proof ID     |                    |                   |              |
 |--- enter --->|                    |                   |              |
 |              |                    |                   |              |
 |              |-- GET /api/v1/proofs/{id} ----------->|              |
 |              |                    |-- SELECT -------->|              |
 |              |                    |<-- proof row -----|              |
 |              |<-- proof details --|                   |              |
 |              |                    |                   |              |
 | view bundle  |                    |                   |              |
 |--- click --->|                    |                   |              |
 |              |-- GET /api/v1/bundles/{batchId} ----->|              |
 |              |                    |-- SELECT -------->|              |
 |              |                    |   batches,        |              |
 |              |                    |   proofs,         |              |
 |              |                    |   anchors,        |              |
 |              |                    |   attestations    |              |
 |              |                    |<-- bundle data ---|              |
 |              |<-- full bundle ----|                   |              |
 |              |                    |                   |              |
 | verify       |                    |                   |              |
 | on-chain     |                    |                   |              |
 |--- click --->|                    |                   |              |
 |              |-- GET /api/v1/verify/{batchId} ------>|              |
 |              |                    |                   |              |
 |              |                    |-- eth_call ---------------------->|
 |              |                    |   verifyProof(                   |
 |              |                    |     merkleRoot,                  |
 |              |                    |     proof,                       |
 |              |                    |     leaf                         |
 |              |                    |   )                              |
 |              |                    |<-- true/false -------------------|
 |              |                    |                   |              |
 |              |<-- verification    |                   |              |
 |              |   result           |                   |              |
 |              |                    |                   |              |
 | see result   |                    |                   |              |
 |<-------------|                    |                   |              |
```

**Data transformations**:

1. **Proof Explorer (React)**: User enters a proof ID or browses recent proofs; frontend makes REST API calls
2. **Proofs Service API**: Queries PostgreSQL for proof records across multiple tables (batches, proofs, anchors, attestations, intents)
3. **Bundle Assembly**: API assembles the full proof bundle from its 4 components:
   - Merkle proofs (L1-L4 chain from account to DN root)
   - Anchor record (transaction hash, chain, block number)
   - Chained proofs (sequential proof hashes for ordering)
   - Governance proofs (G0-G2 authority chain)
4. **On-Chain Verification**: API calls `CertenAnchorV3.verifyProof()` via `eth_call` (read-only, no gas cost) to verify the Merkle proof against the stored root
5. **Proof Explorer**: Displays verification result with full proof breakdown showing each phase's data
