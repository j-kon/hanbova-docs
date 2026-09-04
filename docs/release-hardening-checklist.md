# Hanbova Release-Hardening Checklist

**Scope:** controlled test release and tightly allowlisted pilot only  
**Release posture:** mainnet remains disabled until every required gate below is verified  
**Owner:** release integrator  
**Last reviewed:** 2026-09-04

This checklist is the release gate for the four Hanbova repositories. A checked item must have a reproducible command, test, or manual evidence record. “Implemented” is not the same as “release verified.”

## Integration state

- [ ] App hardening commits are reviewed and integrated onto the current app milestone branch.
- [ ] Backend hardening commits are rebased onto the current backend branch and reviewed for configuration/API compatibility.
- [ ] Documentation and verification counts below are updated from the same clean-checkout revision.
- [ ] No unrelated or uncommitted source changes are included in the release candidate.

## Flutter client

### Required automated checks

```bash
cd hanbova-app
dart format --set-exit-if-changed lib test
flutter analyze
flutter test
```

- [ ] Formatting passes without changing files.
- [ ] `flutter analyze` reports zero issues.
- [ ] Full Flutter test suite passes.
- [ ] Wallet policy tests cover deposit, send, projected-balance, mint, and network/build limits.
- [ ] Wallet context tests prove user, identity, network, mint, and storage-prefix isolation.
- [ ] Ledger tests cover persistence, upsert, restart, refresh, malformed records, and stale/offline reconciliation.
- [ ] Receive tests prove debouncing, stale-response rejection, expiry handling, and idempotent mint execution.
- [ ] Instant-send tests prove quote review, fee/total display, final confirmation, and amountful invoice handling.
- [ ] Protected-send tests prove recipient review, policy revalidation, delivery-pending retry, and refund recovery.
- [ ] Error tests prove that user messages are sanitized and that secrets, tokens, invoices, and JWTs never enter logs or UI errors.

### Manual client checks

- [ ] Test and pilot network badges are visible on Home and payment screens.
- [ ] Mainnet cannot be enabled through runtime state, navigation, or developer UI in an ordinary build.
- [ ] Recovery phrase reveal requires successful OS authentication and fails closed when authentication is unavailable or cancelled.
- [ ] Restore requires an authenticated account and rebuilds the matching wallet context before routing to Home.
- [ ] Activity remains visible after refresh/restart and exposes stale/offline status when reconciliation fails.
- [ ] Unsupported scanner, push notification, biometric-login, live-rate, and recipient capabilities are labeled honestly.
- [ ] Payment and recovery screens work at 1.3x and 2.0x text scale on a 360x640 viewport.
- [ ] Actionable controls have at least 48dp targets, semantic labels, keyboard/focus behavior, and status that does not rely on color alone.

## Rust backend

### Required automated checks

```bash
cd hanbova-backend
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

- [ ] Formatting, clippy, and workspace tests pass in a clean checkout.
- [ ] Ordinary workspace tests do not require PostgreSQL, Docker, a local mint, or undeclared network services.
- [ ] Configuration tests reject missing production database, insecure/default JWT secret, HTTP mint URLs, and mock providers.
- [ ] Production startup fails before binding when database connection or migration fails.
- [ ] Production cannot construct an in-memory repository or mock financial provider.
- [ ] Development/test defaults require an explicit non-production environment.
- [ ] Production router does not mount mock Lightning routes; development/test routes still require authentication.
- [ ] Financial and user-specific routes enforce ownership and return `401` for missing credentials and `403` for insufficient permission.
- [ ] CORS uses an explicit production origin allowlist; wildcard CORS is development-only.
- [ ] Rate limits and narrow input limits cover authentication, password reset, protected messaging, and payment coordination.
- [ ] API errors expose stable sanitized codes while internal details remain in structured logs.

### Explicit integration check

- [ ] Run the CDK/mint integration runner only with its declared mint fixture and record the fixture revision, command, and result.
- [ ] Confirm test mint credentials and URLs are never used by production configuration.

## Deployment and secrets

- [ ] `HANBOVA_ENV=production` is explicit in the deployment environment.
- [ ] `DATABASE_URL`, API host/port, HTTPS `MINT_URL`, and provider mode are injected by deployment configuration.
- [ ] JWT secret is generated outside the repository, is at least 32 bytes, and is not a known/default value.
- [ ] Docker Compose and `.env.example` contain placeholders only; no deployable passwords, signing keys, or JWT secrets are committed.
- [ ] Container-to-container addresses use service names, and the API binds the intended container interface.
- [ ] Logs and crash reporting are checked for mnemonics, private keys, Cashu tokens, refund keys, invoices, and JWTs.
- [ ] Release artifacts are built from clean, pinned revisions of app, backend, protocol, and docs.

## Android and iOS platform security

- [ ] Android backup/data-extraction rules exclude wallet and recovery material as appropriate.
- [ ] Android enables `FLAG_SECURE` while recovery material is visible and clears it afterward.
- [ ] iOS obscures recovery material while the app is inactive or screen capture/recording is active.
- [ ] Release builds use production signing configuration and contain no development network override.
- [ ] Camera permissions are absent when camera scanning is not implemented.

## Evidence record

Record the exact revision and output for every release candidate:

| Check | Revision | Result/date | Evidence |
| --- | --- | --- | --- |
| Flutter format/analyze/test |  |  |  |
| Rust fmt/clippy/test |  |  |  |
| CDK/mint integration |  |  |  |
| Android manual matrix |  |  |  |
| iOS manual matrix |  |  |  |
| Deployment/config review |  |  |  |

The release is **not ready** while any required item is unchecked, while source branches are unreconciled, or while a test result cannot be reproduced from a clean checkout.
