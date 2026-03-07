# API Bridge

The API Bridge is the HTTP gateway between frontend applications and the Accumulate network, providing REST endpoints for ADI management, intent submission, two-phase signing, and multi-chain account deployment.

## API Endpoint Summary

### ADI Operations

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/adi/prepare` | POST | Prepare ADI creation transaction, return hash for signing |
| `/adi/submit` | POST | Submit signed ADI creation transaction |
| `/adi/info` | GET | Get ADI details (key books, accounts, authorities) |
| `/adi/list` | GET | List ADIs associated with a public key |
| `/adi/accounts` | GET | List sub-accounts of an ADI |

### Key Management

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/key/page/prepare` | POST | Prepare key page update (add/remove keys) |
| `/key/page/submit` | POST | Submit signed key page update |
| `/key/book/prepare` | POST | Prepare key book creation |
| `/key/book/submit` | POST | Submit signed key book creation |
| `/key/rotate/prepare` | POST | Prepare key rotation |
| `/key/rotate/submit` | POST | Submit signed key rotation |

### Credits

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/credits/purchase/prepare` | POST | Prepare credit purchase |
| `/credits/purchase/submit` | POST | Submit signed credit purchase |
| `/credits/balance` | GET | Query credit balance |

### Intents

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/intent/create/prepare` | POST | Prepare cross-chain intent |
| `/intent/create/submit` | POST | Submit signed intent |
| `/intent/status` | GET | Check intent execution status |
| `/intent/list` | GET | List intents for an ADI |

### Chain Operations

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/chain/deploy/prepare` | POST | Prepare account deployment on target chain |
| `/chain/deploy/submit` | POST | Submit signed deployment |
| `/chain/balance` | GET | Query balance on target chain |
| `/chain/address` | GET | Get deterministic address for ADI on chain |
| `/chain/supported` | GET | List supported chains and their status |

### Network

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/network/status` | GET | Accumulate network status |
| `/network/version` | GET | API Bridge version |
| `/health` | GET | Health check |
| `/tx/status` | GET | Transaction status by hash |
| `/tx/prepare` | POST | Prepare arbitrary transaction |
| `/tx/submit` | POST | Submit signed transaction |

## Two-Phase Signing Protocol

The API Bridge implements two-phase signing so that private keys never leave the Key Vault extension:

**Phase 1 - Prepare**:
1. Client sends transaction parameters to a `/prepare` endpoint
2. API Bridge constructs the Accumulate transaction envelope
3. API Bridge computes the transaction hash (SHA-256 of the envelope body)
4. Returns: `{txHash, envelope, expiresAt}`

**Phase 2 - Submit**:
1. Client signs `txHash` in Key Vault (Ed25519 signature)
2. Client sends `{envelope, signature, publicKey}` to the `/submit` endpoint
3. API Bridge attaches the external signature to the envelope
4. API Bridge submits the complete transaction to Accumulate
5. Returns: `{txId, status, deliveryStatus}`

## Configuration

Key environment variables from `.env.example`:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8085` | HTTP server port |
| `ACCUM_ENDPOINT` | `http://206.191.154.164/v3` | Accumulate v3 API |
| `ACCUM_ENDPOINT_V2` | `http://206.191.154.164/v2` | Accumulate v2 API |
| `ACCUM_PUBLIC_KEY` | (required) | Accumulate principal public key |
| `ACCUM_PRIV_KEY` | (required) | Accumulate principal private key |
| `ETHEREUM_URL` | (required) | Ethereum RPC endpoint |
| `ANCHOR_CONTRACT_ADDRESS` | (required) | CertenAnchorV3 address |
| `ACCOUNT_FACTORY_ADDRESS` | (required) | CertenAccountFactory address |
| `SOLANA_RPC` | (optional) | Solana RPC endpoint |
| `APTOS_RPC` | (optional) | Aptos REST endpoint |
| `SUI_RPC` | (optional) | Sui JSON-RPC endpoint |
| `NEAR_RPC` | (optional) | NEAR JSON-RPC endpoint |
| `TRON_API` | (optional) | TRON HTTP API endpoint |
| `TON_RPC` | (optional) | TON HTTP API endpoint |
| `CORS_ORIGIN` | `*` | CORS allowed origins |

## Setup

```bash
cd ~/certen/api-bridge
cp .env.example .env
# Edit .env with your configuration
npm install
npm run build
npm start

# Development mode with hot reload
npm run dev
```

## Related Documentation

- [Architecture](architecture.md) - Internal structure and design
- [Ecosystem Overview](../onboarding/02-ecosystem-overview.md) - How API Bridge fits in the protocol
- [Data Flow Walkthrough](../onboarding/04-data-flow-walkthrough.md) - End-to-end scenarios
