# Milestone 3B.1: Controlled Mainnet Pilot & Real User Onboarding - Demonstration Checklist

**Target Protocol**: Cashu NUT-04 (Lightning Minting), NUT-10 & NUT-11 (P2PK Escrow & Refunds), X25519 Encrypted Relay  
**Test Environments**:
1. **Cashu Test Environment**: `https://testnut.cashu.space` (Zero monetary value)
2. **Controlled Mainnet Pilot**: `https://mint.minibits.cash/Bitcoin` (Strictly limited: Max 10,000 sats wallet balance / Max 5,000 sats single send)

> [!CAUTION]
> **Safety Notice**: This is a **controlled small-value developer pilot**, NOT a public mainnet beta. Do not fund with real Bitcoin or use real sats until all safety gates and review are complete.

---

## 1. Real First-User Onboarding Flow Verification

| Step | User Action | Verified Runtime Behavior |
| :--- | :--- | :--- |
| **1. Welcome Screen** | Tap "Create Account" | Navigates to `/signup`. Direct "Restore Wallet" option navigates to `/restore-seed`. |
| **2. Registration** | Enter username (e.g. `alice`), email, password | Authenticates with backend API (`POST /api/v1/auth/register`), saves session JWT. |
| **3. Fail-Closed Init** | Automatic identity & wallet setup | Requires valid `AuthUser` (fails closed if unauthenticated). Generates 12-word BIP-39 mnemonic, derives deterministic P2PK (`secp256k1`) and X25519 transport keys, creates isolated Redb storage (`wallet.redb`), and registers public keys on backend directory (`PUT /api/v1/me/payment-keys`). Tracks explicit publication state (`Published` / `Sync Pending` with retry option). |
| **4. Backup Verification** | Review 12 words & complete 3-word verification quiz | Explains seed safety: "Write these words down and keep them private." No clipboard copy. Verifies words #3, #7, #11 before proceeding. |
| **5. Device Security** | Review Security Hardware | Displays truthful status: "Biometric Hardware Security: Planned • Milestone 4" without false active gating claims. |
| **6. Mint Setup** | Probe Active Mint | Probes mint `/v1/info` capabilities (reachable, sat unit, NUT-04, NUT-07, NUT-10, NUT-11). Displays mint description/motd. Disables proceed if probe fails. |
| **7. Add Bitcoin** | Tap "Deposit Bitcoin" or "Go to Home" | Initial wallet balance starts at **0 sats** (zero synthetic balances or demo transactions). |

---

## 2. Genuine NUT-04 Lightning Deposit & Status Polling

| Step | Action | Runtime Evidence |
| :--- | :--- | :--- |
| **1. Request Mint Quote** | Enter amount (e.g. `1,000 sats`) in `UnifiedDepositSheet` | Enforces max wallet balance cap (`balance + amount <= 10,000 sats`). CDK calls mint `/v1/mint/quote/bolt11`, returns authentic quote ID and Lightning BOLT11 invoice. |
| **2. Display Invoice** | View QR code & copy invoice string | Real QR code rendered (`lnbc...`). Auto-polling timer starts (every 2.5s) calling `checkMintQuoteStatus(quoteId)`. |
| **3. Polling State** | Mint checks quote state without premature minting | Transitions through `UNPAID`. Polling does NOT call `wallet.mint()`. |
| **4. External Lightning Payment** | Pay invoice using external wallet (Phoenix, Strike, Cash App, Alby, or test mint hook) | Mint transitions quote state from `UNPAID` to `PAID`. |
| **5. Single Mint Settlement** | CDK mints proofs upon `PAID` confirmation | Once `checkMintQuoteStatus` returns `isPaid = true`, auto-polling cancels and CDK executes `wallet.mintQuote(quote_id)` **exactly once** under concurrency guard. Proofs stored in `wallet.redb`. |
| **6. Balance Update** | Balance card refreshes | Spendable balance updates in `cashuBalanceProvider`. |

---

## 3. Real Protected Send & Claim Verification (Alice &rarr; Bob)

