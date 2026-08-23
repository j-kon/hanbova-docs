# Hanbova Product & Technical Roadmap

## Milestone Overview

| Milestone | Scope | Status |
| :--- | :--- | :--- |
| **Milestone 1** | Monorepo Bootstrap & Clean Architecture Foundation | `COMPLETED` ✅ |
| **Milestone 2** | Genuine Cashu NUT-10 & NUT-11 Protected Payments | `COMPLETED` ✅ |
| **Milestone 2.5** | Consumer Wallet Experience & Centralized Design System | `COMPLETED` ✅ |
| **Milestone 3A** | Real Two-Device Cashu Test Wallet + Secure Encrypted Delivery | `COMPLETED` ✅ |
| **Milestone 3B** | Lightning Integration & Ecash Swaps (NUT-04/NUT-05) | `COMPLETED` ✅ |
| **Milestone 4** | Production Hardening, Multi-Mint Routing & App Store Readiness | `PLANNED` 📋 |

---

## Milestone 3B Completed Highlights
- Created `CashuLightningBridge` for Cashu NUT-04 (Mint via Lightning) and NUT-05 (Melt via Lightning).
- Exposed `/api/v1/lightning/invoice`, `/api/v1/lightning/pay`, `/api/v1/lightning/mint-quote`, `/api/v1/lightning/melt-quote`.
- Implemented `LightningService` in `hanbova-app` for invoice generation, payments, and quotes.
- Integrated `SendScreen` and `ReceiveScreen` with dynamic Lightning invoice creation and fee parsing.
- Added comprehensive unit and integration tests across both Rust backend (23 tests) and Flutter client (27 tests).
