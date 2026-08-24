# Milestone 3A.2.1: Mobile Integration & Safety Stabilization - Runtime Evidence

**Date of Execution**: August 24, 2026  
**Test Suite Reference**: `crates/hanbova-protected-payments/src/cdk_test.rs`  
**Test Framework**: Rust CDK Integration Test Suite (`cargo test --package hanbova-protected-payments`)  
**Mint Environment**: Controlled local Cashu mint integration using test/mock Lightning settlement  
**CDK Version**: `cdk 0.18.0-rc.0`  
**Client Cryptographic Transport**: `X25519 + ChaCha20-Poly1305 AEAD`  

---

## 1. NUT-04 Mint Funding Evidence

> **Environment Note**: All mint funding tests are executed against a controlled local Cashu mint integration using test/mock Lightning settlement. This is not an external or production Lightning funding test.

| Parameter | Type | Value / Observation |
| :--- | :--- | :--- |
| **Test Mint URL** | Configuration | `http://127.0.0.1:3338` (`HanbovaNetwork.cashuTestMintUrl`) |
| **Payment Method** | Automated Assertion | `bolt11` (`PaymentMethod::from_str("bolt11")`) |
| **Requested Amount** | Automated Assertion | `1,000 sats` (`Amount::from(1000u64)`) |
| **Quote ID** | Runtime Observation | Generated dynamically by mint (Format: `[EXAMPLE: 64d8a9f0e1c2b3a4]`) |
| **Lightning BOLT11 Invoice** | Runtime Observation | Generated dynamically by test mint backend (Format: `[EXAMPLE: lnbc10u1p3...]`) |
| **Quote State Before Settlement** | Automated Assertion | `UNPAID` (`MintQuoteState::Unpaid`) |
| **Settlement Trigger** | Test Action | Invoice settled via test mint mock Lightning settlement hook |
| **CDK Mint Invocation** | Automated Assertion | `alice_wallet.mint(&quote.id, SplitTarget::default(), None).await` &rarr; `Ok()` |
| **Alice Balance After Funding** | Automated Assertion | `assert_eq!(alice_bal_after_fund, Amount::from(1000u64))` |
| **Proof Authenticity** | Runtime Verification | 100% Genuine CDK blinded signatures ($C$). Zero mocked proofs or synthetic balances. |

---

## 2. Scenario A: Alice &rarr; Bob Protected Send & Claim Evidence

| Step / Metric | Type | Value / Runtime State |
| :--- | :--- | :--- |
| **Alice Initial Balance** | Automated Assertion | `1,000 sats` (spendable in `alice_db`) |
| **Payment Amount** | Automated Assertion | `100 sats` (`Amount::from(100u64)`) |
| **Locktime** | Automated Assertion | `unix_time() + 60` (60-second timelock) |
| **NUT-11 Spending Condition** | Protocol Verification | `P2PK(BobPub, locktime, AliceRefundPub, SigFlag::SigInputs)` |
| **Alice Balance After Send** | Automated Assertion | `assert_eq!(alice_bal_after_send, Amount::from(899u64))` *(100 sats sent + 1 sat mint split fee)* |
| **Bob Initial Balance** | Automated Assertion | `assert_eq!(bob_bal_before, Amount::ZERO)` |
| **Third-Party Rejection (Charlie)** | Automated Assertion | `assert!(charlie_claim.is_err())` *(Charlie signature rejected by mint)* |
| **Early Refund Rejection (Alice)** | Automated Assertion | `assert!(early_refund.is_err())` *(Refund before locktime rejected by mint)* |
| **Bob Claim Invocation** | Automated Assertion | `bob_wallet.receive(&token_str, ReceiveOptions { p2pk_signing_keys: vec![bob_sec] }).await` |
| **Bob Received Amount** | Automated Assertion | `assert_eq!(bob_received, Amount::from(99u64))` *(100 sats token - 1 sat swap fee)* |
| **Bob Balance After Claim** | Automated Assertion | `assert_eq!(bob_bal_after, Amount::from(99u64))` |
| **Alice Late-Refund Attempt** | Automated Assertion | `assert!(late_refund.is_err())` *(Rejected: proofs already spent by Bob)* |

---

## 3. Scenario B: Post-Locktime Refund & Late Claim Rejection Evidence

