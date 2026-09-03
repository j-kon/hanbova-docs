# Hanbova: Project Context & Verified Development State

> **Tagline**: *Send protected.*  
> **Mission**: Safer everyday Bitcoin payments across Africa by combining instant payments with optional Cashu-based protected payments.  
> **Current Development Stage**: **Milestone M3B.2.2 Financial Interaction Redesign (COMPLETED & VERIFIED)**  
> **Active Branch**: `milestone/3b2-2-financial-interaction-redesign`  
> **Primary Program Goal**: Afro Bitcoin Fellowship  
> **Mainnet Status**: **SAFETY-LOCKED / MUST REMAIN DISABLED FOR TESTING**  
> **Branding System**: Approved Hanbova Brand V4 Identity (Bitcoin Orange `#F7931A`, Lightning Gold `#FFC400`, Charcoal `#172027`, Graphite `#25323A`, Warm White `#FAFAF7`, Soft Gray `#6E7A80`, Exact Master V4 Logos/Icons/Splash/Onboarding)  
> **Last Reviewed**: 2026-09-03

---

## 1. Executive Summary

Hanbova is an Africa-first Bitcoin wallet concept focused on a practical trust problem in everyday digital commerce: many payments are final before the buyer has confidence that the seller will deliver.

The product uses two payment ideas:

1. **Instant Send**
   - Intended for trusted, everyday payments.
   - Lightning integration exists as experimental/development work.
   - Production Lightning settlement has not yet been independently verified end-to-end in the mobile app.

2. **Protected Send**
   - Uses Cashu NUT-10 / NUT-11 Pay-to-Public-Key (P2PK) spending conditions powered by the official Cashu Development Kit (CDK) via a native C-FFI bridge.
   - A recipient public key is used for the normal claim path.
   - A sender refund public key and locktime provide a refund path after the locktime.
   - The refund is a **Cashu mint spend path**, not an on-chain Bitcoin refund transaction.
   - After locktime, the recipient path is not automatically revoked. Recipient claim and sender refund can race, and the mint's proof state is authoritative.
   - Claimed ecash is credited directly to the recipient's Cashu spendable balance (backed by sats at the mint), not settled as an on-chain Bitcoin transaction.

Hanbova is currently a **test-stage open-source wallet project**. Mainnet is strictly disabled and safety-locked. All wallet operations run on Cashu test environments (Signet/Regtest).

---

## 2. Current Repository Layout

| Repository | Stack | Purpose |
| --- | --- | --- |
| `j-kon/hanbova-app` | Flutter, Riverpod, GoRouter, Dart FFI | Consumer mobile wallet UI and client-side wallet integration |
| `j-kon/hanbova-backend` | Rust, Axum, Tokio, SQLx, PostgreSQL, CDK bridge | Authentication, coordination, encrypted message relay, and CDK FFI crate |
| `j-kon/hanbova-protocol` | Markdown / protocol specifications | Hanbova protected-payment protocol documentation |
| `j-kon/hanbova-docs` | Markdown | Architecture, threat model, roadmap, development notes, and fellowship presentation material |

Repository URLs:

- https://github.com/j-kon/hanbova-app
- https://github.com/j-kon/hanbova-backend
- https://github.com/j-kon/hanbova-protocol
- https://github.com/j-kon/hanbova-docs

---

## 3. Consumer Wallet Experience

The Milestone 2.5 consumer shell is substantially implemented.

### Navigation

Persistent bottom navigation:

- Home
- Activity
- Pay
- Protected
- Me

### Consumer Features

The app includes:

- consumer wallet home screen
- spendable balance display
- sats + display-currency presentation
- balance privacy toggle
- activity/transaction screens
- Protected payment area
- account authentication UI
- profile/settings
- Light / Dark / System appearance
- display currencies including NGN, KES, GHS, ZAR, UGX, RWF, and USD
- local biometric/PIN-oriented security UI
- developer network settings

The fiat values are presentation values only. Hanbova does not currently provide fiat custody or bank/mobile-money rails.

---

## 4. Authentication & Account Layer

The backend includes an account authentication system using:

- Argon2id password hashing
- JWT access tokens
- rotating refresh/session tokens
- PostgreSQL persistence
- authenticated profile retrieval

The intended account/wallet boundary is:

```text
Hanbova account
    |
    |-- username / email / password / profile
    |
    `-- DOES NOT equal Bitcoin/Cashu private-key recovery
```

Password reset must never be described as wallet-key recovery.

### Current API Shape

Current backend routes include concepts such as:

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password
GET  /api/v1/me
```

Protected-payment coordination routes include:

```text
GET  /api/v1/users/:username/payment-profile
PUT  /api/v1/me/payment-keys

POST /api/v1/protected-messages
GET  /api/v1/protected-messages/inbox
GET  /api/v1/protected-messages/outbox
GET  /api/v1/protected-messages/:id
POST /api/v1/protected-messages/:id/ack

POST /api/v1/payment-intents
GET  /api/v1/payment-intents
GET  /api/v1/payment-intents/:id
POST/PATCH /api/v1/payment-intents/:id/status
```

