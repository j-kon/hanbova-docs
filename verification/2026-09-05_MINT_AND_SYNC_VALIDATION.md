# Mint compatibility and mobile sync safety — 2026-09-05

Follow-up to [payment coordination safety](2026-09-05_PAYMENT_COORDINATION_SAFETY.md).

## Mobile changes

- In-flight wallet sync can complete or fail after coordinator disposal without
  notifying a disposed listener. Account context is rechecked before incoming
  payment notifications.
- Inbox/outbox errors and malformed responses propagate instead of being
  reported as a successful empty inbox. Existing reconciliation can show stale
  activity and retry.
- Unlinked envelopes do not prevent processing later messages. Missing or
  invalid intent details cannot fabricate a zero-value transaction or expiry.
- Redelivery preserves local settlement/recovery progress. Backend intent
  refreshes preserve already-settled local amounts, fees, and receipt metadata.
- A new post-lookup check preserves transactions inserted while an inbox intent
  lookup is awaiting its response. A deterministic Completer test reproduced
  the overwrite before this guard and passed after it.
- An expired protection window does not by itself discard a recipient envelope:
  locktime enables a refund path, not automatic settlement or token revocation.

## Verification

These results describe the working tree after the final mobile race fix and
backend test-fixture changes; they are not a production release certification.

| Check | Result |
| --- | --- |
| Flutter analyzer | No issues |
| Full Flutter suite | 330 passed, 1 skipped |
| Focused sync/relay regressions | 15 passed |
| iOS simulator debug build | Succeeded after the final race fix |
| Rust formatting and Clippy | Passed |
| Rust workspace tests | 56 passed, 7 ignored |
| Explicit mint tests on existing Nutshell 0.16.5 | 3 passed, 1 failed |
| Explicit mint tests on isolated Nutshell 0.20.3 | 4 passed |
| Same mint tests using the new Compose fixture | 4 passed |

The seven ignored workspace tests are three PostgreSQL checks and four mint
checks. PostgreSQL checks passed in the preceding coordination pass; they were
not rerun here. CI explicitly runs both groups. GitHub-hosted CI itself has not
been run by this session. Local source checks and tests do not establish native
end-to-end claim/refund behavior.

The earlier execution-approval usage limit was cleared on the scoped retry;
the final race test was observed failing before implementation, then passing.
The rebuilt iOS app was not installed or launched in this pass.

Local temporary evidence logs:

- `/tmp/hanbova-mobile-sync-verified.log`
- `/tmp/hanbova-sync-final-ios-build.log`
- `/tmp/hanbova-backend-sync-verified.log`
- `/tmp/hanbova-local-mint-integration.log`
- `/tmp/hanbova-post-locktime-diagnostic.log`
- `/tmp/hanbova-mint-0203-compatibility.log`
- `/tmp/hanbova-mint-compose-verified.log`

## Mint compatibility finding

The existing development container uses Nutshell 0.16.5 with FakeWallet. Its
installed signature validator switches to refund keys only after locktime.
The recipient-after-locktime test failed consistently with a signature error.
The other three scenarios passed, but its concurrent-spend result alone cannot
prove both spending paths were eligible.

Current [NUT-11](https://github.com/cashubtc/nuts/blob/main/11.md#refund-multisig)
retains the original claim path and adds a refund path after locktime. Inspection
of [Nutshell 0.20.3's validator](https://github.com/cashubtc/nutshell/blob/0.20.3/cashu/mint/conditions.py)
shows both paths; isolated integration tests verified:

1. Correct-key claims succeed; wrong-key claims and early refunds fail.
2. The sender can refund after locktime.
3. The recipient can still claim after locktime; a subsequent refund fails.
4. Concurrent claim/refund attempts produce one successful mint spend with
   consistent wallet balances.

The new `hanbova-backend/docker-compose.mint-test.yml` pins Nutshell 0.20.3 by
multi-architecture image digest. It uses a public test-only key, FakeWallet,
zero swap fees, localhost:3339, and no mounted volumes. Tests accept a local
port via `HANBOVA_TEST_MINT_PORT`; arbitrary external mint URLs are not accepted
by this helper. The caller must still ensure the selected local mint is a
valueless test fixture. Setup and teardown commands are in the backend README.
CI waits for health, runs all four mint scenarios, and tears down the fixture
even when a preceding step fails.

## Remaining release work

- **Existing environment migration:** The original development mint remains on
  0.16.5 and is incompatible with recipient claims after locktime. This pass
  does not upgrade it. Back up its database and key, validate migration and
  rollback on a copy, then explicitly plan the pin change. The upstream
  [0.20.3 release notes](https://github.com/cashubtc/nutshell/releases/tag/0.20.3)
  warn of database migrations.
- **Mint-authoritative reconciliation:** Backend status endpoints still record
  authenticated client reports; they do not independently prove mint settlement.
- **Native integration:** Run claim/refund, account/environment-switch and
  reconnect journeys on the iOS simulator against a compatible environment.
- **Dependency warnings:** `flutter_secure_storage` lacks Swift Package Manager
  support in the installed version; `sqlx-postgres 0.7.4` has a future Rust
  compatibility warning. Neither prevented this pass's builds/checks.

No real payment, production deployment, existing database migration, commit,
or push was performed. Disposable mint fixtures contain no real value and are
removed after verification; existing application containers/data are preserved.
