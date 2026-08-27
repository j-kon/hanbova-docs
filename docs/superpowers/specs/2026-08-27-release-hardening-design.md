# Hanbova Release Hardening Design

**Status:** Approved in chat on 2026-08-27; awaiting written-spec review

## Purpose

This design hardens Hanbova for continued test use and a tightly controlled real-sats pilot. It addresses the security, wallet-state, payment-flow, backend, deployment, trust, accessibility, and test defects identified in the 2026-08-27 cross-cutting audit.

The release remains a controlled pilot. Unsupported product surfaces will be truthful and safe rather than simulated. The work will not add production push notifications, live exchange-rate infrastructure, biometric login, email/phone recipient resolution, or camera scanning.

## Release Gate

The release must remain test-only or tightly allowlisted until all critical requirements in this specification are implemented and their regression tests pass.

Completion requires:

1. Recovery material never fails open.
2. Wallet activity and recovery actions survive refresh and restart.
3. Every wallet operation is scoped to one authenticated user and one immutable network configuration.
4. Pilot caps are enforced below the UI on every payment path.
5. Users review the actual payment amount, fee, recipient, and protection terms before value moves.
6. Production backend startup fails closed and cannot expose mock financial behavior.
7. Unsupported UI capabilities never appear functional.
8. Flutter analysis/tests and Rust formatting/lint/tests pass in a clean checkout without undeclared local services.

## Design Principles

- Financial state is durable, reconcilable, and scoped by wallet context.
- Safety policy lives in domain services, not individual screens.
- Seed, network, mint, and storage namespace must change atomically.
- Production configuration fails closed; development conveniences are explicit.
- User-facing status must be truthful.
- Errors help the user recover without exposing implementation details.
- Existing CDK/Cashu ownership and P2PK/refund-key boundaries are preserved.
- Existing user-authored work is preserved unless a directly overlapping change must be integrated.

## Workstream 1: Flutter Wallet and Security Correctness

### Wallet context

Introduce an immutable `WalletContext` with:

- authenticated `userId`;
- `HanbovaNetwork`;
- active `NetworkConfig`;
- effective mint URL;
- storage prefix;
- matching `WalletCryptoIdentity`.

A `WalletContextController` owns initialization and transitions. Its public state is `loading`, `ready(context)`, or `failure(userMessage)`. No Cashu service is created unless the authenticated user, identity, network, mint, and storage prefix all match the same ready context.

Changing account, network, pilot flavor, or mint invalidates the old context, disposes its CDK service, and enters `loading` until the new identity and service are ready. Screens must disable value-moving actions while the context is not ready.

The normal production build cannot enable mainnet at runtime. Mainnet availability is determined by the existing compile-time pilot flavor. The developer route must reject access outside debug/development mode, and the domain layer must independently reject mainnet when the build flag is absent.

### Durable transaction ledger

Add a `TransactionLedger` interface scoped by `WalletContext.storageKey`. The first implementation stores versioned JSON through `FlutterSecureStorage`, capped to the most recent 500 entries. Transaction models gain strict serialization with unknown-enum/version handling that fails safely and preserves readable records where possible.

Ledger mutations are write-through:

- adding a transaction persists before the UI reports success;
- status changes persist immediately;
- delivery/reconciliation flags persist immediately;
- duplicate IDs are upserts, not duplicate rows.

The transactions provider loads the ledger when the wallet context becomes ready. Pull-to-refresh triggers reconciliation and never clears the ledger. Reconciliation merges:

1. durable local activity;
2. durable protected escrow records;
3. remote protected-message/payment-intent status;
4. current CDK balance where applicable.

If remote reconciliation fails, existing activity stays visible and the UI exposes a stale/offline state. A failed protected-message delivery remains retryable after refresh or restart. Retry loads escrow using the active wallet context's exact storage prefix.

### Seed backup and restore

`BiometricService.authenticate` returns `false` when authentication is unavailable, cancelled, or fails. Recovery phrase reveal requires successful OS authentication. There is no simulator/unsupported-device success fallback.

The backup screen must fail closed if no authenticated wallet context exists. It must never generate an unrelated mnemonic. Backup acknowledgement is persisted per wallet context and restored on restart; it is advisory state, not proof that a backup exists.

The welcome restore entry first requires login or account creation. The mnemonic is validated only after authentication, then restored under the authenticated user's active wallet context. `default_user` is removed. Successful restore invalidates/rebuilds the wallet context and routes to Home only when the new context is ready.

This release continues to state accurately that full ecash recovery after complete device loss is experimental where NUT-13/mint recovery is incomplete.

### Payment policy

Add a pure `WalletPolicy` domain service. It validates:

- positive amounts;
- maximum deposit amount;
- maximum send amount;
- projected maximum wallet balance;
- allowed network/build combination;
- allowlisted mint in pilot mode.

All mint quote, melt quote/payment, instant send, protected send, and deposit entry points invoke the policy. UI checks remain for immediate feedback but are not security boundaries.

### Canonical receive flow

Use one Receive/deposit flow everywhere. The standalone duplicate path is removed or redirects to the canonical flow.

Behavior:

