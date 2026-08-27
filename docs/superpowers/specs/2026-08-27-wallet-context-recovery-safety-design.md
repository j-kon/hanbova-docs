# Wallet Context and Recovery Safety Design

**Status:** Approved in chat on 2026-08-27; awaiting written-spec review

**Parent design:** `docs/superpowers/specs/2026-08-27-release-hardening-design.md`

## Purpose

This design is the first focused delivery from the broader Hanbova release-hardening effort. It prevents an authenticated account or network from using another wallet identity, makes recovery operations fail closed, and makes recovery claims truthful for the controlled pilot.

This work does not implement full Cashu ecash recovery after total device loss. It restores Hanbova's deterministic wallet identity and rebuilds the locally available wallet state. Any balance recovery that depends on mint support remains explicitly experimental.

## Product Decisions

- Backup and restore require a signed-in Hanbova account.
- A restored phrase is bound to the currently authenticated account and active wallet environment.
- Signing out clears wallet material and services from memory but does not delete encrypted recovery material from the device.
- Recovery phrase reveal and replacement require successful operating-system authentication.
- An unavailable, cancelled, or failed authentication attempt is a denial.
- Backup confirmation is advisory state stored separately for each account and wallet environment.

## Wallet Context

Introduce an immutable `WalletContextKey` containing:

- `userId`, taken only from the authenticated session;
- `network`, the active `HanbovaNetwork`;
- `storagePrefix`, taken from the exact active `NetworkConfig`.

The equality and persistence identity of a wallet context use all three values. `storagePrefix` is required because locked mainnet and the controlled mainnet pilot share the `mainnet` enum value but must never share identity, Cashu, escrow, or backup-confirmation storage.

The key exposes one canonical, delimiter-safe storage identifier used by wallet security persistence. Callers do not construct storage keys independently.

Add a nullable `activeWalletContextKeyProvider`. It returns a key only when the auth state is authenticated, the user ID is non-empty, and the active network configuration is enabled. Otherwise it returns `null`.

## Identity Lifecycle

`CryptoIdentityNotifier` watches `activeWalletContextKeyProvider`. When the context changes or becomes unavailable, the notifier immediately stops exposing the previous identity.

Identity creation and restoration read the active context internally rather than accepting caller-supplied account or network arguments. This removes the ability for a screen to initialize an identity for a context other than the current authenticated session.

Each asynchronous identity operation captures its starting context. Before publishing a completed identity into provider state, it re-reads the active context. If the context no longer matches, the result is discarded and reported as a stale-session failure. Secure-storage writes use the captured exact context, so a late operation cannot overwrite another context's material.

`WalletCryptoIdentity` carries its `WalletContextKey`. Legacy `userId` and `network` access can remain as derived getters during migration, but context comparison uses the complete key.

The active Cashu wallet service is constructed only when all of the following are true:

1. an active wallet context exists;
2. a loaded identity exists;
3. the identity's complete context equals the active context;
4. the active network configuration remains enabled;
5. the service receives the same storage prefix and effective mint URL represented by the active configuration.

If any condition is false, the provider returns no wallet service and disposes the prior service. Screens must treat this as “wallet unavailable” and must not offer value-moving actions.

## Secure Storage

Identity storage uses the canonical context identifier. The mnemonic remains the single source for deterministic X25519 and secp256k1 key derivation; obsolete independently stored derived private-key entries are removed only through an explicit, separately tested migration or wallet deletion operation.

The existing encrypted mnemonic for each compatible context remains authoritative. This phase does not move, merge, or delete existing Cashu databases or escrow records.

Backup confirmation is persisted through a small `WalletBackupStore` abstraction backed by `FlutterSecureStorage`. Its key includes the complete wallet context and a schema version. Missing or malformed values resolve to `false`. Completing the backup quiz or successfully restoring a phrase stores `true` for only the active context.

## Backup Flow

The backup screen has four explicit states: loading, ready, unavailable, and error.

1. It requires an active wallet context.
2. It requests the identity for that context.
3. It never generates a mnemonic as fallback behavior.
4. If authentication or identity is unavailable, it shows a recovery action such as signing in or returning to wallet setup; it does not render recovery words.
5. The recovery words remain obscured until OS authentication succeeds.
6. Cancellation, platform errors, unsupported hardware, or an unsecured device keep the words obscured and show a concise safe message.
7. Passing the existing word-confirmation quiz persists backup confirmation for the active context before showing success.

No mnemonic or private key appears in logs, analytics, errors, clipboard automation, or ordinary preferences.

## Restore Flow

Restore is available only when `activeWalletContextKeyProvider` is non-null. The unauthenticated `default_user` namespace is removed.

The flow is:

