# Flutter Wallet and Security Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every Flutter wallet operation context-safe, policy-enforced, durable across restart, explicitly confirmed, and recoverable after partial failure.

**Architecture:** A ready-only `WalletContext` binds the authenticated user, network configuration, mint, storage prefix, and matching identity. A pure `WalletPolicy` guards every service entry point, while a secure versioned transaction ledger and one wallet-scoped sync coordinator replace ephemeral activity and per-screen polling.

**Tech Stack:** Flutter 3/Dart 3, Riverpod 2.5, GoRouter 14, FlutterSecureStorage 9, CDK FFI, flutter_test.

**Spec:** `hanbova-docs/docs/superpowers/specs/2026-08-27-release-hardening-design.md`

## Global Constraints

- Preserve client-side CDK ownership, deterministic key derivation, encrypted envelopes, and sender-held refund keys.
- Preserve the existing `wallet_mainnet_pilot` storage prefix and Minibits pilot mint allowlist.
- The ordinary production build cannot enable mainnet through runtime state or navigation.
- No value-moving operation runs before an authenticated wallet context is ready.
- Every behavioral production change starts with a failing focused test.
- Do not overwrite the user's existing Claim-button layout change; integrate it during the accessibility plan.
- Do not log mnemonic, private key, Cashu token, refund key, invoice, or JWT contents.

---

### Task 1: Central wallet policy

**Files:**
- Create: `hanbova-app/lib/core/cashu/wallet_policy.dart`
- Modify: `hanbova-app/lib/core/cashu/cashu_wallet_service.dart`
- Test: `hanbova-app/test/wallet_policy_test.dart`

**Interfaces:**
- Consumes: `NetworkConfig`, `CashuWalletBalance`.
- Produces: `WalletPolicy.validateDeposit`, `validateSend`, `validateMint`, and `WalletPolicyViolation`.

- [ ] **Step 1: Write failing policy tests**

```dart
test('pilot rejects deposits above the deposit cap', () {
  expect(
    () => WalletPolicy(NetworkConfig.mainnetPilot).validateDeposit(
      amountSats: 10001,
      currentBalanceSats: 0,
    ),
    throwsA(isA<WalletPolicyViolation>()),
  );
});

test('pilot rejects deposits that exceed projected wallet balance', () {
  expect(
    () => WalletPolicy(NetworkConfig.mainnetPilot).validateDeposit(
      amountSats: 1001,
      currentBalanceSats: 9000,
    ),
    throwsA(isA<WalletPolicyViolation>()),
  );
});

test('pilot rejects sends above 5000 sats', () {
  expect(
    () => WalletPolicy(NetworkConfig.mainnetPilot)
        .validateSend(amountSats: 5001),
    throwsA(isA<WalletPolicyViolation>()),
  );
});
```

- [ ] **Step 2: Run the policy test and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_policy_test.dart`

Expected: compilation fails because `wallet_policy.dart` and `WalletPolicy` do not exist.

- [ ] **Step 3: Implement the pure policy**

```dart
final class WalletPolicyViolation implements Exception {
  final String code;
  final String message;
  const WalletPolicyViolation(this.code, this.message);
  @override
  String toString() => message;
}

final class WalletPolicy {
  final NetworkConfig config;
  const WalletPolicy(this.config);

  void validateDeposit({required int amountSats, required int currentBalanceSats}) {
    if (amountSats <= 0) {
      throw const WalletPolicyViolation('invalid_amount', 'Amount must be greater than zero.');
    }
    if (amountSats > config.maxDepositSats) {
      throw WalletPolicyViolation('deposit_limit',
          'Maximum deposit is ${config.maxDepositSats} sats.');
    }
    if (currentBalanceSats + amountSats > config.maxWalletBalanceSats) {
      throw WalletPolicyViolation('wallet_limit',
          'This deposit would exceed the ${config.maxWalletBalanceSats} sat wallet limit.');
    }
  }