1. User enters an amount; no quote is created merely by opening the screen.
2. Input is debounced.
3. Policy validates amount and projected balance before quote creation.
4. Each request receives a monotonically increasing generation ID; stale responses are ignored.
5. The active invoice, amount, expiry, and status are displayed.
6. Mint execution first checks quote status and handles unpaid, paid, expired, and already-minted states idempotently.

### Canonical instant send flow

This release supports BOLT11 invoices only. Copy and parsing must not advertise Lightning Address or Hanbova handles. The parser accepts only prefixes appropriate to enabled networks and normalizes an optional `lightning:` URI prefix.

For amountful invoices, the unused amount input is removed. If amountless invoices are not supported by the current CDK API, they are rejected with a clear message.

Payment is two-step:

1. Create/inspect the melt quote without paying.
2. Show decoded amount, fee reserve, total wallet impact, truncated invoice fingerprint, network, and finality warning.
3. Require explicit confirmation.
4. Revalidate policy immediately before `payMeltQuote`.
5. Persist the transaction result before showing success.

### Canonical protected send flow

This release supports Hanbova usernames only. Phone and email copy is removed.

Payment is two-step:

1. Resolve the normalized username and show the canonical handle/profile.
2. Show amount, expiry, protection/refund semantics, network, and recipient.
3. Require explicit confirmation before creating the intent or locking ecash.
4. Revalidate wallet policy and wallet context immediately before value movement.

If locking succeeds but message delivery fails, the durable ledger and escrow record must show a recoverable delivery-pending state with retry and eventual refund actions.

### Synchronization and errors

Replace per-screen two-second timers with one wallet-scoped `SyncCoordinator`.

- Only one sync may run at a time.
- Sync runs on context readiness, app resume, explicit refresh, and a conservative backoff timer while foregrounded.
- The initial foreground interval is 15 seconds and backs off to 60 seconds after failures.
- Disposing or changing wallet context cancels pending work.
- Sync errors are retained as sanitized status; they are not swallowed.

Define stable user-facing error categories: authentication required, wallet unavailable, offline, invalid payment request, policy limit, insufficient balance, quote expired/unpaid, recipient unavailable, coordination pending, and unexpected error. Raw exceptions go only to structured debug logs and must not expose seeds, tokens, invoices, JWTs, or private keys.

## Workstream 2: Backend and Deployment Hardening

### Typed fail-closed configuration

`AppConfig::from_env` becomes a fallible constructor. Supported environments are explicitly `development`, `test`, and `production`.

Production requires:

- `HANBOVA_ENV=production`;
- explicit `HANBOVA_API_HOST` and `HANBOVA_API_PORT`;
- explicit `DATABASE_URL`;
- a non-default JWT secret of at least 32 bytes;
- explicit `MINT_URL` with HTTPS except for allowlisted development/test hosts;
- non-mock provider selection.

In production, database connection or migration failure terminates startup. In-memory repositories and default secrets are permitted only in development/test and must be logged as such.

Docker Compose uses the exact variable names consumed by Rust. It may provide development defaults through `.env.example`, but committed Compose files must not contain a deployable JWT secret. The API binds `0.0.0.0` inside the container and addresses the mint by service name.

### Provider and route safety

Mock Lightning and protected-payment providers are selected only in development/test. Production startup rejects a mock selection.

Because no production server-side Lightning provider exists in this release, the `/lightning/*` route group is mounted only in development/test. Development/test access still requires the normal authenticated-user extractor so authorization behavior cannot diverge silently. Request ownership and authorization are applied wherever state is user-specific.

### HTTP hardening

- Missing or invalid credentials return `401`; valid credentials without permission return `403`.
- Authentication, password reset, username/profile lookup, protected messaging, and financial coordination endpoints have endpoint-specific rate limits.
- CORS uses an explicit configured origin allowlist; wildcard CORS is allowed only in development.
- Email and username validation are normalized and constrained.
- Message, description, invoice, username, and identifier fields receive narrow length limits below the global body cap.
- Error responses use stable codes and sanitized messages; internal errors remain in structured logs.
- Production password reset cannot claim delivery unless a configured delivery provider accepted the message. Until delivery exists, the endpoint returns an honest unavailable response without exposing token material.

## Workstream 3: Honest UI, Accessibility, and Maintenance

### Capability presentation

For this release:

- QR camera scanning is disabled and labeled “Camera scanning coming soon.” Manual paste remains available.
- The scanner parser no longer claims to import Cashu tokens unless token import is implemented.
- Notifications show an honest empty/coming-soon state; all hardcoded financial events are removed.
- Biometric Login is disabled and labeled “Coming soon.” OS authentication remains in use only for recovery-phrase reveal.
- Verification badges render only from real account verification state; otherwise they are absent.
- Notification settings never display “Enabled” without a persisted, working capability.
- Fiat conversions are labeled “Estimate” and “Development rate” with no live/staleness implication.
- Recipient fields advertise only supported identifier types.

### Accessibility and responsive layout

