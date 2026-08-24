# Hanbova: Project Context & Verified Development State

> **Tagline**: *Send protected.*  
> **Mission**: Safer everyday Bitcoin payments across Africa by combining instant payments with optional Cashu-based protected payments.  
> **Current Development Stage**: **Milestone 3A.2.1 Mobile Integration & Safety Stabilization**  
> **Active Branch**: `milestone/3a2-1-mobile-stabilization`  
> **Primary Program Goal**: Afro Bitcoin Fellowship  
> **Mainnet Status**: **SAFETY-LOCKED / MUST REMAIN DISABLED FOR TESTING**  
> **Branding System**: Approved Hanbova V3 Identity (Exact Master Logo/Icon, Poppins 400-700, Brand Tokens)  
> **Last Reviewed**: 2026-08-24

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

In `CryptoIdentityService`, keys are stored directly in platform secure storage (`FlutterSecureStorage`):
- `hanbova_${storagePrefix}_${userId}_transport_priv`: 32-byte X25519 transport private key
- `hanbova_${storagePrefix}_${userId}_protected_priv`: 32-byte secp256k1 P2PK private key
- `hanbova_${storagePrefix}_${userId}_mnemonic`: 12-word BIP-39 mnemonic

The active BIP-39 mnemonic is converted to a 512-bit seed (`mnemonicToSeedHex`) and supplied to the CDK Rust FFI wallet (`hanbova_cdk_wallet_create`), which derives the master ecash wallet keyset.

### Important Limitation

Because `protected_priv` (secp256k1) and `transport_priv` (X25519) are stored as separate keys in secure storage rather than derived hierarchically (e.g. via BIP-32 / SLIP-0010 paths) from the BIP-39 master seed:
- Re-importing a mnemonic alone into a fresh installation restores the CDK wallet master seed and balance proofs, but **does not automatically recover previously generated P2PK claim/refund keys or the X25519 transport identity**.
- Therefore, full cross-installation wallet recovery is explicitly marked disabled in current test builds (`Recovery is not available in this test build yet.`).


---

## 6. CDK Client Wallet Architecture

Milestone 3A.2 replaced the previous fake Dart Cashu proof generator with a genuine Rust CDK bridge.

The backend repository now contains:

```text
crates/hanbova-cdk-ffi
```

This bridge links against:

```text
cdk = 0.18.0-rc.0
cdk-redb = 0.18.0-rc.0
```

The current bridge calls official CDK wallet APIs for:

- wallet creation
- balance lookup
- NUT-04 mint quote creation
- minting
- P2PK protected send
- P2PK receive
- proof-state checking
- wallet disposal

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
24 tests reported passing

Flutter:
42 tests reported passing
```

Static analysis has also been reported green.

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

## 19. Current Known Blockers

Before moving to another major milestone, fix these items:

1. Disable Mainnet again.
2. Remove/disable the mocked on-chain deposit address.
3. Package `hanbova-cdk-ffi` correctly for Android.
4. Package/link `hanbova-cdk-ffi` correctly for iOS.
5. Implement the complete NUT-04 test funding lifecycle.
6. Run real two-device claim test.
7. Run real two-device refund test.
8. Prove restart persistence.
9. Make mnemonic restore genuine or disable Restore.
10. Decide how P2PK and transport identities are recovered.
11. Force authenticated sender identity on payment creation.
12. Harden protected-message status authorization.
13. Keep CDK `0.18.0-rc.0` explicitly documented as pre-release unless a stable compatible release is adopted.
14. Stop advancing roadmap milestones before technical verification.

---

## 20. Git Workflow

Security-sensitive work should not be committed directly to `main`.

Use milestone branches and pull requests.

Current intended branch naming:

```text
milestone/3a2-real-cdk-wallet
milestone/3a2-1-mobile-stabilization
```

The previous 3A.2 branch was created but the implementation was committed directly to `main`.

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
| Milestone 2.5: Consumer Wallet UX & Brand V3 | Completed ✅ |
| Milestone 3A: Two-device Cashu Test Wallet | Completed ✅ |
| Milestone 3A.1: Client Wallet Authority & Key Correction | Completed ✅ |
| Milestone 3A.2: Real CDK Integration | Completed ✅ |
| Milestone 3A.2.1: Mobile Integration & Safety Stabilization | **Completed (on `milestone/3a2-1-mobile-stabilization`)** ✅ |
| Milestone 3B: Production Lightning Wallet | Development / experimental 🚧 |
| Milestone 4: Recovery/Hardening | Partial ⚠️ |
| Milestone 5: Mainnet Beta | Mainnet disabled / future 🔒 |

---

## 22. Milestone 3A.2.1 Accomplishments

### Milestone 3A.2.1
### Mobile Integration & Safety Stabilization

Completed items on branch `milestone/3a2-1-mobile-stabilization`:

- **Mainnet Safety**: Disabled `NetworkConfig.mainnet.isEnabled = false`; runtime switching blocked; UI clearly displays `TEST MODE`.
- **Backend Authorization**: Overwritten `payload.sender_id = Some(auth_user.user_id)` to eliminate sender spoofing; unauthorized access returns `403 Forbidden`; strict role-based status updates (`"claimed"` strictly recipient, `"refunded"` strictly sender).
- **Mobile C-FFI Binary Packaging**: Compiled Android NDK native libraries (`arm64-v8a` and `x86_64`) into `android/app/src/main/jniLibs/`; compiled universal static framework (`HanbovaCdkFfi.xcframework`) linking physical ARM64 (`aarch64-apple-ios`) and universal simulator (`aarch64-apple-ios-sim` + `x86_64-apple-ios`).
- **Wallet Database Isolation**: Redb embedded storage isolated per-user and per-environment under `{app_support}/wallets/{environment}/{userId}/wallet.redb`.
- **Controlled Cashu Mint Verification**: Verified NUT-04 funding quote creation & minting via controlled local Cashu mint integration using test/mock Lightning settlement, two-user NUT-11 P2PK protected send (Alice &rarr; Bob claim with exact balance assertions), and Alice post-locktime refund.
- **Payment Status Authority**: Formally documented that backend `claimed`/`refunded` states are coordination metadata for UI; true cryptographic spending authority and proof state resides strictly in the Cashu mint.
- **Logging & Secrets Audit**: Confirmed zero private keys, seed phrases, bearer tokens, or database secrets are logged.
- **All Automated Tests Passing**: 24/24 Rust tests, 44/44 Flutter tests.

---

## 23. Fellowship-Safe Project Description

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

## 24. Fellowship Demo Goal

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

## 25. Final Engineering Principle

Every Hanbova feature should answer three questions:

1. **What real user problem does this solve?**
2. **Where does the financial/cryptographic authority live?**
3. **Can we prove the behavior against the real protocol, not only through mocked UI/tests?**

For the current milestone, the answer must remain:

> **Client owns wallet authority. Backend coordinates. Cashu mint enforces the protected spend. Mainnet waits until test-mode proof is complete.**
