# Pending Service

The Certen Pending Service is a background polling service that discovers multi-signature transactions on the Accumulate network that require additional signatures, and syncs them to Firestore for real-time display in the Web App.

## Architecture

```
+-------------------------------------------------------------------+
|  Pending Service                                                    |
|                                                                     |
|  +-------------+     +------------------+     +-----------------+  |
|  |   Poller    |---->|   Accumulate     |---->|   Discovery     |  |
|  |   (timer)   |     |   Client         |     |   Service       |  |
|  +-------------+     +------------------+     +-----------------+  |
|                                                       |             |
|                                                       v             |
|  +-------------+     +------------------+     +-----------------+  |
|  |   State     |<----|   Signing Path   |<----|   Dedup         |  |
|  |   Manager   |     |   Service        |     |   Service       |  |
|  +-------------+     +------------------+     +-----------------+  |
|       |                                                             |
|       v                                                             |
|  +-------------+                                                    |
|  |   Firestore |                                                    |
|  |   Writer    |                                                    |
|  +-------------+                                                    |
+-------------------------------------------------------------------+
         |
         v
    Firestore: /users/{uid}/pendingActions/{actionId}
         |
         v
    Web App (real-time listener)
```

## Source Structure

```
certen-pending-service/
├── src/
│   ├── index.ts                           # Entry point, service bootstrap
│   ├── poller/
│   │   └── poller.ts                      # Polling loop, interval management
│   ├── clients/
│   │   └── accumulate.ts                  # Accumulate v2/v3 API client
│   ├── services/
│   │   ├── pending-discovery.service.ts   # 4-phase discovery algorithm
│   │   ├── signing-path.service.ts        # Resolve key -> signing paths
│   │   ├── state-manager.service.ts       # Track discovered state, write Firestore
│   │   └── dedup.service.ts               # Deduplicate pending actions
│   ├── types/
│   │   └── index.ts                       # TypeScript type definitions
│   └── utils/
│       └── logger.ts                      # Structured logging
├── .env.example                           # Configuration template
├── package.json
└── tsconfig.json
```

## Discovery Algorithm

The discovery process runs on each poll cycle (default: every 30 seconds) and executes 4 phases:

### Phase 1: Resolve Signing Paths

For each monitored user (identified by public key), resolve which ADIs and key pages the user can sign for:

1. Query Accumulate for all ADIs associated with the user's public key
2. For each ADI, traverse key books and key pages
3. Build a map: `publicKey -> [{adi, keyBook, keyPage, threshold}]`

### Phase 2: Process Pending Accounts

For each ADI in the signing paths, check for pending transactions:

1. Query Accumulate v2 API: `POST /v2/query` with `type: "pending"` for each account
2. Collect all pending transaction hashes
3. For each pending transaction, fetch full transaction details

### Phase 3: Scan Signature Chains

For each pending transaction, determine the current signature state:

1. Query the signature chain for the transaction
2. Count existing signatures and identify which keys have signed
3. Compare against the key page threshold to determine if more signatures are needed
4. If the current user's key has NOT signed and the threshold is NOT met, mark as actionable

### Phase 4: Deduplicate

Merge results from multiple signing paths that may reference the same pending transaction:

1. Group by transaction hash
2. Merge signer information
3. Remove transactions that have already been completed since the last poll

## Firestore Data Model

Pending actions are stored at `/users/{uid}/pendingActions/{actionId}`:

```typescript
{
  actionId: string;           // Unique ID (derived from tx hash)
  txHash: string;             // Accumulate transaction hash
  adi: string;                // ADI URL (e.g., "acc://organization.acme")
  account: string;            // Account URL (e.g., "acc://organization.acme/tokens")
  txType: string;             // Transaction type (e.g., "sendTokens", "updateKeyPage")
  keyPage: string;            // Key page URL that requires signing
  threshold: number;          // Required signature count
  currentSignatures: number;  // How many signatures exist
  signers: [{                 // Who has signed
    publicKey: string;
    timestamp: number;
  }];
  needsSignature: boolean;    // Whether this user still needs to sign
  createdAt: number;          // When the pending tx was first discovered
  updatedAt: number;          // Last update timestamp
  expiresAt: number;          // When the pending tx expires on Accumulate
}
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `ACCUMULATE_API` | `http://206.191.154.164/v3` | Accumulate v3 API endpoint |
| `ACCUMULATE_API_V2` | `http://206.191.154.164/v2` | Accumulate v2 API endpoint |
| `POLL_INTERVAL` | `30000` | Polling interval in milliseconds |
| `FIRESTORE_PROJECT_ID` | (required) | Firebase/GCP project ID |
| `GOOGLE_APPLICATION_CREDENTIALS` | (required) | Path to service account JSON |
| `MONITORED_KEYS` | (optional) | Comma-separated public keys to monitor |
| `LOG_LEVEL` | `info` | Log verbosity: debug, info, warn, error |
| `MAX_PENDING_PER_USER` | `100` | Maximum pending actions stored per user |
| `DISCOVERY_TIMEOUT` | `30000` | Timeout for a single discovery cycle (ms) |
| `STALE_ACTION_TTL` | `86400000` | Remove actions older than this (ms, default 24h) |
| `BATCH_SIZE` | `10` | Number of accounts to query in parallel |
| `RETRY_COUNT` | `3` | Retries for failed Accumulate API calls |
| `RETRY_DELAY` | `1000` | Delay between retries (ms) |

## Setup

```bash
cd ~/certen/certen-pending-service

# Configure
cp .env.example .env
# Edit .env:
# - Set GOOGLE_APPLICATION_CREDENTIALS to your Firebase service account JSON path
# - Set FIRESTORE_PROJECT_ID
# - Set ACCUMULATE_API and ACCUMULATE_API_V2

npm install
npm run dev
```

For production, deploy as a long-running process (systemd, Docker, or Cloud Run).

## Related Documentation

- [Ecosystem Overview](../onboarding/02-ecosystem-overview.md) - How Pending Service fits in the protocol
- [Data Flow Walkthrough](../onboarding/04-data-flow-walkthrough.md) - Scenario 2: Multi-sig approval flow
- [Web App](../web-app/README.md) - How the Web App displays pending actions