  void validateSend({required int amountSats}) {
    if (amountSats <= 0) {
      throw const WalletPolicyViolation('invalid_amount', 'Amount must be greater than zero.');
    }
    if (amountSats > config.maxSendSats) {
      throw WalletPolicyViolation('send_limit',
          'Maximum send is ${config.maxSendSats} sats.');
    }
  }
}
```

Inject `WalletPolicy` and a balance callback into `CdkCashuWalletServiceImpl`. Call `validateDeposit` before `walletMintQuote`, and `validateSend` before melt/protected-send execution. Keep UI validation as feedback only.

- [ ] **Step 4: Run focused and existing wallet authority tests**

Run: `cd hanbova-app && flutter test test/wallet_policy_test.dart test/client_wallet_authority_test.dart test/mainnet_safety_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/cashu/wallet_policy.dart lib/core/cashu/cashu_wallet_service.dart test/wallet_policy_test.dart test/client_wallet_authority_test.dart
git -C hanbova-app commit -m "fix: enforce wallet policy at service boundary"
```

### Task 2: Immutable ready-only wallet context

**Files:**
- Create: `hanbova-app/lib/core/wallet/wallet_context.dart`
- Create: `hanbova-app/lib/core/wallet/wallet_context_provider.dart`
- Modify: `hanbova-app/lib/core/cashu/cashu_wallet_provider.dart`
- Modify: `hanbova-app/lib/core/network/network_environment.dart`
- Modify: `hanbova-app/lib/core/crypto/crypto_identity_service.dart`
- Modify: `hanbova-app/lib/features/profile/screens/developer_options_screen.dart`
- Modify: `hanbova-app/lib/app/router.dart`
- Test: `hanbova-app/test/wallet_context_test.dart`
- Test: `hanbova-app/test/mainnet_safety_test.dart`

**Interfaces:**
- Consumes: `authProvider`, `cryptoIdentityProvider`, `activeNetworkConfigProvider`, `selectedMintUrlProvider`.
- Produces: immutable `WalletContext`, `AsyncNotifierProvider<WalletContextController, WalletContext?> walletContextProvider`, and a CDK service that consumes only a ready context.

- [ ] **Step 1: Write failing context-isolation tests**

```dart
test('WalletContext rejects an identity from another network', () {
  expect(
    () => WalletContext.checked(
      userId: 'alice',
      config: NetworkConfig.cashuTest,
      identity: mainnetIdentity,
      mintUrl: NetworkConfig.cashuTest.defaultMintUrl,
    ),
    throwsStateError,
  );
});

test('ordinary builds cannot apply a runtime mainnet override', () async {
  final notifier = NetworkEnvironmentNotifier(storage: fakeStorage);
  await notifier.setNetwork(HanbovaNetwork.mainnet);
  expect(notifier.state, isNot(HanbovaNetwork.mainnet));
});
```

Add `network` and `storagePrefix` fields to the identity fixture so the mismatch is observable.

- [ ] **Step 2: Run the context and mainnet tests and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_context_test.dart test/mainnet_safety_test.dart`

Expected: compilation fails because `WalletContext` does not exist; the runtime override behavior test fails against the current notifier API.

- [ ] **Step 3: Implement context construction and transitions**

```dart
@immutable
final class WalletContext {
  final String userId;
  final NetworkConfig config;
  final String mintUrl;
  final WalletCryptoIdentity identity;

  String get storageKey => '${userId}_${config.storagePrefix}';

  WalletContext.checked({
    required this.userId,
    required this.config,
    required this.mintUrl,
    required this.identity,
  }) {
    if (identity.network != config.network ||
        identity.storagePrefix != config.storagePrefix) {
      throw StateError('Wallet identity does not match active network.');
    }
    if (config.isPilot && mintUrl != config.defaultMintUrl) {
      throw StateError('Pilot mint is not allowlisted.');
    }
  }
}
```

Implement `WalletContextController.build()` to await authenticated user and matching identity. Remove `pilotOverride` from `setNetwork` and remove `mainnetPilotOverrideProvider`; derive mainnet availability only from `NetworkConfig.isMainnetPilotBuild`. Guard `/developer-options` in router redirect and in `DeveloperOptionsScreen` itself.

