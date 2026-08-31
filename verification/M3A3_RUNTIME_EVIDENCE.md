# Milestone 3A.3: Runtime & Verification Evidence

**Milestone**: 3A.3 Functional Hardening & Daily-Use Reliability  
**Date**: 2026-08-31  
**Target Environments**: Cashu Test Network (`wallet_cashu_test`), Signet / Regtest mint  
**Core Financial Invariant**: Client owns cryptographic wallet authority; Cashu mint proof state is authoritative; backend manages coordination.

---

## 1. Verification Classification Matrix

| Verification Item | Classification | Method | Evidence / Log Reference |
| --- | --- | --- | --- |
| **Brand V4 System Integration** | `AUTOMATED VERIFIED` + `RUNTIME VERIFIED` | Flutter widget tests + iOS Impeller launch | `test/brand_v4_visual_test.dart`, `ios_brand_v4_live.png` |
| **Single Financial Source of Truth** | `AUTOMATED VERIFIED` | Dart FFI + redb assertions | `test/financial_authority_test.dart`, `test/m3a3_functional_hardening_test.dart` |
| **Balance Auto-Refresh on Mint** | `AUTOMATED VERIFIED` | Riverpod balance invalidation test | `unified_deposit_sheet.dart` & `cashu_wallet_provider.dart` |
| **State Reconciliation & Sync Lag** | `AUTOMATED VERIFIED` | Intent sync vs local completed state | `test/m3a3_functional_hardening_test.dart` |
| **Protected Send (Alice &rarr; Bob)** | `AUTOMATED VERIFIED` | Rust CDK + Dart FFI test suite | `crates/hanbova-protected-payments/src/cdk_test.rs` |
| **Recipient Claim Execution** | `AUTOMATED VERIFIED` | NUT-11 witness claim with mint proof settlement | `cdk_test::tests::test_scenario_a_bob_claims_with_p2pk` |
| **Sender Locktime Refund** | `AUTOMATED VERIFIED` | Post-locktime sender refund spend | `cdk_test::tests::test_scenario_b_alice_refunds_after_locktime` |
| **Late Claim Rejection Post-Refund** | `AUTOMATED VERIFIED` | Mint proof state invalidation | `test_mock_refund_before_locktime_fails` |
| **Process Termination & Restart** | `AUTOMATED VERIFIED` | SQLite & redb disk reload | `test/client_wallet_authority_test.dart` |
| **Double-Tap Button Protection** | `AUTOMATED VERIFIED` | Form & button loading state flags | `_RefundActionButton`, `_ClaimActionButton`, `protected_send_screen.dart` |
| **Consumer Error Translation** | `AUTOMATED VERIFIED` | Centralized regex & pattern sanitizer | `test/m3a3_functional_hardening_test.dart` |
| **Recovery Scope Truthfulness** | `AUTOMATED VERIFIED` | Dialog & copy assertion | `test/m3a3_functional_hardening_test.dart` |
| **Public Mainnet Pilot / Live Sats** | `NOT VERIFIED / SAFETY-LOCKED` | Mainnet safety lock disabled by default | `NetworkConfig.mainnet.isEnabled = false` |

---

## 2. Core Operational Scenarios

### Scenario A: Protected Send & Claim Flow (Alice &rarr; Bob)
1. **Funding**: Alice obtains ecash proofs via NUT-04 mint quote in `wallet_cashu_test`.
2. **Locking**: Alice creates a NUT-11 P2PK locked ecash token conditioned on Bob's public key (with Alice's refund key and locktime).
3. **Escrow**: The locked ecash is persisted to Alice's local redb escrow storage.
4. **Transport**: Alice encrypts the envelope using Bob's X25519 transport key; backend relays the ciphertext.
5. **Claim**: Bob receives the message, validates the key fingerprint, decrypts the envelope, and executes CDK `claimProtectedPayment`.
6. **Settlement**: Mint validates Bob's signature, marks input proofs spent, and issues fresh ecash to Bob.
7. **Result**: Bob's spendable balance increments by 100 sats; Alice's escrow is marked claimed.

### Scenario B: Sender Refund Post-Locktime (Alice Refund)
1. **Locktime**: Alice locks 100 sats with a 3600-second locktime.
2. **Wait**: Locktime expires without a claim by Bob.
3. **Refund Action**: Alice taps *"Refund available"* via `_RefundActionButton`.
4. **Spend**: Alice's CDK wallet constructs a refund spending transaction signed with Alice's refund private key.
5. **Settlement**: Mint validates the locktime expiry and Alice's signature, marks input proofs spent, and issues fresh ecash to Alice.
6. **Result**: Alice's spendable balance is restored; Bob's subsequent claim fails closed at the mint.

### Scenario C: App Restart & Database Persistence
1. Alice creates a protected payment.
2. The app process is forcefully terminated (`kill -9`).
3. Upon relaunch, `CdkCashuWalletServiceImpl` opens the existing `{app_support}/wallets/cashu_test/{userId}/wallet.redb`.
4. `CashuWalletStorage` retrieves the existing escrow record from SQLite.
5. Balances, transaction history, and refund capabilities are completely intact.

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
