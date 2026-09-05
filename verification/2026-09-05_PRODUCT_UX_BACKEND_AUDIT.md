# Mobile UX and backend quality pass — 2026-09-05

## Scope and design benchmark

Continued the authorized mobile UI/UX and backend audit against Hanbova's V4
brand mapping and product principles: Poppins typography, Bitcoin orange for
actions, charcoal/graphite and warm-white surfaces, clear payment states, and
honest separation of sample data from wallet funds. This is a tested improvement
pass, not a certification of all screens, accessibility, or production readiness.

## Implemented

| Area | Confirmed problem | Change |
| --- | --- | --- |
| Home and Money | Duplicated balance presentation, inconsistent privacy, sample totals in live Money mode | Shared balance card; live wallet balance source; shared privacy setting including legacy preference migration |
| Balance privacy | Secondary asset subtitles and portfolio summaries exposed hidden amounts | Masked these amounts and shared the same setting with activity rows |
| Balance errors | Home crashed on an initial wallet error; lower Money sections misrepresented missing balances as zero | Guarded Home's async value access; consistent unavailable/loading states across wallet-derived amounts |
| Home | Dense balance section and actions; narrow-phone/large-text overflow | Calmer hierarchy, responsive actions and wrapping headings |
| Activity | Weak distinction between completed and uncertain payments; misleading conversion amount | Explicit status badges, uncertainty guidance, both conversion assets without an incoming-payment sign |
| Activity refresh | Screen-level API refresh duplicated the global coordinator | Reused the wallet-context-aware coordinator; saved-activity/retry notice and scrollable empty state |
| Navigation | Floating navigation overlapped the body | Scaffold bottom navigation reserves body space and labels the central Pay action |
| Cards | Real app theme forced infinite button width; narrow rows overflowed | Finite shared button minimum width; flexible card, balance and activity layouts; funding dialog regression checks |
| Pay market selector | Country list overflowed the modal bottom sheet on a phone | Safe-area-aware scrolling; regression selects the last supported country |
| API expiry | Extreme unsigned expiry values could overflow/panic | Checked numeric conversion, duration and timestamp arithmetic; rejected zero/invalid expiry |
| API configuration | Malformed CORS origins accepted before router construction | Validated origin scheme, host, authority, path/query and header safety at startup |
| API rate limiting | One client could consume the global request allowance | Bounded peer-IP windows using socket connection information, ignoring spoofed forwarding headers |
| Protected-message persistence | Existing metadata SELECT fix lacked real database coverage | Isolated PostgreSQL migration/round-trip regression for optional metadata and inbox/outbox isolation; CI database service and explicit test step |
| Product wording | Promised automatic expiry and 100% refunds | Documented refund eligibility, competing claim/refund paths, and possible mint fees |

## Verification

- `flutter analyze --no-pub`: no issues.
- `flutter test --no-pub --reporter expanded`: **315 passed, 1 skipped**.
- Added phone-size checks at 320 logical pixels and 2× text for Home, Money and
  Activity, normal-phone Cards funding checks with both real themes, and
  light/dark navigation rendering checks.
- Added initial wallet loading/error tests for Home and Money and a phone-size
  market selection interaction test. Final review identified the lower Money
  failure-state inconsistency; its regression also exposed Home's async error
  crash. Both were reproduced before fixing.
- Cards tests first reproduced `BoxConstraints forces an infinite width` under
  the application theme; both tests passed after the fix.
- `cargo fmt --all --check`: passed.
- `cargo clippy --workspace --all-targets -- -D warnings`: passed.
- `cargo test --workspace --all-targets`: **50 passed, 3 ignored**.
- Explicit `postgres_message_metadata_round_trips -- --ignored --nocapture`
  against a separate PostgreSQL 16 test container: **1 passed**. This accounts
  for one of the three tests ignored by the ordinary backend suite; the other
  two require a mint integration environment.
- Development Compose rendering and `scripts/validate-compose-config.sh`:
  passed with test-only environment settings.
- iOS simulator: Flutter built and launched on iPhone 17 / iOS 26.5; refreshed
  code with hot restart. Latest native capture shows the redesigned Home screen.
  This is not a claim that every journey was exercised natively.
- Final bounded read-only code review: no remaining critical or important
  findings after the balance-state correction.

Local evidence (temporary, not committed):

- `/tmp/hanbova-app-quality-final.log`
- `/tmp/hanbova-backend-quality-tests.log`
- `/tmp/hanbova-cards-red.log` and `/tmp/hanbova-cards-green.log`
- `/tmp/hanbova-redesign-ios.png`
- `/tmp/hanbova-{home,money,activity}-{light,dark}.png`

Widget captures can be regenerated with
`flutter test --no-pub test/product_visual_test.dart --dart-define=CAPTURE_PRODUCT_SCREENS=true`.
They load Poppins and Material Icons; platform fallback glyphs such as the naira
sign may differ from native iOS rendering.

## Remaining work / release limitations

1. Real provider integrations and their credentials/settlement verification are
   still required. Production stays fail-closed while providers are mocks.
   These checks do not authorize real-money operation.
2. Complete native end-to-end verification of claim/refund races, unreliable
   connectivity and account switching. Atomic status updates and
   protected-message/payment-intent ownership were addressed in the subsequent
   [backend coordination safety pass](2026-09-05_PAYMENT_COORDINATION_SAFETY.md).
   Mint-authoritative reconciliation remains outstanding.
3. Audit remaining screens and all amount surfaces for accessibility and
   privacy; Cards checks here cover normal phone size, not full Dynamic Type
   or VoiceOver conformance. Activity export remains a preview.
4. Live pending totals still need an authoritative aggregate. Home and Money
   now direct the user to check payment status instead of displaying a
   placeholder zero.
5. Rate limiting currently groups users behind the same reverse proxy by the
   proxy's socket address. Configure a trusted-proxy strategy before deploying
   behind one; do not simply trust arbitrary forwarded headers.
6. Existing dependency warnings remain: `sqlx-postgres 0.7.4` future Rust
   incompatibility and iOS plugin lifecycle/Swift Package Manager migration
   notices. No broad dependency upgrade was attempted in this pass.

No production deployment, real payment, push or database migration against a
user-data database was performed by this pass.
