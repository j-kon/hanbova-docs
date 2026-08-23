# Hanbova Threat Model & Security Architecture

## 1. Trust Boundaries

```text
┌─────────────────────────────────────────────────────────────┐
│ Trust Boundary: User Device (Client-Side)                   │
│  - secp256k1 P2PK Private Keys                              │
│  - X25519 Transport Encryption Private Keys                 │
│  - Cashu Proofs & Secret Derivation                         │
│  - Decrypted Protected Payment Envelopes                    │
└──────────────────────────────┬──────────────────────────────┘
                               │
               End-to-End Encrypted Transport
               (X25519 + ChaCha20-Poly1305)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Untrusted / Semi-Trusted: Hanbova Backend & PostgreSQL      │
│  - User Account Profiles (Username, Argon2id Hash)          │
│  - User Public Keys Only (secp256k1 & X25519 Public Keys)   │
│  - Encrypted Message Ciphertexts (v1:... opaque payloads)   │
│  - CANNOT inspect or spend Cashu tokens                     │
└──────────────────────────────┬──────────────────────────────┘
                               │
               Cryptographic P2PK Spending Conditions
               (NUT-10 & NUT-11)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Cashu Mint (Nutshell / CDK)                                 │
│  - Authoritative for proof spend state & double-spend checks│
│  - Enforces P2PK signatures and locktime constraints        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Threat Analysis & Mitigations

| Threat | Risk | Mitigation |
| :--- | :--- | :--- |
| **Backend Database Compromise** | High | PostgreSQL stores **only ciphertext**. Even with full database access, attackers cannot decrypt Cashu tokens or steal funds. |
| **Eavesdropping on Transport** | High | All payload envelopes are encrypted with recipient's X25519 key using ephemeral ECDH + ChaCha20-Poly1305 AEAD before transmission. |
| **Unauthorized Message Access** | Medium | Object-level authorization prevents users (e.g. Charlie) from fetching or querying Bob's messages by ID or enumeration. |
| **Double Spend / Race Condition** | High | Cashu Mint is the single source of truth for proof state (NUT-07). Only one signature (Bob claim or Alice refund) will succeed at the mint. |
| **Accidental Mainnet Connection** | Critical | Mainnet environment is hardcoded as disabled at UI, configuration, and backend layers with fail-closed assertions. |
| **Device Compromise / Key Loss** | High | Private keys reside in secure platform storage (`Keychain` / `Keystore`). App distinguishes session logout from explicit wallet deletion. |