Refactor `cashuWalletServiceProvider` to watch `walletContextProvider`; return `null` while loading/failing and construct `CdkCashuWalletServiceImpl` only from the ready context.

- [ ] **Step 4: Run context, crypto, network, and wallet tests**

Run: `cd hanbova-app && flutter test test/wallet_context_test.dart test/network_environment_test.dart test/mainnet_safety_test.dart test/crypto_test.dart test/client_wallet_authority_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/wallet lib/core/cashu/cashu_wallet_provider.dart lib/core/network/network_environment.dart lib/core/crypto/crypto_identity_service.dart lib/features/profile/screens/developer_options_screen.dart lib/app/router.dart test/wallet_context_test.dart test/network_environment_test.dart test/mainnet_safety_test.dart
git -C hanbova-app commit -m "fix: bind wallet services to immutable context"
```

### Task 3: Versioned durable transaction ledger

**Files:**
- Modify: `hanbova-app/lib/features/transactions/domain/transaction_model.dart`
- Create: `hanbova-app/lib/features/transactions/data/transaction_ledger.dart`
- Create: `hanbova-app/lib/features/transactions/data/secure_transaction_ledger.dart`
- Modify: `hanbova-app/lib/features/transactions/presentation/transactions_provider.dart`
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/protected/presentation/protected_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_provider.dart`
- Test: `hanbova-app/test/transaction_ledger_test.dart`

**Interfaces:**
- Consumes: `WalletContext.storageKey`, `FlutterSecureStorage`.
- Produces: `TransactionModel.toJson/fromJson`, `TransactionLedger.load/upsert/replace`, `TransactionsState(items, isSyncing, isStale, syncMessage)`.

- [ ] **Step 1: Write failing serialization and persistence tests**

```dart
test('transaction round-trips every recovery field', () {
  final decoded = TransactionModel.fromJson(transaction.toJson());
  expect(decoded.id, transaction.id);
  expect(decoded.coordinationSyncPending, isTrue);
  expect(decoded.syncPendingStatus, 'delivery_pending');
  expect(decoded.expiresAt, transaction.expiresAt);
});

test('ledger upserts IDs, keeps newest first, and caps at 500', () async {
  final ledger = SecureTransactionLedger(storage: memoryStorage);
  for (var i = 0; i < 501; i++) {
    await ledger.upsert('alice_wallet_cashu_test', tx(id: 'tx_$i', minute: i));
  }
  await ledger.upsert('alice_wallet_cashu_test', tx(id: 'tx_500', status: TransactionStatus.completed));
  final records = await ledger.load('alice_wallet_cashu_test');
  expect(records, hasLength(500));
  expect(records.first.id, 'tx_500');
  expect(records.first.status, TransactionStatus.completed);
});

test('refresh marks stale on sync failure without deleting activity', () async {
  final notifier = TransactionsNotifier(ledger, 'alice_wallet_cashu_test');
  await notifier.load();
  await notifier.reconcile(sync: () async => throw const SocketException('offline'));
  expect(notifier.state.items, isNotEmpty);
  expect(notifier.state.isStale, isTrue);
});
```

- [ ] **Step 2: Run the ledger test and verify RED**

Run: `cd hanbova-app && flutter test test/transaction_ledger_test.dart`

Expected: compilation fails because serialization, ledger, and `TransactionsState` do not exist.

- [ ] **Step 3: Implement the ledger**

```dart
abstract interface class TransactionLedger {
  Future<List<TransactionModel>> load(String walletKey);
  Future<void> upsert(String walletKey, TransactionModel transaction);
  Future<void> replace(String walletKey, List<TransactionModel> transactions);
}

final class SecureTransactionLedger implements TransactionLedger {
  static const schemaVersion = 1;
  static const maxEntries = 500;
  final FlutterSecureStorage storage;
  const SecureTransactionLedger({required this.storage});

