# Hanbova Product & Technical Roadmap

## Milestone Overview

| Milestone | Scope | Status |
| :--- | :--- | :--- |
| **Milestone 1** | Monorepo Bootstrap & Clean Architecture Foundation | `COMPLETED` ✅ |
| **Milestone 2** | Genuine Cashu NUT-10 & NUT-11 Protected Payments | `COMPLETED` ✅ |
| **Milestone 2.5** | Consumer Wallet Experience & Centralized Design System | `COMPLETED` ✅ |
| **Milestone 3A** | Real Two-Device Cashu Test Wallet + Secure Encrypted Delivery | `COMPLETED` ✅ |
| **Milestone 3B** | Breez Spark Lightning Integration & Testnet Liquidity | `UPCOMING` ⏳ |
| **Milestone 4** | Production Hardening, Multi-Mint Routing & App Store Readiness | `PLANNED` 📋 |

---

## Milestone 3A Completed Highlights
- Pinned Nutshell (`0.16.5`) and CDK (`0.18.0-rc.0`).
- Multi-network environment support (`Local Development`, `Cashu Test`, `Mainnet [Disabled]`).
- Dual client-side cryptographic identities (`secp256k1` for P2PK spending + `X25519` for encrypted envelope delivery).
- End-to-end encrypted envelope transport (Ephemeral X25519 + ChaCha20-Poly1305 AEAD). Zero bearer token or private key custody on server.
- Object-level authorized inbox/outbox messaging and public key directory.
- Two-device test suite and reproducible claim/refund verification.

---

## Milestone 3B Preparation: Breez Spark Integration
- Integrate Breez Spark SDK for non-custodial Lightning payments and on-chain swap ins/outs.
- Connect valueless Bitcoin test networks (Signet / Mutinynet).
- Ensure seamless interoperability between Lightning balances and Cashu protected escrows.