- Every actionable control has a minimum 48-by-48 logical-pixel hit target.
- Custom gesture-only controls are replaced by semantic Material controls or wrapped with correct button semantics, labels, focus behavior, and tooltips.
- The center action is named “Pay” and visually distinguishes itself from the separate Send shortcut.
- Critical status does not rely only on color.
- Screens used in payment and recovery flows remain usable at 1.3x and 2.0x text scaling on a small phone viewport.
- Existing Claim-button layout intent is preserved, but its target height is raised from 36 to 48.

### Localization boundary

Add Flutter localization delegates, `supportedLocales`, and an English ARB catalog. Move strings touched by this hardening work into localization resources. This establishes the architecture without inventing unvalidated translations. Additional languages are a separate product/research task.

### Maintenance

- Resolve all current analyzer findings.
- Split large presentation files only where necessary to isolate the new review, capability, or synchronization components; do not perform a broad visual rewrite.
- Remove `google_fonts` if it remains unused after the work.
- Use the package version as the single runtime version source; remove hardcoded profile version text.
- Update README, roadmap, verification context, and test counts to match the implemented state.
- Add Android data-extraction/backup rules appropriate for wallet material. While the recovery phrase is revealed, enable Android `FLAG_SECURE`; on iOS, obscure the phrase whenever screen capture/recording is active and whenever the app is inactive. Add release-signing and network-security checks for both platforms.

## Data Ownership and Migration

All new client persistence keys include a schema version and wallet-context key derived from authenticated user ID plus storage prefix. No raw mnemonic, private key, Cashu token, or refund key is written to logs or ordinary preferences.

On first launch after upgrade:

- Existing secure identity and Cashu/escrow storage remain authoritative.
- The ledger starts empty because prior transactions were not durable.
- Reconciliation imports discoverable escrow and incoming coordination records without fabricating completed history.
- Existing pilot storage prefixes remain unchanged.

Malformed ledger data is quarantined under a diagnostic key and replaced with a safe empty ledger; escrow/Cashu storage is not deleted. The UI reports that activity could not be loaded and offers retry/support diagnostics.

## Testing Strategy

Every behavioral change follows red-green-refactor with the smallest focused regression test first.

### Flutter unit and provider tests

- authentication unavailable/cancelled/failed/successful seed reveal;
- backup status persistence per wallet context;
- restore refuses unauthenticated/shared-user storage;
- wallet context never mixes identity/network/mint/prefix;
- production build cannot runtime-enable mainnet;
- policy rejects each cap bypass at service boundaries;
- ledger serialization, upsert, restart, refresh, malformed data, and context isolation;
- mainnet-pilot failed-delivery retry uses `wallet_mainnet_pilot`;
- stale receive quote responses are ignored;
- send confirmation uses quote amount and fee, not a detached field;
- supported BOLT11 prefix/URI parsing;
- sync in-flight exclusion, lifecycle triggers, and backoff;
- sanitized error mapping.

### Flutter widget tests

- canonical Receive and both Send review flows;
- unsupported features show honest disabled/empty states;
- no fake account/financial/security status is rendered;
- accessibility labels and 48dp targets;
- payment/recovery screens at 1.3x and 2.0x text scale;
- test and pilot network warnings remain prominent.

### Rust tests

- configuration rejects every missing/insecure production requirement;
- development defaults remain explicit and testable;
- production rejects mock providers and persistence fallback;
- Docker variable names are covered by a configuration smoke test or validation script;
- financial routes require authentication;
- CORS allowlist, status codes, rate limits, input lengths, and sanitized errors;
- ordinary workspace tests require no external mint;
- external CDK scenarios run under an explicit integration feature/script with a provisioned mint.

### Final verification

- `dart format --set-exit-if-changed lib test`
- `flutter analyze`
- `flutter test`
- `cargo fmt --all -- --check`
- `cargo clippy --workspace --all-targets -- -D warnings`
- `cargo test --workspace`
- the explicit CDK integration runner with its mint fixture
- Android portrait and small-screen manual checks for Home, Receive, Instant Send review, Protected Send review, activity recovery, Profile, and recovery phrase protection

## Documentation Deliverables

- Update architecture and threat model with `WalletContext`, ledger reconciliation, policy boundaries, production configuration, and mock-provider restrictions.
- Update product/roadmap copy to distinguish implemented, disabled, experimental, and future features.
- Record exact verification commands and counts from the final run.
- Keep recovery limitations and controlled-pilot limits explicit.

## Non-Goals

- Production camera QR scanning.
- Push notification delivery or notification preferences.
- Biometric login or biometric transaction signing.
- Live exchange-rate infrastructure.
- Phone/email/Lightning Address recipient resolution.
- New translations beyond the English localization foundation.
- Full NUT-13 ecash recovery.
- Broad redesign of the established visual language.

## Implementation Decomposition

The implementation plan will be divided into three independently reviewable phases while preserving one release gate:

1. Flutter wallet context, persistence, recovery, payment policy, canonical payment flows, and sync.
2. Rust/backend configuration, provider, routing, HTTP, deployment, and test hardening.
3. Honest capability UI, accessibility/localization, platform hardening, documentation, and final cross-system verification.

Each phase must keep its repository testable. A phase may not claim release readiness independently; release readiness requires all three phases and the final end-to-end matrix.
