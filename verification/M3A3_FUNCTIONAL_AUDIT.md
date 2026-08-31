# Milestone 3A.3: Comprehensive Functional Audit

**Milestone**: 3A.3 Functional Hardening & Daily-Use Reliability  
**Date**: 2026-08-31  
**Scope**: Full End-to-End User Journey Audit across `hanbova-app`, `hanbova-backend`, and protocol contracts.  
**Core Authority Rule**:  
- **CLIENT OWNS WALLET AUTHORITY** (BIP-39 mnemonic, Secp256k1 P2PK keys, X25519 transport keys, local Cashu CDK wallet instance with redb persistence).  
- **BACKEND COORDINATES** (Encrypted envelope relay, payment intent coordination metadata, user directory lookup).  
- **CASHU MINT ENFORCES FINANCIAL SPENDING STATE** (Authoritative proof state via NUT-07, quote validation via NUT-04).

---

## User Journey Audit Summary

| Step | User Action / Flow | Classification | Current State & Findings | Hardening Required in M3A3 |
| --- | --- | --- | --- | --- |
| 1 | **App Launch & Splash** | `WORKING` | Brand V4 icon, dark/light theme detection, authenticates session routing to `/home` or `/welcome`. | None. Fully verified. |
| 2 | **Welcome Onboarding** | `WORKING` | 3 production photographic slides, Brand V4 typography (*"Send protected."*), action buttons navigate to `/signup`, `/login`, `/restore-seed`. | None. Fully verified. |
| 3 | **Registration** | `WORKING` | Form validation, creates user record, issues JWT, routes to `/wallet-setup`. | Error messages sanitized to consumer-friendly format. |
| 4 | **Login** | `WORKING` | Accepts `@username` or registered email address, issues JWT, navigates to target path. | Ensure complete in-memory identity invalidation on switch. |
| 5 | **Wallet Setup & Mnemonic** | `WORKING` | Generates BIP-39 mnemonic, derives deterministic P2PK/X25519 keys, initializes isolated redb database. | Fail closed if CDK FFI unavailable. |
| 6 | **Key Publication & Verification** | `PARTIAL` | Publishes public keys to backend for active environment. | If publication fails due to network, present clear retry/sync pending state without blocking offline wallet access. |
| 7 | **Cashu Wallet Initialization** | `WORKING` | Initializes `CashuWalletService` via native C-FFI bridge with per-user, per-environment isolated database. | Verified across macOS, iOS Simulator, Android. |
| 8 | **Home Dashboard & Balance** | `WORKING` | Displays spendable, locked protected balance, and total sats in fiat (NGN/KES/GHS/ZAR/USD). | Ensure instant automatic refresh after mint, claim, and refund events. |
| 9 | **Add Bitcoin (NUT-04)** | `PARTIAL` | Creates mint quote, displays Lightning invoice/QR, polls quote, mints ecash on payment. | Add double-tap protection on mint button and idempotent quote claim handling. |
| 10 | **Protected Send** | `PARTIAL` | Looks up recipient, validates environment keys, creates P2PK-locked token via CDK, persists escrow, encrypts envelope, relays via backend. | If relay drops post-lock, preserve escrow, record `deliveryPending: true`, and provide deduplicated retry. |
| 11 | **Recipient Inbox & Claim** | `PARTIAL` | Retrieves message, verifies recipient fingerprint, decrypts envelope, executes CDK `receive_token`, updates backend. | If backend coordination fails post-claim, keep local financial state claimed and mark `coordinationSyncPending`. |
| 12 | **Sender Refund Path** | `WORKING` | Strictly post-locktime. Uses sender refund key to spend escrowed ecash back to spendable balance via CDK. | Prevent duplicate spend attempts and handle race against late recipient claims. |
| 13 | **Activity Feed** | `WORKING` | Real wallet transactions (Instant, Protected Send, Claim, Refund, Mint). Filter tabs work cleanly. | Distinguish financial state from coordination status. |
| 14 | **Transaction Details** | `PARTIAL` | Shows receipt breakdown and status. | Explicitly separate Financial State (Claimed/Refunded/Escrowed), Delivery State (Delivered/Pending), and Coordination State (Synced/Pending). |
| 15 | **App Restart & Persistence** | `WORKING` | Balances, escrows, transaction records, and cryptographic identities persist in local databases. | Re-verify in real two-app restart test. |
| 16 | **Logout / Account Switching** | `PARTIAL` | Logging out clears auth state. | Harden tear-down of Riverpod in-memory state and wallet context to ensure User B never accesses User A's data. |
| 17 | **Restore Wallet Flow** | `MISLEADING` | Mnemonic restores deterministic P2PK/X25519 keys, but UI copy must not promise full proof recovery from a clean install without local redb or NUT-13. | Update Restore Wallet copy to state exact recovery scope truthfully. |
| 18 | **Error Presentation** | `PARTIAL` | Raw `Bad state:` prefixes stripped in previous step, but remaining Dart/FFI errors need clean consumer translation. | Implement centralized error translator with zero internal leaks. |
| 19 | **Double-Tap Safety** | `PARTIAL` | Financial buttons need universal loading & debounce guards. | Guard all financial action buttons against concurrent execution. |

---

## Key Hardening Directives for M3A3

1. **Financial Source of Truth**:
   - Financial balances must come exclusively from CDK/redb (`CashuWalletService.getBalance()`) or Cashu mint proof states.
   - Backend coordination metadata must never override or falsify financial truth.
2. **State Separation**:
   - Every transaction item and detail screen must maintain clear separation between:
     - **Financial State**: `Spendable`, `Locked Escrow`, `Claimed`, `Refunded`, `Spent`.
     - **Delivery State**: `Delivered`, `Delivery Pending`, `Relay Failed`.
     - **Coordination State**: `Synced`, `Sync Pending`.
3. **Double-Tap & Concurrency Guards**:
   - Disable buttons while async financial operations are in progress.
4. **Truthful Recovery Scope**:
   - Accurately communicate that mnemonic restore recovers deterministic signing keys and account identity, while proof recovery from a fresh device requires local database backup until NUT-13 proof restoration is supported.
