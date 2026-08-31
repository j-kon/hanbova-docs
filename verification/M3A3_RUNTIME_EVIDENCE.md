# Milestone 3A.3: Runtime & Verification Evidence

**Milestone**: 3A.3 Functional Hardening & Daily-Use Reliability  
**Date**: 2026-08-31  
**Target Environments**: Cashu Test Network (`wallet_cashu_test`), Signet / Regtest mint  
**Core Financial Invariant**: Client owns cryptographic wallet authority; Cashu mint proof state is authoritative; backend manages coordination.

---

## 1. Verification Classification Matrix

Every item and scenario is classified as exactly one of: `AUTOMATED VERIFIED`, `RUNTIME VERIFIED`, or `NOT VERIFIED`.

| Verification Item | Classification | Method / Evidence |
| --- | --- | --- |
| **Brand V4 Design System & Visual Assets** | `RUNTIME VERIFIED` | Live iOS simulator execution (`ios_m3a3_live.png`) |
| **Single Financial Source of Truth (redb / CDK)** | `AUTOMATED VERIFIED` | `test/financial_authority_test.dart`, `test/m3a3_functional_hardening_test.dart` |
| **Balance Auto-Refresh on Mint / Invalidation** | `AUTOMATED VERIFIED` | `unified_deposit_sheet.dart` & `cashu_wallet_provider.dart` tests |
| **State Reconciliation & Sync Lag Resilience** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (preserving settled local state) |
| **Canonical Transaction Deduplication** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (idempotent upsert by ID) |
| **Refunded Incoming Status Mapping** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` (refunded mapped to TransactionStatus.refunded) |
| **Retry Delivery Key Rotation Rejection** | `AUTOMATED VERIFIED` | `test/financial_authority_test.dart`, `test/m3a3_functional_hardening_test.dart` |
| **Double-Tap & Concurrency Safe Buttons** | `AUTOMATED VERIFIED` | `_RefundActionButton`, `_ClaimActionButton`, `protected_send_screen.dart` |
| **Consumer Error Translation & Secret Redaction** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` |
| **Truthful Recovery Scope Copy** | `AUTOMATED VERIFIED` | `test/m3a3_functional_hardening_test.dart` |
| **Two-App Live Runtime Claim (Alice &rarr; Bob)** | `NOT VERIFIED` | Requires simultaneous two-device test-mint session |
| **Two-App Live Runtime Refund (Alice Refund)** | `NOT VERIFIED` | Requires simultaneous two-device test-mint session |
| **Two-App Live Runtime Restart Persistence** | `NOT VERIFIED` | Requires simultaneous two-device test-mint session |
| **CDK Scenario Tests (Local Mint Required)** | `NOT VERIFIED` | Ignored in standard `cargo test` (`cdk_test.rs`, requires running local test mint) |
| **Public Mainnet Pilot / Live Sats** | `NOT VERIFIED` | Mainnet safety lock strictly active (`NetworkConfig.mainnet.isEnabled = false`) |

---

## 2. Core Operational Scenarios

### Scenario A: Protected Send & Claim Flow (Alice &rarr; Bob)
- **Protocol Flow**: Alice funds ecash &rarr; creates NUT-11 P2PK locked token &rarr; encrypts envelope with Bob's X25519 transport key &rarr; relays via backend &rarr; Bob decrypts, validates fingerprint, and executes CDK `claimProtectedPayment` &rarr; mint verifies P2PK witness &rarr; Bob balance increments.
- **Verification Status**: `NOT VERIFIED` in live two-device runtime session. (CDK integration tests in `crates/hanbova-protected-payments/src/cdk_test.rs` require a running local test mint and are ignored in standard cargo test suite).

### Scenario B: Sender Refund Post-Locktime (Alice Refund)
- **Protocol Flow**: Alice locks sats with locktime &rarr; locktime passes &rarr; Alice taps *"Refund available"* via `_RefundActionButton` &rarr; CDK signs refund spend transaction with Alice refund private key &rarr; mint settles &rarr; Alice spendable balance restored &rarr; Bob subsequent claim rejected.
- **Verification Status**: `NOT VERIFIED` in live two-device runtime session.

### Scenario C: App Restart & Database Persistence
- **Protocol Flow**: Alice creates protected payment &rarr; process forcefully terminated &rarr; relaunch opens existing `{app_support}/wallets/cashu_test/{userId}/wallet.redb` &rarr; SQLite retrieves escrow record &rarr; balances and refund capabilities persist intact.
- **Verification Status**: `NOT VERIFIED` in live two-device runtime session.

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
