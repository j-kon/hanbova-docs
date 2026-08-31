# Milestone 3A.3: Secret & Sensitive Data Logging Audit

**Milestone**: 3A.3 Functional Hardening & Daily-Use Reliability  
**Date**: 2026-08-31  
**Audit Scope**: Complete static and runtime audit of `hanbova-app`, `hanbova-backend`, and all FFI bridge crates for unintentional exposure or logging of sensitive cryptographic secrets.

---

## 1. Audit Directives & Classification

| Secret Category | Target Definition | Required Handling | Audit Finding |
| --- | --- | --- | --- |
| **BIP-39 Mnemonic** | 12-word master recovery phrase | In-memory only during generation/restore; zero disk/console logging | **CLEAN ✅** (Zero log occurrences) |
| **Wallet Master Seed** | 64-byte derived binary/hex seed | Scoped to FFI boundary and secure storage; zero logging | **CLEAN ✅** (Zero log occurrences) |
| **P2PK Private Keys** | Secp256k1 private keys | Scoped to native CDK redb engine and cryptographic service; zero logging | **CLEAN ✅** (Zero log occurrences) |
| **X25519 Transport Secret** | Ephemeral/static transport private keys | Encrypted envelope engine in-memory only; zero logging | **CLEAN ✅** (Zero log occurrences) |
| **JWT Bearer Tokens** | Authentication bearer tokens | Sanitized in network interceptors & error handlers; redacted | **CLEAN ✅** (Zero raw token logging) |
| **Cashu Ecash Proofs** | Raw blinded secrets / unspent token strings | Redacted from user-facing error banners and log messages | **CLEAN ✅** (Tokens truncated or encapsulated) |
| **Database File Paths** | Local redb / SQLite filesystem URIs | Sanitized via `ConsumerErrorTranslator` | **CLEAN ✅** (Paths sanitized in UI) |

---

## 2. Codebase Audit Results

1. **`hanbova-app/lib`**:
   - `print()` and `debugPrint()` invocations: **0 log leaks found**.
   - Public keys: Only public keys and SHA-256 key fingerprints (`CryptoIdentityNotifier.computeFingerprint`) are exchanged or displayed.
   - Error messages: `ConsumerErrorTranslator` dynamically redacts hex sequences $\ge 32$ characters and local filesystem path substrings.
2. **`hanbova-backend/crates`**:
   - `hanbova-api`: Standard structured logging without request body token dumps.
   - `hanbova-cdk-ffi`: Native C-FFI exports sanitize and return structured JSON result payloads.
   - `hanbova-protected-payments`: Example CLI outputs display strictly public keys and amounts.

---

## 3. Conclusion

The application and backend maintain strict cryptographic confidentiality. No bearer assets, private keys, seed phrases, or authorization credentials are exposed in stdout, system logs, or user-facing UI error messages.