  String _key(String walletKey) => 'hanbova_ledger_v1_$walletKey';
  // Decode {"version":1,"transactions":[...]}; quarantine malformed JSON
  // to `hanbova_ledger_corrupt_<timestamp>_$walletKey`, then return [].
}
```

Make notifier mutations asynchronous and write-through. Update all transaction mutation call sites in Home, Protected, Send, Receive, and Protected Send to await persistence before reporting success. Replace Home's `ref.invalidate(transactionsProvider)` with `await notifier.reconcile(...)`. Keep ledger items visible during sync failure and expose a stale/offline banner.

- [ ] **Step 4: Run ledger, activity export, and Home tests**

Run: `cd hanbova-app && flutter test test/transaction_ledger_test.dart test/activity_export_test.dart test/widget_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/transactions lib/features/home/presentation/home_screen.dart lib/features/protected/presentation/protected_screen.dart lib/features/send/presentation/send_screen.dart lib/features/receive/presentation/receive_screen.dart lib/features/protected_send/presentation/protected_send_provider.dart test/transaction_ledger_test.dart test/activity_export_test.dart test/widget_test.dart
git -C hanbova-app commit -m "fix: persist and reconcile wallet activity"
```

### Task 4: Fail-closed seed reveal, durable backup status, authenticated restore

**Files:**
- Modify: `hanbova-app/lib/core/security/biometric_service.dart`
- Create: `hanbova-app/lib/core/security/wallet_backup_repository.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Modify: `hanbova-app/lib/features/security/presentation/restore_seed_screen.dart`
- Modify: `hanbova-app/lib/app/router.dart`
- Test: `hanbova-app/test/wallet_security_test.dart`
- Test: `hanbova-app/test/auth_widget_test.dart`

**Interfaces:**
- Consumes: ready `WalletContext`, `LocalAuthentication`, `FlutterSecureStorage`.
- Produces: `WalletBackupRepository.isAcknowledged/setAcknowledged`, fail-closed `BiometricService.authenticate`, authenticated restore routing.

- [ ] **Step 1: Write failing recovery tests**

```dart
test('authentication unavailable never authorizes seed reveal', () async {
  final service = BiometricService(auth: FakeLocalAuth(available: false));
  expect(await service.authenticate(reason: 'Reveal phrase'), isFalse);
});

test('backup acknowledgement is isolated by wallet key', () async {
  final repo = WalletBackupRepository(memoryStorage);
  await repo.setAcknowledged('alice_wallet_cashu_test', true);
  expect(await repo.isAcknowledged('alice_wallet_cashu_test'), isTrue);
  expect(await repo.isAcknowledged('alice_wallet_mainnet_pilot'), isFalse);
});

testWidgets('restore requires authentication before accepting mnemonic', (tester) async {
  await tester.pumpWidget(unauthenticatedRestoreApp());
  expect(find.text('Sign in to restore your wallet'), findsOneWidget);
  expect(find.byType(TextField), findsNothing);
});
```

- [ ] **Step 2: Run security tests and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_security_test.dart test/auth_widget_test.dart`

Expected: unavailable authentication returns true; backup status is memory-only; unauthenticated restore shows the phrase form.

- [ ] **Step 3: Implement fail-closed security behavior**

Change the unavailable branch to `return false`. Replace `walletBackupStatusProvider` with a context-keyed async repository/provider. Remove random-mnemonic fallback from Backup. If context is not ready, show a blocking “Wallet unavailable—sign in again” state.

Classify `/restore-seed` as an authenticated route. Welcome's Restore action stores only an intended post-auth route, sends the user to login, and opens Restore after authentication. Remove `default_user`; require `walletContext.userId`, rebuild context after restore, and route Home only after readiness.

- [ ] **Step 4: Run security, auth, and crypto tests**

Run: `cd hanbova-app && flutter test test/wallet_security_test.dart test/auth_widget_test.dart test/bip39_test.dart test/crypto_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/security lib/features/security/presentation lib/app/router.dart test/wallet_security_test.dart test/auth_widget_test.dart
git -C hanbova-app commit -m "fix: fail closed around wallet recovery"
```

### Task 5: Wallet-scoped synchronization and pilot retry recovery

**Files:**
- Create: `hanbova-app/lib/core/sync/wallet_sync_coordinator.dart`
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/protected/presentation/protected_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_provider.dart`
- Modify: `hanbova-app/lib/core/cashu/cashu_wallet_storage.dart`
- Test: `hanbova-app/test/wallet_sync_test.dart`
- Test: `hanbova-app/test/protected_delivery_recovery_test.dart`