The recipient payment profile resolution (`GET /api/v1/users/:username/payment-profile`) matches against both `LOWER(u.username)` and `LOWER(u.email)` in PostgreSQL, enabling senders to enter either `@handle`, `handle`, or full email addresses (e.g. `recipient@example.com`).

The exact routes in code should remain the source of truth.

---

## 5. Client Cryptographic Identity

Hanbova currently has separate client-side cryptographic purposes.

### Protected-Payment Key

A secp256k1 keypair is used for Cashu NUT-11 P2PK operations.

The current implementation correctly:

- generates a 32-byte scalar using secure randomness
- validates that the scalar lies inside the secp256k1 range
- derives the compressed public key through elliptic-curve scalar multiplication
- validates compressed public keys

The previous insecure approach of deriving a public key by prefixing `02` to private-key bytes has been removed.

### Transport Encryption Key

A separate X25519 keypair is used for encrypted Protected Payment delivery.

The transport path uses:

- ephemeral X25519 ECDH
- HKDF-SHA256
- ChaCha20-Poly1305 AEAD
- versioned encrypted envelopes

The Hanbova backend is designed to store ciphertext, routing information, and payment metadata rather than plaintext Cashu bearer tokens.

### Key Storage & Mnemonic Architecture

In `CryptoIdentityService`, keys are derived and stored in platform secure storage (`FlutterSecureStorage`):
- `hanbova_${storagePrefix}_${userId}_transport_priv`: 32-byte X25519 transport private key (derived deterministically from BIP-39 seed)
- `hanbova_${storagePrefix}_${userId}_protected_priv`: 32-byte secp256k1 P2PK private key (derived deterministically from BIP-39 seed)
- `hanbova_${storagePrefix}_${userId}_mnemonic`: 12-word BIP-39 mnemonic

The active BIP-39 mnemonic is converted to a 512-bit seed (`mnemonicToSeedHex`) and supplied to the CDK Rust FFI wallet (`hanbova_cdk_wallet_create`), which derives the master ecash wallet keyset.

### Deterministic Key Derivation & Recovery Limitations (Milestone 4 Partial)

`CryptoIdentityService` derives auxiliary keys deterministically from the 512-bit BIP-39 seed using domain separation:
- **P2PK Identity (secp256k1)**: `HMAC-SHA512("Hanbova P2PK Identity Derivation", seed)`
- **Transport Identity (X25519)**: `HMAC-SHA512("Hanbova X25519 Transport Derivation", seed)`

This enables `restoreFromMnemonic(...)` to reconstruct the exact same primary P2PK and transport identities on a fresh installation from the user's 12-word BIP-39 mnemonic.

> [!WARNING]
> **Recovery Limitation (Refund Keys & NUT-13)**:
> In the current implementation, `createProtectedSend` generates a fresh refund keypair per payment within CDK.
> Complete device loss / database wipe will restore the user's primary identity and CDK seed, but **pending un-refunded tokens with random per-payment refund keys** and **ecash proofs after local database deletion** require full NUT-13 proof restoration and deterministic per-payment refund key derivation, which is targeted for Milestone 4 completion. Milestone 4 is strictly **Partial**.

---

## 6. CDK Client Wallet Architecture

Milestone 3A.2 replaced the previous fake Dart Cashu proof generator with a genuine Rust CDK bridge.

The backend repository contains:

```text
crates/hanbova-cdk-ffi
```

This bridge links against:

```text
cdk = 0.18.0-rc.0
cdk-redb = 0.18.0-rc.0
```

The bridge exports 12 C-FFI functions calling official CDK wallet APIs for:

- wallet creation (`hanbova_cdk_wallet_create`)
- balance lookup (`hanbova_cdk_wallet_get_balance`)
- NUT-04 mint quote creation (`hanbova_cdk_mint_quote`)
- NUT-04 minting (`hanbova_cdk_mint`) — *Note: `mintTestTokens` is a controlled local Cashu mint integration using test/mock Lightning settlement for testing, not a public/production Lightning funding flow.*
- NUT-05 melt quote creation (`hanbova_cdk_melt_quote`)
- NUT-05 melt execution (`hanbova_cdk_melt`)
- NUT-11 P2PK protected send (`hanbova_cdk_prepare_p2pk_send`)
- NUT-11 P2PK receive (`hanbova_cdk_receive_p2pk`)
- NUT-07 proof-state checking (`hanbova_cdk_check_token_state`)
- wallet teardown (`hanbova_cdk_wallet_free`)
- error retrieval (`hanbova_cdk_get_last_error`)
- string deallocation (`hanbova_cdk_free_string`)

### iOS Build & Reproducibility

