# Consensus and Proofs

This document covers the CometBFT consensus mechanics used by Certen validators and the 9-phase proof cycle that generates cryptographic evidence for cross-chain anchoring.

## CometBFT Consensus

Certen validators use CometBFT (formerly Tendermint) for Byzantine Fault Tolerant consensus. The consensus algorithm provides instant finality: once a block is committed, it is final and cannot be reverted.

### Consensus Rounds

Each block goes through four stages:

```
Propose -> Prevote -> Precommit -> Commit
   |          |           |          |
   v          v           v          v
 Proposer   All vote    All vote   Block is
 creates    on block    to commit  finalized
 block      validity    block      and applied
```

**Propose**: A designated proposer (rotated round-robin among validators) creates a block containing proof operations. The proposer runs `PrepareProposal` to select which intents to process and which proof phases to execute.

**Prevote**: All validators receive the proposed block and run `ProcessProposal` to verify its validity. Each validator sends a prevote (yes/no) based on whether the proposed operations are valid.

**Precommit**: If 2/3+ validators prevoted yes, each validator sends a precommit. This stage ensures that a supermajority agrees on the block.

**Commit**: If 2/3+ validators precommitted, the block is finalized. All validators run `FinalizeBlock` to execute the proof operations and update their local state.

### BFT Threshold

With 7 validators, the BFT threshold requires 5 (ceil(7 * 2/3) + 1) validators to agree:

| Total Validators | Byzantine Tolerance | Required Agreement |
|-----------------|--------------------|--------------------|
| 4 | 1 faulty | 3 must agree |
| 7 | 2 faulty | 5 must agree |
| 10 | 3 faulty | 7 must agree |

### Validator Set Management

The validator set is configured at genesis and can be updated through governance proposals. Each validator has:

- **CometBFT key**: Ed25519 key for consensus voting
- **BLS key**: BLS12-381 key for proof bundle signing
- **Voting power**: Weight in consensus (typically equal across validators)

## 9-Phase Proof Cycle

The proof cycle generates a chain of cryptographic evidence from individual account state on Accumulate through to an on-chain anchor on a destination blockchain.

```
+--------+    +--------+    +--------+    +--------+
|   L1   |--->|   L2   |--->|   L3   |--->|   L4   |
| Account|    |  BPT   |    |  Root  |    |   DN   |
| Merkle |    |  Root  |    | Anchor |    | Anchor |
+--------+    +--------+    +--------+    +--------+
                                               |
                                               v
+--------+    +--------+    +--------+    +--------+
|   G0   |--->|   G1   |--->|   G2   |    | (from  |
| Key    |    | Key    |    | Auth   |    |  L4)   |
| Page   |    | Book   |    | Chain  |    +--------+
+--------+    +--------+    +--------+
                                 |
                                 v
                          +--------+    +----------+
                          |  BLS   |--->|  Anchor  |
                          | Aggr.  |    | On-Chain |
                          +--------+    +----------+
```

### Phase L1: Account Merkle

**Purpose**: Prove that a specific account's state hash is included in its partition's account tree.

**Process**:
1. Query Accumulate API for the target account state (e.g., `acc://organization.acme/intents`)
2. Retrieve the account's state hash from the response
3. Obtain the Merkle tree of the account's partition
4. Construct an inclusion proof (list of sibling hashes from leaf to root)

**Output**: `MerkleProof{leaf: accountHash, path: []Hash, root: partitionAccountRoot}`

### Phase L2: BPT (Binary Patricia Trie)

**Purpose**: Prove that the partition's account root is included in the partition's BPT root.

**Process**:
1. Using the partition account root from L1
2. Query the BPT structure for the partition
3. Construct a BPT inclusion proof

**Output**: `BPTProof{key: partitionAccountRoot, path: []BPTNode, root: partitionBPTRoot}`

### Phase L3: Root Anchor

**Purpose**: Prove that the partition's BPT root is anchored into the Directory Network.

**Process**:
1. Using the partition BPT root from L2
2. Query the DN for the partition's anchor record
3. Construct a proof that the partition root was included in a DN anchor batch

**Output**: `AnchorProof{partitionRoot: bptRoot, anchorEntry: DNAnchorRecord}`

### Phase L4: DN Anchor

**Purpose**: Prove that the DN state root (containing all partition anchors) is final.

**Process**:
1. Using the DN anchor record from L3
2. Verify that the DN block containing this anchor has been finalized
3. Capture the DN state root hash at the anchor height

**Output**: `DNProof{dnRoot: stateRootHash, height: blockHeight, timestamp: blockTimestamp}`

### Phase G0: Key Page

**Purpose**: Prove the state of the key page that authorized the operation.

**Process**:
1. Query Accumulate for the key page state (e.g., `acc://organization.acme/book0/1`)
2. Extract: list of public keys, signature threshold, key page version
3. Construct a Merkle proof of the key page state in its partition

**Output**: `KeyPageProof{keys: []PublicKey, threshold: uint, version: uint, merkleProof: MerkleProof}`

### Phase G1: Key Book