### Prerequisites:
- **Alice**: Funded wallet with &ge; 100 sats.
- **Bob**: Registered recipient account (`@bob`) with published P2PK and X25519 public keys.

| Sequence | Actor | Action & Verification |
| :--- | :--- | :--- |
| **1. Create Protected Send** | **Alice** | Enters `@bob`, `100 sats`, memo, and timelock. Provider strictly enforces `amountSats <= 5,000 sats` (pilot send cap). |
| **2. Local Escrow Lock** | **Alice App** | CDK executes `createProtectedSend`, locking ecash proofs under `P2PK(BobPub, locktime, AliceRefundPub, SigFlag::SigInputs)`. |
| **3. Encrypted Relay** | **Alice App** | Cashu token is encrypted with Bob's X25519 public key using ChaCha20-Poly1305 and relayed to backend. |
| **4. Incoming Discovery** | **Bob** | Bob opens Protected tab &rarr; Incoming. Fetches encrypted message from backend. |
| **5. Decrypt & Witness Sign** | **Bob App** | Decrypts Cashu token locally using private X25519 key; signs NUT-11 spending witness using private Secp256k1 P2PK key. |
| **6. Mint Settlement** | **Bob App** | CDK calls mint to receive proofs. Mint verifies P2PK witness signature and issues new proofs to Bob. |
| **7. Balance Reconciliation** | **Bob & Alice** | Bob spendable balance increases (+99 sats after swap fee). Alice transaction transitions to `Claimed`. |

---

## 4. Real Refund Verification (Locktime Expiration Scenario)

### Accurate NUT-11 Spending Semantics:
- **Before locktime**: Recipient spending path is valid.
- **After locktime**: Recipient spending path remains valid; sender refund path ALSO becomes valid.
- **Settlement Rule**: **First valid mint spend wins.** There is never an "automatic refund".

| Sequence | Actor | Action & Verification |
| :--- | :--- | :--- |
| **1. Short Locktime Send** | **Alice** | Alice sends `100 sats` to `@bob` with **30-second** dev locktime. |
| **2. Unclaimed Wait** | **Alice & Bob** | Bob does not claim the payment during the protection window. |
| **3. Expiration Trigger** | **Alice App** | After 30 seconds, UI badge updates to **"Locktime expired • Refund available"**. |
| **4. Self-Service Refund** | **Alice** | Alice taps **Refund**. CDK signs with `AliceRefundPriv` and submits proofs to mint for refund swap. |
| **5. Balance Restored** | **Alice App** | Alice balance is restored to spendable redb proofs. Backend status transitions to `refunded`. |
| **6. Bob Late Claim Rejection** | **Bob** | Bob attempts late claim &rarr; Mint rejects spent proofs (double-spend protection). |

---

## 5. Controlled Mainnet Pilot Constraints & Boundaries

| Parameter | Controlled Pilot Rule | Enforcement Location |
| :--- | :--- | :--- |
| **Activation Flag** | `--dart-define=MAINNET_DEMO_PILOT=true` or Developer Options opt-in | `NetworkConfig.isMainnetPilotBuild` / `mainnetPilotOverrideProvider` |
| **Active Config Provider** | `activeNetworkConfigProvider` centralizes pilot state | `network_environment.dart` |
| **Allowlisted Mint** | `https://mint.minibits.cash/Bitcoin` | `cashuWalletServiceProvider` (strictly forces allowlist mint, ignores custom selected mints) |
| **Mint Screen Restrictions** | Adding custom mints & switching mints disabled | `MintsScreen` |
| **Max Wallet Balance Cap** | `10,000 sats` | `UnifiedDepositSheet` (`balance + amount > config.maxWalletBalanceSats`) |
| **Max Single Send Cap** | `5,000 sats` | `ProtectedSendNotifier` & `SendScreen` (NUT-05 melt) |
| **Token Import Gate** | Arbitrary Cashu token import disabled in pilot | `UnifiedDepositSheet._claimCashuToken` |
| **Safety Warning** | Prominent Warning Badge on Home & Protected Screens | `HomeScreen`, `ProtectedScreen`, `MainnetSafetyDialog` |
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