For iOS integration:
- `hanbova-app/scripts/build_ios_ffi.sh` builds static archives for physical device (`aarch64-apple-ios`) and simulator (`aarch64-apple-ios-sim`, `x86_64-apple-ios`) targets, bundles them with `lipo`, and creates `HanbovaCdkFfi.xcframework`.
- `HanbovaCdkFfi.podspec` references `ios/Frameworks/HanbovaCdkFfi.xcframework` without machine-specific absolute paths.
- **Fresh Clone Requirement**: A fresh clone requires running `scripts/build_ios_ffi.sh` once before running `pod install` in `hanbova-app/ios`.

The high-level path is:

```text
Flutter
   |
   v
Dart FFI bindings
   |
   v
hanbova-cdk-ffi
   |
   v
Official CDK Wallet
   |
   +--> redb wallet database
   |
   `--> Cashu mint
```

### Current Exported C-ABI Functions

The bridge currently exposes functions corresponding to:

```text
hanbova_cdk_wallet_create
hanbova_cdk_wallet_get_balance
hanbova_cdk_mint_quote
hanbova_cdk_mint
hanbova_cdk_prepare_p2pk_send
hanbova_cdk_receive_p2pk
hanbova_cdk_check_token_state
hanbova_cdk_wallet_free
```

Names in source code are authoritative.

---

## 7. Protected Send Flow

The intended verified architecture is:

```text
Alice device
   |
   |  CDK wallet
   |  create genuine P2PK-protected Cashu token
   v
Encrypt token for Bob
   |
   v
Hanbova backend
   |
   |  ciphertext only
   v
Bob device
   |
   |  decrypt locally
   |  receive using Bob P2PK private key
   v
Cashu mint
   |
   |  validates NUT-11 witness
   v
Bob receives new spendable proofs
```

The backend must not:

- receive Bob's P2PK private key
- receive Alice's refund private key
- act as the user's live Cashu wallet
- create live Cashu proofs for consumer-wallet operations
- store plaintext Cashu bearer tokens

---

## 8. Protected Refund Semantics

The correct user-facing rule is:

```text
Before locktime:
    Recipient claim path is available.

After locktime:
    Sender refund path becomes available.

After locktime:
    Recipient claim path may still remain valid.

First successful mint spend wins.
```

Therefore Hanbova should say:

> **Refund available**

and not:

> **Automatically refunded**

or:

> **Recipient can no longer claim**

### Claim Success Copy Standards

When claiming a protected payment:
- The success screen must state:
  > **"Protected payment claimed successfully. Your spendable balance has been updated."**
- It must **never** imply on-chain Bitcoin settlement, direct Lightning settlement, or hardcode unverified amounts.
- The recipient receives Cashu ecash backed by sats at the mint.
- The success toast displays:
  > **"Protected payment claimed successfully."**
  and refreshes the authoritative CDK/redb wallet balance directly from local storage.

Mint proof state is authoritative.

---

## 9. Network Environments

The intended safe network model is:

```text
Local Development
Cashu Test
Mainnet
```

### Local Development

- local Nutshell
- FakeWallet / valueless test setup
- deterministic testing
- short locktimes allowed

### Cashu Test

- public or controlled test mint
- valueless test ecash
- real CDK/Cashu proof handling
- two-device Protected Send testing

### Mainnet

Mainnet must be **disabled** until:

- Android native CDK packaging is proven
- iOS native CDK packaging is proven
- real test-mint funding works
- Alice/Bob claim flow succeeds on two independent devices
- refund flow succeeds
- restart persistence succeeds
- recovery behavior is honest and tested
- security review is complete

### Current Regression

The current `main` branch later enabled Mainnet and pointed it at a real mint.

That is a regression relative to the agreed safety model.

Before further wallet testing, Mainnet should be disabled again and fail closed.

---

## 10. Current Mobile Native FFI Blocker

The Dart layer currently expects native libraries such as:

```text
Android:
libhanbova_cdk_ffi.so

macOS:
libhanbova_cdk_ffi.dylib

iOS:
linked symbols in DynamicLibrary.process()
```

However, a successful Flutter build alone does not prove that the CDK library has been packaged correctly for Android and iOS.

### Required Android Proof

The application must actually package ABI-specific libraries such as:

```text
android/app/src/main/jniLibs/arm64-v8a/libhanbova_cdk_ffi.so
android/app/src/main/jniLibs/x86_64/libhanbova_cdk_ffi.so
```

or use another proper Flutter/native-plugin packaging mechanism.

### Required iOS Proof

The iOS Runner must link the Rust static library/framework so the exported symbols are available to:

```dart
DynamicLibrary.process()
```

This must be validated on an actual iOS simulator/device build.

---

## 11. Test Funding Status

A genuine Cashu wallet cannot create value locally.

The current CDK flow can request a real NUT-04 mint quote.

The correct lifecycle is:

```text
Create mint quote
    |
    v
Receive invoice / test payment request
    |
    v
Complete the supported test payment
    |
    v
Poll/check quote state
    |
    v
