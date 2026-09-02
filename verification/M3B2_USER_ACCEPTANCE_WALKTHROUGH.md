# Hanbova M3B.2 Product Experience User Acceptance Walkthrough

## Executive Summary

This document reports the comprehensive visual, UX, and product-flow validation of the **Hanbova M3B.2** consumer release on both **Android Emulator** (`sdk gphone64 arm64`) and **iOS Simulator** (`iPhone 17`) using deterministic mock/sandbox providers.

```text
============================================================
HANBOVA M3B.2 USER ACCEPTANCE AUDIT
============================================================
Target Devices: Android Emulator (API 37) & iOS Simulator (iOS 26.5)
Providers Mode: Deterministic Mock / Sandbox (Bitnob & DT One)
Mainnet Pilot: SUSPENDED (wallet_mainnet_pilot untouched)
Branch: milestone/3b2-consumer-product-experience
Status: COMPLETE — READY FOR HUMAN PRODUCT REVIEW
============================================================
```

---

## Evaluation Categories

- `PASS`: Requirement fully satisfied, verified on device, adheres to Brand V4 and security guidelines.
- `POLISH`: Minor visual styling, spacing, or animation enhancement recommended for future iterations.
- `MEDIUM`: Moderate UX friction with no financial or security impact.
- `HIGH`: Major UX degradation or confusing financial ambiguity.
- `BLOCKER`: Prevents core journey completion, causes crash/hang, or leaks sensitive secrets.

---

## Detailed Journey Audits

### Journey A: First-Time User Onboarding & Residence (`PASS`)
- **Flow**: `Welcome` $\rightarrow$ `Get Started` $\rightarrow$ `Country of Residence Picker` $\rightarrow$ `Account Creation` $\rightarrow$ `Wallet Setup` $\rightarrow$ `Biometric Enablement` $\rightarrow$ `12-Word Seed Backup & Verification Quiz` $\rightarrow$ `Home Dashboard`.
- **Copy & Jargon**: Clean, friendly, trust-building copy. Zero mentions of `CDK`, `NUT-04`, `P2PK`, or `proofs`.
- **Residence Context**: Selecting Nigeria (`NG`) or Kenya (`KE`) immediately sets `identityCountry` without being coupled to roaming/travel spend locations.
- **Recovery Security**: Words are masked by default, require explicit reveal, and pass through a 3-word verification quiz. Recovery words are never logged.
- **Keyboard & Navigation**: Clean text-input focus, smooth keyboard dismiss on scroll, reliable back navigation on both platforms.

### Journey B: Home Dashboard & Primary Actions (`PASS`)
- **Visual Hierarchy**:
  1. Primary Balance Card (Dominant Sats & Fiat conversion with Send/Receive/Scan).
  2. Quick Actions row (Spend & Bills, Travel Hub).
  3. Protected Payments summary banner (Active count & protected amount).
  4. Quick Claim Code banner.
  5. Recent Activity feed with direct link to unified timeline.
- **Brand V4**: Charcoal background (`#172027`), Bitcoin Orange (`#F7931A`), Lightning Gold (`#FFC400`), warm cards, sharp typography. Secondary features never overwhelm primary wallet functions.

### Journey C: Unified Activity & Filters (`PASS`)
- **Transaction Families**: Deterministic sample timeline populated with all 10 transaction families across 6 categories:
  - *Money In*: Bitcoin Received, Protected Claim, Protected Refund, Card Refund
  - *Money Out*: Bitcoin Sent, Bank Transfer Payout, Mobile Money Payout
  - *Protected*: Protected Payment (*"Waiting for recipient"*, *"Refund available"*)
  - *Bills*: Airtime, Mobile Data, Electricity, Water, TV, Internet
  - *Travel*: eSIM Purchase, eSIM Top-up
  - *Cards*: Card Funding, Card Payment
- **Filter Chips**: `All`, `Money In`, `Money Out`, `Protected`, `Bills`, `Travel`, `Cards`.
- **Search Query**: Real-time filtering matching counterparty, biller name, plan name, reference, and description.
- **Advanced Filters**: Status filter, destination country filter, date range, amount range.
- **CSV Export**: Standard RFC-4180 CSV export with rich consumer headers.