1. collect and normalize exactly 12 BIP-39 words;
2. validate the phrase and checksum locally;
3. require successful OS authentication with wording that clearly authorizes replacement of the active wallet identity;
4. capture the active wallet context and show a final replacement warning;
5. persist the phrase under that exact context;
6. derive the deterministic transport and protected-payment identities;
7. discard the result if the session context changed during the operation;
8. publish the recovered public keys to the authenticated account;
9. invalidate and rebuild the Cashu wallet service for the same context;
10. persist backup confirmation and show the outcome.

The final result has truthful partial states:

- **Identity restored:** local identity, public-key publication, and wallet rebuild succeeded.
- **Identity restored; sync pending:** local identity and wallet rebuild succeeded, but public-key publication failed. The user receives a retry action and is warned that receiving protected payments may be unavailable until sync completes.
- **Restore failed:** validation, OS authentication, secure persistence, derivation, context validation, or wallet rebuild failed. No full-success message or Home navigation is shown.

Successful copy states: “Your wallet identity has been restored. Full ecash balance recovery after complete device loss remains experimental.” The UI must not claim that an ecash balance was recovered unless the wallet service can prove it.

## Failure and Concurrency Rules

- All recovery security decisions fail closed.
- Raw platform and storage exceptions are converted to stable user-facing categories and may be logged only after secret-bearing values are removed.
- Only the newest operation for the current context may update identity state.
- A context change cancels or invalidates backup/restore UI work and prevents navigation based on stale completion.
- Retry operations re-read the active authenticated context instead of reusing an old screen argument.
- Local phrase replacement is not rolled back after a later key-publication failure; that outcome is the explicit “sync pending” state. Secure-storage or derivation failure occurs before publication and is a failed restore.

## Testing Contract

Every behavioral change follows red-green-refactor.

### Provider and service tests

- no active context exists while unauthenticated, loading, errored, disabled, or using an empty user ID;
- context equality distinguishes users, test environments, locked mainnet, and mainnet pilot by exact storage prefix;
- an identity for user A cannot construct a wallet for user B;
- an identity from one storage prefix cannot construct a wallet for another;
- logout and network changes stop exposing the old identity and dispose the old wallet service;
- an asynchronous identity result is discarded when auth or network context changes before completion;
- identity creation and restore cannot accept an arbitrary caller-supplied account ID;
- backup confirmation persists across provider-container restart and remains isolated by complete context;
- missing or malformed backup confirmation fails to `false`.

### Security and widget tests

- biometric/device authentication returns `false` when unavailable, cancelled, rejected, or errored and `true` only on explicit OS success;
- backup cannot display or generate words without an authenticated, matching identity;
- failed authentication never reveals words;
- restore cannot run while signed out and never uses `default_user`;
- restore requires device authentication and a final replacement confirmation;
- a context change during restore blocks success navigation;
- public-key publication failure produces “sync pending,” not full success;
- wallet-rebuild failure produces a failed outcome and does not navigate Home;
- successful restore uses the active context's exact storage prefix and displays the ecash-recovery limitation.

### Regression verification

- existing deterministic key-derivation tests continue to pass;
- existing Cashu storage and protected-payment tests continue to pass;
- `dart format --set-exit-if-changed lib test` passes;
- `flutter analyze` reports no findings;
- `flutter test` passes in full.

## Migration and Compatibility

Current secure-storage mnemonic keys already include a user ID and a derived storage prefix. The implementation must first characterize existing keys for local, Cashu test, compile-time pilot, and runtime pilot configurations with tests.

If the new canonical key differs from a compatible existing key, migration copies the mnemonic only after validating it and deriving the expected identity. The original key remains until the new identity and wallet initialize successfully. There is no cross-user or cross-prefix migration, and no migration from locked-mainnet storage into pilot storage.

The existing user-authored Claim-button sizing change is outside this phase and remains untouched.

## Non-Goals

- Full NUT-13 or mint-assisted ecash balance recovery.
- Transaction-ledger persistence.
- Payment review flows or payment-cap enforcement.
- Runtime mainnet-pilot removal, except where exact context isolation requires recognizing its storage prefix.
- Biometric login or biometric transaction signing.
- Broad UI redesign, localization, or accessibility remediation.
- Deleting wallet material on logout.

## Completion Criteria

This phase is complete only when:

1. no wallet service can exist with a mismatched account or storage environment;
2. backup and restore cannot access or create recovery material while unauthenticated;
3. recovery phrase reveal and replacement fail closed under every non-successful OS-authentication result;
4. stale asynchronous work cannot reactivate a previous wallet context;
5. backup confirmation survives restart and is context-isolated;
6. restore outcomes and ecash limitations are truthful;
7. all focused and full Flutter verification commands pass.
