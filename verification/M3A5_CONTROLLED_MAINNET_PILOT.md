# Milestone 3A.5: Controlled Mainnet Pilot Verification Evidence

**Milestone**: 3A.5 Controlled Mainnet Pilot (First Real Sats)  
**Status**: `SUSPENDED AT FUNDING GATE` ⏸️  
**Date**: 2026-09-02  
**Branches**:  
- `hanbova-app`: `milestone/3a5-controlled-mainnet-pilot`  
- `hanbova-docs`: `milestone/3a5-controlled-mainnet-pilot`  
- `hanbova-backend`: `main`  
**Pilot Environment**:
- Storage Namespace: `wallet_mainnet_pilot`
- Mint Endpoint: `https://mint.minibits.cash/Bitcoin` (`cdk-mintd/0.17.6`)
- Mint Pubkey: `03c21ef6f899f661356103552c1945579b0f36f96f9f7d2190ca0cbdab3add4af9`
- Total Pilot Cap: 1,000 sats (Pilot run target: 250 sats)
- Standard Public Release: `DISABLED` (Mainnet locked on default builds)

---

## 1. Executive Summary & Verification Matrix

The Controlled Mainnet Pilot is a strictly gated, non-public verification milestone designed to execute genuine Bitcoin sat transfers over the Cashu ecash protocol via Minibits mint (`https://mint.minibits.cash/Bitcoin`).

Execution has been cleanly paused and suspended at the external Lightning funding gate prior to sat settlement.

| Item / Requirement | Verification Tier | Evidence & Method | Status |
| :--- | :--- | :--- | :--- |
| **1. Controlled Mainnet Pilot Build** | `COMPILE DEFINED` | `flutter build apk --debug --dart-define=MAINNET_DEMO_PILOT=true` and iOS simulator build enable pilot configuration with prominent UI warning. | **PASS** ✅ |
| **2. Default Build Mainnet Lock** | `REGRESSION VERIFIED` | Standard builds without define flags enforce `NetworkConfig.mainnet.isEnabled = false`, `maxWalletBalanceSats = 0`, defaulting to `local` network. | **PASS** ✅ |
| **3. Storage Isolation** | `STORAGE AUDITED` | Redb isolated at `hanbova_cdk_${userId}_wallet_mainnet_pilot`; FlutterSecureStorage isolated with `wallet_mainnet_pilot` prefix. Zero test balance bleed. | **PASS** ✅ |
| **4. Minibits Connectivity** | `API VERIFIED` | `GET /v1/info` queried live: `cdk-mintd/0.17.6`, unit `sat`, mint pubkey confirmed reachable. | **PASS** ✅ |
| **5. NUT-04 Support** | `SPEC CONFIRMED` | NUT-04 (bolt11 mint quotes) active on Minibits mint. Quote creation verified with live BOLT11 generation. | **CONFIRMED** ✅ |
| **6. NUT-11 Support** | `SPEC CONFIRMED` | NUT-11 (P2PK spending conditions) confirmed supported on Minibits mint info. | **CONFIRMED** ✅ |
| **7. Real Mainnet Funding** | `FUNDING GATE` | Suspended prior to external Lightning payment. Current quote permitted to expire naturally. | **PENDING** ⏸️ |
| **8. NUT-04 Real Mint** | `CDK RUNTIME` | Not executed (awaiting real sat funding). | **NOT EXECUTED** ⏸️ |
| **9. Mainnet Claim 100 Sats** | `CDK RUNTIME` | Not executed (awaiting real sat funding). | **NOT EXECUTED** ⏸️ |
| **10. Mainnet Refund 100 Sats** | `CDK RUNTIME` | Not executed (awaiting real sat funding). | **NOT EXECUTED** ⏸️ |
| **11. Late Claim Rejection** | `CDK RUNTIME` | Not executed (awaiting real sat funding). | **NOT EXECUTED** ⏸️ |

---

## 2. Suspension Details & Pilot Resumption Protocol

- **Reason**: Controlled pilot paused before external Lightning funding. No real sats were spent or lost.
- **Invoice State**: The initial test quote (`01a0638e-6c8a-7271-b171-359b24ae4cd0`) will expire naturally. It must **not** be reused when the pilot resumes.
- **Future Resumption Protocol**:
  1. Boot backend, Android pilot app, and iOS pilot app.
  2. Generate exactly one fresh 250-sat Minibits NUT-04 quote.
  3. Settle invoice via external Lightning wallet.
  4. Complete Scenario A (100-sat Protected Claim) and Scenario B (100-sat Refund & Late Claim Rejection).

---

## 3. Storage Prefix Isolation Proof

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

---

## 4. Omissions & Public Release Declaration

- **NUT-05 (Melt / Lightning Send)**: `NOT TESTED`
- **NUT-13 (Mnemonic Recovery)**: `NOT COMPLETE`
- **MAINNET PUBLIC RELEASE**: `DISABLED`

> **CRITICAL INVARIANT**: Mainnet is strictly locked on all default builds (`NetworkConfig.mainnet.isEnabled = false`, `maxWalletBalanceSats = 0`). Access to the Controlled Pilot requires the explicit compile-time flag `MAINNET_DEMO_PILOT=true`.
