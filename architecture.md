# Hanbova Technical Architecture

## 1. System Overview

Hanbova is structured into four focused, decoupled project repositories:

```text
┌────────────────────────────────────────────────────────┐
│               hanbova-app (Flutter/Dart)               │
│  - Presentation (Dark/Light Design System, Shell)      │
│  - Riverpod State & Network Environment Switcher       │
│  - Lightning Service (BOLT11 Receive, Pay, NUT-04/05)  │
│  - Client-Side Cryptographic Identities:               │
│      1. Protected Payment Key (secp256k1 P2PK)         │
│      2. Transport Encryption Key (X25519)              │
│  - End-to-End Encrypted Envelope Service (ChaCha20)    │
└──────────────────────────┬─────────────────────────────┘
                           │ Authenticated REST (/api/v1)
                           ▼
┌────────────────────────────────────────────────────────┐
│               hanbova-backend (Rust / Axum)            │
│  - Tokio Async Runtime                                 │
│  - Tower HTTP Middleware (CORS, Trace, Request Limits) │
│  - Argon2id + JWT + Rotating Refresh Tokens            │
│  - Lightning REST Routes (/api/v1/lightning/...)       │
│  - Public Key Routing Directory (/api/v1/users/...)    │
│  - Encrypted Envelope Relay (/api/v1/protected-msg...) │
│  - PostgreSQL Persistence via SQLx (Ciphertext Only)   │
└────────────┬─────────────────────────────┬─────────────┘
             │                             │
             ▼                             ▼
┌───────────────────────────┐ ┌───────────────────────────┐
│hanbova-protected-payments │ │     hanbova-lightning     │
│ - ProtectedPaymentProvider│ │ - LightningProvider trait │
│ - CashuProtectedPayment   │ │ - CashuLightningBridge    │
│ - CDK Wallet & Redb DB    │ │ - NUT-04 Mint Quotes      │
│ - NUT-10 & NUT-11 P2PK    │ │ - NUT-05 Melt Quotes      │
└────────────┬──────────────┘ └────────────┬──────────────┘
             │                             │
             │ (Talks to Mint via HTTP)    │
             ▼                             ▼
   ┌───────────────────┐         ┌───────────────────┐
   │ Cashu Nutshell /  │         │ Lightning Node /  │
   │ CDK Mint (Docker) │         │ LSP / Breez SDK   │
   │ (testnut.cashu.sp)│         │                   │
   └───────────────────┘         └───────────────────┘
             │
             └──────────────┬──────────────┘
                            ▼
             ┌─────────────────────────────┐
             │     crates/hanbova-core     │
             │ - Pure Domain Models        │
             │ - PaymentType, PaymentStatus│
             │ - SatoshiAmount, Invariants │
             └─────────────────────────────┘
```

---

## 2. Dual Payment Rails: Instant Send vs Protected Send

### 1. Instant Send (Bitcoin Lightning)
- **Settlement**: Immediate final settlement on the Lightning Network (sub-second).
- **Format**: BOLT11 invoices (`lnbc...` / `lntb...`) and Lightning Addresses (`name@domain.com`).
- **Use cases**: In-person merchants, coffee, digital services, small instant tips.

### 2. Protected Send (Cashu P2PK Escrow)
- **Settlement**: Conditional ecash escrow locked to recipient's public key with sender refund path and locktime.
- **Spending Conditions**: NUT-10 Spending Conditions & NUT-11 Pay-to-Public-Key.
- **Use cases**: P2P trades, informal commerce, distance deliveries, unverified merchants.

---

## 3. Ecash-Lightning Interoperability (NUT-04 & NUT-05 Swaps)

```text
       ┌──────────────────────┐
       │ Lightning Payment    │
       │ (External / Invoice) │
       └──────────┬───────────┘
                  │
        NUT-04    │    NUT-05
        Deposit   │    Withdraw / Melt
                  ▼
       ┌──────────────────────┐
       │ Hanbova Ecash Escrow │
       │ (Spendable/Protected)│
       └──────────────────────┘
```

1. **Mint via Lightning (NUT-04)**:
   - App requests a mint quote for $X$ sats $\rightarrow$ mint generates BOLT11 invoice.
   - Payer pays invoice via Lightning $\rightarrow$ mint issues signed blind proofs.
2. **Melt via Lightning (NUT-05)**:
   - App presents an external BOLT11 invoice $\rightarrow$ mint calculates payment amount + fee reserve.
   - App spends ecash proofs to the mint $\rightarrow$ mint executes Lightning payment and settles invoice.
