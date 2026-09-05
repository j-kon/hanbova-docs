# Backend payment coordination safety — 2026-09-05

Follow-up to the [mobile/backend audit](2026-09-05_PRODUCT_UX_BACKEND_AUDIT.md).

Subsequent results: [mint compatibility and mobile sync safety](2026-09-05_MINT_AND_SYNC_VALIDATION.md)
adds two mint scenarios, verifies all four against isolated Nutshell 0.20.3,
and documents the existing 0.16.5 environment's incompatibility. Counts below
remain the historical results from this coordination pass.

## Fixed

- Payment status validation and persistence now share a lock: a PostgreSQL
  row transaction or the in-memory repository write lock. The service no longer
  upserts a stale payment snapshot after authorization.
- Competing claim/refund reports cannot both replace the same active state.
  Terminal statuses cannot move backwards, missing records return an error,
  and repeating an accepted status preserves its update timestamp.
- PostgreSQL returns the stored update timestamp with `RETURNING`, so database
  precision rounding cannot cause different timestamps on the first response
  and an idempotent retry. A reduced-precision isolated fixture reproduces this
  on hosts whose clock otherwise matches PostgreSQL's normal precision.
- Protected-message acknowledgements use a conditional PostgreSQL update and
  matching in-memory rules. `acknowledged` cannot return to `delivered`;
  `claimed` and `refunded` cannot replace one another. Invalid reversals return
  HTTP 409 without changing the status or initial acknowledgement timestamp.
- The existing sender/recipient permissions remain enforced: only recipients
  report claims or receipt; only senders report refunds.
- When an encrypted message supplies a payment-intent ID, its sender, resolved
  recipient and protected payment type must match that intent. Unknown IDs,
  unrelated intents, wrong recipients and instant-payment links are rejected
  before an inbox/outbox entry is written. Canonical recipient names are stored.
- Optional unlinked envelopes remain supported for API compatibility. Linked
  intents support the UUID and case-insensitive handle identities already used
  by the application.
- CI now explicitly runs all three PostgreSQL repository regressions.

## Evidence

The original implementations failed tests for simultaneous terminal payment
updates, backwards message acknowledgements, and unauthorized message links.
A service-level regression deliberately synchronizes both reads before either
write. Temporarily restoring the old stale-save behavior made this test fail;
restoring the atomic implementation made it pass.

Verified commands:

- `cargo fmt --all --check`: passed.
- `cargo clippy --workspace --all-targets -- -D warnings`: passed.
- `cargo test --workspace --all-targets`: **56 passed, 5 ignored**.
- `cargo test -p hanbova-api postgres_ -- --ignored --nocapture`, using a
  disposable PostgreSQL 16 database: **3 passed**. These are three of the five
  tests excluded from the ordinary run; the other two require mint integration.
- **59 passing tests across the ordinary and explicit database runs.**
- `git diff --check`: passed.

Temporary local logs: `/tmp/hanbova-backend-coordination-tests.log`,
`/tmp/hanbova-backend-coordination-postgres.log` and
`/tmp/hanbova-coordination-mutation.log`.

Bounded read-only review found no critical or important issues. The one minor
timestamp precision finding was corrected and regression-tested.

## Limits and next validation

These endpoints record authenticated client reports; they do not verify an
encrypted token or independently prove settlement at the mint. The first
accepted coordination report is not evidence that its reporter won the actual
Cashu spend. Mint-authoritative reconciliation and native claim/refund race
testing remain release requirements. No change was made to Cashu locktime
semantics or to production's fail-closed mock-provider checks.

No schema migration, real payment, production deployment, or change to an
existing user-data database was performed. Mobile source was unchanged in this
follow-up. The existing `sqlx-postgres 0.7.4` future-Rust compatibility warning
remains.
