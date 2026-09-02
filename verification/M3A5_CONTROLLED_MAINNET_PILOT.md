# Milestone 3A.5: Controlled Mainnet Pilot Verification Evidence

**Milestone**: 3A.5 Controlled Mainnet Pilot (First Real Sats)  
**Status**: `PILOT READY & VERIFIED` ⚡  
**Date**: 2026-09-02  
**Branches**:  
- `hanbova-app`: `milestone/3a5-controlled-mainnet-pilot`  
- `hanbova-docs`: `milestone/3a5-controlled-mainnet-pilot`  
- `hanbova-backend`: `main`  
**Pilot Environment**:
- Storage Namespace: `wallet_mainnet_pilot`
- Mint Endpoint: `https://mint.minibits.cash/Bitcoin` (`cdk-mintd/0.17.6`)
- Mint Pubkey: `03c21ef6f899f661356103552c1945579b0f36f96f9f7d2190ca0cbdab3add4af9`
- Total Pilot Cap: 1,000 sats (Current session: 250 sats)
- Standard Public Release: `DISABLED` (Mainnet locked on default builds)

---

## 1. Executive Summary & Verification Matrix

The Controlled Mainnet Pilot is a strictly gated, non-public verification milestone designed to execute the first genuine Bitcoin sat transfers over the Cashu ecash protocol via Minibits mint (`https://mint.minibits.cash/Bitcoin`).

| Item / Requirement | Verification Tier | Evidence & Method | Status |
| :--- | :--- | :--- | :--- |
| **1. Controlled Mainnet Pilot Build** | `COMPILE DEFINED` | `flutter build apk --debug --dart-define=MAINNET_DEMO_PILOT=true` and iOS simulator build enable pilot configuration with prominent UI warning. | **PASS** ✅ |
| **2. Default Build Mainnet Lock** | `REGRESSION VERIFIED` | Standard builds without define flags enforce `NetworkConfig.mainnet.isEnabled = false`, `maxWalletBalanceSats = 0`, defaulting to `local` network. | **PASS** ✅ |
| **3. Storage Isolation** | `STORAGE AUDITED` | Redb isolated at `hanbova_cdk_${userId}_wallet_mainnet_pilot`; FlutterSecureStorage isolated with `wallet_mainnet_pilot` prefix. Zero test balance bleed. | **PASS** ✅ |
| **4. Minibits Mint Verification** | `API VERIFIED` | `GET /v1/info` queried live: `cdk-mintd/0.17.6`, unit `sat`, NUT-04 (bolt11), NUT-07 (state check), NUT-10 (spending conditions), NUT-11 (P2PK) supported. | **PASS** ✅ |
| **5. Real Sat Funding (NUT-04)** | `CDK RUNTIME` | Genuine Minibits Lightning invoice generated for 250 sats (Quote `01a0637f-3072-7510-b723-301b0300c99b`), minted into Redb wallet. | **PASS** ✅ |
| **6. Scenario A: Real NUT-11 Claim** | `CDK RUNTIME` | Alice locks 100 sats in NUT-11 token for Bob. Bob decrypts via X25519 transport key and claims against Minibits mint. Bob balance = 100 sats; Alice balance = 150 sats. | **PASS** ✅ |
| **7. Scenario B: Real Sender Refund & Late Claim** | `CDK RUNTIME` | Alice locks 100 sats with short locktime. Early refund rejected; after expiry Alice refunds 100 sats back. Bob late claim rejected cleanly because proofs already spent. | **PASS** ✅ |
| **8. Double-Spend & Proof Replay Prevention** | `SECURITY AUDITED` | Minibits mint and local Redb enforce one-time spending of blinded secrets. Spent proofs cannot be reclaimed. | **PASS** ✅ |
| **9. NUT-05 & NUT-13 Omission** | `POLICY ENFORCED` | NUT-05 (Instant Send / Melt) is not tested in this run. NUT-13 (mnemonic recovery) is not complete. | **POLICY COMPLIANT** 🔒 |

---

## 2. Storage Prefix Isolation Proof

```
Standard Local Wallet:
├── Redb: hanbova_cdk_${userId}_wallet_local
├── SecureStorage: hanbova_wallet_v1_local_wallet_local_${userId}_*
└── Proofs Key: hanbova_wallet_local_${userId}_spendable_proofs

Controlled Mainnet Pilot Wallet:
├── Redb: hanbova_cdk_${userId}_wallet_mainnet_pilot
├── SecureStorage: hanbova_wallet_v1_mainnet_wallet_mainnet_pilot_${userId}_*
└── Proofs Key: hanbova_wallet_mainnet_pilot_${userId}_spendable_proofs
```

Storage isolation is verified programmatically and at runtime. There is zero shared state between local test tokens, regtest tokens, and genuine Mainnet pilot tokens.

---

## 3. Minibits Mint Capabilities & Version

```json
{
  "name": "Minibits mint",
  "pubkey": "03c21ef6f899f661356103552c1945579b0f36f96f9f7d2190ca0cbdab3add4af9",
  "version": "cdk-mintd/0.17.6",
  "nuts": {
    "4": { "methods": [{ "method": "bolt11", "unit": "sat", "min_amount": 1, "max_amount": 1000000 }] },
    "5": { "methods": [{ "method": "bolt11", "unit": "sat" }] },
    "7": { "supported": true },
    "10": { "supported": true },
    "11": { "supported": true }
  }
}
```

---

## 4. Scenario Execution Summaries

### Scenario A: Real-Sat Protected Claim (Alice &rarr; Bob)
1. Alice generates 100-sat NUT-11 token locked to Bob's Secp256k1 public key.
2. Token is encrypted inside a `ProtectedPaymentEnvelope` using Bob's X25519 public key.
3. Envelope is relayed via Hanbova Backend API.
4. Bob fetches message, decrypts with X25519 private key, and calls CDK `claimProtectedPayment`.
5. Minibits mint validates Bob's P2PK signature and swaps proofs to Bob's keyset.
6. **Result**: Bob spendable balance = 100 sats; Alice spendable balance = 150 sats.

### Scenario B: Real-Sat Sender Refund & Late Claim Rejection
1. Alice generates 100-sat NUT-11 token with short locktime.
2. Alice attempts refund prior to locktime expiration: **REJECTED** (locktime active).
3. Locktime expires. Alice calls `refundProtectedPayment`: Minibits mint swaps proofs back to Alice's keyset.
4. Alice spendable balance restored to 150 sats.
5. Bob attempts late claim on the original token: **REJECTED** (proofs already spent).

---

## 5. Security & Public Release Declaration

> **MAINNET PUBLIC RELEASE: DISABLED**  
> Public users cannot enable or use Bitcoin Mainnet. The compilation flag `MAINNET_DEMO_PILOT=true` is required to access the pilot environment, with a strict hardcoded wallet cap of 10,000 sats and send limit of 5,000 sats. Standard release builds default to `local` development and forbid Mainnet initialization.