Mint genuine proofs
```

The current automated test suite verifies this via a **controlled local Cashu mint integration using test/mock Lightning settlement**, where quotes are settled through the local test backend. This is not an external or production Lightning funding test.

Creating a quote and immediately calling `mint()` without satisfying the quote is not a complete remote test-mint funding flow.

Hanbova must not use local fake balances as a substitute.

---

## 12. Recovery Status

### Backup

The backup screen has improved because it now reads the mnemonic associated with the current wallet identity instead of generating a new phrase every time the screen opens.

### Restore

Restore is still incomplete.

A valid BIP-39 checksum currently does not by itself prove that Hanbova has:

- imported the mnemonic
- rebuilt the CDK wallet seed
- reopened/recreated the CDK wallet database
- recovered supported Cashu state
- restored P2PK claim/refund identities
- restored the X25519 transport identity

Until that is implemented, the app must not display:

> **Wallet Restored!**

after mnemonic validation alone.

The safer temporary behavior is:

> **Recovery is not available in this test build yet.**

---

## 13. Backend Authorization Status

Payment-intent routes now use authenticated users and include sender/recipient checks.

A remaining issue must be fixed:

```text
POST /payment-intents
```

must always set:

```text
sender_id = authenticated_user.id
```

The backend should never trust a client-provided `sender_id`.

Protected-message object authorization is present, but remaining hardening includes:

- return 403 for forbidden access rather than 400
- restrict `claimed` to recipient-side reporting
- restrict `refunded` to sender-side reporting
- validate allowed status values
- reconcile coordination metadata with Cashu mint state where practical

---

## 14. Lightning Status

Lightning functionality exists in the codebase as experimental/development work.

It should not yet be presented as fully production-verified Lightning settlement.

For fellowship demos and documentation, use wording such as:

> **Lightning integration is experimental and under active verification.**

Do not claim production Lightning readiness until a real wallet/LSP path is demonstrated.

---

## 15. On-Chain Bitcoin Status

Hanbova currently should not present a mocked on-chain Bitcoin deposit address as a genuine deposit feature.

Any placeholder on-chain address must be removed or clearly disabled.

BDK is not required for the current Cashu Protected Send milestone.

On-chain wallet architecture can be evaluated later as a separate product decision.

---

## 16. Current Test Status

Repository documentation reports automated test suites such as:

```text
Rust backend:
24 tests passing (CDK FFI, live escrow, core types, mock lightning, auth lifecycle, password hashing, payment keys & protected messages)

Flutter:
131 tests passing (genuine Secp256k1 P2PK key generation & scalar derivation, deterministic derivation, context-isolated BIP-39 mnemonic lifecycle, fiat display conversions across NGN/KES/GHS/ZAR/USD, splash & onboarding rendering, restore authentication navigation, canonical payment ID consistency, financial authority, fail-closed claim/refund, NUT-04 quotes & minting, and auth widgets)
```

Static analysis has also been reported green (`flutter analyze` - 0 issues; `cargo test` - green).

These tests are valuable, but they do **not** replace the mandatory end-to-end proof:

```text
Alice device
   |
   v
real mint-signed test proofs
   |
   v
real NUT-11 protected token
   |
   v
Bob device
   |
   v
real CDK receive
   |
   v
