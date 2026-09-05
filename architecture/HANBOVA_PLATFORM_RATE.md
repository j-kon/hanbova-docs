# Hanbova Platform Rate Architecture

## 1. Overview

**Hanbova Rate** represents the current customer-facing settlement and conversion rate that Hanbova can actually offer through its configured backend liquidity and payout provider.

Example customer-facing display:
```
Hanbova Rate                       ● Live
$1 = ₦1,365.00
USDT → NGN
Updated just now
```

---

## 2. Fundamental Distinctions

Hanbova Rate is strictly isolated from other financial and display systems in the application:

| System | Role | Authority / Source | Live Money Movement? |
| :--- | :--- | :--- | :--- |
| **Hanbova Rate** | Indicative platform settlement/conversion rate offered to customers | Configured provider (`PlatformRateProvider` / Bitnob) via backend service | **No** (Informational only) |
| **Display Currency Conversion** (`currency_provider.dart`) | Converts the user's authoritative balance into a familiar local fiat denomination for UI view | Client-side reference exchange rates | **No** (Display view only) |
| **Transaction Quote** | Fresh, locked-in provider quote obtained when money is about to move | Real-time quote endpoint (`/api/v1/payouts/quotes`, etc.) | **Yes** (Executable for 15 minutes) |
| **Generic Market FX** | Central Bank (e.g. CBN) or global forex benchmark rates | External market trackers | **No** (Benchmark only) |
| **Historical Transaction Rate** | Frozen snapshot of the effective rate at the moment a transaction was confirmed | Immutable transaction receipt/ledger record | **No** (Historical audit) |

---

## 3. Architecture & Data Flow

To ensure security, resilience, and theme independence, requests follow a strict unidirectional hierarchy:

```
Flutter Client (HanbovaRateCard / Riverpod)
    ↓
Hanbova API (GET /api/v1/rates/hanbova?market=NG)
    ↓
Hanbova Rate Service (45s TTL Cache + 10m Stale Window)
    ↓
PlatformRateProvider (BitnobRateProvider)
    ↓
Bitnob Trading / Payouts Quotes API (HMAC-SHA256)
```

### Security & Secret Isolation
- Bitnob API credentials (`BITNOB_CLIENT_ID`, `BITNOB_CLIENT_SECRET`) reside **exclusively on the backend**.
- Client requests are authenticated using standard HMAC-SHA256 (`CLIENT_ID:TIMESTAMP:NONCE:PAYLOAD`).
- Secrets and signatures are **never** logged, serialized into API payloads, or transmitted to Flutter.

---

## 4. Backend Provider Abstraction

The rate layer is provider-neutral, implemented via the `PlatformRateProvider` trait:

```rust
#[async_trait]
pub trait PlatformRateProvider: Send + Sync {
    async fn get_rate(
        &self,
        market: &str,
        settlement_asset: &str,
        target_currency: &str,
    ) -> ProviderResult<HanbovaRate>;

    fn provider_id(&self) -> &'static str;
}
```

### Provider Modes:
1. **`mock`**: Returns deterministic reference rates (e.g., 1,365.00 NGN per USD/USDT). Marked `is_live: false`, `is_stale: false`.
2. **`sandbox`**: Uses Bitnob Sandbox API (`https://sandboxapi.bitnob.co`) with configured credentials.
3. **`production`**: Queries live Bitnob Quotes API (`https://api.bitnob.co`) with full HMAC signing.
   - **Production Safety Rule**: If a production provider fails or is unconfigured, it **never** silently falls back to mock rates. It strictly returns an unavailable error (`503 Service Unavailable`).

---

## 5. Caching & Resilience Policies

To prevent thundering herds on provider APIs and widget rebuild storms:
- **Fresh Cache TTL**: **45 seconds** (within the 30–60s specification).
- **Stale Fallback Window**: **10 minutes (600s)**.
  - If the provider temporarily fails (network hiccup, upstream rate limit) while a rate is within this window, the backend returns the last known rate with:
    `"is_stale": true`, `"is_live": false`
  - The UI transparently presents this as `"Last updated 4m ago"` and `"● Stale"`.
- **Unavailable State**: If no trustworthy rate exists within the 10-minute window, the API returns:
  ```json
  HTTP 503
  {
    "error": "Rate temporarily unavailable",
    "code": "rate_unavailable"
  }
  ```
  The UI presents `"Rate temporarily unavailable"` with a retry action.

---

## 6. API Specification

### Endpoint
`GET /api/v1/rates/hanbova`

### Query Parameters
- `market`: Country ISO code (default: `"NG"`)
- `asset`: Settlement asset (default: `"USDT"`)
- `currency`: Target fiat quote currency (default: `"NGN"`)

### Sample Response (200 OK)
```json
{
  "market": "NG",
  "base": "USD",
  "quote": "NGN",
  "display": "$1 = ₦1,365.00",
  "settlement_asset": "USDT",
  "rate": 1365.00,
  "provider": "bitnob",
  "is_live": true,
  "is_stale": false,
  "updated_at": "2026-09-05T20:00:00Z",
  "expires_at": null
}
```

---

## 7. Flutter Domain & UI Component

### Domain Layer (`lib/core/rates/`)
- `hanbova_rate.dart`: Core domain model, status enum (`live`, `stale`, `loading`, `unavailable`, `demo`), and state wrapper.
- `hanbova_rate_service.dart`: Client communicating with `/rates/hanbova`.
- `hanbova_rate_provider.dart`: Riverpod `StateNotifierProvider` managing background polling (45s) and Demo mode integration.

### Reusable UI Widget (`HanbovaRateCard`)
- **Standard Card**:
  - Brand V4 compact card design with dark/light mode compatibility.
  - Header with icon and status pill.
  - Large rate readout (`$1 = ₦1,365.00`).
  - Footer with settlement corridor (`USDT → NGN`) and relative timestamp (`Updated just now`, `Last updated 4m ago`).
- **Inline Card (`HanbovaRateCard.inline()`)**:
  - Streamlined one-line version for confirmation sheets (`Airtime`, `Bills`, `eSIM`, `Payouts`).

### Screen Integrations
- **Home**: Integrated directly below Action Rail with pull-to-refresh.
- **Money**: Positioned above the Assets section.
- **Convert**: Embedded prominently alongside trade inputs.
- **Payment Confirmation Sheet**: Inline version displayed when settling in NGN.
