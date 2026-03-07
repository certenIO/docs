# Web App Architecture

The Certen Web App is a React 18 SPA built with Vite, using Material-UI for components, Firebase for backend services, and a custom message protocol for Key Vault integration.

## Application Structure

```
certen-web-app/
├── src/
│   ├── App.tsx                    # Root component, router, context providers
│   ├── main.tsx                   # Entry point, React DOM render
│   ├── router.tsx                 # React Router configuration, lazy loading
│   ├── theme.ts                   # Material-UI theme customization
│   ├── firebase.ts                # Firebase initialization
│   ├── components/                # UI components (see README for inventory)
│   ├── contexts/                  # React context providers (8 contexts)
│   ├── services/                  # API client services (10+ services)
│   ├── hooks/                     # Custom React hooks
│   ├── types/                     # TypeScript type definitions
│   ├── utils/                     # Utility functions
│   └── assets/                    # Static assets, images
├── functions/
│   ├── src/
│   │   ├── index.ts               # Cloud Functions entry, Express app
│   │   ├── middleware/
│   │   │   ├── auth.ts            # Firebase auth token verification
│   │   │   └── rateLimit.ts       # Per-user rate limiting
│   │   └── routes/
│   │       ├── user.ts            # User profile endpoints
│   │       └── webhook.ts         # External webhook handlers
│   ├── package.json
│   └── tsconfig.json
├── public/                        # Static files
├── firebase.json                  # Firebase project configuration
├── firestore.rules                # Firestore security rules
├── .env.example                   # Environment template
├── package.json
├── tsconfig.json
├── vite.config.ts
└── Dockerfile
```

## React Application Structure

### Routing

Pages are lazy-loaded to reduce initial bundle size:

```
App
├── / (Landing)
├── /login (Login)
├── /dashboard (Dashboard)          [Protected]
├── /adi/create (CreateAdi)         [Protected]
├── /adi/:url (AdiDetail)           [Protected]
├── /adi/:url/keys (KeyManagement)  [Protected]
├── /adi/:url/accounts (Accounts)   [Protected]
├── /adi/:url/intents (Intents)     [Protected]
├── /pending (PendingActions)       [Protected]
├── /proofs (ProofExplorer)         [Protected]
├── /proofs/:id (ProofDetail)       [Protected]
├── /settings (Settings)            [Protected]
└── /about (About)
```

Protected routes redirect to `/login` if the user is not authenticated.

### State Management

The app uses React Context API combined with Firestore real-time listeners. No external state library (Redux, Zustand) is used.

```
App
└── ThemeProvider (MUI)
    └── AuthContext.Provider
        └── ApiContext.Provider
            └── KeyVaultContext.Provider
                └── AdiContext.Provider
                    └── PendingContext.Provider
                        └── ChainContext.Provider
                            └── NotificationContext.Provider
                                └── Router
```

**AuthContext**: Wraps Firebase Auth. Listens to `onAuthStateChanged` and provides `user`, `isAuthenticated`, `login()`, `logout()` to all descendants.

**PendingContext**: Subscribes to a Firestore collection (`/users/{uid}/pendingActions`) with `onSnapshot`. Updates state in real-time when the Pending Service discovers new pending actions.

**KeyVaultContext**: Detects the Key Vault extension by checking for `window.certen`. Listens for connection/disconnection events. Provides `signHash()`, `getPublicKeys()`, `isConnected`.

## Key Vault Integration

The Web App communicates with the Key Vault Chrome extension via the `window.certen` message protocol:

```
Web App (page)                Content Script              Service Worker
    |                              |                          |
    |-- window.postMessage ------->|                          |
    |   {type: "CERTEN_REQUEST",   |                          |
    |    method: "signHash",       |                          |
    |    params: {hash, keyType}}  |                          |
    |                              |-- chrome.runtime ------->|
    |                              |   .sendMessage()         |
    |                              |                          |
    |                              |                          |-- process
    |                              |                          |   request
    |                              |                          |
    |                              |                          |-- show popup
    |                              |                          |   (if approval
    |                              |                          |    needed)
    |                              |                          |
    |                              |<-- response -------------|
    |<-- window.postMessage -------|                          |
    |   {type: "CERTEN_RESPONSE",  |                          |
    |    result: {signature}}      |                          |
```

**Available methods**:
- `getPublicKeys()` - List available public keys and their types
- `signHash(hash, keyType)` - Sign a hash with a specific key
- `getAddresses(chainType)` - Get addresses for a chain type
- `isLocked()` - Check if the vault is locked
- `connect()` - Request connection approval from user

## Cloud Functions Backend

The `functions/` directory contains Firebase Cloud Functions deployed as an Express app:

- **Auth middleware**: Verifies Firebase ID tokens on incoming requests
- **Rate limiting**: Per-user request limits (stored in Firestore)
- **User routes**: Profile creation, preferences update
- **Webhook routes**: External service callbacks

Cloud Functions are deployed alongside the app via `firebase deploy --only functions`.

## Material-UI Theming

Custom MUI theme configured in `theme.ts`:
- Dark and light mode support via `ThemeContext`
- Custom color palette aligned with Certen branding
- Component overrides for consistent styling (buttons, cards, tables)
- Responsive breakpoints for mobile/tablet/desktop

## Build and Deployment

```bash
# Development
npm run dev                # Vite dev server on port 3000

# Production build
npm run build              # Output to dist/

# Preview production build
npm run preview

# Deploy to Firebase Hosting
firebase deploy --only hosting

# Deploy Cloud Functions
firebase deploy --only functions

# Deploy everything
firebase deploy
```

**Vite configuration** (`vite.config.ts`):
- React plugin with Fast Refresh
- Path aliases (`@/` maps to `src/`)
- Environment variable prefix: `VITE_`
- Build target: ES2020
- Chunk splitting for vendor libraries
