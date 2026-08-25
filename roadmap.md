# Hanbova Product & Technical Roadmap

## Milestone Overview

| Milestone | Scope | Status |
| :--- | :--- | :--- |
| **Milestone 1** | Monorepo Bootstrap & Clean Architecture Foundation | `COMPLETED` ✅ |
| **Milestone 2** | Genuine Cashu NUT-10 & NUT-11 Protected Payments | `COMPLETED` ✅ |
| **Milestone 2.5** | Consumer Wallet Experience & Centralized Design System | `COMPLETED` ✅ |
| **Milestone 3A** | Real Two-Device Cashu Test Wallet + Secure Encrypted Delivery | `COMPLETED` ✅ |
| **Milestone 3A.1** | Security Correction: Move Cashu Wallet Authority to Client & Genuine Secp256k1 | `COMPLETED` ✅ |
| **Milestone 3A.2** | Real CDK Client Wallet Integration via Thin Rust C-FFI Bridge & Zero-Custody Backend Hardening | `COMPLETED` ✅ |
| **Milestone 3A.2.1** | Mobile Integration & Safety Stabilization (Android NDK / iOS FFI, Mainnet Lock, Authorization 403) | `COMPLETED` ✅ (on `milestone/3a2-1-mobile-stabilization`) |
| **Milestone 3B** | Lightning Integration & Ecash Swaps (NUT-04/NUT-05) | `Development / experimental` 🚧 |
| **Milestone 4** | Production Hardening, BIP-39 Backup, Biometrics & Multi-Mint | `Partial` ⚠️ |
| **Milestone 5** | Public Mainnet Beta, On-Chain Swaps & App Store Publishing Readiness | `Mainnet disabled / future` 🔒 |

---

## Milestone 3A.2.1 Completed Highlights (Current Active Branch: `milestone/3a2-1-mobile-stabilization`)
- **Mainnet Safety Hardening**: Strict safety lock active (`NetworkConfig.mainnet.isEnabled = false`); runtime switching blocked; UI clearly displays test environment badge (`TEST MODE`).
- **Backend Authorization & Spoofing Elimination**: Unconditionally overwritten `payload.sender_id = Some(auth_user.user_id)`; returns HTTP 403 Forbidden for unauthorized access; enforces strict role state transitions (`"claimed"` strictly recipient, `"refunded"` strictly sender).
- **Mobile C-FFI Binary Packaging**: Compiled Android NDK native libraries (`arm64-v8a` and `x86_64`) into `android/app/src/main/jniLibs/`; compiled universal static framework (`HanbovaCdkFfi.xcframework`) linking physical ARM64 (`aarch64-apple-ios`) and universal simulator (`aarch64-apple-ios-sim` + `x86_64-apple-ios`).
- **Wallet Database Isolation**: Redb embedded storage isolated per-user and per-environment under `{app_support}/wallets/{environment}/{userId}/wallet.redb`.
- **Financial Authority & Consistency Gate**:
  - Replaced all mock/synthetic claim pathways in `ClaimPaymentScreen` with genuine CDK NUT-11 witness settlements.
  - Eliminated simulated transaction seed data (`TransactionsNotifier` initializes strictly empty `[]`; demo seeder guarded by `kDebugMode`).
  - Strict Cashu ecash token secrecy enforced (tokens never placed in `TransactionModel` presentation metadata).
  - Delivery failure resilience: relay failures preserve locked ecash in client secure storage, allowing non-duplicating retry and locktime refunds.
  - Recipient binding regression fix: dynamic input edits clear cached recipient keys and re-resolve new recipient keys deterministically.
- **Payment Status Authority**: Documented backend states (`claimed`, `refunded`) as coordination metadata; true cryptographic spending authority resides strictly in the Cashu mint.
- **Automated Verification**: 24/24 Rust backend workspace tests pass, 0 clippy warnings, `flutter analyze` 0 issues, 63/63 Flutter tests pass.

---

