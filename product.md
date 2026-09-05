# Hanbova Product Vision

**Tagline**: *Send protected.*

---

## 1. Problem Statement: Everyday Payments in Africa

Digital peer-to-peer and commerce payments across African markets face recurring challenges rooted in rigid, instant-finality rails:

1. **Typo and Wrong-Recipient Loss**: Sending funds to an incorrect phone number or address usually results in total loss with no self-service recovery mechanism.
2. **Informal Online Commerce Fraud**: Buyers are reluctant to prepay unknown online merchants on social platforms (e.g. Instagram, WhatsApp), while sellers are reluctant to ship goods without advance payment.
3. **Fake Payment Screenshots**: Merchants frequently face fraudulent transfer SMS receipts or manipulated screenshots before bank settlement clears.
4. **Dormant / Abandoned Transfers**: Senders initiate transfers to contacts who never claim or register, leaving funds locked in limbo.
5. **Lack of Conditional Settlement**: Everyday users lack access to simple, non-custodial escrow tools for services, freelance milestones, or deliveries.

---

## 2. The Solution: Dual Payment Tracks

Hanbova introduces a clear, two-track consumer mental model:

### A. Instant Send
* **Mechanism**: Direct Bitcoin / Lightning payment.
* **Settlement**: Immediate finality upon invoice payment.
* **Best for**: In-person merchants, trusted friends, bill payments, and low-value instant transactions.

### B. Protected Send
* **Mechanism**: Escrowed / timelocked payment (using Cashu P2PK and timelock primitives).
* **Settlement**: Conditional. Funds are locked safely until the recipient provides cryptographic claim proof.
* **Safety Features**:
  * **Protection Window**: A chosen locktime (e.g. 24h or 3 days) determines when the sender's refund path becomes available. It does not automatically cancel the recipient's claim path or return funds.
  * **Self-Service Refund**: After locktime, the sender can request a refund of unspent locked funds. The recipient may still claim until the mint accepts a spend; the first accepted claim or refund wins. Applicable mint fees may reduce the recovered amount.
  * **Proof of Commitment**: The recipient can independently verify that funds are locked before releasing goods.

---

## 3. Guiding Product Principles

* **Calm, Human-Centric Fintech**: Avoid intimidating cryptocurrency jargon, cliché orange styling, or complex blockchain parameters.
* **African-First Context**: Designed for low-bandwidth networks, intermittent connectivity, and mobile-first daily usage.
* **Non-Custodial Foundations**: Users retain custody and sovereignty over their keys; the platform acts as an enabler and protocol coordinator.
