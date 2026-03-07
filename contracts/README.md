# Smart Contracts

The Certen smart contracts provide on-chain anchoring of Accumulate state, BLS ZK proof verification, and ERC-4337 account abstraction for ADI-governed cross-chain accounts.

## Core Contracts (EVM)

| Contract | Purpose | Key Features |
|----------|---------|-------------|
| `CertenAnchorV3` | Stores Accumulate state roots | Merkle root storage, Merkle verification, pausable, access-controlled |
| `BLSZKVerifier` | Verifies Groth16 ZK proofs of BLS aggregate signatures | ~220K gas per verification, stateless |
| `CertenAccountV2` | ERC-4337 smart account linked to ADI governance | Intent execution, governance validation |
| `CertenAccountV3` | Updated account with multi-chain support | Enhanced intent protocol, batch execution |
| `CertenAccountFactory` | Deploys ADI-linked accounts via CREATE2 | Deterministic addresses, ADI-to-account mapping |

## Platform Status

| Platform | Language | Status | Framework |
|----------|----------|--------|-----------|
| EVM (Ethereum, L2s) | Solidity | Production | Foundry |
| Solana | Rust (Anchor) | In development | Anchor |
| CosmWasm | Rust | In development | cosmwasm-std |
| NEAR | Rust | Planned | near-sdk |
| TRON | Solidity | Planned | Hardhat + TronBox |
| TON | FunC | Planned | Blueprint |
| Aptos | Move | Planned | Aptos CLI |
| Sui | Move | Planned | Sui CLI |

## Setup

### EVM (Foundry)

```bash
cd ~/certen/certen-contracts/evm

# Install dependencies
forge install

# Build
forge build

# Test
forge test

# Test with verbosity
forge test -vvv

# Gas report
forge test --gas-report

# Local deployment
anvil &
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast

# Testnet deployment (Sepolia)
forge script script/Deploy.s.sol \
  --rpc-url $SEPOLIA_RPC \
  --private-key $DEPLOYER_KEY \
  --broadcast \
  --verify
```

### Solana (Anchor)

```bash
cd ~/certen/certen-contracts/solana

# Build
anchor build

# Test
anchor test

# Deploy to devnet
anchor deploy --provider.cluster devnet
```

### CosmWasm

```bash
cd ~/certen/certen-contracts/cosmwasm

# Build
cargo build --release --target wasm32-unknown-unknown

# Optimize
docker run --rm -v "$(pwd)":/code cosmwasm/rust-optimizer:0.15.0

# Test
cargo test
```

### NEAR

```bash
cd ~/certen/certen-contracts/near

# Build
cargo build --release --target wasm32-unknown-unknown

# Test
cargo test

# Deploy
near deploy --accountId certen.testnet --wasmFile target/wasm32-unknown-unknown/release/certen.wasm
```

## Related Documentation

- [Architecture](architecture.md) - Contract design, inheritance, deployment scripts
- [Validator Consensus & Proofs](../validator/consensus-and-proofs.md) - How proofs are generated before anchoring
- [Architecture Deep Dive](../onboarding/03-architecture-deep-dive.md) - On-chain trust boundary