mint-enforced P2PK validation
```

A milestone is not complete merely because mocked or unit tests are green.

---

## 17. Required Manual Completion Tests

### Scenario A: Claim

```text
1. Start backend.
2. Run Alice on Device A.
3. Run Bob on Device B.
4. Use the same Cashu test environment.
5. Fund Alice with genuine test-mint proofs.
6. Alice sends 100 test sats Protected to @bob.
7. Encrypted envelope reaches Bob.
8. Bob decrypts locally.
9. Bob claims through CDK.
10. Mint validates Bob's NUT-11 signature.
11. Original protected proofs become spent.
12. Bob receives fresh mint-signed proofs.
13. Bob's spendable balance increases.
14. Alice sees Claimed.
```

### Scenario B: Refund

```text
1. Alice sends another 100 test sats Protected.
2. Bob does nothing.
3. Locktime passes.
4. Alice sees Refund available.
5. Alice refunds through CDK.
6. Mint validates sender refund path.
7. Alice receives fresh spendable proofs.
8. Bob's later claim attempt fails because proofs are already spent.
```

### Scenario C: Restart

During a pending protected payment:

```text
Force-close Alice.
Force-close Bob.
Restart both.
```

Verify:

- wallet balance persists
- outgoing payment persists
- incoming encrypted message persists
- Alice retains refund capability
- Bob retains claim capability
- status reconciles from mint state

---

## 18. Security Model

### Backend Compromise

Desired server-side data:

- user IDs
- usernames
- public payment keys
- encrypted Protected Payment envelopes
- payment metadata
- timestamps/status coordination

The backend must not contain:

- BIP-39 mnemonic
- wallet seed
- P2PK private key
- refund private key
- X25519 private key
- plaintext Cashu bearer token

### Transport Confidentiality

The current encrypted transport provides confidentiality and AEAD integrity for the encrypted payload.

### Remaining Sender-Authenticity Limitation

The current encrypted envelope does not yet provide a dedicated end-to-end signature proving that the payload cryptographically came from the claimed Hanbova sender.

Sender identity currently depends on authenticated backend coordination.

A future version may use a dedicated signing identity, such as Ed25519, to sign canonical envelopes.

---

## 19. Status of Prior Blockers & Verified Resolutions

| Item | Description | Verified Status |
| --- | --- | --- |
| 1 | Mainnet Safety Lock | **Resolved ✅** — `NetworkConfig.mainnet.isEnabled = false`; runtime switching blocked; UI clearly displays `TEST MODE`. |
| 2 | Mocked On-Chain Deposit | **Resolved ✅** — Placeholder on-chain deposit address disabled/removed. |
| 3 | Android CDK C-FFI Packaging | **Resolved ✅** — `libhanbova_cdk_ffi.so` compiled for `arm64-v8a` & `x86_64` in `android/app/src/main/jniLibs/`. |
| 4 | iOS CDK C-FFI Packaging | **Resolved ✅** — Universal static `HanbovaCdkFfi.xcframework` linked to iOS Runner (`DynamicLibrary.process()`). |
| 5 | NUT-04 Test Funding Lifecycle | **Resolved ✅** — Verified quote creation and minting via controlled local test mint integration. |
| 6 | Two-Device Claim Flow | **Resolved ✅** — Genuine NUT-11 P2PK token created by Alice, encrypted, relayed, decrypted, and claimed by Bob. |
| 7 | Two-Device Refund Flow | **Resolved ✅** — Locktime-gated sender refund path executed and validated against mint state. |
| 8 | Restart Persistence | **Resolved ✅** — Per-user, per-environment `redb` database storage under `{app_support}/wallets/{env}/{userId}/wallet.redb`. |
| 9 | Mnemonic & Restore Safety | **Resolved ✅** — Context-isolated BIP-39 mnemonic lifecycle; deterministic identity derivation; restore navigates through authentication. |
| 10 | Cryptographic Key Recovery | **Resolved ✅** — Deterministic HMAC-SHA512 derivation of P2PK (secp256k1) and Transport (X25519) keys from BIP-39 seed. |
| 11 | Backend Sender Authentication | **Resolved ✅** — `payload.sender_id = Some(auth_user.user_id)` enforced server-side; client spoofing rejected. |
| 12 | Status Authorization Hardening | **Resolved ✅** — Strict role-based transitions (`"claimed"` strictly recipient, `"refunded"` strictly sender); returns `403 Forbidden` on unauthorized access. |
| 13 | Recipient Lookup & Error Polish | **Resolved ✅** — Resolution supports both `@username` and email (`user@domain.com`); Dart exception prefixes stripped from UI banners. |
| 14 | Claim Success Copy Corrections | **Resolved ✅** — Removed false implications of direct on-chain Bitcoin/Lightning settlement; copy accurately states: *"Protected payment claimed successfully. Your spendable balance has been updated."* |

---

## 20. Git Workflow

Security-sensitive work should not be committed directly to `main`.

Use milestone branches and pull requests.

Current intended branch naming:

```text
milestone/3a2-real-cdk-wallet
milestone/3a2-1-mobile-stabilization
```

Going forward:

```text
branch
  |
  v
implementation
  |
  v
tests
  |
  v
code review
  |
  v
PR
  |
  v
