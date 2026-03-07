# Proofs Service Architecture

The Proofs Service is a Go backend with a React frontend, using PostgreSQL for proof artifact storage and a repository pattern for data access.

## Backend Structure

```
proofs_service/
├── cmd/
│   ├── proof-service/
│   │   └── main.go                # API server entry point
│   └── migrate/
│       └── main.go                # Database migration tool
├── pkg/
│   ├── config/
│   │   └── config.go              # Environment-based configuration
│   ├── database/
│   │   ├── database.go            # Connection pool, migration runner
│   │   ├── migrations/            # SQL migration files (001-009)
│   │   ├── batch_repo.go          # Batch repository
│   │   ├── proof_repo.go          # Proof repository
│   │   ├── anchor_repo.go         # Anchor repository
│   │   ├── attestation_repo.go    # Attestation repository
│   │   ├── intent_repo.go         # Intent repository
│   │   └── factory.go             # Repository factory
│   └── server/
│       ├── server.go              # HTTP server setup, middleware
│       ├── handlers/
│       │   ├── proof_handler.go    # Proof query handlers
│       │   ├── bundle_handler.go   # Bundle assembly handlers
│       │   ├── anchor_handler.go   # Anchor record handlers
│       │   ├── verify_handler.go   # Verification handlers
│       │   ├── intent_handler.go   # Intent query handlers
│       │   ├── attestation_handler.go # Attestation handlers
│       │   └── stats_handler.go    # Statistics handlers
│       └── middleware/
│           ├── cors.go             # CORS middleware
│           ├── ratelimit.go        # Rate limiting
│           └── logging.go          # Request logging
├── frontend/                       # React Proof Explorer (see below)
├── .env.example
├── docker-compose.yml
├── Dockerfile
└── go.mod / go.sum
```

## Repository Pattern

Each database table has a corresponding repository that encapsulates all SQL operations:

```
RepositoryFactory
  |
  +-- BatchRepository
  |     .Create(batch)
  |     .GetByID(id)
  |     .GetLatest()
  |     .List(filter, pagination)
  |
  +-- ProofRepository
  |     .Create(proof)
  |     .GetByID(id)
  |     .GetByBatchID(batchId)
  |     .GetByChain(chain)
  |     .GetByADI(adiUrl)
  |     .List(filter, pagination)
  |
  +-- AnchorRepository
  |     .Create(anchor)
  |     .GetByID(id)
  |     .GetByTxHash(txHash)
  |     .GetByChain(chain)
  |     .List(filter, pagination)
  |
  +-- AttestationRepository
  |     .Create(attestation)
  |     .GetByBatchID(batchId)
  |     .List(filter, pagination)
  |
  +-- IntentRepository
        .Create(intent)
        .GetByID(id)
        .GetByADI(adiUrl)
        .UpdateStatus(id, status)
        .List(filter, pagination)
```

The `RepositoryFactory` creates all repositories from a shared `*sql.DB` connection pool.

## Database Schema

9 migrations define the schema:

| Migration | Table/Change | Description |
|-----------|-------------|-------------|
| 001 | `batches` | Proof batch metadata (id, timestamp, accumulate_height, status) |
| 002 | `proofs` | Individual phase proofs (batch_id, phase, data, merkle_root) |
| 003 | `anchors` | On-chain anchor records (batch_id, chain, tx_hash, block_number) |
| 004 | `attestations` | Validator attestations (batch_id, validator_id, signature) |
| 005 | `intents` | Cross-chain intent tracking (adi, chain, status, tx_hash) |
| 006 | Indexes | Performance indexes on batch_id, chain, adi, timestamp |
| 007 | `status` column | Add status/error tracking to proofs table |
| 008 | `bundle_size` | Add computed bundle size to batches |
| 009 | `audit_log` | Audit log for verification requests |

### Key Tables

