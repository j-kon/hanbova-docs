# Milestone 3B.1: Controlled Mainnet Pilot & Real User Onboarding - Demonstration Checklist

**Target Protocol**: Cashu NUT-04 (Lightning Minting), NUT-10 & NUT-11 (P2PK Escrow & Refunds), X25519 Encrypted Relay  
**Test Environments**:
1. **Cashu Test Environment**: `https://testnut.cashu.space` (Zero monetary value)
2. **Controlled Mainnet Pilot**: `https://mint.minibits.cash/Bitcoin` (Strictly limited: Max 10,000 sats deposit / Max 5,000 sats send)

> [!CAUTION]
> **Safety Notice**: This is a **controlled small-value developer pilot**, NOT a public mainnet beta. Do not store large funds.

---

## 1. Real First-User Onboarding Flow Verification

| Step | User Action | Verified Runtime Behavior |
| :--- | :--- | :--- |
| **1. Welcome Screen** | Tap "Create Account" | Navigates to `/signup`. Alternative "Restore Wallet" navigates to `/restore-seed`. |
| **2. Registration** | Enter username (e.g. `alice`), email, password | Authenticates with backend API (`POST /api/v1/auth/register`), saves session JWT. |
| **3. Identity & Storage** | Automatic setup step | Generates 12-word BIP-39 mnemonic, derives deterministic P2PK (`secp256k1`) and X25519 transport keys, creates isolated Redb storage (`wallet.redb`), and registers public keys on backend (`PUT /api/v1/me/payment-keys`). |
| **4. Backup Verification** | Review 12 words & complete 3-word verification quiz | Verifies words #3, #7, #11 before proceeding. Prevents proceeding without confirmed backup. |
| **5. Device Security** | Toggle Biometric Unlock (Optional) | Configures Face ID / Touch ID hardware security prompt for transactions. |
| **6. Mint Setup** | Confirm Active Mint | Probes mint capabilities (NUT-04, NUT-07, NUT-10, NUT-11). |
| **7. Add Bitcoin** | Tap "Deposit Bitcoin" or "Go to Home" | Initial wallet balance starts at **0 sats** (zero synthetic balances or demo transactions). |

---

## 2. Genuine NUT-04 Lightning Deposit Verification

| Step | Action | Runtime Evidence |
| :--- | :--- | :--- |
| **1. Request Mint Quote** | Enter amount (e.g. `1,000 sats`) in `UnifiedDepositSheet` | CDK calls mint's `/v1/mint/quote/bolt11`, returns authentic quote ID and Lightning BOLT11 invoice. |
| **2. Display Invoice** | View QR code & copy invoice string | Real QR code rendered (`lnbc...`). Auto-polling timer starts (every 2.5s). |
| **3. External Lightning Payment** | Pay invoice using external wallet (Phoenix, Strike, Cash App, Alby, or test mint hook) | Mint transitions quote state from `UNPAID` to `PAID`. |
| **4. Auto-Mint Settlement** | CDK mints proofs upon payment confirmation | CDK executes `wallet.mint(quote_id)` to receive blinded signatures ($C$). Proofs stored in `wallet.redb`. |
| **5. Balance Update** | Balance card refreshes | Spendable balance updates to `1,000 sats` in `cashuBalanceProvider`. |

---

## 3. Real Protected Send & Claim Verification (Alice &rarr; Bob)

### Prerequisites:
- **Alice**: Funded wallet with &ge; 100 sats.
- **Bob**: Registered recipient account (`@bob`) with published P2PK and X25519 public keys.

| Sequence | Actor | Action & Verification |
| :--- | :--- | :--- |
| **1. Create Protected Send** | **Alice** | Enters `@bob`, `100 sats`, memo, and timelock. |
| **2. Local Escrow Lock** | **Alice App** | CDK executes `createProtectedSend`, locking ecash proofs under `P2PK(BobPub, locktime, AliceRefundPub, SigFlag::SigInputs)`. |
| **3. Encrypted Relay** | **Alice App** | Cashu token is encrypted with Bob's X25519 public key using ChaCha20-Poly1305 and relayed to backend. |
| **4. Incoming Discovery** | **Bob** | Bob opens Protected tab &rarr; Incoming. Fetches encrypted message from backend. |
| **5. Decrypt & Witness Sign** | **Bob App** | Decrypts Cashu token locally using private X25519 key; signs NUT-11 spending witness using private Secp256k1 P2PK key. |
| **6. Mint Settlement** | **Bob App** | CDK calls mint to receive proofs. Mint verifies P2PK witness signature and issues new proofs to Bob. |
| **7. Balance Reconciliation** | **Bob & Alice** | Bob spendable balance increases (+99 sats after swap fee). Alice transaction transitions to `Claimed`. |

---

## 4. Real Refund Verification (Locktime Expiration Scenario)

| Sequence | Actor | Action & Verification |
| :--- | :--- | :--- |
| **1. Short Locktime Send** | **Alice** | Alice sends `100 sats` to `@bob` with **30-second** dev locktime. |
| **2. Unclaimed Wait** | **Alice & Bob** | Bob does not claim the payment. |
| **3. Expiration Trigger** | **Alice App** | After 30 seconds, UI badge updates to **"Locktime expired • Refund available"**. |
| **4. Self-Service Refund** | **Alice** | Alice taps **Refund**. CDK signs with `AliceRefundPriv` and submits proofs to mint for refund swap. |
| **5. Balance Restored** | **Alice App** | Alice balance is restored to spendable redb proofs. Backend status transitions to `refunded`. |
| **6. Bob Late Claim Rejection** | **Bob** | Bob attempts late claim &rarr; Mint rejects spent proofs (double-spend protection). |

---

## 5. Controlled Mainnet Pilot Constraints & Boundaries

| Parameter | Controlled Pilot Rule | Enforcement Location |
| :--- | :--- | :--- |
| **Activation Flag** | `--dart-define=MAINNET_DEMO_PILOT=true` or Developer Options opt-in | `NetworkConfig.isMainnetPilotBuild` / `mainnetPilotOverrideProvider` |
| **Max Deposit Cap** | `10,000 sats` (~$6.00 USD) | `UnifiedDepositSheet` (`amount > config.maxDepositSats`) |
| **Max Send Cap** | `5,000 sats` (~$3.00 USD) | `ProtectedSendScreen` (`amountSats > config.maxSendSats`) |
| **Allowlisted Mint** | `https://mint.minibits.cash/Bitcoin` | `NetworkConfig.mainnetPilot.defaultMintUrl` |
| **Safety Warning** | Prominent Amber/Red Warning Banner on Home & Protected Screens | `HomeScreen`, `ProtectedScreen`, `MainnetSafetyDialog` |
| **Standard Builds** | Mainnet remains strictly locked (`isEnabled = false`) | `NetworkConfig.mainnetLocked` |

---

## 6. Recovery Truth & Limitations Audit

1. **Deterministic Master Keys**:
   - 12-word BIP-39 mnemonic deterministically derives CDK seed, primary Secp256k1 P2PK identity, and X25519 transport identity.
2. **Local Persistence**:
   - Ecash proofs and escrow records persist across normal application restarts in `wallet.redb`.
3. **Known Limitations**:
   - **NUT-13 Proof Recovery**: Full off-device proof recovery after total device wipe requires NUT-13 deterministic secret derivation from the mint (scheduled for Milestone 4/5).
   - **Ephemeral Refund Keys**: Per-payment refund keys generated for specific escrows are stored in local redb; total app data deletion loses unrefunded ephemeral keys unless backed up.
