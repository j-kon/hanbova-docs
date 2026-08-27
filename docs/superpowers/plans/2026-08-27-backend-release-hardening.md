# Backend and Deployment Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Rust API and container configuration fail closed, production-safe, authenticated, abuse-resistant, and testable without undeclared local services.

**Architecture:** Fallible typed configuration determines environment capabilities before any socket is bound. Production requires PostgreSQL and real providers; development/test may select in-memory and mock implementations explicitly. HTTP policies are centralized in middleware and validation helpers, while external-mint tests move behind an explicit integration feature.

**Tech Stack:** Rust 2021, Axum 0.7, Tokio, SQLx/PostgreSQL, Tower/Tower HTTP, Docker Compose, Cargo test/clippy/fmt.

**Spec:** `hanbova-docs/docs/superpowers/specs/2026-08-27-release-hardening-design.md`

## Global Constraints

- Production never uses a default JWT secret, in-memory repository, mock financial provider, wildcard CORS, or HTTP mint URL.
- Development/test conveniences require an explicit non-production environment.
- Server-side `/lightning/*` routes remain development/test-only in this release and still require authentication.
- API error bodies use stable public codes and never expose database, provider, token, or stack details.
- Ordinary `cargo test --workspace` runs without PostgreSQL, Docker, or a local mint.
- Every behavioral production change starts with a failing focused test.

---

### Task 1: Fallible typed configuration

**Files:**
- Modify: `hanbova-backend/services/api/src/config.rs`
- Modify: `hanbova-backend/services/api/src/main.rs`
- Modify: `hanbova-backend/services/api/src/handlers/health.rs`
- Modify: `hanbova-backend/services/api/src/state.rs`
- Test: inline `#[cfg(test)]` module in `hanbova-backend/services/api/src/config.rs`

**Interfaces:**
- Produces: `Environment`, `ProviderMode`, `AppConfig::from_iter`, `AppConfig::from_env() -> Result<AppConfig, ConfigError>`, and `AppConfig::validate`.
- Consumes: process environment only in `from_env`; tests use deterministic key/value iterators.

- [ ] **Step 1: Write failing production configuration tests**

```rust
#[test]
fn production_rejects_missing_database_and_default_secret() {
    let vars = [
        ("HANBOVA_ENV", "production"),
        ("HANBOVA_API_HOST", "0.0.0.0"),
        ("HANBOVA_API_PORT", "8080"),
        ("JWT_SECRET", "short"),
        ("MINT_URL", "https://mint.example.com"),
        ("PROVIDER_MODE", "production"),
    ];
    let error = AppConfig::from_iter(vars).unwrap_err();
    assert!(error.to_string().contains("DATABASE_URL"));
    assert!(error.to_string().contains("JWT_SECRET"));
}

#[test]
fn production_rejects_http_mint_and_mock_provider() {
    let error = AppConfig::from_iter(valid_production_vars()
        .into_iter()
        .chain([("MINT_URL", "http://mint.example.com"), ("PROVIDER_MODE", "mock")]))
        .unwrap_err();
    assert!(error.to_string().contains("HTTPS"));
    assert!(error.to_string().contains("mock"));
}
```

- [ ] **Step 2: Run configuration tests and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api config::tests -- --nocapture`

Expected: compilation fails because the typed environment/provider APIs do not exist and `from_env` is infallible.

- [ ] **Step 3: Implement configuration types and aggregate validation**

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Environment { Development, Test, Production }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ProviderMode { Mock, Production }

#[derive(Debug, thiserror::Error)]
#[error("invalid configuration: {problems}")]
pub struct ConfigError { pub problems: String }

impl AppConfig {
    pub fn from_env() -> Result<Self, ConfigError> {
        Self::from_iter(std::env::vars())
    }

    pub fn from_iter<I, K, V>(vars: I) -> Result<Self, ConfigError>
    where
        I: IntoIterator<Item = (K, V)>, K: Into<String>, V: Into<String>,
    {
        // Build a HashMap, parse exact names, collect every validation problem,
        // and return one ConfigError when any production invariant fails.
    }
}
```

