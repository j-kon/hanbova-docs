# Hanbova Multi-Asset & Provider Architecture

This document formalizes the frontend and product architecture for Hanbova M3B.2.2 Multi-Asset Wallet, Stablecoin & Conversion UX Extension.

> [!IMPORTANT]
> **Architectural Intent Only**:
> This document describes intended future roles for backend and third-party infrastructure.
> Hanbova makes **no claim of live provider connectivity** at this milestone.
> All client code binds strictly to normalized domain abstractions (`AssetBalance`, `ConversionQuote`, `ConversionPair`), avoiding direct UI coupling to specific vendors.

---

## 1. Intended Provider Roles

### Bitnob
- **Core Role**: Digital asset custody and conversion infrastructure where legally approved.
- **Intended Capabilities**:
  - Bitcoin on-chain and Lightning liquidity orchestration.
  - Regulated stablecoin custodial wallets (USDT, USDC).
  - Conversion quoting and execution engine for crypto-to-crypto pairs:
    - BTC ↔ USDT
    - BTC ↔ USDC
    - USDT ↔ USDC
  - Future debit card settlement and local fiat off-ramp settlement rails.

### Flutterwave
- **Core Role**: African fiat payment rails and local banking integration.
- **Intended Capabilities**:
  - Local currency pay-ins and collections (Bank Transfer, Cards, USSD).
  - Cross-border merchant settlement and payout routing (e.g. NGN, KES, GHS, RWF).
  - Fiat ↔ Stablecoin liquidity bridges where regulatory compliance permits.
  - Stablecoin network transfers to African enterprise endpoints.

### DT One
- **Core Role**: Digital everyday services and telecommunications gateway.
- **Intended Capabilities**:
  - Mobile airtime top-ups across Africa.
  - Mobile data bundle activations.
  - Prepaid electricity token generation (e.g. KPLC, AEDC, EKEDC).
  - Water and TV subscription bill payment (e.g. DStv, GOtv, Startimes).
  - International roaming eSIM provisioning.

---

## 2. Prevention of Duplicate Provider Balances

In Hanbova's consumer wallet model, balances are presented strictly by **Asset**, not by provider:
- **Bitcoin**: Unified Cashu e-cash and Lightning balance.
- **USDT**: Single normalized Tether balance.
- **USDC**: Single normalized USD Coin balance.

Consumer interfaces must **never** expose fragmented or competing provider balances such as:
- ❌ `Bitnob USDT Balance: $500`
- ❌ `Flutterwave USDT Balance: $750`

Instead, Hanbova's future backend service will act as an authoritative balance aggregator and ledger. The client UI renders exactly one consolidated, verified balance per asset.

---

## 3. Truthfulness & Non-Demo Guardrails

1. **Normal Mode**:
   - Bitcoin displays genuine synced Cashu/Lightning balance.
   - USDT and USDC display zero balance with clear feature lifecycle state badges (`Coming soon`, `Setup required`, `Verification required`).
   - Normal mode **never fabricates** stablecoin balances.
2. **Demo Mode**:
   - Provides deterministic sample balances:
     - Bitcoin: `1,800,000 sats`
     - USDT: `$1,250.00`
     - USDC: `$750.00`
   - Every demo screen and component prominently bears:
     `DEMO MODE • SAMPLE DATA • NO REAL MONEY`.
3. **Protected Send**:
   - Strictly reserved for **Bitcoin / Cashu** locktime escrow.
   - Protected stablecoins are disallowed without a cryptographically verified escrow protocol.