**Purpose**: Prove the structure of the key book containing the key page.

**Process**:
1. Query Accumulate for the key book state (e.g., `acc://organization.acme/book0`)
2. Extract: list of key pages, key book URL, authority settings
3. Construct a Merkle proof of the key book state

**Output**: `KeyBookProof{pages: []URL, authority: URL, merkleProof: MerkleProof}`

### Phase G2: Authority

**Purpose**: Prove the full authority chain from ADI to key book to key page.

**Process**:
1. Query the ADI for its authority configuration
2. Resolve any delegation chains (key pages delegating to other ADIs)
3. Construct proofs for each link in the authority chain

**Output**: `AuthorityProof{adi: URL, authorities: []Authority, delegations: []DelegationProof}`

### Phase BLS: Aggregate Signature

**Purpose**: Create a BLS aggregate signature over the proof bundle and generate a ZK proof of its validity.

**Process**:
1. Each validator hashes the complete proof bundle (L1-L4, G0-G2)
2. Each validator signs the hash with its BLS12-381 private key
3. Signatures are collected during CometBFT consensus (attached to prevote/precommit messages)
4. Signatures from 5+ validators are aggregated into a single BLS aggregate signature
5. A gnark Groth16 circuit verifies the aggregate signature and generates a ZK-SNARK proof

**gnark Circuit**:
```
Inputs:
  - message: the proof bundle hash (public)
  - aggregateSignature: BLS aggregate signature (public)
  - validatorPublicKeys: BLS public keys of signing validators (public)

Circuit logic:
  1. Verify BLS pairing: e(aggregateSignature, G2) == e(H(message), sum(validatorPublicKeys))
  2. Output: Groth16 proof (A, B, C points) + public inputs

Verification cost: ~220K gas on EVM (constant regardless of validator count)
```

**Output**: `BLSProof{aggregateSignature: BLSSig, zkProof: Groth16Proof, publicInputs: []uint256}`

### Phase Anchor: On-Chain Commit

**Purpose**: Submit the proof bundle to destination chain smart contracts.

**Process**:
1. Construct the anchor transaction data:
   - `merkleRoot`: SHA-256 hash binding all Merkle proofs
   - `zkProof`: Groth16 proof bytes (A, B, C curve points)
   - `publicInputs`: Array of public inputs for the verifier
2. For each target chain, use the chain strategy to submit the anchor:
   - EVM: Call `CertenAnchorV3.anchor(merkleRoot, zkProof, publicInputs)`
   - Solana: Invoke the Certen program with anchor instruction
   - Others: Chain-specific anchor transactions
3. Wait for transaction confirmation on each chain
4. Record anchor results (tx hash, block number, chain) in PostgreSQL

**Output**: `AnchorResult{chain: string, txHash: Hash, blockNumber: uint64, status: string}`

## Proof Bundle Format

The complete proof bundle generated by a proof cycle:

```
ProofBundle {
  batchId:        UUID            // Unique identifier for this proof cycle
  timestamp:      uint64          // Unix timestamp of generation
  accumulateHeight: uint64        // Accumulate block height at proof time

  merkleProofs: {
    L1: MerkleProof              // Account -> Partition
    L2: BPTProof                 // Partition -> BPT Root
    L3: AnchorProof              // BPT Root -> DN Anchor
    L4: DNProof                  // DN Anchor -> DN Root
  }

  governanceProofs: {
    G0: KeyPageProof             // Key page state
    G1: KeyBookProof             // Key book structure
    G2: AuthorityProof           // Authority chain
  }

  consensus: {
    aggregateSignature: bytes    // BLS12-381 aggregate
    signerBitmap:       uint64   // Bitmap of which validators signed
    zkProof:            bytes    // Groth16 proof of BLS validity
    publicInputs:       []bytes  // Verifier public inputs
  }

  anchors: [{
    chain:       string          // Target chain name
    txHash:      bytes           // Anchor transaction hash
    blockNumber: uint64          // Confirmation block number
    contractAddr: bytes          // CertenAnchorV3 address on this chain
  }]

  merkleRoot:    bytes32         // SHA-256 root binding all proofs
}
```

## Per-Chain Anchoring

Each chain has specific anchoring mechanics:

| Chain Type | Transaction Format | Finality | Typical Gas/Cost |
|-----------|-------------------|----------|-----------------|
| EVM (Ethereum) | `anchor(root, proof, inputs)` | ~12 min (64 blocks) | ~250K gas |
| EVM (L2s) | Same as Ethereum | ~2 sec (1 block) | ~250K gas (cheaper) |
| Solana | Program instruction | ~0.4 sec | ~5000 compute units |
| Cosmos | `ExecuteContract` msg | ~6 sec | ~200K gas |
| Aptos | Move function call | ~1 sec | ~1000 gas units |
| Sui | Move transaction | ~2 sec | ~1000 gas units |
| NEAR | `FunctionCall` action | ~1 sec | ~10 TGas |
| TRON | `TriggerSmartContract` | ~3 sec | ~250K energy |