## Milestone 3A.2 Completed Highlights
- **Thin Rust C-FFI Bridge (`crates/hanbova-cdk-ffi`)**: Wraps official `cdk` (`0.18.0-rc.0`) and persistent embedded `cdk-redb` storage with isolated per-user, per-network database paths.
- **Genuine Client-Side Cashu Wallet (`CdkCashuWalletServiceImpl`)**: Powered by official CDK over Dart FFI bindings. Eliminates all simulated fake proofs, random C points, fake keyset IDs, and fake mint balance checks.
- **Genuine NUT-04, NUT-07, NUT-11**: Official CDK mint quotes & swaps, NUT-11 P2PK locked send with recipient public key, sender refund public key, locktime, and `SigFlag::SigInputs`.
- **Zero-Custody Authorization Hardening**: Backend enforces authenticated `AuthUser` ownership on all payment intents, restricting visibility strictly to sender and recipient. Enforces strict state transitions (recipient claims, sender refunds, unauthorized third parties receive 403 Forbidden). Validates secp256k1 (33-byte compressed) and X25519 (32-byte) public keys.
- **Deterministic Persistent Mnemonic Derivation**: Active wallet mnemonic is stored persistently in secure storage and derived into 512-bit seed for CDK wallet. Backup screen displays active wallet mnemonic.

---

## Milestone 3A.1 Completed Highlights
- **PointyCastle Secp256k1 Service**: Genuine cryptographically secure 256-bit scalar generation in range $[1, n-1]$ with elliptic-curve scalar multiplication ($Q = G \cdot d$) to derive standard 33-byte compressed public keys.
- **Client-Side Wallet Authority**: Client wallet directly owns and stores all spendable Cashu proofs and NUT-11 escrow keys across restarts (`CashuWalletService` & `CashuWalletStorage`).
- **Zero-Custody Backend**: Completely stripped `claim_proof` and private keys from backend API models and database. Backend serves strictly as a zero-custody coordination and encrypted message relay service.
- **Automated Test Coverage**: 1,000-key uniqueness and validity tests, P2PK claim, unauthorized claim rejection, and post-locktime refund tests all passing.

---

## Milestone 3B Highlights (Active Development 🚧)
- **NUT-05 Lightning Melt Support**:
  - Implemented `hanbova_cdk_melt_quote` and `hanbova_cdk_melt` in `crates/hanbova-cdk-ffi`.
  - Exported all 12 C-FFI symbols across macOS dylib, Android JNI `.so`, and iOS universal `HanbovaCdkFfi.xcframework`.
  - Implemented `createMeltQuote` and `payMeltQuote` in `CashuWalletService` returning typed `MeltQuoteResult` and `MeltExecutionResult`.
  - Automated scenario tests passing in `client_wallet_authority_test.dart`.

---

## Milestone 4 Partial Highlights
- **Deterministic Key Hierarchy & Full Mnemonic Recovery**:
  - Domain-separated HMAC-SHA512 derivation from BIP-39 mnemonic seed (`Hanbova P2PK Identity Derivation` & `Hanbova X25519 Transport Derivation`).
  - Single 12-word mnemonic phrase fully recovers CDK wallet balance, secp256k1 P2PK identity, and X25519 transport identity on fresh installations.
- **BIP-39 Mnemonic Lifecycle**: 2048-word BIP-39 English dictionary and `MnemonicService` for 12-word seed phrase generation, validation, and autocomplete.
- **Backup & Verification**: Interactive `BackupSeedScreen` with tap-to-reveal, screenshot security warnings, and 3-word verification quiz.
- **Restore UI**: `RestoreSeedScreen` with 12-word entry, live autocomplete suggestions, and checksum verification.
- **Biometric Security**: `BiometricService` wrapping platform Face ID, Touch ID, and hardware key security.
- **Multi-Mint Ready**: `MintsScreen` for multi-mint management, active mint switching, and live NUT-11 capability probe validation.