**Interfaces:**
- Consumes: ready wallet context, protected inbox/message service, intent repository, ledger, escrow storage.
- Produces: `WalletSyncCoordinator.syncNow`, `onAppLifecycleChanged`, `WalletSyncState`; retry always uses `context.config.storagePrefix`.

- [ ] **Step 1: Write failing concurrency and storage-prefix tests**

```dart
test('coordinator never overlaps sync requests', () async {
  final gate = Completer<void>();
  var calls = 0;
  final coordinator = WalletSyncCoordinator(runSync: () async {
    calls++;
    await gate.future;
  });
  final first = coordinator.syncNow();
  final second = coordinator.syncNow();
  expect(calls, 1);
  gate.complete();
  await Future.wait([first, second]);
});

test('pilot retry reads escrow from wallet_mainnet_pilot', () async {
  await notifier.retryDelivery('payment_1');
  expect(storage.lastRequestedPrefix, 'wallet_mainnet_pilot');
});
```

- [ ] **Step 2: Run recovery tests and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_sync_test.dart test/protected_delivery_recovery_test.dart`

Expected: coordinator type is missing; retry requests escrow without a prefix.

- [ ] **Step 3: Implement one coordinator and context-exact retry**

```dart
final class WalletSyncCoordinator {
  final Future<void> Function() runSync;
  Future<void>? _inFlight;
  Timer? _timer;
  Duration _interval = const Duration(seconds: 15);

  Future<void> syncNow() => _inFlight ??= runSync().whenComplete(() => _inFlight = null);
  void markFailure() => _interval = const Duration(seconds: 60);
  void markSuccess() => _interval = const Duration(seconds: 15);
  void dispose() => _timer?.cancel();
}
```

Move key publishing/inbox reconciliation out of both screens and into a wallet-context provider that observes app lifecycle. In retry, read `walletContextProvider.requireValue` and pass `storagePrefix: context.config.storagePrefix` to `getEscrowRecord`; use `context.mintUrl` in the envelope.

- [ ] **Step 4: Run focused recovery and navigation tests**

Run: `cd hanbova-app && flutter test test/wallet_sync_test.dart test/protected_delivery_recovery_test.dart test/widget_test.dart`

Expected: PASS with no two-second screen timers remaining.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/sync lib/features/home/presentation/home_screen.dart lib/features/protected/presentation/protected_screen.dart lib/features/protected_send/presentation/protected_send_provider.dart lib/core/cashu/cashu_wallet_storage.dart test/wallet_sync_test.dart test/protected_delivery_recovery_test.dart
git -C hanbova-app commit -m "fix: centralize wallet sync and recovery"
```

### Task 6: Canonical receive flow with race-safe quotes

**Files:**
- Create: `hanbova-app/lib/features/receive/domain/deposit_controller.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/wallet/presentation/unified_deposit_sheet.dart`
- Modify: `hanbova-app/lib/app/router.dart`
- Test: `hanbova-app/test/deposit_flow_test.dart`

**Interfaces:**
- Consumes: wallet context, wallet service, wallet policy, balance provider.
- Produces: `DepositState` and `DepositController.setAmount/createQuote/checkAndMint`; every receive entry opens the same screen/sheet backed by this controller.

- [ ] **Step 1: Write failing quote-race and cap tests**

