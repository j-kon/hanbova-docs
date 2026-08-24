# Hanbova Product & Technical Roadmap

## Milestone Overview

| Milestone | Scope | Status |
| :--- | :--- | :--- |
| **Milestone 1** | Monorepo Bootstrap & Clean Architecture Foundation | `COMPLETED` ✅ |
| **Milestone 2** | Genuine Cashu NUT-10 & NUT-11 Protected Payments | `COMPLETED` ✅ |
| **Milestone 2.5** | Consumer Wallet Experience & Centralized Design System | `COMPLETED` ✅ |
| **Milestone 3A** | Real Two-Device Cashu Test Wallet + Secure Encrypted Delivery | `COMPLETED` ✅ |
| **Milestone 3A.1** | Security Correction: Move Cashu Wallet Authority to Client & Genuine Secp256k1 | `COMPLETED` ✅ |
| **Milestone 3B** | Lightning Integration & Ecash Swaps (NUT-04/NUT-05) | `COMPLETED` ✅ |
| **Milestone 4** | Production Hardening, BIP-39 Backup, Biometrics & Multi-Mint | `COMPLETED` ✅ |
| **Milestone 5** | Public Mainnet Beta, On-Chain Swaps & App Store Publishing | `PLANNED` 📋 |

---

## Milestone 3A.1 Completed Highlights
- **PointyCastle Secp256k1 Service**: Genuine cryptographically secure 256-bit scalar generation in range $[1, n-1]$ with elliptic-curve scalar multiplication ($Q = G \cdot d$) to derive standard 33-byte compressed public keys.
- **Client-Side Wallet Authority**: Client wallet directly owns and stores all spendable Cashu proofs and NUT-11 escrow keys across restarts (`CashuWalletService` & `CashuWalletStorage`).
- **Zero-Custody Backend**: Completely stripped `claim_proof` and private keys from backend API models and database. Backend serves strictly as a zero-custody coordination and encrypted message relay service.
- **Automated Test Coverage**: 1,000-key uniqueness and validity tests, P2PK claim, unauthorized claim rejection, and post-locktime refund tests all passing (39 Flutter tests, 23 Backend tests).

## Milestone 4 Completed Highlights
- Complete 2048-word BIP-39 English dictionary and `MnemonicService` for 12-word seed phrase generation, validation, and autocomplete.
- Interactive `BackupSeedScreen` with tap-to-reveal, screenshot security warnings, and 3-word verification quiz.
- `RestoreSeedScreen` with 12-word entry, live autocomplete suggestions, and checksum verification.
- `BiometricService` wrapping platform Face ID, Touch ID, and hardware key security.
- `MintsScreen` for multi-mint management, active mint switching, and live NUT-11 capability probe validation.
- All test suites green: 23 backend tests, 31 Flutter client tests.
