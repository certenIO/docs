# Web Application

The Certen Web App is the primary user interface for the Certen Protocol, providing ADI management, multi-signature transaction coordination, intent creation, and proof visualization.

## Page Inventory

| Route | Page | Description |
|-------|------|-------------|
| `/` | Landing | Protocol overview, getting started |
| `/login` | Login | Firebase authentication (email, Google) |
| `/dashboard` | Dashboard | Overview of user's ADIs, pending actions, recent activity |
| `/adi/create` | Create ADI | Step-by-step ADI creation wizard |
| `/adi/:url` | ADI Detail | ADI info, key books, accounts, governance |
| `/adi/:url/keys` | Key Management | Add/remove keys, rotate, set thresholds |
| `/adi/:url/accounts` | Chain Accounts | Deploy and manage accounts on target chains |
| `/adi/:url/intents` | Intents | Create and track cross-chain intents |
| `/pending` | Pending Actions | Multi-sig transactions awaiting user's signature |
| `/proofs` | Proof Explorer | Browse and verify proof bundles |
| `/proofs/:id` | Proof Detail | Detailed proof bundle view with verification |
| `/settings` | Settings | User preferences, connected Key Vault |
| `/about` | About | Protocol documentation and links |

## Component Organization

| Directory | Purpose |
|-----------|---------|
| `src/components/adi/` | ADI creation, detail, and management components |
| `src/components/auth/` | Login, signup, protected route wrappers |
| `src/components/chain/` | Chain account deployment and management |
| `src/components/common/` | Shared UI components (buttons, modals, tables) |
| `src/components/dashboard/` | Dashboard widgets and activity feed |
| `src/components/intent/` | Intent creation form and status tracking |
| `src/components/key/` | Key page management, key rotation |
| `src/components/layout/` | App shell, navigation, sidebar |
| `src/components/pending/` | Pending action list and approval flow |
| `src/components/proof/` | Proof explorer, proof detail, verification |
| `src/components/settings/` | User settings and Key Vault connection |

## Context Providers

| Context | Purpose |
|---------|---------|
| `AuthContext` | Firebase auth state, user session |
| `AdiContext` | Current ADI selection, ADI list for user |
| `KeyVaultContext` | Key Vault extension connection state, available keys |
| `PendingContext` | Real-time pending actions from Firestore |
| `ChainContext` | Supported chains, chain status, RPC health |
| `ThemeContext` | Dark/light mode, MUI theme customization |
| `NotificationContext` | Toast notifications, error display |
| `ApiContext` | API Bridge client configuration |

## Service Layer

| Service | Purpose |
|---------|---------|
| `AdiService` | ADI CRUD via API Bridge |
| `KeyService` | Key page operations via API Bridge |
| `CreditService` | Credit purchase and balance |
| `IntentService` | Intent creation and status |
| `ChainService` | Multi-chain account operations |
| `TxService` | Generic transaction prepare/submit |
| `AuthService` | Firebase authentication |
| `FirestoreService` | Firestore data subscriptions |
| `KeyVaultService` | Key Vault message protocol client |
| `ProofService` | Proof queries via Proofs Service API |

## Firebase Integration

| Service | Usage |
|---------|-------|
| **Firebase Auth** | User authentication (email/password, Google SSO) |
| **Firestore** | Real-time data: pending actions, user preferences, ADI metadata |
| **Cloud Functions** | Backend logic in `functions/` directory (auth middleware, rate limiting, data validation) |
| **Firebase Hosting** | Production deployment target |

## Setup

```bash
cd ~/certen/certen-web-app
cp .env.example .env
```

Configure `.env`:

```env
VITE_API_BASE_URL=http://localhost:8085
VITE_PROOFS_API_URL=http://localhost:8080
VITE_FIREBASE_API_KEY=your-api-key
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
VITE_FIREBASE_APP_ID=your-app-id
```

```bash
npm install
npm run dev

# With Firebase emulators (for local development)
firebase emulators:start &
npm run dev
```

## Related Documentation

- [Architecture](architecture.md) - React app structure, state management, build system
- [Key Vault](../key-vault/README.md) - Extension integration details
- [API Bridge](../api-bridge/README.md) - Backend API endpoints
