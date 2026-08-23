# Hanbova Technical Architecture

## 1. System Overview

Hanbova is structured into four focused, decoupled project repositories:

```text
┌────────────────────────────────────────────────────────┐
│               hanbova-app (Flutter/Dart)               │
│  - Presentation (Dark/Light Design System, Shell)      │
│  - Riverpod State & Network Environment Switcher       │
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
│  - Public Key Routing Directory (/api/v1/users/...)    │
│  - Encrypted Envelope Relay (/api/v1/protected-msg...) │
│  - PostgreSQL Persistence via SQLx (Ciphertext Only)   │
└────────────┬─────────────────────────────┬─────────────┘
             │                             │
             ▼                             ▼
┌───────────────────────────┐ ┌───────────────────────────┐
│hanbova-protected-payments │ │     hanbova-lightning     │
│ - ProtectedPaymentProvider│ │ - LightningProvider trait │
│ - CashuProtectedPayment   │ │ - BOLT11 Invoices         │
│ - CDK Wallet & Redb DB    │ │ - Breez SDK Adapter       │
│ - NUT-10 & NUT-11 P2PK    │ │                           │
└────────────┬──────────────┘ └────────────┬──────────────┘
             │                             │
             │ (Talks to Mint via HTTP)    │
             ▼                             ▼
   ┌───────────────────┐         ┌───────────────────┐
   │ Cashu Nutshell /  │         │ Lightning Node /  │
   │ CDK Mint (Docker) │         │ FakeWallet / LND  │
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

## 2. Cryptographic Identities & Transport Architecture

Every Hanbova user maintains two separate cryptographic identities generated and held exclusively client-side:

### 1. Protected Payment Identity (`secp256k1`)
- **Public Key**: `02<32-byte-pubkey>`
- **Purpose**: Cashu NUT-11 Pay-to-Public-Key spending path and refund authorization path.
- **Custody**: Device keychain/keystore. Never transmitted to backend.

### 2. Transport Encryption Identity (`X25519`)
- **Public Key**: 32-byte public key (hex)
- **Purpose**: End-to-end authenticated encryption of protected payment envelopes between users.
- **Custody**: Device keychain/keystore. Never transmitted to backend.

---

## 3. End-to-End Encrypted Protected Payment Lifecycle

```text
Alice Device (Sender)                     Hanbova Server                     Bob Device (Recipient)
       │                                        │                                      │
1. Type @bob                                    │                                      │
       │─── GET /users/bob/payment-profile ────>│                                      │
       │<── Bob secp256k1 & X25519 pubkeys ─────│                                      │
       │                                        │                                      │
2. Create Cashu P2PK Token                      │                                      │
   (locked to Bob pubkey,                       │                                      │
    Alice refund key, locktime)                 │                                      │
       │                                        │                                      │
3. Encrypt Envelope (v1)                        │                                      │
   (Ephemeral X25519 ECDH + HKDF                │                                      │
    + ChaCha20-Poly1305 AEAD)                   │                                      │
       │                                        │                                      │
4. Send Encrypted Envelope                      │                                      │
       │─── POST /protected-messages ──────────>│ (Stores ciphertext only,            │
       │    (Ciphertext payload only)           │  cannot decrypt Cashu token)         │
       │                                        │                                      │
5. Bob Polls Inbox                              │                                      │
       │                                        │<── GET /protected-messages/inbox ────│
       │                                        │─── Encrypted Ciphertext ────────────>│
       │                                        │                                      │
6. Bob Decrypts Locally                         │                                      │
   (Bob X25519 private key                      │                                      │
    recovers Cashu token)                       │                                      │
       │                                        │                                      │
7. Bob Claims at Cashu Mint                     │                                      │
   (Signs with Bob secp256k1                    │                                      │
    private key -> Mint confirms)               │                                      │
       │                                        │                                      │
8. Acknowledge Claim                            │                                      │
       │                                        │<── POST /protected-messages/:id/ack ─│
```

---

## 4. Network Isolation & Security Principles

1. **Zero Bearer Token Leakage**: Plaintext Cashu tokens and private keys are never sent to or stored in backend databases.
2. **Environment Isolation**:
   - `Local Development`: Local Nutshell mint (`http://127.0.0.1:3338`).
   - `Cashu Test`: Public test mint (`https://testnut.cashu.space`) with valueless test ecash.
   - `Mainnet`: Hardcoded as disabled. Fail-closed on any attempt to select.
3. **Locktime Semantics**: Locktime activation enables sender self-service refund without invalidating recipient claim path until one spend wins at the mint.