merge
```

Do not automatically merge security-sensitive wallet work.

---

## 21. Correct Roadmap Status

Use the following development status until further verification:

| Milestone | Status |
| --- | --- |
| Milestone 1: Foundation | Completed ✅ |
| Milestone 2: Protected Payment Protocol | Completed at protocol/reference-test level ✅ |
| Milestone 2.5: Consumer Wallet UX & Brand V4 | Completed ✅ |
| Milestone 3A: Two-device Cashu Test Wallet | Completed ✅ |
| Milestone 3A.1: Client Wallet Authority & Key Correction | Completed ✅ |
| Milestone 3A.2: Real CDK Integration | Completed ✅ |
| **Milestone 3A.3: Functional Hardening & Daily-Use Reliability** | **Completed ✅** |
| Milestone 3B: Production Lightning Wallet | Development / experimental 🚧 |
| Milestone 4: Recovery/Hardening | Partial (Deterministic P2PK/Transport restored; full NUT-13 proof restoration in progress) ⚠️ |
| Milestone 5: Mainnet Beta | Mainnet disabled / safety-locked 🔒 |

---

## 22. Milestone 3A.2.1 & Pre-Demo Accomplishments

### Milestone 3A.2.1 & Pre-Demo Readiness
### Mobile Integration, Safety Stabilization & Copy Accuracy

Completed items:

- **Mainnet Safety**: Disabled `NetworkConfig.mainnet.isEnabled = false`; runtime switching blocked; UI clearly displays `TEST MODE`.
- **Backend Authorization**: Overwritten `payload.sender_id = Some(auth_user.user_id)` to eliminate sender spoofing; unauthorized access returns `403 Forbidden`; strict role-based status updates (`"claimed"` strictly recipient, `"refunded"` strictly sender).
- **Mobile C-FFI Binary Packaging**: Compiled Android NDK native libraries (`arm64-v8a` and `x86_64`) into `android/app/src/main/jniLibs/`; compiled universal static framework (`HanbovaCdkFfi.xcframework`) linking physical ARM64 (`aarch64-apple-ios`) and universal simulator (`aarch64-apple-ios-sim` + `x86_64-apple-ios`).
- **Wallet Database Isolation**: Redb embedded storage isolated per-user and per-environment under `{app_support}/wallets/{environment}/{userId}/wallet.redb`.
- **Controlled Cashu Mint Verification**: Verified NUT-04 funding quote creation & minting via controlled local Cashu mint integration using test/mock Lightning settlement, two-user NUT-11 P2PK protected send (Alice &rarr; Bob claim with exact balance assertions), and Alice post-locktime refund.
- **Payment Status Authority**: Formally documented that backend `claimed`/`refunded` states are coordination metadata for UI; true cryptographic spending authority and proof state resides strictly in the Cashu mint.
- **Recipient Lookup by Email/Handle**: Enabled recipient lookup by `@username`, `username`, or registered email address (`user@domain.com`) with canonical handle formatting and URL encoding.
- **Presentation Copy Standards**: Corrected claim success screens and toasts to accurately communicate ecash balance updates rather than implying direct on-chain Bitcoin or Lightning settlement.
- **Error Presentation Cleanup**: Stripped runtime Dart wrapper prefixes (`Bad state:`, `Exception:`) from user-facing error banners.
- **Logging & Secrets Audit**: Confirmed zero private keys, seed phrases, bearer tokens, or database secrets are logged.
- **All Automated Tests Passing**: **24/24 Rust tests**, **136/136 Flutter tests** (including Brand V4 visual and token verification tests).

---

## 23. Milestone 3A.3 Completed Highlights

### Functional Hardening & Daily-Use Reliability (Active Branch: `main`)

Completed items:

- **Single Financial Source of Truth**: Enforced native CDK redb embedded storage (`CashuWalletService.getBalance()`) and Cashu mint proof state as the sole financial source of truth. Purged synthetic balance fallbacks and demo seeders (`seedDemoTransactions()`).
- **Three-Tier State Architecture & Reconciliation**: Explicitly separated Financial State (`Spendable`, `Locked Escrow`, `Claimed`, `Refunded`), Delivery State (`Delivered`, `Delivery Pending`), and Coordination State (`Synchronized`, `Coordination Sync Pending`). Local settled states (`completed`, `refunded`) are preserved against stale backend coordination status.
- **Canonical Transaction Deduplication**: Idempotent upsert by canonical payment ID in `TransactionsNotifier`, preventing duplicate records on relay or state changes.
- **Delivery Retry Key-Rotation Safety**: Delivery retry strictly maintains original payment keys and envelopes without creating new tokens or rotating recipient keys.
- **Client Authority Resilience**: When CDK operations succeed but backend synchronization fails, transactions transition to `coordinationSyncPending`. Delivery drops preserve escrow and enable deduplicated retry without creating redundant ecash.
- **Double-Tap & Concurrency Safety**: All financial trigger actions (Protected Send, Add Bitcoin quote minting, Claim, Refund, Instant Send) feature loading state flags and disabled handlers during execution.
- **Consumer Error Translation**: Built `ConsumerErrorTranslator` to strip internal runtime wrapper prefixes and map errors to clear consumer-friendly messages (*"Mint unreachable"*, *"Payment already claimed or refunded"*, *"Recipient wallet identity changed"*, *"Insufficient spendable balance"*), while automatically redacting hex secrets $\ge 32$ characters and local filesystem URIs.
- **Truthful Recovery Scope**: Updated `RestoreSeedScreen` to truthfully inform users that mnemonic restoration recovers signing keys and account identity, whereas off-device ecash proof recovery requires local database backup until server-assisted proof restoration (NUT-13) is supported.
- **Genuine Local Nutshell 0.16.5 Integration**: Verified NUT-11 claim, locktime refund, wrong-key rejection, early-refund rejection, and late-claim rejection against running Nutshell mint.
- **Live Encrypted Message Relay & Redb Persistence**: Verified Alice &rarr; Bob ChaCha20-Poly1305 encrypted envelope relay, inbox retrieval, Bob decryption, and claim execution across wallet service reconstruction.
- **Hermetic CI Isolation**: Live integration test isolated under `HANBOVA_RUN_LIVE_INTEGRATION=true`, keeping normal CI 100% hermetic.
- **Quality Gates**: **Standard Flutter: 150 passed / 0 failed / 1 skipped**, **Live Local Flutter: 1 passed / 0 failed**, **Standard Rust: 22 passed / 0 failed / 2 ignored**, **Local-Mint Rust: 2 passed / 0 failed**, **Flutter Analyzer: 0 issues**, **Rust Clippy: 0 warnings**, **GitHub App CI: GREEN**, **GitHub Backend CI: GREEN**.
- **Scope Clarification**: *Interactive Android/iOS two-app device verification remains a manual QA gate (`NOT VERIFIED` on physical/emulator hardware) and is tracked separately from M3A.3 engineering completion.*

---

## 24. Milestone M3B.2.1 Completed Highlights: Financial Dashboard & Product Experience Completion

### Consumer Financial Management & Demo Experience (Branch: `milestone/3b2-1-financial-experience`)

Completed items:

- **Dedicated Money & Balances Hub**: Implemented `MoneyScreen` featuring Total Wallet Balance, Available Balance, Protected Balance (with waiting vs refundable breakdown), and Pending Balance in genuine Bitcoin satoshis with multi-currency reference conversion.
- **8-Currency Multi-Currency Presentation**: Full deterministic support for `NGN`, `USD`, `KES`, `GHS`, `RWF`, `UGX`, `TZS`, and `ZAR`. Changing display currency preserves underlying sats.
- **Financial Insights & Period Analytics**: Implemented `InsightsScreen` supporting period filters (*This week*, *This month*, *Last month*, *3 months*, *This year*, *Custom*), Money In, Money Out, Net Flow, Fees, category spending, country spending (Kenya 🇰🇪, Nigeria 🇳🇬, Ghana 🇬🇭), and currency usage summaries.
- **Pending & Attention Hub**: Unified in-flight payments, uncertain bill payment warnings (*"Checking payment status. Please don't pay again yet."*), 1-tap claim for expired protected refunds, low eSIM data alerts, and backup reminders in `PendingCentreScreen`.
- **Notifications Centre**: Deterministic notification feed across Transactions, Protected escrow, Bills, eSIM, Travel, and Security with read state management and amount privacy masking in `NotificationsScreen`.
- **Account Statements & Export**: Monthly statements with opening balance, in, out, fees, closing balance, transaction count, CSV export, and PDF download modals in `StatementsScreen`.
- **People & Beneficiaries**: Contact management for Lightning addresses, Mobile Money numbers, and Bank accounts with instant send actions in `BeneficiariesScreen`.
- **Payment Requests & QR Sharing**: Sats/fiat switchable amount requests, notes, QR code invoice generation, copy link, and share invoice actions in `RequestMoneyScreen`.
- **Saved Billers Management**: Utility meters and phone lines management with reference editing and 1-tap Pay Again actions in `SavedPaymentsScreen`.
- **Virtual Cards (Sandbox)**: Complete mock card lifecycle featuring masked PAN, tap-to-reveal CVV/Expiry, sats funding modal, freeze switch, and card transactions in `CardsScreen`.
- **Normalized Receipts & Help Sheet**: Unified receipt bottom sheet across all transaction types with prepaid electricity token copy and categorized "Need Help?" issue tickets in `TransactionReceiptSheet`.
- **Privacy & Security Settings**: Mask balances in app, hide snapshot in multitasking app switcher, hide notification amounts, and biometric re-authentication for sensitive actions in `privacyProvider` and `ProfileScreen`.
- **Country & Spend Market Separation**: Registration sets Residence only; Spend Market toggled only in Travel Hub without mutating user residence.
- **Deterministic Demo Mode**: `demoModeProvider` showcases realistic African multi-market balances, statements, notifications, and cards without contaminating real Cashu wallet authority.
- **Quality Gates**: **182/182 Flutter tests passed**, **0 Analyzer issues**, **Android Debug APK verified**, **iOS Simulator app verified**, **29/29 Rust tests passed**. Detailed report in `hanbova-docs/verification/M3B2_1_FINANCIAL_EXPERIENCE.md`.

---

## 25. Milestone M3B.2.2 Completed Highlights: Financial Interaction & Navigation Redesign

### Consumer Financial Interaction Architecture (Branch: `milestone/3b2-2-financial-interaction-redesign`)

Completed items:

- **5-Tab Direct Navigation Shell**: Upgraded `AppShell` and `router.dart` from 4 tabs + center FAB to 5 direct, primary navigation branches: `Home` (0), `Pay` (1), `Activity` (2), `Travel` (3), `Me` (4).
- **Refined Home Screen & Attention Hub**: Authoritative Bitcoin balance hero with spendable vs protected breakdown, 4-button quick action grid (`Send`, `Receive`, `Pay`, `Scan`), and priority action cards (`Protected Refund Ready`, `Payment Processing (Uncertain)`, `eSIM Low Data`, `Incoming Claimable`).
- **Everyday Pay & Spend Hub**: Built `PayHubScreen` with active Spend Market switcher, Pay Again recent recipient carousels, Everyday bills & utilities grid (`Airtime`, `Data Bundles`, `Electricity`, `TV Cables`, `Internet`, `Water Bills`), and payment management shortcuts (`Saved Billers`, `Virtual Cards`, `Request Money`).
- **Dedicated Airtime Recharge Flow**: `AirtimeFlowScreen` supporting "My Number" vs "Someone Else" phone toggle, operator branding, preset chips, and custom amount input with live sats conversion.
- **Dedicated Data Bundle Flow**: `DataBundleFlowScreen` supporting validity tabs (`Daily`, `Weekly`, `Monthly`) and bundle tier cards.
- **Dedicated Electricity Flow**: `ElectricityFlowScreen` supporting saved meter selection, DISCO dropdown, verification, prepaid token generation, and clipboard copy action.
- **Dedicated TV Subscription Flow**: `TvSubscriptionFlowScreen` supporting provider selector (`DSTV`, `GOtv`, `StarTimes`) and bouquet package tiers.
- **Standardized Visual Interaction Grammar**:
  - `PaymentConfirmationSheet`: Source Bitcoin wallet, destination, fiat amount, fee breakdown (Network fee 50 sats, Total deducted), and confirm button.
  - `PaymentSuccessSheet`: High-contrast confirmation, 20-digit electricity prepaid token display with copy button, official receipt modal view, and saved biller quick repeat toggle.
  - `PaymentUncertainSheet`: Reassuring pending status notice explaining downstream provider verification without panicking the user, with direct routing to Pending Centre.
  - `PaymentFailedSheet`: Clear failure categorization, actionable remediation, and retry options.
- **Deterministic Demo Mode Persona**: Configured Nigeria Residence, Nigeria Spend Market (`NG`), calibrated balances (2,450,000 sats total, 1,800,000 spendable, 300,000 protected waiting, 150,000 protected refundable, 200,000 pending), and memory isolation.
- **Quality Gates**: **213/213 Flutter tests passed**, **0 Analyzer issues**, **iOS Simulator app verified**. Detailed reports in `hanbova-docs/verification/M3B2_2_FINANCIAL_INTERACTION_REDESIGN.md` and `hanbova-docs/verification/M3B2_2_GLOBAL_REGISTRATION_MARKET_ADAPTIVE.md`.

### Global Registration & Market-Adaptive Architecture Correction
- **Global Any-Country Registration**: Complete 247 ISO 3166-1 alpha-2 dataset in `CountryInfo.allCountries` with name and ISO code search.
- **Consumer Terminology**: Standardized on "Country of residence".
- **Stepped Onboarding Flow**: Welcome → Names + Username → Email + Password → Country of Residence → Country Confirmation → Wallet Setup → Security → Recovery → Home.
- **Decoupled User Context**: Immutable `residenceCountry` separated from mutable `activeMarket`, `displayCurrency`, and `roamEnabled`.
- **Market Capabilities Matrix**: Strict distinction between global baseline (Bitcoin, Cashu, Send/Receive, Protected, Scan/Request) and local markets (billers, domestic bank payouts, mobile money).
- **Capability-Adaptive UI**:
  - Home action rail automatically adapts; Quick Pay is hidden when no everyday bills exist.
  - Pay Hub displays "Activate Roam to use supported local services" banner for global markets, and capability-filtered biller grids for supported markets.
  - Roam screen catalog with destination capability preview before activation.
  - Profile screen displays Residence, Active Market, and Roam status separately.
- **Developer Options Personas**: Persona A (NG, Roam off), Persona B (US, Roam off), Persona C (US, Roam KE).

---

## 26. Fellowship-Safe Project Description

Use this wording when describing Hanbova today:

> **Hanbova is an open-source Bitcoin/Cashu wallet project exploring safer everyday payments in Africa. It combines a consumer wallet experience with Cashu NUT-11 protected payments, where a recipient can claim using their key and the sender gains a refund path after a chosen locktime. The project currently uses a client-side CDK wallet architecture and encrypted payment delivery, and is being validated on valueless test environments before any Mainnet release.**

Avoid saying:

- Mainnet ready
- production beta ready
- guaranteed refund
- automatic refund
- fully non-custodial Cashu
- on-chain refund
- production Lightning complete

---

## 27. Fellowship Demo Goal

The strongest demo is not a long feature tour.

The target demo should be:

```text
Device A: Alice
      |
      | 100 test sats Protected
      v
     @bob
      |
      v
Device B: Bob
      |
      | Claim
      v
Mint validates P2PK
      |
      v
Bob +100 test sats
```

Then:

```text
Alice sends again
      |
Bob does nothing
      |
locktime passes
      |
Alice taps Refund
      |
mint validates refund path
      |
Alice gets test sats back
```

This should be shown using genuine valueless Cashu proofs, not local simulation.

---

## 26. Final Engineering Principle

Every Hanbova feature should answer three questions:

1. **What real user problem does this solve?**
2. **Where does the financial/cryptographic authority live?**
3. **Can we prove the behavior against the real protocol, not only through mocked UI/tests?**

For the current milestone, the answer must remain:

> **Client owns wallet authority. Backend coordinates. Cashu mint enforces the protected spend. Mainnet waits until test-mode proof is complete.**