```dart
test('opening receive does not create a quote', () async {
  final controller = DepositController(wallet: wallet, policy: policy, balance: () async => balance);
  await controller.initialize();
  expect(wallet.createMintQuoteCalls, 0);
});

test('stale quote response cannot replace the newest amount', () async {
  final first = controller.createQuote(1000);
  final second = controller.createQuote(2000);
  wallet.completeQuote(amount: 2000, id: 'new');
  wallet.completeQuote(amount: 1000, id: 'old');
  await Future.wait([first, second]);
  expect(controller.state.quoteId, 'new');
  expect(controller.state.amountSats, 2000);
});

test('controller applies projected balance cap before requesting quote', () async {
  await expectLater(controller.createQuote(1001), throwsA(isA<WalletPolicyViolation>()));
  expect(wallet.createMintQuoteCalls, 0);
});
```

- [ ] **Step 2: Run deposit tests and verify RED**

Run: `cd hanbova-app && flutter test test/deposit_flow_test.dart`

Expected: `DepositController` is missing and the current screen creates a quote during `initState`.

- [ ] **Step 3: Implement the controller and consolidate navigation**

Use a generation counter:

```dart
Future<void> createQuote(int amountSats) async {
  final generation = ++_generation;
  final current = await balance();
  policy.validateDeposit(amountSats: amountSats, currentBalanceSats: current.totalSats);
  state = DepositState.loading(amountSats);
  final quote = await wallet.createMintQuote(amountSats);
  if (generation != _generation) return;
  state = DepositState.ready(amountSats: amountSats, quote: quote);
}
```

Remove automatic quote creation and per-keystroke direct calls. Debounce UI input by 400ms. Route every Receive action to this canonical flow. Before mint, query quote status and handle unpaid/paid/expired/already-minted idempotently.

- [ ] **Step 4: Run deposit and wallet tests**

Run: `cd hanbova-app && flutter test test/deposit_flow_test.dart test/client_wallet_authority_test.dart test/widget_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/receive lib/features/wallet/presentation/unified_deposit_sheet.dart lib/app/router.dart test/deposit_flow_test.dart test/widget_test.dart
git -C hanbova-app commit -m "fix: unify and guard receive quotes"
```

### Task 7: Quote-first instant-send review

**Files:**
- Create: `hanbova-app/lib/features/send/domain/lightning_request_parser.dart`
- Create: `hanbova-app/lib/features/send/domain/instant_send_controller.dart`
- Create: `hanbova-app/lib/features/send/presentation/instant_send_review.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Test: `hanbova-app/test/instant_send_flow_test.dart`

**Interfaces:**
- Consumes: wallet context, Cashu wallet, wallet policy, ledger.
- Produces: `LightningRequestParser.parse(raw, network)`, `InstantSendController.prepare/pay`, and `InstantSendQuote` review model.

- [ ] **Step 1: Write failing parser and confirmation tests**

```dart
test('parser strips lightning URI and enforces active-network prefix', () {
  expect(LightningRequestParser.parse('lightning:LNTB1ABC', HanbovaNetwork.cashuTest), 'lntb1abc');
  expect(
    () => LightningRequestParser.parse('lnbc1main', HanbovaNetwork.cashuTest),
    throwsA(isA<InvalidLightningRequest>()),
  );
});

test('prepare creates a quote but does not pay it', () async {
  final review = await controller.prepare('lntb1invoice');
  expect(review.amountSats, 500);
  expect(review.feeReserveSats, 5);
  expect(wallet.payCalls, 0);
});