Implement `Display` for `Environment` as lowercase `development`, `test`, or `production`, and provide `is_development`/`is_production` helpers. Development defaults remain `127.0.0.1:8080`, in-memory repository, local HTTP mint, and mock providers only when `HANBOVA_ENV=development` or `test` is explicit. Remove the implicit environment default. Update `main`, health/version handlers, state construction, and every test fixture to use the typed fields and `AppConfig::from_iter` rather than struct literals.

- [ ] **Step 4: Run focused configuration tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api config::tests`

Expected: PASS for complete production vars and explicit development/test configurations; every missing/insecure production case fails.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/config.rs services/api/src/main.rs services/api/src/handlers/health.rs services/api/src/state.rs
git -C hanbova-backend commit -m "fix: validate production configuration"
```

### Task 2: Fail-closed database startup and state construction

**Files:**
- Create: `hanbova-backend/services/api/src/startup.rs`
- Modify: `hanbova-backend/services/api/src/main.rs`
- Modify: `hanbova-backend/services/api/src/state.rs`
- Test: inline tests in `hanbova-backend/services/api/src/startup.rs`

**Interfaces:**
- Consumes: validated `AppConfig`.
- Produces: `connect_database(&AppConfig) -> anyhow::Result<Option<PgPool>>` and `AppState::try_new(config, pool) -> anyhow::Result<AppState>`.

- [ ] **Step 1: Write failing startup-policy tests**

```rust
#[tokio::test]
async fn production_refuses_missing_pool() {
    let config = production_config();
    let error = AppState::try_new(config, None).unwrap_err();
    assert!(error.to_string().contains("PostgreSQL"));
}

#[tokio::test]
async fn development_allows_in_memory_state() {
    let state = AppState::try_new(development_config(), None).unwrap();
    assert!(state.db_pool.is_none());
}
```

- [ ] **Step 2: Run startup tests and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api startup::tests`

Run: `cd hanbova-backend && cargo test -p hanbova-api state::tests`

Expected: `try_new` and startup module do not exist.

- [ ] **Step 3: Implement environment-dependent startup**

```rust
pub async fn connect_database(config: &AppConfig) -> anyhow::Result<Option<PgPool>> {
    let Some(url) = config.database_url.as_deref() else {
        anyhow::ensure!(!config.is_production(), "PostgreSQL is required in production");
        return Ok(None);
    };
    let pool = PgPoolOptions::new().max_connections(10)
        .acquire_timeout(Duration::from_secs(5)).connect(url).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;
    Ok(Some(pool))
}
```

`main` propagates every production connection/migration error and exits before binding. `AppState::try_new` rejects missing production pool and any mock provider in production. Preserve `AppState::new_for_test` as an explicit test constructor.

- [ ] **Step 4: Run startup and existing API tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api`

