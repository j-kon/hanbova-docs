# Milestone 3A.3: Verification Classification & Runtime Evidence

**Milestone**: 3A.3 Functional Hardening & Daily-Use Reliability  
**Date**: 2026-08-31  
**Target Environments**: Local Development (`wallet_local`), Cashu Test Network (`wallet_cashu_test`), Signet / Regtest mint  
**Core Financial Invariant**: Client owns cryptographic wallet authority; Cashu mint proof state is authoritative; backend manages coordination.

---

## 1. Verification Classification Matrix

Every item and scenario is classified into exactly one of four rigorous verification tiers:
- `AUTOMATED VERIFIED`: Unit, widget, property, and state reconciliation tests passing in hermetic CI.
- `LIVE LOCAL INTEGRATION VERIFIED`: End-to-end integration tests executed against a running local Nutshell test mint and live Hanbova API backend.
- `DEVICE RUNTIME VERIFIED`: Real mobile UI / visual assets captured live on simulated or physical mobile hardware.
- `NOT VERIFIED`: Requires manual two-device physical coordination or future milestone activation.

| Verification Item | Classification | Method / Evidence |
| :--- | :--- | :--- |
| **Brand V4 Design System & Visual Assets** | `DEVICE RUNTIME VERIFIED` | Live iOS simulator & Android emulator execution (`ios_live_both.png`, `android_live_both.png`) |
| **Single Financial Source of Truth (redb / CDK)** | `AUTOMATED VERIFIED` | `test/financial_authority_test.dart`, `test/m3a3_functional_hardening_test.dart` |
| **Balance Auto-Refresh on Mint / Invalidation** | `AUTOMATED VERIFIED` | `unified_deposit_sheet.dart` & `cashu_wallet_provider.dart` tests |
| **State Reconciliation & Sync Lag Resilience** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (preserving settled local state) |
| **Canonical Transaction Deduplication** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (idempotent upsert by ID) |
| **Refunded Incoming Status Mapping** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (refunded mapped to TransactionStatus.refunded) |
| **Retry Delivery Key Rotation Rejection** | `AUTOMATED VERIFIED` | `test/financial_authority_test.dart`, `test/m3a3_functional_hardening_test.dart` |
| **Double-Tap & Concurrency Safe Buttons** | `AUTOMATED VERIFIED` | `_RefundActionButton`, `_ClaimActionButton`, `protected_send_screen.dart` |
| **Consumer Error Translation & Secret Redaction** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` |
| **Truthful Recovery Scope Copy** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` |
| **Local-Mint CDK Claim (`cdk_test.rs`)** | `LIVE LOCAL INTEGRATION VERIFIED` | `cargo test -p hanbova-protected-payments cdk_test -- --ignored` |
| **Local-Mint CDK Refund (`cdk_test.rs`)** | `LIVE LOCAL INTEGRATION VERIFIED` | `cargo test -p hanbova-protected-payments cdk_test -- --ignored` |
| **Two-Wallet Live Encrypted Relay & Claim (Scenario A)** | `LIVE LOCAL INTEGRATION VERIFIED` | `test/live_local_integration_test.dart` (live against local Nutshell mint & Hanbova API) |
| **Two-Wallet Live Refund & Late Claim Rejection (Scenario B)** | `LIVE LOCAL INTEGRATION VERIFIED` | `test/live_local_integration_test.dart` (live against local Nutshell mint) |
| **Two-Wallet Redb Reconstruction & Relay Persistence (Scenario C)** | `LIVE LOCAL INTEGRATION VERIFIED` | `test/live_local_integration_test.dart` (wallet service reconstruction & inbox decryption) |
| **Actual Two-App Device Claim (Alice &rarr; Bob)** | `NOT VERIFIED` | Requires interactive multi-device testing session |
| **Actual Two-App Device Refund (Alice Refund)** | `NOT VERIFIED` | Requires interactive multi-device testing session |
| **Actual Two-App Mobile Force-Close / Reopen Persistence** | `NOT VERIFIED` | Requires OS process kill & launch on mobile device |
| **Public Mainnet Pilot / Live Sats** | `NOT VERIFIED` | Mainnet safety lock strictly active (`NetworkConfig.mainnet.isEnabled = false`) |

---

## 2. Operational Scenarios Validated

### Scenario A: Real Protected Claim (Alice &rarr; Bob)
- **Protocol Flow**: Alice funds ecash &rarr; creates NUT-11 P2PK locked token &rarr; encrypts envelope with Bob's X25519 transport key &rarr; relays via backend &rarr; Bob decrypts, validates fingerprint, and executes CDK `claimProtectedPayment` &rarr; mint verifies P2PK witness &rarr; Bob balance increments &rarr; Alice subsequent refund attempt rejected because mint marks proofs spent.
- **Verification Status**: `LIVE LOCAL INTEGRATION VERIFIED`
- **Execution Evidence**:
  - `hanbova-backend`: `test_scenario_a_bob_claims_with_p2pk` passed on local Nutshell 0.16.5 mint.
  - `hanbova-app`: `test/live_local_integration_test.dart` executed full live lifecycle:
    1. Alice funded 1000 sats from local mint via CDK `createMintQuote` + `mintQuote`.
    2. Alice locked 100 sats in NUT-11 token for Bob (`spendable: 900`, `escrow: 100`).
    3. Alice encrypted envelope via ChaCha20-Poly1305 + X25519 ECDH + HKDF.
    4. Alice relayed message through backend relay API (`POST /api/v1/protected-messages`).
    5. Bob fetched inbox from backend relay (`GET /api/v1/protected-messages/inbox`).
    6. Bob decrypted envelope using his transport keypair.
    7. Bob claimed 100 sats via CDK `claimProtectedPayment`.
    8. Bob spendable balance became 100 sats.
    9. Alice refund failed because mint marks proofs spent.
    10. Activity list contains exactly 1 canonical transaction.