test('confirm revalidates policy and persists paid result', () async {
  await controller.confirm(preparedQuote);
  expect(wallet.payCalls, 1);
  expect(ledger.records.single.amountSats, 500);
});
```

- [ ] **Step 2: Run instant-send tests and verify RED**

Run: `cd hanbova-app && flutter test test/instant_send_flow_test.dart`

Expected: parser/controller/review types do not exist.

- [ ] **Step 3: Implement prepare → review → confirm**

Remove the detached amount controller. Label the input “BOLT11 Lightning invoice”; remove address/username promises. `prepare` parses, calls `createMeltQuote`, validates quote amount, and returns a review object without paying. The review screen shows amount, fee reserve, total, network, invoice fingerprint, and finality warning. `confirm` revalidates the active context and policy, calls `payMeltQuote`, then write-through persists the result.

- [ ] **Step 4: Run instant send, Lightning, and wallet tests**

Run: `cd hanbova-app && flutter test test/instant_send_flow_test.dart test/lightning_test.dart test/client_wallet_authority_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/send test/instant_send_flow_test.dart test/lightning_test.dart
git -C hanbova-app commit -m "fix: confirm Lightning quote before payment"
```

### Task 8: Verified-recipient protected-send review

**Files:**
- Create: `hanbova-app/lib/features/protected_send/domain/protected_send_draft.dart`
- Create: `hanbova-app/lib/features/protected_send/presentation/protected_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_provider.dart`
- Test: `hanbova-app/test/protected_send_review_test.dart`

**Interfaces:**
- Consumes: `UserPaymentProfile`, wallet context, wallet policy, protected-send notifier.
- Produces: immutable `ProtectedSendDraft` and `prepareDraft/confirmDraft`; locking occurs only in `confirmDraft`.

- [ ] **Step 1: Write failing review-boundary tests**

```dart
test('prepare resolves username without creating intent or locking funds', () async {
  final draft = await notifier.prepareDraft(
    username: '@amina', amountSats: 2500, description: 'Order', expirationSeconds: 86400,
  );
  expect(draft.recipient.username, 'amina');
  expect(repository.createCalls, 0);
  expect(wallet.lockCalls, 0);
});

test('confirm rejects a draft created for a stale wallet context', () async {
  await expectLater(
    notifier.confirmDraft(draftFromCashuTest, activeContext: mainnetContext),
    throwsA(isA<StateError>()),
  );
  expect(wallet.lockCalls, 0);
});
```

- [ ] **Step 2: Run protected review tests and verify RED**

Run: `cd hanbova-app && flutter test test/protected_send_review_test.dart`

Expected: draft and prepare/confirm APIs do not exist.

- [ ] **Step 3: Implement two-step protected send**

Narrow form copy to “Hanbova username.” `prepareDraft` normalizes/resolves the username and snapshots wallet context key, amount, description, and expiry. The review displays canonical handle, amount, expiry, network, claim/refund race, and delivery behavior. `confirmDraft` proves the context key is unchanged, revalidates policy, then runs the current create-intent → lock → persist → deliver sequence.

- [ ] **Step 4: Run protected-send and wallet recovery tests**

Run: `cd hanbova-app && flutter test test/protected_send_review_test.dart test/protected_delivery_recovery_test.dart test/client_wallet_authority_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/protected_send test/protected_send_review_test.dart test/protected_delivery_recovery_test.dart
git -C hanbova-app commit -m "fix: review protected recipient before locking funds"
```

### Task 9: Stable sanitized client errors

**Files:**
- Create: `hanbova-app/lib/core/errors/user_facing_error.dart`
- Modify: `hanbova-app/lib/core/errors/app_failure.dart`
- Modify: `hanbova-app/lib/core/networking/api_client.dart`
- Modify: `hanbova-app/lib/features/wallet/presentation/unified_deposit_sheet.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_provider.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/claim_payment_screen.dart`
- Modify: `hanbova-app/lib/features/security/presentation/restore_seed_screen.dart`
- Modify: `hanbova-app/lib/features/auth/providers/auth_provider.dart`
- Modify: `hanbova-app/lib/features/auth/screens/wallet_setup_screen.dart`
- Modify: `hanbova-app/lib/features/auth/screens/forgot_password_screen.dart`
- Modify: `hanbova-app/lib/features/auth/screens/reset_password_screen.dart`
- Test: `hanbova-app/test/user_facing_error_test.dart`

**Interfaces:**
- Consumes: `AppFailure.code`, `WalletPolicyViolation.code`, network/timeout exceptions, known wallet state errors.
- Produces: `UserFacingError(code, message, retryable)` and `UserFacingErrorMapper.from(Object)`; screens render only mapped messages.

- [ ] **Step 1: Write failing sanitization tests**

```dart
test('socket exception maps to retryable offline message', () {
  final error = UserFacingErrorMapper.from(const SocketException('Connection refused: token=cashuBsecret'));
  expect(error.code, UserErrorCode.offline);
  expect(error.retryable, isTrue);
  expect(error.message, 'You appear to be offline. Check your connection and try again.');
  expect(error.message, isNot(contains('cashuBsecret')));
});

