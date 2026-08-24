# Hanbova: Comprehensive Project Overview & Architecture Guide

> **Tagline**: *Send protected.*  
> **Mission**: Safer everyday Bitcoin payments across Africa. Send instantly or protect money with conditional claim windows.  
> **Current Version**: `v0.5.0-beta`  
> **Target Audience / Program**: Afro Bitcoin Fellowship & Production Beta  

---

## 1. Executive Summary & Problem Statement

### The Problem: Irreversibility in Everyday Commerce
Across African corridors, peer-to-peer commerce and online trade face acute trust deficits:
- **Instant Finality Risk**: When buyers pay upfront via Lightning or on-chain Bitcoin, transactions are irreversible. If goods are not delivered or are defective, the buyer has zero recourse.
- **Delivery Risk**: When merchants ship goods before receiving payment, buyers often default or refuse payment upon arrival.
- **The Centralization Trap**: Lack of conditional escrow on Bitcoin rails forces ordinary merchants and consumers back onto centralized, permissioned mobile-money or bank rails that charge exorbitant fees and freeze accounts arbitrarily.

### The Solution: Hanbova Dual-Track Wallet
Hanbova unifies two complementary payment mechanisms in a single consumer mobile wallet:
1. **Instant Send (Bitcoin Lightning Network)**: Sub-second final settlement via BOLT11 invoices and Lightning addresses for trusted, everyday microtransactions (coffee, groceries, airtime).
2. **Protected Send (Cashu NUT-10 / NUT-11 P2PK Escrow)**: Cryptographic conditional escrow locking ecash proofs to the recipient's public key with a sender-specified claim window (locktime).
   - If the recipient delivers goods within the window, they sign with their private key and claim the funds.
   - If the recipient defaults or fails to claim, the sender executes an **on-chain self-service refund** after locktime expiry without needing third-party arbitration.

---

## 2. System Architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          HANBOVA MOBILE APP (Flutter)                       │
│  - Presentation Layer: Clean Dark/Light Design System                       │
│  - Dual Payment Rails: Instant Send (Lightning) vs Protected Send (Cashu)   │
│  - Unified Deposit Sheet: Lightning Quotes (NUT-04) / On-Chain / Ecash      │
│  - Client Cryptographic Identity Service:                                   │
│      * BIP-39 12-Word Mnemonic Phrase -> 512-Bit HMAC-SHA512 Seed           │
│      * PointyCastle Secp256k1 (P2PK Escrow Keypairs)                        │
│      * X25519 ECDH + ChaCha20-Poly1305 AEAD (End-to-End Encryption)         │
│  - Dart FFI Bridge (CdkFfiBindings) -> In-Process Native Execution          │
└──────────────────────┬───────────────────────────────┬──────────────────────┘
                       │ Dart FFI                      │ Authenticated HTTPS
                       ▼                               ▼
┌─────────────────────────────────────────┐  ┌────────────────────────────────┐
│  crates/hanbova-cdk-ffi (Rust cdylib)   │  │   hanbova-backend (Axum API)   │
│  - Official Cashu Dev Kit (cdk 0.18-rc) │  │  - Zero-Custody Coordinator    │
│  - Persistent Embedded Storage (redb)   │  │  - Argon2id + JWT + AuthUser   │
│  - Client Wallet Authority              │  │  - Encrypted Relay Service     │
│  - NUT-04, NUT-07, NUT-10, NUT-11 P2PK  │  │  - PostgreSQL 16 via SQLx      │
└──────────────────────┬──────────────────┘  └────────────────────────────────┘
                       │ HTTP                                        │
                       ▼                                             ▼
