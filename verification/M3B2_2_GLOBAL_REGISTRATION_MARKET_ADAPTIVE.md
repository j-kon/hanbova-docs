# Milestone M3B.2.2 Architecture Correction: Global Registration & Market-Adaptive Experience

## Overview
This architectural correction rectifies the geographical and product architecture of Hanbova M3B.2.2.
It establishes a true global registration experience, decouples user home residence from operational market and roaming state, and enforces capability-driven UI rendering across the application.

## Negative Constraints Strictly Preserved
- **Do not merge**: All changes are committed solely to feature branch `milestone/3b2-2-financial-interaction-redesign`.
- **Do not integrate DT One**: No DT One APIs or credentials added.
- **Do not integrate Bitnob**: No Bitnob APIs or credentials added.
- **Do not implement KYC**: KYC status modeled as clean enum placeholder without mock identity verification flows.
- **Do not enable Mainnet**: Valueless test environment and demo mode strictly preserved.

---

## 1. Global Registration & Country Selection
- **Global ISO Dataset**: Complete 247 ISO 3166-1 alpha-2 countries and territories in `CountryInfo.allCountries`.
- **Search Capability**: Case-insensitive substring search matching country name, 2-letter ISO code, and dial code.
- **Consumer Language**: "Country of residence" (no "Citizenship", "Nationality", or "Supported country").
- **Stepped Onboarding Order**:
  1. Welcome
  2. First Name + Last Name + Username
  3. Email + Password
  4. Country of Residence (Global search list)
  5. Country Explanation Confirmation
  6. Wallet Setup → Security → Recovery → Home

---

## 2. Market Capability Matrix
- **Global Baseline**:
  - Bitcoin (Lightning & on-chain)
  - Send & Receive
  - Scan QR & Request Money
  - Protected Payments (Escrow, locktime, claim/refund)
  - Private Cashu e-cash
  - Activity & Balances
  - Security, Recovery, and Profile
  - Stablecoins explicitly marked **"Coming soon"** with zero fake balances
- **Supported Local Markets (NG, KE, GH, RW, UG, TZ, ZA)**:
  - Exposes domestic bill payments (Airtime, Data, Electricity, TV, Internet, Water) based strictly on each market's verified capabilities.
  - Exposes local payout rails (Bank payouts, Mobile Money) only where supported.
- **Unsupported / Global Markets (e.g. US, Senegal, UK, etc.)**:
  - Automatically hides African biller grids and domestic payout buttons.
  - Replaces them with global action rails and an invitation card to **"Activate Roam to use supported local services"**.

---

## 3. Normalized Country State & Roam Mode
- **State Model (`UserCountryContext`)**:
  - `residenceCountry`: Immutable user home country set at registration.
  - `activeMarket`: Current operational market (equals `residenceCountry` when Roam is off).
  - `displayCurrency`: Active display currency (e.g., NGN, USD, KES, GHS).
  - `roamEnabled`: Boolean flag indicating whether Roam is active.
- **Roam Behavior**:
  - Activating Roam to a supported destination (e.g., Kenya `KE`):
    - `residenceCountry` remains unchanged.
    - `activeMarket` switches to destination.
    - `displayCurrency` switches to destination currency.
    - Destination market capabilities become active.
  - Deactivating Roam restores `activeMarket = residenceCountry` and display currency to residence currency.

---

## 4. UI Adaptivity & Verified Surfaces

| UI Surface | Global Market (e.g. US, SN) | Supported Market (e.g. NG, KE) | Roaming User (e.g. US in KE) |
|---|---|---|---|
| **Home Action Rail** | Send, Receive, Protected, Scan, Request, More | Send, Receive, Protected, Scan, Request, Airtime, More | Send, Receive, Protected, Scan, Request, Airtime, More |
| **Home Quick Pay** | **Hidden** (no everyday bills) | Displays local supported bill icons | Displays destination bill icons |
| **Pay Hub** | Send (BTC, Protected), Wallet (BTC, Cashu, Stablecoin Coming Soon), Roam Banner | Send Money (BTC, Protected, Bank, MoMo), Pay Again, Everyday Bills | Destination everyday bills & local payout rails |
| **Profile Screen** | Displays Residence (`US`), Active Market (`US`), Roam (`Off`) | Displays Residence (`NG`), Active Market (`NG`), Roam (`Off`) | Displays Residence (`US`), Active Market (`KE (Roam)`), Roam (`Active`) |

---

## 5. Demo Personas
Integrated into Developer Options (`/developer-options`):
- **Persona A**: Nigeria Residence (`NG`), Roam Off, Full Nigerian capability matrix.
- **Persona B**: United States Residence (`US`), Roam Off, Clean global baseline (no biller grids, USD currency).
- **Persona C**: United States Residence (`US`), Roam On to Kenya (`KE`), Kenyan capability matrix (KES currency, M-Pesa, Kenyan bills).

---

## 6. Verification Quality Gates
- **Unit & Widget Tests**: **213/213 passed** (`flutter test --exclude-tags=live-network`), including all 16 architecture tests in `test/global_registration_market_adaptive_test.dart`.
- **Static Analysis**: **0 issues found** via `flutter analyze`.
- **Formatting**: Clean (`dart format --output=none --set-exit-if-changed .`).
- **iOS Simulator Execution**: Verified on iPhone 17 simulator (`EAE647A5-91D1-46C7-8855-1902F9F2E421`).