### Scenario B: Real Sender Refund & Late Claim Rejection
- **Protocol Flow**: Alice locks sats with locktime &rarr; locktime passes &rarr; Alice executes refund via CDK `refundProtectedPayment` &rarr; CDK signs refund spend transaction with Alice refund private key &rarr; mint settles &rarr; Alice spendable balance restored &rarr; Bob subsequent late claim rejected because proofs were already spent by Alice refund.
- **Verification Status**: `LIVE LOCAL INTEGRATION VERIFIED`
- **Execution Evidence**:
  - `hanbova-backend`: `test_scenario_b_alice_refunds_after_locktime` passed on local Nutshell 0.16.5 mint.
  - `hanbova-app`: `test/live_local_integration_test.dart` executed full live lifecycle:
    1. Alice created 100 sat protected payment with 2-second locktime (`spendable: 800`).
    2. Early refund before locktime rejected by mint (`StateError`).
    3. Waited 3 seconds for locktime expiry.
    4. Alice executed refund &rarr; received 100 sats (`spendable: 900`).
    5. Bob late claim attempt rejected by mint (`StateError`) because proofs were spent by Alice refund.

### Scenario C: Redb Wallet Reconstruction & Backend Relay Persistence
- **Protocol Flow**: Alice creates protected payment &rarr; Alice encrypts and relays token via backend &rarr; wallet services disposed & nullified &rarr; Alice wallet reconstructed pointing to existing Redb directory (balance persists) &rarr; Bob reconstructs transport identity from seed &rarr; Bob fetches relayed message from backend &rarr; Bob decrypts envelope &rarr; Bob reconstructs CDK wallet & claims token &rarr; claim succeeds.
- **Verification Status**: `LIVE LOCAL INTEGRATION VERIFIED`
- **Execution Evidence**:
  - `hanbova-app`: `test/live_local_integration_test.dart` executed:
    1. Alice created 100 sat protected send with 60-second locktime (`spendable: 800`).
    2. Alice encrypted and relayed message via backend API.
    3. Disposed and nullified both wallet service instances.
    4. Reopened Alice with fresh instance pointing to existing Redb directory: verified Alice spendable balance (800 sats) persisted.
    5. Rederived Bob X25519 transport keypair from Bob seed.
    6. Bob fetched inbox afresh from backend API, located message, and decrypted envelope.
    7. Reconstructed Bob CDK wallet instance pointing to existing Redb directory.
    8. Bob executed claim using decrypted token &rarr; claimed 100 sats (`Bob spendable balance: 200 sats`).
- **Scope Note**: This integration test verifies Redb balance persistence and backend relay decryption across service reconstruction in Dart. Actual mobile OS process termination and SQLite persistent escrow storage on device is classified as a separate manual/device runtime gate (`NOT VERIFIED` until on-device session).

---

## 3. Recovery Truthfulness Specification

- **Identity Recovery (`MnemonicService.restoreFromMnemonic`)**:
  - Restores 512-bit master CDK wallet seed.
  - Derives deterministic primary secp256k1 P2PK keypair.
  - Derives deterministic X25519 transport encryption keypair.
- **Proof Recovery Scope**:
  - **Local Device**: Persisted in encrypted SQLite & local redb embedded database.
  - **Fresh Device Restore**: Requires local database transfer or future server-assisted proof restoration (NUT-13).
  - User copy in `RestoreSeedScreen` truthfully states: *"Your signing keys and account identity have been restored. Ecash proofs stored locally on another device require local database transfer until server-assisted proof restoration (NUT-13) is supported."*

---

## 4. Test Suite Execution Summary

### Flutter Test Suites
- **Standard Hermetic Suite**: **150 passed / 0 failed / 1 skipped** (100% passing across 108 test suites; zero external dependencies)
- **Live Local Integration Suite**: **1 passed / 0 failed** (explicitly executed via `HANBOVA_RUN_LIVE_INTEGRATION=true flutter test test/live_local_integration_test.dart`)
- **Flutter Analyzer**: **0 issues found**
- **Flutter Formatter**: Clean across 109 Dart files (`dart format --output=none --set-exit-if-changed .`)
- **Android APK Build**: `build/app/outputs/flutter-apk/app-debug.apk` built successfully

### Rust Backend Test Suites
- **Standard Test Suite**: **22 passed / 0 failed / 2 ignored** (`cargo test --workspace --all-targets`)
- **Local-Mint Test Suite**: **2 passed / 0 failed** (`cargo test -p hanbova-protected-payments cdk_test -- --ignored`)
- **Rust Clippy**: **0 warnings** (`cargo clippy --workspace --all-targets -- -D warnings`)
- **Rust Formatter**: Clean (`cargo fmt --all -- --check`)
- **Mint Fee Assumption Note**: Assertions in `cdk_test.rs` are pinned to the local `cashubtc/nutshell:0.16.5` FakeWallet configuration (zero split/swap fees).

### Safety & Governance
- **Mainnet Protection**: `NetworkConfig.mainnet.isEnabled = false` (Compile-time and runtime safety locked).