**`batches`**:
```sql
CREATE TABLE batches (
    id              UUID PRIMARY KEY,
    timestamp       TIMESTAMP NOT NULL,
    accumulate_height BIGINT NOT NULL,
    merkle_root     BYTEA NOT NULL,
    status          VARCHAR(20) DEFAULT 'completed',
    bundle_size     INTEGER,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

**`proofs`**:
```sql
CREATE TABLE proofs (
    id          UUID PRIMARY KEY,
    batch_id    UUID REFERENCES batches(id),
    phase       VARCHAR(10) NOT NULL,  -- L1, L2, L3, L4, G0, G1, G2, BLS, Anchor
    data        JSONB NOT NULL,        -- Phase-specific proof data
    merkle_root BYTEA,
    status      VARCHAR(20) DEFAULT 'valid',
    error       TEXT,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

**`anchors`**:
```sql
CREATE TABLE anchors (
    id           UUID PRIMARY KEY,
    batch_id     UUID REFERENCES batches(id),
    chain        VARCHAR(50) NOT NULL,
    tx_hash      VARCHAR(128) NOT NULL,
    block_number BIGINT,
    contract_addr VARCHAR(128),
    status       VARCHAR(20) DEFAULT 'confirmed',
    created_at   TIMESTAMP DEFAULT NOW()
);
```

## Proof Bundle Format

When the API assembles a full proof bundle (via `/api/v1/bundles/{batchId}`), it combines data from multiple tables into a 4-component structure:

```
ProofBundle {
  batch: {
    id, timestamp, accumulateHeight, merkleRoot, status
  },

  merkleProofs: {
    L1: { phase: "L1", data: {...}, merkleRoot },
    L2: { phase: "L2", data: {...}, merkleRoot },
    L3: { phase: "L3", data: {...}, merkleRoot },
    L4: { phase: "L4", data: {...}, merkleRoot }
  },

  governanceProofs: {
    G0: { phase: "G0", data: {...} },
    G1: { phase: "G1", data: {...} },
    G2: { phase: "G2", data: {...} }
  },

  consensus: {
    BLS: { phase: "BLS", data: {aggregateSignature, zkProof, publicInputs} }
  },

  anchors: [{
    chain, txHash, blockNumber, contractAddr, status
  }],

  attestations: [{
    validatorId, signature, timestamp
  }]
}
```

## REST API Design

### Handler Pattern

Each handler follows a consistent pattern:

1. Parse and validate request parameters (path params, query params, body)
2. Call the appropriate repository method
3. Transform the database result into the API response format
4. Return JSON response with appropriate HTTP status code

### Error Responses

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Proof batch not found",
    "details": {"batchId": "uuid-1234"}
  }
}
```

| HTTP Status | Error Code | Meaning |
|------------|------------|---------|
| 400 | `BAD_REQUEST` | Invalid request parameters |
| 404 | `NOT_FOUND` | Resource not found |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |

## Frontend: Proof Explorer

```
frontend/
├── src/
│   ├── App.tsx                    # Root component
│   ├── main.tsx                   # Entry point
│   ├── api/
│   │   └── client.ts              # Proofs Service API client
│   ├── components/
│   │   ├── ProofList.tsx           # Paginated proof list
│   │   ├── ProofDetail.tsx         # Single proof breakdown
│   │   ├── BundleView.tsx          # Full bundle visualization
│   │   ├── AnchorList.tsx          # Anchor record table
│   │   ├── VerificationResult.tsx  # On-chain verification display
│   │   └── StatsOverview.tsx       # System statistics dashboard
│   ├── types/
│   │   └── index.ts               # TypeScript types
│   └── utils/
│       └── format.ts              # Date, hash formatting
├── package.json
├── tsconfig.json
└── vite.config.ts
```

Built with React 18, Vite, and Ant Design component library. The frontend communicates exclusively with the Proofs Service REST API.

**Build and run**:

```bash
cd frontend
npm install
npm run dev      # Dev server (port 3000)
npm run build    # Production build to dist/
```