### Journey D: Contextual Transaction Details (`PASS`)
- **Context Relevance**: Detail screens dynamically adapt to the transaction type:
  - *Bitcoin*: Counterparty/invoice, network fee, status, timestamp.
  - *Protected*: Locktime expiry timestamp, masked claim code with reveal/copy actions, protection terms explanation card.
  - *Prepaid Electricity*: Dedicated prepaid meter token card with 1-tap copy button (`4819-2049-1829-4019-3918`).
  - *eSIM*: Plan name, ICCID receipt reference, travel country rail.
  - *Payout*: Payment rail (M-Pesa / Mobile Money vs Bank Transfer), destination phone/account, fee breakdown.
  - *Card*: Merchant, USD fiat value, card funding method.

### Journey E: Spend Hub & Utility Bill Payments (`PASS`)
- **Flow**: Category discovery $\rightarrow$ Biller selection $\rightarrow$ Account/Meter number input $\rightarrow$ Validation $\rightarrow$ Real-time quote $\rightarrow$ Confirmation sheet $\rightarrow$ Processing state $\rightarrow$ Adaptive receipt with 20-digit token $\rightarrow$ Save biller for future refill.
- **Uncertain Payment Notice**: Prominently displays *"Checking payment status — please do not attempt duplicate payment."*
- **Saved Billers**: Pay Again carousel allows 1-tap refills for electricity and airtime.

### Journey F: Travel Hub & Destination Market Switching (`PASS`)
- **Residence Permanence**: User begins with `identityCountry = NG`, `spendCountry = NG`.
- **Destination Switch**: Switching destination to Kenya (`KE`) displays the explicit modal notice:
  > *"Notice: Your country of residence will not change. Only the destination utilities, local eSIM packages, and spend corridors adapt."*
- **State Integrity**: `identityCountry` remains `NG`, `spendCountry` becomes `KE`, and `displayCurrency` syncs to `KES`.
- **Rail Adaptation**: Immediately loads Kenyan utility billers (KPLC Prepaid, Safaricom) and M-Pesa payout corridors.

### Journey G: eSIM Store & Roaming Management (`PASS`)
- **Package Selection**: Clean regional cards (e.g. East Africa 5GB, Kenya 3GB).
- **Subordination of Jargon**: Technical SM-DP+ and LPA strings are subordinate to primary "Install eSIM" (QR Code & Direct iOS/Android installation URLs) and "Top Up" actions.
- **My eSIM Tracker**: Live remaining data progress bar (`4.2 GB / 5.0 GB remaining`), validity countdown, and 1-tap top-up modal.

### Journey H: Financial Edge & Failure States (`PASS`)
- **States Verified**: `loading`, `pending`, `processing`, `completed`, `failed`, `cancelled`, `refunded`, `payment uncertain`, `offline`, `unsupported`, `provider unavailable`.
- **Clean Communication**: Failed quotes or network timeouts display user-actionable instructions without raw exception traces.

### Journey I: Profile, Security & Recovery Audit (`PASS`)
- **Hub Overview**: Displays permanent Residence origin alongside active travel destination market.
- **Recovery Phrase Protection**:
  - Accessing the 12-word recovery phrase requires explicit re-authentication (device biometrics / PIN).
  - Words are blurred/masked by default (`••••••••`) until the user explicitly taps reveal.
  - Zero recovery phrases logged to console, disk, or analytics.

---

## Findings Summary

| Category | Count | Items |
| :--- | :---: | :--- |
| **BLOCKER** | **0** | None |
| **HIGH PRIORITY** | **0** | None |
| **MEDIUM** | **0** | None |
| **POLISH** | **0** | None |
| **PASS** | **12** | Journeys A–I, Visual Quality, Android Parity, iOS Parity |

---

## Platform Consistency

- **Android Emulator (`API 37`)**: Smooth Material 3 scrolling, bottom sheets respect system navigation bars, zero layout overflows.
- **iOS Simulator (`iOS 26.5`)**: SafeArea handling for Dynamic Island and home indicator, smooth Cupertino-style modal sheets and text inputs.