| Step / Metric | Type | Value / Runtime State |
| :--- | :--- | :--- |
| **Alice Starting Balance** | Automated Assertion | `1,000 sats` |
| **Second Payment Amount** | Automated Assertion | `100 sats` |
| **Locktime** | Automated Assertion | `unix_time() + 2` (2-second test locktime) |
| **Alice Balance After Send** | Automated Assertion | `assert_eq!(alice_bal_sent, Amount::from(899u64))` |
| **Locktime Expiration** | Runtime Action | `tokio::time::sleep(Duration::from_secs(3)).await` (`now > locktime`) |
| **Alice Refund Trigger** | Automated Assertion | `alice_wallet.receive(&token_str, ReceiveOptions { p2pk_signing_keys: vec![alice_refund_sec] }).await` |
| **Alice Refund Received** | Automated Assertion | `assert_eq!(refund_received, Amount::from(99u64))` *(100 sats token - 1 sat swap fee)* |
| **Alice Balance After Refund** | Automated Assertion | `assert_eq!(alice_bal_refunded, Amount::from(998u64))` *(899 + 99 sats)* |
| **Bob Late-Claim Attempt** | Automated Assertion | Bob attempts claim with valid `bob_sec` after refund |
| **Bob Late-Claim Result** | Automated Assertion | `assert!(bob_claim.is_err())` *(Mint double-spend protection rejects spent proofs)* |

---

## 4. Payment Status Authority & State Reconcilation

| Layer | Role & Authority |
| :--- | :--- |
| **Backend State (`claimed` / `refunded`)** | **Coordination Metadata Only**. Provides UI notification triggers and encrypted relay status. Backend status is non-authoritative. |
| **Client ACK** | **Transport Confirmation Only**. Does NOT constitute cryptographic proof of settlement. |
| **Cashu Mint Proof State (NUT-07)** | **Cryptographically Authoritative**. Proof spending status is validated on-mint via `hanbova_cdk_check_token_state`. If mint reports proofs spent, claim/refund is final. |

---

## 5. Recovery Ground Truth Audit

| Identity / Component | Storage Mechanism | Recovery Status on Fresh Installation |
| :--- | :--- | :--- |
| **BIP-39 Mnemonic** | `FlutterSecureStorage` (`hanbova_${prefix}_${userId}_mnemonic`) | Restorable via user entry of 12-word phrase. |
| **CDK Wallet Master Seed** | Derived via PBKDF2 (`mnemonicToSeedHex`) | Restored from mnemonic; generates master ecash seed. |
| **Secp256k1 P2PK Private Key** | `FlutterSecureStorage` (`hanbova_${prefix}_${userId}_protected_priv`) | **NOT derived from mnemonic**; stored as independent key. Loss of secure storage wipes key unless exported. |
| **X25519 Transport Key** | `FlutterSecureStorage` (`hanbova_${prefix}_${userId}_transport_priv`) | **NOT derived from mnemonic**; stored as independent key. |
| **Local Proof Persistence** | Embedded Redb (`wallet.redb`) | Persists across app restarts. Wiped on fresh install. |
| **Off-Device Proof Recovery (Wipe)** | NUT-13 Mint Restoration | **NOT VERIFIED / NOT IMPLEMENTED IN CURRENT BUILD**. |
| **Fresh Install Recovery Status** | Safe UI Gate | **Recovery disabled in current test build** (`Recovery is not available in this test build yet.`). |

---

## 6. Mobile Platform Runtime Verification

| Platform Target | Integration Status | Verification Evidence |
| :--- | :--- | :--- |
| **Android ARM64 (`arm64-v8a`)** | **VERIFIED** | `libhanbova_cdk_ffi.so` (15.9 MB) packaged in `jniLibs/arm64-v8a/` and dynamically loaded via `DynamicLibrary.open('libhanbova_cdk_ffi.so')`. |
| **Android x86_64 (`x86_64`)** | **VERIFIED** | `libhanbova_cdk_ffi.so` (16.3 MB) packaged in `jniLibs/x86_64/` and dynamically loaded. |
| **iOS Simulator (`arm64` + `x86_64`)** | **VERIFIED** | Universal static framework `HanbovaCdkFfi.xcframework` linked into Runner. `nm -gU Runner.debug.dylib` confirms all 10 `hanbova_cdk_*` symbols resolved for `DynamicLibrary.process()`. |
| **iOS Physical Device (`aarch64-apple-ios`)** | **NOT VERIFIED** | Compiled static library (`libhanbova_cdk_ffi.a`, 62 MB) packaged into `HanbovaCdkFfi.xcframework` slice `ios-arm64`. Physical device runtime is **NOT VERIFIED** (no physical iPhone connected to test harness). |
