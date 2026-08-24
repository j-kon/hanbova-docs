# Hanbova Development Guide & Two-Device Testing

## 1. Prerequisites
- **Rust Toolchain**: `1.84+` (`cargo`, `clippy`, `rustfmt`)
- **Flutter SDK**: `3.29+` with iOS/Android build tools
- **Docker & Docker Compose**: For local PostgreSQL, Nutshell mint, and containerized API
- **Xcode / Android Studio**: For running simulators and physical device deployments

---

## 2. Quickstart: Running the Complete Stack

### Step 1: Start Containerized Infrastructure
```bash
cd hanbova-backend
docker compose up -d
```
This boots:
- **PostgreSQL 16**: Port `5434` (mapped to `5432` internally)
- **Nutshell Cashu Mint**: Port `3338` (serving NUT-00 through NUT-11 with fake wallet backend)
- **Hanbova API**: Port `8080` (zero-custody encrypted relay)

### Step 2: Build the CDK C-FFI Shared Library (for Desktop/Ffi targets)
```bash
cd hanbova-backend
cargo build --release -p hanbova-cdk-ffi
```

### Step 3: Run Backend Tests
```bash
cd hanbova-backend
cargo test --workspace --all-targets
```

### Step 4: Run Flutter Client Tests & Launch App
```bash
cd hanbova-app
flutter pub get
flutter test
flutter run -d "iPhone 17" # or your Android emulator / device
```

---

## 3. Automated Test Verification

| Repository | Test Command | Coverage |
| :--- | :--- | :--- |
| **`hanbova-backend`** | `cargo test --workspace` | **24/24 Passed** (Core invariants, Auth, CDK FFI, Protected Payments, Relay) |
| **`hanbova-backend`** | `cargo clippy --workspace --all-targets -- -D warnings` | **0 Warnings** |
| **`hanbova-app`** | `flutter test` | **42/42 Passed** (Secp256k1 P2PK, BIP-39, CDK FFI, E2EE, UI, Mainnet Safety) |
| **`hanbova-app`** | `flutter analyze` | **0 Issues** |

---

## 4. Two-Device Testing Procedure

### Scenario A: Successful Protected Payment Claim
1. **Device A (Alice)**:
   - Sign up as `@alice`.
   - Ensure network is set to **Cashu Test** (`https://testnut.cashu.space`) or **Local Development**.
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