test('unknown CDK exception never exposes raw details', () {
  final error = UserFacingErrorMapper.from(StateError('ffi error token=cashuBsecret'));
  expect(error.code, UserErrorCode.unexpected);
  expect(error.message, 'Something went wrong. Your wallet state was not discarded.');
  expect(error.message, isNot(contains('ffi')));
});
```

- [ ] **Step 2: Run error-mapping tests and verify RED**

Run: `cd hanbova-app && flutter test test/user_facing_error_test.dart`

Expected: mapper types do not exist and `ApiClient` embeds raw exceptions in failure messages.

- [ ] **Step 3: Implement code-based mapping and remove raw exception rendering**

```dart
enum UserErrorCode {
  authenticationRequired, walletUnavailable, offline, invalidPayment,
  policyLimit, insufficientBalance, quoteExpired, quoteUnpaid,
  recipientUnavailable, coordinationPending, unexpected,
}

final class UserFacingError {
  final UserErrorCode code;
  final String message;
  final bool retryable;
  const UserFacingError(this.code, this.message, {this.retryable = false});
}
```

Map stable backend `AppFailure.code` values first, typed domain failures second, `SocketException`/`TimeoutException` third, and all unknown errors to `unexpected`. `ApiClient` stores the original error only for debug logging and uses “Network request failed” as its public message. Replace every listed screen/provider's `$e`, `e.toString()`, and `replaceAll('Exception:')` rendering with `UserFacingErrorMapper.from(e).message`. Keep authentication session expiry as its own stable code rather than substring matching.

- [ ] **Step 4: Run error, auth, receive, and send tests**

Run: `cd hanbova-app && flutter test test/user_facing_error_test.dart test/auth_widget_test.dart test/deposit_flow_test.dart test/instant_send_flow_test.dart test/protected_send_review_test.dart`

Expected: PASS and `rg -n "errorMessage: e\.toString|Text\([^)]*\\$e|Network request failed: \\$e|Network connection failed: \\$e" lib` returns no matches.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/errors lib/core/networking/api_client.dart lib/features/wallet/presentation/unified_deposit_sheet.dart lib/features/receive lib/features/send lib/features/protected_send lib/features/security/presentation/restore_seed_screen.dart lib/features/auth test/user_facing_error_test.dart
git -C hanbova-app commit -m "fix: sanitize wallet and network errors"
```

### Task 10: Flutter wallet phase verification

**Files:**
- Verify: all Dart files modified by Tasks 1–8.

**Interfaces:**
- Consumes: Tasks 1–9.
- Produces: analyzer-clean, formatted, passing wallet/security phase.

- [ ] **Step 1: Run formatter check and format changed Dart files**

Run: `cd hanbova-app && dart format --set-exit-if-changed lib test`

If it reports changes, run `dart format lib test`, inspect `git diff`, then rerun the check.

- [ ] **Step 2: Run static analysis**

Run: `cd hanbova-app && flutter analyze`

Expected: `No issues found!`

- [ ] **Step 3: Run the full Flutter test suite**

Run: `cd hanbova-app && flutter test`

Expected: all tests pass.

- [ ] **Step 4: Inspect scope and user change preservation**

Run: `git -C hanbova-app status --short && git -C hanbova-app diff --check && git -C hanbova-app diff HEAD~8 --stat`

Expected: no untracked artifacts, no whitespace errors, and the Claim control retains the user's horizontal layout intent with accessibility sizing handled by the UI plan.

- [ ] **Step 5: Confirm the phase ends on committed task boundaries**

Run: `git -C hanbova-app status --short`

Expected: only the pre-existing Claim-button change remains when Task 4 of the UI plan has not yet integrated it; otherwise the worktree is clean. Any verification correction must be added to and amended into the task commit that introduced it, then that task's focused test must be rerun.
