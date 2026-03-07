# API Bridge Architecture

The API Bridge is a TypeScript/Express application that translates HTTP requests into Accumulate transactions and manages multi-chain account deployment.

## Application Structure

```
api-bridge/
├── src/
│   ├── server.ts                  # Express app, route registration, middleware
│   ├── AccumulateService.ts       # Accumulate API client (v2 + v3)
│   ├── CertenIntentService.ts     # Intent creation, 4-blob construction
│   ├── AdiStorageService.ts       # ADI metadata and caching
│   ├── TwoPhaseSigningService.ts  # Prepare/submit transaction flow
│   ├── CreditService.ts          # Credit purchase and balance queries
│   ├── routes/
│   │   ├── adi.ts                 # ADI CRUD endpoints
│   │   ├── key.ts                 # Key management endpoints
│   │   ├── credit.ts              # Credit endpoints
│   │   ├── intent.ts              # Intent endpoints
│   │   ├── chain.ts               # Multi-chain endpoints
│   │   ├── tx.ts                  # Generic transaction endpoints
│   │   └── network.ts             # Network status endpoints
│   ├── chains/
│   │   ├── ChainHandler.ts        # Chain handler interface
│   │   ├── EVMHandler.ts          # EVM chain handler (ethers.js)
│   │   ├── SolanaHandler.ts       # Solana handler (@solana/web3.js)
│   │   ├── AptosHandler.ts        # Aptos handler (aptos SDK)
│   │   ├── SuiHandler.ts          # Sui handler (@mysten/sui.js)
│   │   ├── NEARHandler.ts         # NEAR handler (near-api-js)
│   │   ├── TRONHandler.ts         # TRON handler (tronweb)
│   │   └── TONHandler.ts          # TON handler (ton SDK)
│   ├── middleware/
│   │   ├── cors.ts                # CORS configuration
│   │   ├── errorHandler.ts        # Global error handling
│   │   └── rateLimiter.ts         # Rate limiting
│   └── utils/
│       ├── crypto.ts              # Hashing, encoding utilities
│       └── validation.ts          # Request validation helpers
├── .env.example                   # Configuration template (137 lines)
├── package.json
├── tsconfig.json
└── Dockerfile
```

## Express Server Structure

The server (`server.ts`) initializes middleware, registers route modules, and starts the HTTP listener:

```
Request Flow:
  Client -> CORS -> Rate Limiter -> Route Handler -> Service Layer -> Accumulate/Chain API
                                                                   |
                                                                   v
  Client <- Error Handler <- JSON Response <- Service Result
```

**Middleware stack** (applied in order):
1. `cors()` - CORS headers based on `CORS_ORIGIN` env var
2. `express.json()` - Parse JSON request bodies
3. `rateLimiter()` - Per-IP rate limiting
4. Route-specific handlers
5. `errorHandler()` - Catch-all error formatting

## Service Layer

### AccumulateService

The core client for Accumulate API communication. Wraps both v2 and v3 API calls:

- **v3 API**: Transaction construction, submission, account queries
- **v2 API**: Legacy queries for pending transactions, signature chains
- **Transaction building**: Constructs typed Accumulate transactions (CreateIdentity, CreateTokenAccount, WriteData, etc.)
- **Hash computation**: Computes SHA-256 transaction hashes for two-phase signing

### CertenIntentService

Constructs the 4-blob intent data structure for cross-chain operations:

```
Blob 0 (Metadata):     version, actionType, targetChain, timestamp
Blob 1 (Cross-Chain):  destination, value, calldata, gasLimit
Blob 2 (Governance):   authority, requiredSigs, signers, threshold
Blob 3 (Replay):       nonce, expiry, chainNonce, intentHash
```

The service:
1. Validates intent parameters against the target chain's requirements
2. Resolves the ADI's governance structure to populate Blob 2
3. Generates a unique nonce and computes the intent hash
4. Constructs an Accumulate `WriteData` transaction to store the intent

### AdiStorageService

Caches ADI metadata (key books, key pages, authorities) to reduce Accumulate API calls. Provides lookup by ADI URL or by public key.

### TwoPhaseSigningService

Manages the prepare/submit lifecycle:
- Stores pending prepared transactions with expiration (5 minutes)
- Validates that submitted signatures match the prepared transaction hash
- Attaches external signatures to Accumulate transaction envelopes
- Handles multi-sig scenarios where multiple signatures arrive over time

## Chain Handler System

The `ChainHandler` interface defines operations for each target blockchain:

```typescript
interface ChainHandler {
  name: string;
  deployAccount(adi: string, salt: Buffer): Promise<string>;  // returns address
  getBalance(address: string): Promise<BigNumber>;
  getAddress(adi: string, salt: Buffer): Promise<string>;      // deterministic
  estimateDeployGas(adi: string): Promise<BigNumber>;
  isDeployed(address: string): Promise<boolean>;
}
```

### EVMHandler

Shared implementation for all 7 EVM chains. Uses ethers.js to:
- Deploy `CertenAccountV3` via `CertenAccountFactory.createAccount()` (CREATE2)
- Query balances via `provider.getBalance()`
- Compute deterministic addresses via `ethers.utils.getCreate2Address()`
- Estimate gas for deployments

### Sponsored Deployment (NEAR, TON)

Some chains require the API Bridge to pay for account deployment (the user doesn't have native tokens on the destination chain yet). The API Bridge holds a small balance on these chains and sponsors the deployment transaction. The cost is recouped through Accumulate credits.

## Error Handling and Retry Patterns

- **Accumulate API errors**: Mapped to HTTP status codes (404 for unknown ADI, 400 for invalid parameters, 500 for network errors)
- **Chain RPC errors**: Retried up to 3 times with exponential backoff
- **Transaction failures**: Detailed error messages returned including Accumulate delivery status codes
- **Rate limiting**: 100 requests per minute per IP address (configurable)