┌─────────────────────────────────────────┐  ┌────────────────────────────────┐
│           Cashu Mint (Nutshell)         │  │     Lightning Node / LSP       │
│  - Blinded Signatures (BDHKE)           │  │  - BOLT11 Invoices             │
│  - NUT-11 P2PK Spending Verification    │  │  - Sub-second Lightning Pay    │
└─────────────────────────────────────────┘  └────────────────────────────────┘
```

---

## 3. Cryptographic Invariants & Protocols

### 3.1 Genuine Client-Side Secp256k1 Key Generation
All P2PK private keys are generated on-device using a cryptographically secure random number generator (`Random.secure()`). The scalar $d$ is strictly validated within the valid curve range $[1, n-1]$. The corresponding public key $Q = d \cdot G$ is calculated via PointyCastle elliptic curve scalar multiplication and serialized as standard 33-byte compressed hex (`02` or `03` prefix).

### 3.2 End-to-End Encrypted Message Relay (E2EE)
When a sender creates a Protected Payment, the locked ecash token payload is encrypted on-device before leaving the phone:
1. Sender generates an ephemeral X25519 keypair.
2. Performs Diffie-Hellman key exchange with the recipient's registered X25519 public key to derive a 256-bit symmetric shared secret.
3. Encrypts the Cashu token string using **ChaCha20-Poly1305 AEAD** with a unique 12-byte nonce.
4. Uploads the encrypted envelope (`ephemeral_pubkey`, `nonce`, `ciphertext`) to the backend.
5. The backend stores and relays ciphertext only; it **never sees private keys, claim codes, or decrypted tokens**.

### 3.3 Official CDK C-FFI Bridge & Client Wallet Authority
The mobile app communicates with the official Cashu Dev Kit (`cdk = "0.18.0-rc.0"`) through a lightweight C-ABI export layer (`crates/hanbova-cdk-ffi`):
- **Functions Exported**: `hanbova_cdk_wallet_create`, `hanbova_cdk_wallet_total_balance`, `hanbova_cdk_mint_quote`, `hanbova_cdk_mint_tokens`, `hanbova_cdk_send_p2pk`, `hanbova_cdk_receive_token`, `hanbova_cdk_check_token_state`, `hanbova_cdk_wallet_free`.
- **Database Storage**: Isolated `cdk-redb` database file per user and network (`hanbova_cdk_<user>_cashu_test.redb` vs `hanbova_cdk_<user>_mainnet.redb`).
- **P2PK Send Witness**: Sets recipient pubkey, sender refund pubkey, locktime, and `SigFlag::SigInputs` ensuring mathematical enforceability at the mint.

---

## 4. Repositories in the Ecosystem

| Repository | Tech Stack | Purpose |
| :--- | :--- | :--- |
| **`j-kon/hanbova-app`** | Flutter 3.29, Riverpod, GoRouter, Dart FFI | Cross-platform mobile wallet client for iOS and Android |
| **`j-kon/hanbova-backend`** | Rust 1.84, Axum, Tokio, SQLx, Postgres, CDK | Zero-custody backend relay, CDK FFI crate, and protected escrow services |
| **`j-kon/hanbova-protocol`** | Markdown / Formal Specs | Protocol specifications for HPP-01 (Encrypted Transport) and HPP-02 (Escrow Lifecycle) |
| **`j-kon/hanbova-docs`** | Markdown / Mermaid | Technical architecture, threat model, presentation pitch, and roadmap |

---

## 5. Security & Threat Mitigation Matrix

| Potential Threat | Architecture Mitigation |
| :--- | :--- |
| **Server Database Compromise** | Zero-Custody: The server stores only X25519 ciphertexts. No private keys, BIP-39 seeds, or Cashu proofs exist on the backend. |
| **Recipient Inaction / Default** | NUT-11 Timelocked Escrow: The sender can claim a self-service refund immediately once locktime expires. |
| **Man-in-the-Middle Attack** | End-to-End Authenticated Encryption: Ephemeral X25519 + ChaCha20-Poly1305 AEAD ensures only the recipient's private key can decrypt tokens. |
| **Device Loss or Failure** | 12-Word BIP-39 Mnemonic Backup with 3-word verification quiz and PBKDF2 seed derivation (`HMAC-SHA512`). |
| **Mainnet Accidental Loss** | Mainnet Safety Dialog requiring explicit risk disclosures, backup verification, and isolated database namespaces. |

---

## 6. Key API Endpoints (`hanbova-backend`)

### Authentication (`/api/v1/auth`)
- `POST /signup`: Register user with username, password, Secp256k1 public key, and X25519 public key.
- `POST /login`: Authenticate and receive JWT access token + refresh token.
- `POST /refresh`: Rotate refresh token and obtain fresh access token.

### Public Directory (`/api/v1/users`)
- `GET /:handle/payment-keys`: Fetch user's registered Secp256k1 P2PK and X25519 transport public keys.

### Payment Intents & Coordination (`/api/v1/payment-intents`)
- `POST /`: Create protected payment intent (authenticated, validates keys).
- `GET /:id`: Retrieve payment intent details (restricted strictly to sender or recipient).
- `POST /:id/status`: Update status transition (`claimed` by recipient or `refunded` by sender after locktime).

### Encrypted Message Relay (`/api/v1/protected-messages`)
- `POST /send`: Upload encrypted message envelope for recipient.
- `GET /inbox`: Retrieve incoming encrypted messages for authenticated user.
- `GET /outbox`: Retrieve outgoing encrypted messages sent by authenticated user.

---

## 7. Verification & Automated Test Status

```text
======================================================================
              HANBOVA AUTOMATED TEST VERIFICATION SUITE
======================================================================
🦀 Rust Backend Workspace:
   - Command:  cargo test --workspace --all-targets
   - Result:   24 / 24 PASSED (100% Green)
   - Clippy:   0 Warnings (cargo clippy --workspace -- -D warnings)

📱 Flutter Mobile Client:
   - Command:  flutter test
   - Result:   42 / 42 PASSED (100% Green)
   - Analyzer: 0 Issues (flutter analyze)

📦 Production Release Artifacts:
   - Android APK:   build/app/outputs/flutter-apk/app-release.apk (56.1 MB)
   - iOS Bundle:    build/ios/iphoneos/Runner.app (18.9 MB)
   - CDK C-FFI:     crates/hanbova-cdk-ffi/target/release/libhanbova_cdk_ffi.dylib
======================================================================
```

---

## 8. Fellowship Pitch Summary (3-Minute Script)

1. **Introduction**: Introduce Hanbova and the African trust barrier in digital commerce.
2. **Instant Lightning Demo**: Show sub-second BOLT11 invoice payment with haptic feedback.
3. **Protected Send Demo**: Send a 24-hour protected escrow to `@bob`. Show Bob receiving the notification, inspecting the locktime, and claiming the payment upon delivery.
4. **Seed Backup & Security**: Demonstrate the 12-word BIP-39 backup flow with interactive verification quiz.
5. **Conclusion**: Highlight open source deliverables, 66/66 passing tests, and mainnet beta readiness.
