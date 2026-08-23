# Hanbova Development Guide & Two-Device Testing

## 1. Prerequisites
- **Rust toolchain**: `1.75+` (`cargo`, `clippy`, `rustfmt`)
- **Flutter SDK**: `3.22+` with iOS/Android toolchains
- **Docker & Docker Compose**: for local PostgreSQL and Nutshell mint

---

## 2. Starting Local Services

```bash
# 1. Start PostgreSQL and Nutshell Mint
cd hanbova-backend
docker compose up -d

# 2. Run Backend API
cargo run --bin hanbova-api
```

---

## 3. Two-Device Testing Procedure (Milestone 3A)

### Scenario A: Successful Protected Payment Claim
1. **Device A (Alice)**:
   - Sign up as `@alice`.
   - Open **Developer Options** -> Ensure network is set to **Cashu Test** (`https://testnut.cashu.space`) or **Local Development**.
   - Navigate to **Pay** -> **Send Protected**.
   - Enter `@bob`, Amount `100` test sats, Protection Window `60 seconds`.
   - Confirm and send.
2. **Device B (Bob)**:
   - Sign up as `@bob`.
   - Open **Protected** -> **Incoming** tab.
   - See incoming `100 test sats` from `@alice`.
   - Tap **Claim**.
   - Observe Bob's spendable balance increase by `+100 test sats`.
3. **Device A (Alice)**:
   - Status updates to **Claimed by @bob**.

---

### Scenario B: Self-Service Sender Refund after Locktime
1. **Device A (Alice)**:
   - Send `100 test sats` protected to `@bob` with `30s` or `60s` locktime.
2. **Device B (Bob)**:
   - Does nothing / does not claim.
3. **Device A (Alice)**:
   - Watch live countdown in **Protected -> Active**.
   - When locktime passes, action changes to **Refund available**.
   - Tap **Refund**.
   - Funds are returned to Alice's spendable balance (`+100 test sats`).
4. **Device B (Bob)**:
   - Payment status updates to **Refunded to Alice**. Cannot be claimed.
