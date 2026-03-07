# Key Vault Architecture

The Key Vault is a Chrome Manifest V3 extension with four entry points: service worker (background), content script, injected provider, and popup UI.

## Manifest V3 Architecture

```
+-------------------------------------------------------------------+
|  Web Page (certen-web-app)                                         |
|  +-------------------------------------------------------------+  |
|  |  Injected Provider (window.certen)                           |  |
|  |  Injected by content script into page context                |  |
|  +-------------------------------------------------------------+  |
|       | window.postMessage                                         |
|       v                                                            |
|  +-------------------------------------------------------------+  |
|  |  Content Script (content.ts)                                 |  |
|  |  Runs in isolated world, bridges page <-> extension          |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
        | chrome.runtime.sendMessage
        v
+-------------------------------------------------------------------+
|  Service Worker (background.ts)                                    |
|  - Message router                                                  |
|  - Key storage manager                                             |
|  - Signing operations                                              |
|  - Vault lock/unlock state                                         |
+-------------------------------------------------------------------+
        | chrome.runtime.sendMessage (to popup)
        v
+-------------------------------------------------------------------+
|  Popup UI (popup.tsx - React)                                      |
|  - Vault creation / import                                         |
|  - Password entry (unlock)                                         |
|  - Transaction approval                                            |
|  - Key management                                                  |
|  - Settings                                                        |
+-------------------------------------------------------------------+
```

## Entry Points and Build

The Webpack configuration (`webpack.config.js`) produces 4 bundles:

| Entry Point | Source | Output | Context |
|------------|--------|--------|---------|
| Service Worker | `src/background/index.ts` | `dist/background.js` | Extension background |
| Content Script | `src/content/index.ts` | `dist/content.js` | Page isolated world |
| Injected Provider | `src/injected/provider.ts` | `dist/injected.js` | Page main world |
| Popup | `src/popup/index.tsx` | `dist/popup.js` | Extension popup |

## Vault Crypto Modules

### `src/vault/encryption.ts`

Handles encryption/decryption of the keystore:

- **Encrypt**: `AES-256-GCM(plaintext, key)` -> `{ciphertext, iv, tag}`
- **Decrypt**: `AES-256-GCM(ciphertext, key, iv, tag)` -> `plaintext`
- Uses Web Crypto API (`crypto.subtle`) for all operations

### `src/vault/keyDerivation.ts`

Password-based key derivation:

- **Algorithm**: PBKDF2-SHA512
- **Iterations**: 600,000 (configurable, OWASP 2023 minimum)
- **Salt**: 32 bytes random (generated on vault creation, stored with keystore)
- **Output**: 256-bit AES key for keystore encryption

### `src/vault/signing.ts`

Signing operations for supported key types:

- **Ed25519**: Uses `@noble/ed25519` for signing
- **secp256k1**: Uses `@noble/secp256k1` for signing
- **BLS12-381**: Uses `@noble/bls12-381` for signing
- All signing is done in the service worker (never in page context)

### `src/vault/hdWallet.ts`

HD wallet key derivation:

- **Mnemonic generation**: BIP-39 with 128 or 256 bits of entropy
- **Seed derivation**: PBKDF2(mnemonic + passphrase) per BIP-39
- **Key derivation**: SLIP-0010 for Ed25519, BIP-32 for secp256k1
- **Path parsing**: Supports standard BIP-44 derivation paths

## Message Protocol

Communication flows through three layers:

### Page -> Content Script

The injected provider (`window.certen`) sends messages via `window.postMessage`:

```typescript
// Injected provider sends
window.postMessage({
  type: "CERTEN_REQUEST",
  id: "uuid-1234",
  method: "signHash",
  params: {
    hash: "0xabcdef...",
    keyType: "ed25519",
    derivationPath: "m/44'/281'/0'/0/0"
  }
}, "*");

// Content script forwards response
window.addEventListener("message", (event) => {
  if (event.data.type === "CERTEN_RESPONSE") {
    // {id: "uuid-1234", result: {signature: "0x..."}}
  }
});
```

### Content Script -> Service Worker

```typescript
// Content script forwards
chrome.runtime.sendMessage({
  type: "CERTEN_REQUEST",
  id: "uuid-1234",
  method: "signHash",
  params: { ... }
});

// Service worker responds via callback
```

### Service Worker -> Popup

For operations requiring user approval (signing, connecting):

```typescript
// Service worker opens popup
chrome.action.openPopup();

// Popup reads pending request from storage
chrome.storage.session.get("pendingRequest");

// User approves/rejects
// Popup sends result back to service worker
chrome.runtime.sendMessage({
  type: "APPROVAL_RESULT",
  id: "uuid-1234",
  approved: true
});
```

## Two-Phase Signing Flow (Extension Perspective)

```
1. Web App calls window.certen.signHash(txHash)
2. Injected provider posts message to content script
3. Content script forwards to service worker
4. Service worker checks if vault is unlocked
   - If locked: opens popup for password entry
   - If unlocked: proceeds
5. Service worker opens popup for signing approval
6. Popup displays: "Sign transaction 0xabcd...?"
7. User clicks Approve
8. Service worker:
   a. Loads encrypted keystore from chrome.storage.local
   b. Decrypts with session key (derived from password on unlock)
   c. Derives the signing key from HD path
   d. Signs txHash with the appropriate algorithm
   e. Clears plaintext key from memory
9. Service worker sends signature to content script
10. Content script posts signature to page
11. Injected provider resolves the promise with signature
```

## Key Storage Format

The encrypted keystore stored in `chrome.storage.local`:

```
{
  version: 2,
  salt: "base64...",              // PBKDF2 salt (32 bytes)
  iterations: 600000,             // PBKDF2 iteration count
  encryptedMnemonic: {
    ciphertext: "base64...",      // AES-256-GCM encrypted mnemonic
    iv: "base64...",              // 12-byte initialization vector
    tag: "base64..."             // 16-byte authentication tag
  },
  accounts: [{
    name: "Account 1",
    derivationPaths: {
      accumulate: "m/44'/281'/0'/0/0",
      ethereum: "m/44'/60'/0'/0/0",
      solana: "m/44'/501'/0'/0/0"
    },
    addresses: {
      accumulate: "ed25519:0x1234...",
      ethereum: "0xabcd...",
      solana: "Abc123..."
    }
  }],
  settings: {
    autoLockTimeout: 300,         // seconds
    defaultKeyType: "ed25519"
  }
}
```

The mnemonic is the only secret stored. All keys are re-derived from the mnemonic on unlock. Public keys and addresses are stored unencrypted for display purposes.

## Lock/Unlock Lifecycle

1. **Vault creation**: User enters password -> PBKDF2 derives AES key -> mnemonic encrypted -> stored
2. **Unlock**: User enters password -> PBKDF2 derives AES key -> mnemonic decrypted -> session key stored in `chrome.storage.session` (cleared on browser close)
3. **Auto-lock**: Timer fires after inactivity -> session key cleared -> vault locked
4. **Sign request while locked**: Popup opens for password entry -> unlock -> sign -> response
