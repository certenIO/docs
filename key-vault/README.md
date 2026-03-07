# Key Vault

The Key Vault is a Chrome Manifest V3 extension that provides secure client-side key storage and transaction signing for the Certen Protocol.

## Security Model

- **Keys never leave the extension**: Private keys are stored in the extension's isolated storage and are never exposed to web pages or network requests
- **Encryption at rest**: All keys encrypted with AES-256-GCM
- **Key derivation**: PBKDF2-SHA512 with 600,000 iterations (OWASP 2023 recommendation)
- **Lock/unlock lifecycle**: Vault locks automatically after inactivity timeout; requires password to unlock

## Supported Key Types

| Key Type | Curve | Usage |
|----------|-------|-------|
| Ed25519 | Curve25519 | Accumulate transaction signing |
| secp256k1 | secp256k1 | EVM transaction signing |
| BLS12-381 | BLS12-381 | Validator consensus (advanced) |

## HD Wallet

Key derivation follows industry standards:

- **Mnemonic**: BIP-39 (12 or 24 words)
- **Derivation paths**: BIP-44 / SLIP-0010
- **Account structure**: `m/44'/{coinType}'/0'/0/{index}`

| Chain Type | Coin Type | Derivation Path |
|-----------|-----------|-----------------|
| Accumulate | 281 | `m/44'/281'/0'/0/0` |
| Ethereum/EVM | 60 | `m/44'/60'/0'/0/0` |
| Solana | 501 | `m/44'/501'/0'/0/0` |
| Cosmos | 118 | `m/44'/118'/0'/0/0` |
| Aptos | 637 | `m/44'/637'/0'/0/0` |
| Sui | 784 | `m/44'/784'/0'/0/0` |
| NEAR | 397 | `m/44'/397'/0'/0/0` |
| TRON | 195 | `m/44'/195'/0'/0/0` |
| TON | 607 | `m/44'/607'/0'/0/0` |

## Multi-Chain Address Generation

From a single mnemonic, the Key Vault derives keys and generates addresses for 9 chain types. Each chain type uses its native address encoding:

- **Accumulate**: Ed25519 public key hex
- **Ethereum/EVM**: `0x` + keccak256(secp256k1 pubkey)[12:]
- **Solana**: Base58-encoded Ed25519 public key
- **Cosmos**: Bech32-encoded (`cosmos1...`)
- **Aptos**: `0x` + SHA3-256(Ed25519 pubkey + 0x00)
- **Sui**: `0x` + Blake2b(flag + Ed25519 pubkey)
- **NEAR**: Implicit account (hex-encoded Ed25519 pubkey)
- **TRON**: Base58Check-encoded (starts with `T`)
- **TON**: Workchain-aware address format

## Setup

```bash
cd ~/certen/key-vault-signer
npm install
npm run build
```

Load the extension in Chrome:
1. Navigate to `chrome://extensions/`
2. Enable "Developer mode" (toggle in top-right)
3. Click "Load unpacked"
4. Select the `dist/` directory

The extension icon appears in the Chrome toolbar. Click it to create a new vault or import an existing mnemonic.

## Related Documentation

- [Architecture](architecture.md) - Manifest V3 architecture, message protocol, crypto modules
- [Web App](../web-app/README.md) - How the Web App integrates with Key Vault
- [Data Flow Walkthrough](../onboarding/04-data-flow-walkthrough.md) - Two-phase signing in context