Expected: PASS; development test app still uses explicit in-memory state.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/startup.rs services/api/src/main.rs services/api/src/state.rs
git -C hanbova-backend commit -m "fix: fail closed during API startup"
```

### Task 3: Remove mock financial routes from production and authenticate development routes

**Files:**
- Modify: `hanbova-backend/services/api/src/routes/mod.rs`
- Modify: `hanbova-backend/services/api/src/routes/lightning.rs`
- Modify: `hanbova-backend/services/api/src/auth/handlers.rs`
- Modify: `hanbova-backend/services/api/src/main.rs`
- Test: inline API tests in `hanbova-backend/services/api/src/main.rs`

**Interfaces:**
- Consumes: `AppConfig.environment`, `AuthUser` extractor.
- Produces: `create_api_router(&AppConfig)` which excludes `/lightning/*` in production; authenticated handlers in development/test.

- [ ] **Step 1: Write failing route-policy tests**

```rust
#[tokio::test]
async fn development_lightning_route_requires_authentication() {
    let response = setup_test_app().oneshot(Request::builder()
        .method("POST").uri("/api/v1/lightning/invoice")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"amount_sats":1000}"#)).unwrap()).await.unwrap();
    assert_eq!(response.status(), StatusCode::UNAUTHORIZED);
}

#[tokio::test]
async fn production_router_does_not_mount_lightning_routes() {
    let response = setup_production_router_without_startup().oneshot(Request::builder()
        .method("POST").uri("/api/v1/lightning/invoice")
        .body(Body::empty()).unwrap()).await.unwrap();
    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}
```

- [ ] **Step 2: Run route tests and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api development_lightning_route_requires_authentication`

Run: `cd hanbova-backend && cargo test -p hanbova-api production_router_does_not_mount_lightning_routes`

Expected: development route accepts unauthenticated requests and production router mounts it.

- [ ] **Step 3: Gate routes and normalize authentication rejection**

Change every Lightning handler signature to include `_auth_user: AuthUser`. Make `AuthUser::from_request_parts` map missing/malformed/expired tokens to `ApiError::Unauthorized`, producing HTTP 401. Build routes conditionally:

```rust
pub fn create_api_router(config: &AppConfig) -> Router<AppState> {
    let router = Router::new()
        .merge(health::router())
        .merge(auth::router())
        .merge(protected_messages::router())
        .nest("/payment-intents", payment_intents::router());
    if config.is_production() { router } else { router.merge(lightning::router()) }
}
```

- [ ] **Step 4: Run route and auth lifecycle tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api`

Expected: PASS; invalid login remains non-enumerating, while bearer-auth failures are 401.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/routes services/api/src/auth/handlers.rs services/api/src/main.rs
git -C hanbova-backend commit -m "fix: isolate and authenticate mock financial routes"
```

### Task 4: Input constraints and sanitized errors

**Files:**
- Create: `hanbova-backend/services/api/src/validation.rs`
- Modify: `hanbova-backend/services/api/src/error.rs`
- Modify: `hanbova-backend/services/api/src/auth/service.rs`
- Modify: `hanbova-backend/services/api/src/routes/protected_messages.rs`
- Modify: `hanbova-backend/services/api/src/models/payment_intent_dto.rs`
- Modify: `hanbova-backend/services/api/src/models/protected_message_dto.rs`
- Test: inline tests in `validation.rs` and API tests in `main.rs`

**Interfaces:**
- Produces: `validate_username`, `validate_email`, `validate_description`, `validate_ciphertext`, `PublicErrorCode`.
- Constraints: username 3–32 lowercase ASCII letters/digits/underscore; email 3–254 chars with one `@`; descriptions ≤500 chars; invoice ≤20,000 chars; encrypted payload ≤131,072 chars.

- [ ] **Step 1: Write failing validation/error tests**

```rust
#[test]
fn username_rejects_spaces_unicode_and_overlength() {
    assert!(validate_username("bad name").is_err());
    assert!(validate_username("amína").is_err());
    assert!(validate_username(&"a".repeat(33)).is_err());
    assert_eq!(validate_username("Alice_7").unwrap(), "alice_7");
}

#[tokio::test]
async fn oversized_protected_message_returns_sanitized_400() {
    let response = authenticated_post("/api/v1/protected-messages", json!({
        "ciphertext": "x".repeat(131_073)
    })).await;
    assert_eq!(response.status(), StatusCode::BAD_REQUEST);
    assert_eq!(json_body(response).await["error"], "FIELD_TOO_LONG");
}
```

- [ ] **Step 2: Run validation tests and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api validation::tests`

Run: `cd hanbova-backend && cargo test -p hanbova-api oversized_protected_message_returns_sanitized_400`

Expected: validation module is missing and oversized fields reach repositories.

- [ ] **Step 3: Implement exact validation and public error mapping**

```rust
pub fn validate_username(raw: &str) -> Result<String, ApiError> {
    let value = raw.trim().trim_start_matches('@').to_ascii_lowercase();
    let valid = (3..=32).contains(&value.len())
        && value.bytes().all(|b| b.is_ascii_lowercase() || b.is_ascii_digit() || b == b'_');
    valid.then_some(value).ok_or_else(|| ApiError::bad_request(
        "INVALID_USERNAME", "Username must be 3–32 letters, numbers, or underscores."))
}
```

Add code-bearing `ApiError::BadRequest { code, message }`. Log internal causes with tracing, but return only stable code/message. Apply validators before service/repository calls. Login failures remain the same generic public message.

- [ ] **Step 4: Run API and domain tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/validation.rs services/api/src/error.rs services/api/src/auth/service.rs services/api/src/routes/protected_messages.rs services/api/src/models services/api/src/main.rs
git -C hanbova-backend commit -m "fix: constrain API input and sanitize errors"
```

### Task 5: CORS allowlist and endpoint rate limiting

**Files:**
- Create: `hanbova-backend/services/api/src/middleware/rate_limit.rs`
- Modify: `hanbova-backend/services/api/src/middleware/mod.rs`
- Modify: `hanbova-backend/services/api/src/routes/auth.rs`
- Modify: `hanbova-backend/services/api/src/routes/protected_messages.rs`
- Modify: `hanbova-backend/services/api/src/main.rs`
- Test: inline tests in `middleware/rate_limit.rs` and API tests in `main.rs`

**Interfaces:**
- Produces: `cors_layer(&AppConfig)`, `RateLimiter::allow(key, bucket, now)`, and Axum middleware returning 429.
- Limits per IP: register/login/forgot/reset 10 requests per 60 seconds; profile lookup and protected-message creation 60 per 60 seconds; development health/version remain unlimited.

- [ ] **Step 1: Write failing CORS and limiter tests**

```rust
#[test]
fn limiter_blocks_the_eleventh_auth_request_in_window() {
    let limiter = RateLimiter::default();
    let now = Instant::now();
    for _ in 0..10 { assert!(limiter.allow("127.0.0.1", Bucket::Auth, now)); }
    assert!(!limiter.allow("127.0.0.1", Bucket::Auth, now));
    assert!(limiter.allow("127.0.0.1", Bucket::Auth, now + Duration::from_secs(61)));
}

#[tokio::test]
async fn production_cors_rejects_unlisted_origin() {
    let response = production_test_app().oneshot(Request::builder()
        .method("OPTIONS").uri("/api/v1/health")
        .header("origin", "https://attacker.example")
        .header("access-control-request-method", "GET")
        .body(Body::empty()).unwrap()).await.unwrap();
    assert!(response.headers().get("access-control-allow-origin").is_none());
}
```

- [ ] **Step 2: Run middleware tests and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api middleware::rate_limit::tests`

Run: `cd hanbova-backend && cargo test -p hanbova-api production_cors_rejects_unlisted_origin`

Expected: limiter is missing and production CORS returns wildcard allowance.

- [ ] **Step 3: Implement dependency-free fixed-window limiting and configured CORS**

Use `Arc<Mutex<HashMap<(String, Bucket), Window>>>`, prune expired entries on access, and derive the key from `ConnectInfo<SocketAddr>`. Return `ApiError::TooManyRequests` with `Retry-After: 60`. Attach buckets to the exact route groups.

Parse `CORS_ALLOWED_ORIGINS` as a comma-separated list of `HeaderValue`s. `development` may call `.allow_origin(Any)`; test/production uses `AllowOrigin::list(config.cors_allowed_origins.clone())` and startup rejects an empty production list.

- [ ] **Step 4: Run middleware and API tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api`

Expected: PASS, including deterministic limiter tests that inject `Instant` instead of sleeping.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/middleware services/api/src/routes/auth.rs services/api/src/routes/protected_messages.rs services/api/src/main.rs services/api/src/config.rs services/api/src/error.rs
git -C hanbova-backend commit -m "fix: restrict API origins and request rates"
```

### Task 6: Honest production password reset behavior

**Files:**
- Modify: `hanbova-backend/services/api/src/auth/service.rs`
- Modify: `hanbova-backend/services/api/src/auth/models.rs`
- Modify: `hanbova-backend/services/api/src/auth/handlers.rs`
- Test: API tests in `hanbova-backend/services/api/src/main.rs`

**Interfaces:**
- Produces: production `PASSWORD_RESET_UNAVAILABLE` response; development/test keeps a reset token only in the explicit `dev_reset_token` field.

- [ ] **Step 1: Write failing production reset test**

```rust
#[tokio::test]
async fn production_reset_is_honestly_unavailable_without_delivery_provider() {
    let response = production_test_app().oneshot(post_json(
        "/api/v1/auth/forgot-password", json!({"email":"alice@example.com"}))).await.unwrap();
    assert_eq!(response.status(), StatusCode::SERVICE_UNAVAILABLE);
    let body = json_body(response).await;
    assert_eq!(body["error"], "PASSWORD_RESET_UNAVAILABLE");
    assert!(body.get("dev_reset_token").is_none());
}
```

- [ ] **Step 2: Run reset test and verify RED**

Run: `cd hanbova-backend && cargo test -p hanbova-api production_reset_is_honestly_unavailable_without_delivery_provider`

Expected: current service returns a success message claiming a reset link was issued.

- [ ] **Step 3: Implement environment-honest behavior**

Before creating a token in production, return `ApiError::ServiceUnavailable { code: "PASSWORD_RESET_UNAVAILABLE", message: "Password reset delivery is not available. Contact support." }`. Keep non-enumerating development behavior and the dev token only when `Environment != Production`.

- [ ] **Step 4: Run auth lifecycle tests**

Run: `cd hanbova-backend && cargo test -p hanbova-api test_auth_full_lifecycle`

Run: `cd hanbova-backend && cargo test -p hanbova-api production_reset_is_honestly_unavailable_without_delivery_provider`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add services/api/src/auth services/api/src/error.rs services/api/src/main.rs
git -C hanbova-backend commit -m "fix: report unavailable password reset delivery"
```

### Task 7: Correct container configuration and secret handling

**Files:**
- Modify: `hanbova-backend/docker-compose.yml`
- Modify: `hanbova-backend/.env.example`
- Create: `hanbova-backend/scripts/validate-compose-config.sh`
- Modify: `hanbova-backend/README.md`

**Interfaces:**
- Consumes: exact configuration names from Task 1.
- Produces: development Compose environment with `HANBOVA_ENV`, `HANBOVA_API_HOST`, `HANBOVA_API_PORT`, `MINT_URL`, `PROVIDER_MODE`, `CORS_ALLOWED_ORIGINS`, and externally supplied `JWT_SECRET`.

- [ ] **Step 1: Write a failing Compose validation script**

```bash
#!/usr/bin/env bash
set -euo pipefail
compose_file="${1:-docker-compose.yml}"
required=(HANBOVA_ENV HANBOVA_API_HOST HANBOVA_API_PORT DATABASE_URL JWT_SECRET MINT_URL PROVIDER_MODE CORS_ALLOWED_ORIGINS)
for name in "${required[@]}"; do
  grep -q -- "${name}=" "$compose_file" || { echo "missing $name" >&2; exit 1; }
done
grep -q -- 'HOST=' "$compose_file" && { echo 'legacy HOST variable present' >&2; exit 1; }
grep -q -- 'CASHU_MINT_URL=' "$compose_file" && { echo 'legacy CASHU_MINT_URL present' >&2; exit 1; }
grep -Eq 'JWT_SECRET=[^$]' "$compose_file" && { echo 'hardcoded JWT secret present' >&2; exit 1; }
```

- [ ] **Step 2: Run the script and verify RED**

Run: `cd hanbova-backend && bash scripts/validate-compose-config.sh`

Expected: FAIL for missing exact names and hardcoded JWT secret.

- [ ] **Step 3: Correct Compose and examples**

Set:

```yaml
environment:
  HANBOVA_ENV: development
  HANBOVA_API_HOST: 0.0.0.0
  HANBOVA_API_PORT: 8080
  DATABASE_URL: postgres://${POSTGRES_USER:-hanbova}:${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-hanbova}
  JWT_SECRET: ${JWT_SECRET:?set JWT_SECRET}
  MINT_URL: http://cashu-mint:3338
  PROVIDER_MODE: mock
  CORS_ALLOWED_ORIGINS: ${CORS_ALLOWED_ORIGINS:-http://localhost:3000}
```

Update `.env.example` with a clearly non-production development secret instruction rather than a usable committed secret. Document `docker compose --env-file .env up --build`.

- [ ] **Step 4: Run validation and render Compose**

Run: `cd hanbova-backend && POSTGRES_PASSWORD=test-only JWT_SECRET=test-only-secret-at-least-32-bytes docker compose config >/tmp/hanbova-compose.yml && bash scripts/validate-compose-config.sh /tmp/hanbova-compose.yml`

Expected: PASS and rendered API target uses `cashu-mint:3338` plus `0.0.0.0:8080`.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add docker-compose.yml .env.example scripts/validate-compose-config.sh README.md
git -C hanbova-backend commit -m "fix: align container configuration and secrets"
```

### Task 8: Hermetic default tests and explicit CDK integration runner

**Files:**
- Modify: `hanbova-backend/crates/hanbova-protected-payments/Cargo.toml`
- Modify: `hanbova-backend/crates/hanbova-protected-payments/src/lib.rs`
- Create: `hanbova-backend/scripts/run-cdk-integration.sh`
- Modify: `hanbova-backend/README.md`

**Interfaces:**
- Produces: Cargo feature `integration-cdk`; default tests exclude external mint; explicit script verifies port 3338 before running integration tests.

- [ ] **Step 1: Record the failing clean-workspace test**

Run: `cd hanbova-backend && cargo test --workspace`

Expected before implementation: FAIL in both `cdk_test` scenarios with connection refused when no mint is running.

- [ ] **Step 2: Add the explicit feature gate**

```toml
[features]
default = []
integration-cdk = []
```

```rust
#[cfg(all(test, feature = "integration-cdk"))]
pub mod cdk_test;
```

Create `scripts/run-cdk-integration.sh` that runs `curl --fail http://127.0.0.1:3338/v1/info` and exits with “start the development mint with docker compose up cashu-mint” when unavailable; otherwise run `cargo test -p hanbova-protected-payments --features integration-cdk cdk_test -- --nocapture`.

- [ ] **Step 3: Run default tests and verify GREEN**

Run: `cd hanbova-backend && cargo test --workspace`

Expected: PASS without a local mint.

- [ ] **Step 4: Verify explicit integration runner behavior**

Run without mint: `cd hanbova-backend && bash scripts/run-cdk-integration.sh`

Expected: deterministic non-zero exit with the documented startup instruction, not a Rust panic.

Run with mint: `cd hanbova-backend && docker compose up -d cashu-mint && bash scripts/run-cdk-integration.sh`

Expected: both CDK scenarios pass.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-backend add crates/hanbova-protected-payments/Cargo.toml crates/hanbova-protected-payments/src/lib.rs scripts/run-cdk-integration.sh README.md
git -C hanbova-backend commit -m "test: separate external CDK integration suite"
```

### Task 9: Backend phase verification

**Files:**
- Verify: all Rust, Compose, shell, and documentation files modified by Tasks 1–8.

**Interfaces:**
- Consumes: Tasks 1–8.
- Produces: formatted, lint-clean, hermetic backend phase.

- [ ] **Step 1: Check Rust formatting**

Run: `cd hanbova-backend && cargo fmt --all -- --check`

Expected: PASS. If it fails, run `cargo fmt --all`, inspect the diff, then rerun the check.

- [ ] **Step 2: Run strict clippy**

Run: `cd hanbova-backend && cargo clippy --workspace --all-targets -- -D warnings`

Expected: PASS. Record the existing `sqlx-postgres 0.7.4` future-incompatibility separately; do not suppress it.

- [ ] **Step 3: Run hermetic workspace tests**

Run: `cd hanbova-backend && cargo test --workspace`

Expected: PASS with no external services.

- [ ] **Step 4: Validate deployment files**

Run: `cd hanbova-backend && POSTGRES_PASSWORD=test-only JWT_SECRET=test-only-secret-at-least-32-bytes docker compose config >/tmp/hanbova-compose.yml && bash scripts/validate-compose-config.sh /tmp/hanbova-compose.yml`

Expected: PASS.

- [ ] **Step 5: Check repository scope**

Run: `git -C hanbova-backend status --short && git -C hanbova-backend diff --check`

Expected: no generated/untracked artifacts and no whitespace errors.
