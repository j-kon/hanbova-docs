# Wallet Context and Recovery Safety Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ensure Hanbova can expose and operate only the wallet identity belonging to the authenticated account and exact wallet environment, while making backup and restore authenticated, persistent, fail-closed, and truthful.

**Architecture:** A nullable `WalletContextKey` derived only from authenticated Riverpod state and the exact active `NetworkConfig` becomes the authority for identity and backup storage. Context-aware identity operations reject stale asynchronous results, the Cashu provider requires exact context equality, and focused backup/restore application services keep security and partial-success behavior outside large presentation widgets.

**Tech Stack:** Flutter 3/Dart 3, Riverpod 2.5, GoRouter 14, FlutterSecureStorage 9, local_auth 2.3, cryptography 2.7, CDK FFI, flutter_test.

**Spec:** `hanbova-docs/docs/superpowers/specs/2026-08-27-wallet-context-recovery-safety-design.md`

## Global Constraints

- Backup and restore require a signed-in Hanbova account.
- Use authenticated user ID plus the exact active storage prefix; locked mainnet and `wallet_mainnet_pilot` never share identity or acknowledgement storage.
- Signing out clears in-memory identity and wallet services but does not delete encrypted device storage.
- Unavailable, cancelled, failed, or errored OS authentication returns denial.
- The mnemonic remains the sole source for deterministic X25519 and secp256k1 identity derivation.
- Full Cashu ecash recovery after total device loss remains explicitly experimental.
- Do not log mnemonic, private key, Cashu token, refund key, invoice, access token, or JWT contents.
- Do not move, merge, or delete existing Cashu databases or protected escrow records.
- Preserve the user's uncommitted Claim-button sizing change in `lib/features/protected/presentation/protected_screen.dart` without editing those lines.
- Every production behavior change begins with a focused failing test or a failing static-analysis check.

---

### Task 1: Define the authoritative wallet context key

**Files:**
- Create: `hanbova-app/lib/core/wallet/wallet_context.dart`
- Test: `hanbova-app/test/wallet_context_test.dart`

**Interfaces:**
- Consumes: `AuthState`, `HanbovaNetwork`, `NetworkConfig`, `authProvider`, `activeNetworkConfigProvider`.
- Produces: `WalletContextKey`, `WalletContextKey.fromSession(AuthState, NetworkConfig)`, `String storageId`, `String identityStoragePrefix`, `String legacyIdentityStoragePrefix`, and `Provider<WalletContextKey?> activeWalletContextKeyProvider`.

- [ ] **Step 1: Write failing context derivation and isolation tests**

```dart
final alice = UserProfile(
  id: 'alice/id',
  username: 'alice',
  handle: '@alice',
  email: 'alice@example.test',
  firstName: 'Alice',
  lastName: 'Test',
  displayName: 'Alice Test',
  emailVerified: true,
  createdAt: DateTime.utc(2026),
);

test('wallet context requires an authenticated non-empty user and enabled config', () {
  expect(
    WalletContextKey.fromSession(AuthState.unauthenticated(), NetworkConfig.local),
    isNull,
  );
  expect(
    WalletContextKey.fromSession(
      AuthState.authenticated(alice, 'token'),
      NetworkConfig.mainnetLocked,
    ),
    isNull,
  );
});

test('storage identity distinguishes locked mainnet from mainnet pilot', () {
  final pilot = WalletContextKey(
    userId: alice.id,
    network: HanbovaNetwork.mainnet,
    storagePrefix: NetworkConfig.mainnetPilot.storagePrefix,
  );
  final locked = WalletContextKey(
    userId: alice.id,
    network: HanbovaNetwork.mainnet,
    storagePrefix: NetworkConfig.mainnetLocked.storagePrefix,
  );
  expect(pilot, isNot(locked));
  expect(pilot.storageId, isNot(locked.storageId));
  expect(pilot.identityStoragePrefix, isNot(locked.identityStoragePrefix));
});

test('storage IDs are delimiter-safe for arbitrary backend user IDs', () {
  final key = WalletContextKey(
    userId: 'alice/user:one',
    network: HanbovaNetwork.cashuTest,
    storagePrefix: NetworkConfig.cashuTest.storagePrefix,
  );
  expect(key.storageId, isNot(contains('alice/user:one')));
  expect(key.storageId, startsWith('v1_'));
});
```

- [ ] **Step 2: Run the focused test and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_context_test.dart`

Expected: compilation fails because `core/wallet/wallet_context.dart` and `WalletContextKey` do not exist.

- [ ] **Step 3: Implement the immutable context and provider**

```dart
@immutable
final class WalletContextKey {
  final String userId;
  final HanbovaNetwork network;
  final String storagePrefix;

  const WalletContextKey({
    required this.userId,
    required this.network,
    required this.storagePrefix,
  });

  static WalletContextKey? fromSession(AuthState auth, NetworkConfig config) {
    final userId = auth.user?.id.trim() ?? '';
    if (!auth.isAuthenticated || userId.isEmpty || !config.isEnabled) return null;
    return WalletContextKey(
      userId: userId,
      network: config.network,
      storagePrefix: config.storagePrefix,
    );
  }

  static String _encode(String value) =>
      base64Url.encode(utf8.encode(value)).replaceAll('=', '');

  String get storageId =>
      'v1_${network.name}_${_encode(storagePrefix)}_${_encode(userId)}';
  String get identityStoragePrefix => 'hanbova_wallet_$storageId';
  String get legacyIdentityStoragePrefix => 'hanbova_${storagePrefix}_$userId';

  @override
  bool operator ==(Object other) =>
      other is WalletContextKey &&
      other.userId == userId &&
      other.network == network &&
      other.storagePrefix == storagePrefix;

  @override
  int get hashCode => Object.hash(userId, network, storagePrefix);
}

final activeWalletContextKeyProvider = Provider<WalletContextKey?>((ref) {
  return WalletContextKey.fromSession(
    ref.watch(authProvider),
    ref.watch(activeNetworkConfigProvider),
  );
});
```

- [ ] **Step 4: Run and format the focused change**

Run: `cd hanbova-app && dart format lib/core/wallet/wallet_context.dart test/wallet_context_test.dart && flutter test test/wallet_context_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/wallet/wallet_context.dart test/wallet_context_test.dart
git -C hanbova-app commit -m "fix: define authenticated wallet context"
```

### Task 2: Isolate identity persistence and migrate compatible mnemonic keys

**Files:**
- Create: `hanbova-app/lib/core/crypto/wallet_identity_store.dart`
- Test: `hanbova-app/test/wallet_identity_store_test.dart`

**Interfaces:**
- Consumes: `WalletContextKey`, `FlutterSecureStorage`.
- Produces: `StoredMnemonicSource`, `StoredMnemonic`, `WalletIdentityStore.read`, `write`, `delete`, and `walletIdentityStoreProvider`.

- [ ] **Step 1: Write failing canonical, legacy, and deletion tests**

```dart
setUp(() => FlutterSecureStorage.setMockInitialValues({}));

test('canonical mnemonic is isolated by complete wallet context', () async {
  const storage = FlutterSecureStorage();
  final store = SecureWalletIdentityStore(storage: storage);
  await store.write(testContext, validMnemonic);

  final loaded = await store.read(testContext);
  expect(loaded?.mnemonic, validMnemonic);
  expect(loaded?.source, StoredMnemonicSource.canonical);
  expect(await store.read(pilotContext), isNull);
});

test('read returns a compatible legacy mnemonic without deleting it', () async {
  FlutterSecureStorage.setMockInitialValues({
    '${testContext.legacyIdentityStoragePrefix}_mnemonic': validMnemonic,
  });
  final store = SecureWalletIdentityStore(storage: const FlutterSecureStorage());
  final loaded = await store.read(testContext);
  expect(loaded?.source, StoredMnemonicSource.legacy);
  expect(loaded?.mnemonic, validMnemonic);
});

test('delete removes only the requested context identity keys', () async {
  final store = SecureWalletIdentityStore(storage: const FlutterSecureStorage());
  await store.write(testContext, validMnemonic);
  await store.write(pilotContext, validMnemonic);
  await store.delete(testContext);
  expect(await store.read(testContext), isNull);
  expect(await store.read(pilotContext), isNotNull);
});
```

- [ ] **Step 2: Run the storage test and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_identity_store_test.dart`

Expected: compilation fails because `WalletIdentityStore` and `SecureWalletIdentityStore` do not exist.

- [ ] **Step 3: Implement the injectable secure store**

```dart
enum StoredMnemonicSource { canonical, legacy }

final class StoredMnemonic {
  final String mnemonic;
  final StoredMnemonicSource source;
  const StoredMnemonic(this.mnemonic, this.source);
}

abstract interface class WalletIdentityStore {
  Future<StoredMnemonic?> read(WalletContextKey context);
  Future<void> write(WalletContextKey context, String mnemonic);
  Future<void> delete(WalletContextKey context);
}

final class SecureWalletIdentityStore implements WalletIdentityStore {
  final FlutterSecureStorage storage;
  const SecureWalletIdentityStore({required this.storage});

  @override
  Future<StoredMnemonic?> read(WalletContextKey context) async {
    final canonical = await storage.read(
      key: '${context.identityStoragePrefix}_mnemonic',
    );
    if (canonical != null && canonical.trim().isNotEmpty) {
      return StoredMnemonic(canonical, StoredMnemonicSource.canonical);
    }
    final legacy = await storage.read(
      key: '${context.legacyIdentityStoragePrefix}_mnemonic',
    );
    return legacy == null || legacy.trim().isEmpty
        ? null
        : StoredMnemonic(legacy, StoredMnemonicSource.legacy);
  }

  @override
  Future<void> write(WalletContextKey context, String mnemonic) => storage.write(
        key: '${context.identityStoragePrefix}_mnemonic',
        value: mnemonic,
      );

  @override
  Future<void> delete(WalletContextKey context) async {
    for (final prefix in [
      context.identityStoragePrefix,
      context.legacyIdentityStoragePrefix,
    ]) {
      await storage.delete(key: '${prefix}_mnemonic');
      await storage.delete(key: '${prefix}_transport_priv');
      await storage.delete(key: '${prefix}_protected_priv');
    }
  }
}

final walletIdentityStoreProvider = Provider<WalletIdentityStore>((ref) {
  return const SecureWalletIdentityStore(storage: FlutterSecureStorage());
});
```

Do not delete the legacy mnemonic during a read or migration. Task 3 writes the validated phrase to the canonical key only after deterministic derivation succeeds.

- [ ] **Step 4: Run and format the storage change**

Run: `cd hanbova-app && dart format lib/core/crypto/wallet_identity_store.dart test/wallet_identity_store_test.dart && flutter test test/wallet_identity_store_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/crypto/wallet_identity_store.dart test/wallet_identity_store_test.dart
git -C hanbova-app commit -m "fix: isolate wallet identity storage"
```

### Task 3: Bind cryptographic identity lifecycle to the active context

**Files:**
- Modify: `hanbova-app/lib/core/crypto/crypto_identity_service.dart`
- Modify: `hanbova-app/test/crypto_test.dart`
- Create: `hanbova-app/test/crypto_identity_context_test.dart`

**Interfaces:**
- Consumes: `activeWalletContextKeyProvider`, `WalletIdentityStore`, `MnemonicService`, deterministic derivation helpers.
- Produces: context-bearing `WalletCryptoIdentity`, `getOrCreateIdentity()`, `requireIdentity()`, `restoreFromMnemonic({required String mnemonic})`, `deleteWalletKeys()`, `WalletContextUnavailableException`, `WalletIdentityUnavailableException`, and `StaleWalletContextException`.

- [ ] **Step 1: Write failing lifecycle tests with a controllable fake store**

```dart
final testContextProvider = StateProvider<WalletContextKey?>((ref) => aliceContext);

test('requireIdentity refuses a missing wallet instead of creating one', () async {
  final store = FakeWalletIdentityStore();
  final container = ProviderContainer(overrides: [
    activeWalletContextKeyProvider.overrideWith(
      (ref) => ref.watch(testContextProvider),
    ),
    walletIdentityStoreProvider.overrideWithValue(store),
  ]);
  addTearDown(container.dispose);

  await expectLater(
    container.read(cryptoIdentityProvider.notifier).requireIdentity(),
    throwsA(isA<WalletIdentityUnavailableException>()),
  );
  expect(store.writes, isEmpty);
});

test('legacy mnemonic migrates only after validation and derivation', () async {
  final store = FakeWalletIdentityStore(
    values: {aliceContext: const StoredMnemonic(validMnemonic, StoredMnemonicSource.legacy)},
  );
  final container = identityContainer(store);
  addTearDown(container.dispose);

  final identity = await container
      .read(cryptoIdentityProvider.notifier)
      .requireIdentity();
  expect(identity.context, aliceContext);
  expect(store.writes[aliceContext], validMnemonic);
  expect(store.deletes, isEmpty);
});

test('late identity result cannot reactivate a previous account', () async {
  final gate = Completer<void>();
  final store = FakeWalletIdentityStore(beforeRead: () => gate.future);
  final container = identityContainer(store);
  addTearDown(container.dispose);

  final operation = container
      .read(cryptoIdentityProvider.notifier)
      .getOrCreateIdentity();
  container.read(testContextProvider.notifier).state = bobContext;
  gate.complete();

  await expectLater(operation, throwsA(isA<StaleWalletContextException>()));
  expect(container.read(cryptoIdentityProvider).valueOrNull, isNull);
  expect(store.writes.containsKey(bobContext), isFalse);
});
```

- [ ] **Step 2: Run lifecycle tests and verify RED**

Run: `cd hanbova-app && flutter test test/crypto_identity_context_test.dart test/crypto_test.dart`

Expected: compilation fails because identities do not carry `context`, existing methods accept caller-supplied account/network values, and the exception types do not exist.

- [ ] **Step 3: Refactor the notifier around captured contexts**

```dart
final class WalletContextUnavailableException implements Exception {}
final class WalletIdentityUnavailableException implements Exception {}
final class StaleWalletContextException implements Exception {}

class WalletCryptoIdentity {
  final WalletContextKey context;
  final String protectedPaymentPubkey;
  final String transportEncryptionPubkey;
  final SimpleKeyPair transportKeyPair;
  final String protectedPaymentPrivkeyHex;
  final String mnemonic;
  final String walletSeedHex;

  String get userId => context.userId;
  HanbovaNetwork get network => context.network;
  String get storagePrefix => context.storagePrefix;

  const WalletCryptoIdentity({
    required this.context,
    required this.protectedPaymentPubkey,
    required this.transportEncryptionPubkey,
    required this.transportKeyPair,
    required this.protectedPaymentPrivkeyHex,
    required this.mnemonic,
    required this.walletSeedHex,
  });
}

class CryptoIdentityNotifier extends AsyncNotifier<WalletCryptoIdentity?> {
  @override
  Future<WalletCryptoIdentity?> build() async {
    ref.watch(activeWalletContextKeyProvider);
    return null;
  }

  WalletContextKey _captureContext() {
    final context = ref.read(activeWalletContextKeyProvider);
    if (context == null) throw WalletContextUnavailableException();
    return context;
  }

  void _publishIfCurrent(
    WalletContextKey captured,
    WalletCryptoIdentity identity,
  ) {
    if (ref.read(activeWalletContextKeyProvider) != captured) {
      throw StaleWalletContextException();
    }
    state = AsyncValue.data(identity);
  }
}
```

Implement one private `_deriveIdentity(context, mnemonic)` helper. `requireIdentity()` returns a matching in-memory identity, otherwise reads storage and throws when absent. `getOrCreateIdentity()` uses the same load path but generates and writes a mnemonic only when storage is empty. `restoreFromMnemonic()` validates and normalizes the phrase, writes it to the captured canonical context, derives it, and publishes it only if the context still matches. When `read()` returns a legacy candidate, validate and derive first, then copy to the canonical key without deleting the legacy key.

Remove the unused derived-private-key reads and the now-unused `dart:convert` import. Change `deleteWalletKeys()` to capture the active context and delegate only that context to `WalletIdentityStore.delete`.

- [ ] **Step 4: Run focused crypto and migration tests**

Run: `cd hanbova-app && dart format lib/core/crypto/crypto_identity_service.dart test/crypto_test.dart test/crypto_identity_context_test.dart && flutter test test/crypto_identity_context_test.dart test/crypto_test.dart test/wallet_identity_store_test.dart`

Expected: PASS, including the existing deterministic X25519 test.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/crypto/crypto_identity_service.dart test/crypto_test.dart test/crypto_identity_context_test.dart
git -C hanbova-app commit -m "fix: bind crypto identity to wallet context"
```

### Task 4: Require exact context equality before constructing or using Cashu

**Files:**
- Modify: `hanbova-app/lib/core/cashu/cashu_wallet_provider.dart`
- Modify: `hanbova-app/lib/features/auth/screens/wallet_setup_screen.dart`
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_provider.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/claim_payment_screen.dart`
- Modify: `hanbova-app/lib/features/protected/presentation/protected_screen.dart` outside the user-edited Claim-button lines
- Modify: `hanbova-app/test/auth_widget_test.dart`
- Modify: `hanbova-app/test/financial_authority_test.dart`
- Create: `hanbova-app/test/cashu_wallet_context_test.dart`

**Interfaces:**
- Consumes: `activeWalletContextKeyProvider`, context-bearing `WalletCryptoIdentity`, `activeNetworkConfigProvider`.
- Produces: `CashuWalletServiceFactory`, `cashuWalletServiceFactoryProvider`, and a nullable `cashuWalletServiceProvider` that constructs CDK only for an exact identity/context match; consumers use `getOrCreateIdentity()` only during setup and `requireIdentity()` everywhere else.

- [ ] **Step 1: Write failing mismatch tests**

```dart
test('Cashu provider stays unavailable for another account identity', () async {
  final container = walletContainer(
    activeContext: aliceContext,
    identity: await identityFor(bobContext),
  );
  addTearDown(container.dispose);
  expect(container.read(cashuWalletServiceProvider), isNull);
});

test('Cashu provider stays unavailable across mainnet storage prefixes', () async {
  final container = walletContainer(
    activeContext: pilotContext,
    identity: await identityFor(lockedMainnetContext),
  );
  addTearDown(container.dispose);
  expect(container.read(cashuWalletServiceProvider), isNull);
});

test('account switch disposes the prior Cashu service immediately', () async {
  final factory = RecordingCashuWalletServiceFactory();
  final container = walletContainer(
    activeContext: aliceContext,
    identity: await identityFor(aliceContext),
    factory: factory,
  );
  addTearDown(container.dispose);

  final first = container.read(cashuWalletServiceProvider)
      as RecordingCashuWalletService;
  container.read(activeContextStateProvider.notifier).state = bobContext;
  await container.pump();

  expect(first.wasDisposed, isTrue);
  expect(container.read(cashuWalletServiceProvider), isNull);
});
```

Update `MockCryptoIdentityNotifier` so `getOrCreateIdentity()` and `requireIdentity()` take no account/network arguments and its fixture carries `WalletContextKey`.

- [ ] **Step 2: Run mismatch and consumer tests and verify RED**

Run: `cd hanbova-app && flutter test test/cashu_wallet_context_test.dart test/auth_widget_test.dart test/financial_authority_test.dart`

Expected: the mismatch tests fail against the non-null-only provider guard, and consumer tests fail to compile against the new identity API.

- [ ] **Step 3: Add the exact provider guard and migrate callers**

```dart
typedef CashuWalletServiceFactory = CashuWalletService Function({
  required WalletContextKey context,
  required WalletCryptoIdentity identity,
  required String mintUrl,
  required CashuWalletStorage storage,
});

final cashuWalletServiceFactoryProvider = Provider<CashuWalletServiceFactory>(
  (ref) => ({
    required context,
    required identity,
    required mintUrl,
    required storage,
  }) =>
      CdkCashuWalletServiceImpl(
        userId: context.userId,
        network: context.network,
        walletSeedHex: identity.walletSeedHex,
        p2pkPrivateKeyHex: identity.protectedPaymentPrivkeyHex,
        p2pkPublicKeyHex: identity.protectedPaymentPubkey,
        storagePrefix: context.storagePrefix,
        mintUrl: mintUrl,
        storage: storage,
      ),
);

final cashuWalletServiceProvider = Provider<CashuWalletService?>((ref) {
  final context = ref.watch(activeWalletContextKeyProvider);
  final identity = ref.watch(cryptoIdentityProvider).valueOrNull;
  final config = ref.watch(activeNetworkConfigProvider);
  final selectedMint = ref.watch(selectedMintUrlProvider);

  if (context == null ||
      identity == null ||
      identity.context != context ||
      !config.isEnabled ||
      config.network != context.network ||
      config.storagePrefix != context.storagePrefix) {
    return null;
  }

  final mintUrl = config.isPilot
      ? config.defaultMintUrl
      : (selectedMint ?? config.defaultMintUrl);
  final service = ref.watch(cashuWalletServiceFactoryProvider)(
    context: context,
    identity: identity,
    mintUrl: mintUrl,
    storage: ref.watch(cashuWalletStorageProvider),
  );
  ref.onDispose(service.dispose);
  return service;
});
```

Use `getOrCreateIdentity()` only in `WalletSetupScreen`. Home key publication, protected send, claim, and incoming Claim actions call `requireIdentity()`. Replace Home's `_keysSynced` boolean with `String? _syncedContextId`; publish keys whenever `activeWalletContextKeyProvider.storageId` differs so an account/environment switch cannot inherit the prior sync flag.

- [ ] **Step 4: Run wallet authority and UI tests**

Run: `cd hanbova-app && dart format lib/core/cashu/cashu_wallet_provider.dart lib/features/auth/screens/wallet_setup_screen.dart lib/features/home/presentation/home_screen.dart lib/features/protected_send/presentation/protected_send_provider.dart lib/features/protected_send/presentation/claim_payment_screen.dart lib/features/protected/presentation/protected_screen.dart test/auth_widget_test.dart test/financial_authority_test.dart test/cashu_wallet_context_test.dart && flutter test test/cashu_wallet_context_test.dart test/auth_widget_test.dart test/financial_authority_test.dart test/client_wallet_authority_test.dart`

Expected: PASS. Confirm `git diff -- lib/features/protected/presentation/protected_screen.dart` still contains the user's existing Claim-button change plus only the identity-call migration outside that button block.

- [ ] **Step 5: Commit only the planned app changes**

```bash
git -C hanbova-app add lib/core/cashu/cashu_wallet_provider.dart lib/features/auth/screens/wallet_setup_screen.dart lib/features/home/presentation/home_screen.dart lib/features/protected_send/presentation/protected_send_provider.dart lib/features/protected_send/presentation/claim_payment_screen.dart test/auth_widget_test.dart test/financial_authority_test.dart test/cashu_wallet_context_test.dart
git -C hanbova-app add -p lib/features/protected/presentation/protected_screen.dart
git -C hanbova-app commit -m "fix: reject mismatched wallet services"
```

At the interactive staging prompt, stage only the `getOrCreateIdentity(...)` to `requireIdentity()` hunk. Answer `n` for the pre-existing Claim-button `minimumSize: Size(72, 36)` hunk so the user's change remains unstaged and uncommitted.

### Task 5: Make operating-system authentication fail closed

**Files:**
- Modify: `hanbova-app/lib/core/security/biometric_service.dart`
- Create: `hanbova-app/test/biometric_service_test.dart`

**Interfaces:**
- Consumes: `local_auth.LocalAuthentication` through an injectable `LocalAuthGateway`.
- Produces: `PluginLocalAuthGateway`, testable `BiometricService`, and `authenticate({required String reason})` that returns true only for explicit plugin success.

- [ ] **Step 1: Write failing authentication outcome tests**

```dart
test('unsupported device is denied without opening a prompt', () async {
  final gateway = FakeLocalAuthGateway(supported: false, canCheck: false);
  final service = BiometricService(gateway: gateway);
  expect(await service.authenticate(reason: 'Reveal phrase'), isFalse);
  expect(gateway.authenticateCalls, 0);
});

test('cancelled or rejected prompt is denied', () async {
  final gateway = FakeLocalAuthGateway(authResult: false);
  expect(
    await BiometricService(gateway: gateway)
        .authenticate(reason: 'Replace wallet identity'),
    isFalse,
  );
});

test('platform errors are denied and explicit success is accepted', () async {
  expect(
    await BiometricService(gateway: FakeLocalAuthGateway(throwsPlatformError: true))
        .authenticate(reason: 'Reveal phrase'),
    isFalse,
  );
  expect(
    await BiometricService(gateway: FakeLocalAuthGateway(authResult: true))
        .authenticate(reason: 'Reveal phrase'),
    isTrue,
  );
});
```

- [ ] **Step 2: Run the biometric test and verify RED**

Run: `cd hanbova-app && flutter test test/biometric_service_test.dart`

Expected: compilation fails because `LocalAuthGateway` is absent; the current unsupported-device branch also authorizes access.

- [ ] **Step 3: Add the gateway and fail-closed implementation**

```dart
abstract interface class LocalAuthGateway {
  Future<bool> canCheckBiometrics();
  Future<bool> isDeviceSupported();
  Future<bool> authenticate(String reason);
}

final class PluginLocalAuthGateway implements LocalAuthGateway {
  final LocalAuthentication auth;
  PluginLocalAuthGateway({LocalAuthentication? auth})
      : auth = auth ?? LocalAuthentication();

  @override
  Future<bool> canCheckBiometrics() => auth.canCheckBiometrics;
  @override
  Future<bool> isDeviceSupported() => auth.isDeviceSupported();
  @override
  Future<bool> authenticate(String reason) => auth.authenticate(
        localizedReason: reason,
        options: const AuthenticationOptions(
          stickyAuth: true,
          biometricOnly: false,
        ),
      );
}

class BiometricService {
  final LocalAuthGateway gateway;
  BiometricService({LocalAuthGateway? gateway})
      : gateway = gateway ?? PluginLocalAuthGateway();

  Future<bool> authenticate({required String reason}) async {
    try {
      final available = await gateway.canCheckBiometrics() ||
          await gateway.isDeviceSupported();
      if (!available) return false;
      return await gateway.authenticate(reason);
    } on PlatformException {
      return false;
    } catch (_) {
      return false;
    }
  }
}
```

- [ ] **Step 4: Run and format biometric tests**

Run: `cd hanbova-app && dart format lib/core/security/biometric_service.dart test/biometric_service_test.dart && flutter test test/biometric_service_test.dart`

Expected: PASS for unsupported, cancellation, rejection, platform error, and success.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/security/biometric_service.dart test/biometric_service_test.dart
git -C hanbova-app commit -m "fix: fail closed for recovery authentication"
```

### Task 6: Persist backup confirmation and make Backup fail closed

**Files:**
- Create: `hanbova-app/lib/core/security/wallet_backup_store.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Modify: `hanbova-app/lib/features/auth/screens/wallet_setup_screen.dart`
- Modify: `hanbova-app/lib/features/profile/screens/profile_screen.dart`
- Modify: `hanbova-app/test/mainnet_safety_test.dart`
- Create: `hanbova-app/test/wallet_backup_test.dart`

**Interfaces:**
- Consumes: `WalletContextKey`, `activeWalletContextKeyProvider`, `FlutterSecureStorage`, `cryptoIdentityProvider`, `biometricServiceProvider`.
- Produces: `WalletBackupStore`, `SecureWalletBackupStore`, `WalletBackupStatusNotifier.confirm()`, and `AsyncNotifierProvider<WalletBackupStatusNotifier, bool> walletBackupStatusProvider`.

- [ ] **Step 1: Write failing persistence and Backup widget tests**

```dart
test('confirmation persists and is isolated by complete context', () async {
  final store = SecureWalletBackupStore(storage: const FlutterSecureStorage());
  await store.setConfirmed(aliceTestContext, true);
  expect(await store.isConfirmed(aliceTestContext), isTrue);
  expect(await store.isConfirmed(alicePilotContext), isFalse);

  final reloaded = SecureWalletBackupStore(storage: const FlutterSecureStorage());
  expect(await reloaded.isConfirmed(aliceTestContext), isTrue);
});

test('missing and malformed confirmation fail to false', () async {
  FlutterSecureStorage.setMockInitialValues({
    'hanbova_backup_v1_${aliceTestContext.storageId}': 'not-a-bool',
  });
  final store = SecureWalletBackupStore(storage: const FlutterSecureStorage());
  expect(await store.isConfirmed(aliceTestContext), isFalse);
});

testWidgets('Backup never generates a phrase while wallet is unavailable', (tester) async {
  await tester.pumpWidget(backupApp(activeContext: null));
  await tester.pumpAndSettle();
  expect(find.text('Wallet unavailable'), findsOneWidget);
  expect(find.text('Tap to Reveal 12 Words'), findsNothing);
  expect(fakeIdentityStore.writes, isEmpty);
});

testWidgets('failed device authentication keeps recovery words hidden', (tester) async {
  await tester.pumpWidget(backupApp(authResult: false));
  await tester.pumpAndSettle();
  await tester.tap(find.text('Tap to Reveal 12 Words'));
  await tester.pump();
  expect(find.text('••••••••'), findsWidgets);
  expect(find.text('Authentication was not completed.'), findsOneWidget);
});
```

- [ ] **Step 2: Run backup tests and verify RED**

Run: `cd hanbova-app && flutter test test/wallet_backup_test.dart test/mainnet_safety_test.dart`

Expected: storage types do not exist; Backup currently generates an unrelated mnemonic when auth is absent; the status provider is memory-only.

- [ ] **Step 3: Implement the versioned store and context-aware notifier**

```dart
abstract interface class WalletBackupStore {
  Future<bool> isConfirmed(WalletContextKey context);
  Future<void> setConfirmed(WalletContextKey context, bool value);
}

final class SecureWalletBackupStore implements WalletBackupStore {
  final FlutterSecureStorage storage;
  const SecureWalletBackupStore({required this.storage});
  String _key(WalletContextKey context) =>
      'hanbova_backup_v1_${context.storageId}';

  @override
  Future<bool> isConfirmed(WalletContextKey context) async =>
      await storage.read(key: _key(context)) == 'true';

  @override
  Future<void> setConfirmed(WalletContextKey context, bool value) =>
      storage.write(key: _key(context), value: value.toString());
}

final class WalletBackupStatusNotifier extends AsyncNotifier<bool> {
  @override
  Future<bool> build() async {
    final context = ref.watch(activeWalletContextKeyProvider);
    if (context == null) return false;
    return ref.watch(walletBackupStoreProvider).isConfirmed(context);
  }

  Future<void> confirm() async {
    final context = ref.read(activeWalletContextKeyProvider);
    if (context == null) throw WalletContextUnavailableException();
    await ref.read(walletBackupStoreProvider).setConfirmed(context, true);
    if (ref.read(activeWalletContextKeyProvider) != context) {
      throw StaleWalletContextException();
    }
    state = const AsyncValue.data(true);
  }
}
```

Backup calls `requireIdentity()` and never imports or calls `MnemonicService.generateMnemonic`. Add explicit loading/unavailable/error states. On failed authentication, keep `_isRevealed` false and show the fixed safe message. Make quiz verification asynchronous and await `walletBackupStatusProvider.notifier.confirm()` before success. Wallet setup also awaits `confirm()` after its word quiz succeeds. Profile uses the following deterministic presentation rule, treating loading/error as not confirmed:

```dart
final isBackedUp =
    ref.watch(walletBackupStatusProvider).valueOrNull ?? false;
```

- [ ] **Step 4: Run backup, setup, profile, and safety tests**

Run: `cd hanbova-app && dart format lib/core/security/wallet_backup_store.dart lib/features/security/presentation/backup_seed_screen.dart lib/features/auth/screens/wallet_setup_screen.dart lib/features/profile/screens/profile_screen.dart test/wallet_backup_test.dart test/mainnet_safety_test.dart && flutter test test/wallet_backup_test.dart test/mainnet_safety_test.dart test/auth_widget_test.dart`

Expected: PASS and no generated mnemonic is written by Backup.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/security/wallet_backup_store.dart lib/features/security/presentation/backup_seed_screen.dart lib/features/auth/screens/wallet_setup_screen.dart lib/features/profile/screens/profile_screen.dart test/wallet_backup_test.dart test/mainnet_safety_test.dart
git -C hanbova-app commit -m "fix: persist and protect wallet backup state"
```

### Task 7: Orchestrate authenticated restore with truthful partial outcomes

**Files:**
- Create: `hanbova-app/lib/features/security/application/restore_wallet_controller.dart`
- Modify: `hanbova-app/lib/features/security/presentation/restore_seed_screen.dart`
- Modify: `hanbova-app/lib/features/auth/screens/welcome_screen.dart`
- Modify: `hanbova-app/lib/features/auth/screens/sign_in_screen.dart`
- Modify: `hanbova-app/lib/app/router.dart`
- Create: `hanbova-app/test/restore_wallet_controller_test.dart`
- Create: `hanbova-app/test/restore_seed_widget_test.dart`

**Interfaces:**
- Consumes: active wallet context, `BiometricService`, `CryptoIdentityNotifier`, `ApiClient`, `cashuWalletServiceProvider`, `WalletBackupStatusNotifier`.
- Produces: `RestoreWalletOutcome.restored`, `RestoreWalletOutcome.syncPending`, `RestoreWalletResult`, `RestoreWalletFailure`, `RestoreWalletController.restore(String mnemonic)`, and `retryPublicKeySync()`.

- [ ] **Step 1: Write failing controller and route-intent tests**

```dart
test('restore refuses an unauthenticated context before touching storage', () async {
  final controller = restoreController(activeContext: null);
  await expectLater(
    controller.restore(validMnemonic),
    throwsA(isA<RestoreWalletFailure>().having(
      (failure) => failure.code,
      'code',
      'authentication_required',
    )),
  );
  expect(fakeIdentityStore.writes, isEmpty);
});

test('restore reports sync pending after local success and publication failure', () async {
  final controller = restoreController(publicationFails: true);
  final result = await controller.restore(validMnemonic);
  expect(result.outcome, RestoreWalletOutcome.syncPending);
  expect(fakeWallet.getBalanceCalls, 1);
  expect(fakeBackupStore.confirmedContexts, [aliceContext]);
});

test('wallet rebuild failure does not report success', () async {
  final controller = restoreController(walletUnavailable: true);
  await expectLater(
    controller.restore(validMnemonic),
    throwsA(isA<RestoreWalletFailure>().having(
      (failure) => failure.code,
      'code',
      'wallet_unavailable',
    )),
  );
});

test('context change during restore blocks success and confirmation', () async {
  final controller = restoreController(changeContextDuringRestore: true);
  await expectLater(
    controller.restore(validMnemonic),
    throwsA(isA<RestoreWalletFailure>().having(
      (failure) => failure.code,
      'code',
      'session_changed',
    )),
  );
  expect(fakeBackupStore.confirmedContexts, isEmpty);
});

test('retry rereads the current identity and clears a transient sync failure', () async {
  final controller = restoreController(publicationFailuresBeforeSuccess: 1);
  final result = await controller.restore(validMnemonic);
  expect(result.outcome, RestoreWalletOutcome.syncPending);
  expect(await controller.retryPublicKeySync(), isTrue);
  expect(fakeCrypto.requireIdentityCalls, 1);
});

testWidgets('welcome restore action sends the user through sign in', (tester) async {
  await tester.pumpWidget(routerApp(initialLocation: '/welcome'));
  await tester.tap(find.text('Sign in to restore with phrase'));
  await tester.pumpAndSettle();
  expect(find.byType(SignInScreen), findsOneWidget);
  expect(currentLocation(), '/login?next=%2Frestore-seed');
});

testWidgets('restore form is absent without an authenticated context', (tester) async {
  await tester.pumpWidget(restoreScreenApp(activeContext: null));
  await tester.pumpAndSettle();
  expect(find.text('Sign in to restore your wallet'), findsOneWidget);
  expect(find.byType(TextField), findsNothing);
});

testWidgets('restore requires explicit replacement confirmation', (tester) async {
  await tester.pumpWidget(restoreScreenApp(activeContext: aliceContext));
  await enterValidMnemonic(tester);
  await tester.tap(find.text('Restore Wallet'));
  await tester.pumpAndSettle();
  expect(
    find.text(
      'This replaces the wallet identity for the signed-in account in this wallet environment.',
    ),
    findsOneWidget,
  );
  await tester.tap(find.text('Cancel'));
  await tester.pumpAndSettle();
  expect(fakeRestoreController.restoreCalls, 0);
});
```

- [ ] **Step 2: Run restore tests and verify RED**

Run: `cd hanbova-app && flutter test test/restore_wallet_controller_test.dart test/restore_seed_widget_test.dart`

Expected: the controller/result types do not exist; the welcome action currently links directly to `/restore-seed`; the screen still uses `default_user`.

- [ ] **Step 3: Implement the restore application service**

```dart
enum RestoreWalletOutcome { restored, syncPending }

final class RestoreWalletResult {
  final RestoreWalletOutcome outcome;
  final WalletCryptoIdentity identity;
  const RestoreWalletResult(this.outcome, this.identity);
}

final class RestoreWalletFailure implements Exception {
  final String code;
  final String message;
  const RestoreWalletFailure(this.code, this.message);
}

final class RestoreWalletController {
  final Ref ref;
  const RestoreWalletController(this.ref);

  Future<RestoreWalletResult> restore(String mnemonic) async {
    final context = ref.read(activeWalletContextKeyProvider);
    if (context == null) {
      throw const RestoreWalletFailure(
        'authentication_required',
        'Sign in to restore your wallet.',
      );
    }
    final authorized = await ref.read(biometricServiceProvider).authenticate(
          reason: 'Authenticate to replace this wallet identity',
        );
    if (!authorized) {
      throw const RestoreWalletFailure(
        'authentication_denied',
        'Authentication was not completed.',
      );
    }

    final identity = await ref
        .read(cryptoIdentityProvider.notifier)
        .restoreFromMnemonic(mnemonic: mnemonic);
    if (ref.read(activeWalletContextKeyProvider) != context) {
      throw const RestoreWalletFailure(
        'session_changed',
        'Your account or wallet environment changed. Start restore again.',
      );
    }

    var outcome = RestoreWalletOutcome.restored;
    try {
      await ref.read(cryptoIdentityProvider.notifier).publishPublicKeys(
            apiClient: ref.read(apiClientProvider),
            identity: identity,
          );
    } catch (_) {
      outcome = RestoreWalletOutcome.syncPending;
    }

    final wallet = ref.read(cashuWalletServiceProvider);
    if (wallet == null) {
      throw const RestoreWalletFailure(
        'wallet_unavailable',
        'The wallet identity was saved, but the wallet could not be opened.',
      );
    }
    await wallet.getBalance();
    await ref.read(walletBackupStatusProvider.notifier).confirm();
    ref.invalidate(cashuBalanceProvider);
    return RestoreWalletResult(outcome, identity);
  }
}
```

Map mnemonic validation, secure-storage, derivation, stale-context, and CDK errors to the stable failure codes described by the spec without including raw mnemonic or private-key values. Implement `retryPublicKeySync()` by re-reading the active context and `requireIdentity()` before publication; return `false` when the context changes or publication fails.

- [ ] **Step 4: Implement the authenticated UI and safe post-login intent**

Before calling the controller, validate the 12 words and show a final dialog: “This replaces the wallet identity for the signed-in account in this wallet environment.” Continue only on explicit confirmation.

Use these fixed result messages:

```dart
const restoredMessage =
    'Your wallet identity has been restored. Full ecash balance recovery '
    'after complete device loss remains experimental.';
const syncPendingMessage =
    'Your wallet identity was restored, but payment-key sync is pending. '
    'Receiving protected payments may be unavailable until sync completes.';
```

For `syncPending`, show `Retry sync` and `Continue` actions. For a failed outcome, remain on Restore and do not navigate Home. Remove `default_user` completely.

Change the welcome CTA to `Sign in to restore with phrase` and navigate to `/login?next=%2Frestore-seed`. In the router, allow only the exact `/restore-seed` value as a post-login destination; all other `next` values resolve to `/home`.

```dart
String safePostLoginPath(Uri uri) =>
    uri.queryParameters['next'] == '/restore-seed' ? '/restore-seed' : '/home';
```

Use the same helper in the authenticated-login redirect and `SignInScreen` success path so the router refresh cannot discard the restore intent.

- [ ] **Step 5: Run restore controller, widget, routing, and auth tests**

Run: `cd hanbova-app && dart format lib/features/security/application/restore_wallet_controller.dart lib/features/security/presentation/restore_seed_screen.dart lib/features/auth/screens/welcome_screen.dart lib/features/auth/screens/sign_in_screen.dart lib/app/router.dart test/restore_wallet_controller_test.dart test/restore_seed_widget_test.dart && flutter test test/restore_wallet_controller_test.dart test/restore_seed_widget_test.dart test/auth_widget_test.dart`

Expected: PASS for authentication denial, session change, full restore, sync pending, retry, wallet failure, safe login redirect, and truthful copy.

- [ ] **Step 6: Commit**

```bash
git -C hanbova-app add lib/features/security/application/restore_wallet_controller.dart lib/features/security/presentation/restore_seed_screen.dart lib/features/auth/screens/welcome_screen.dart lib/features/auth/screens/sign_in_screen.dart lib/app/router.dart test/restore_wallet_controller_test.dart test/restore_seed_widget_test.dart test/auth_widget_test.dart
git -C hanbova-app commit -m "fix: make wallet restore authenticated and truthful"
```

### Task 8: Close static-analysis gaps and run the complete safety gate

**Files:**
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Modify: `hanbova-app/test/crypto_test.dart` only if formatting still differs
- Modify: `hanbova-docs/docs/superpowers/specs/2026-08-27-wallet-context-recovery-safety-design.md` after verification

**Interfaces:**
- Consumes: all outputs from Tasks 1–7.
- Produces: zero Flutter analyzer findings, formatted Dart, a passing full Flutter suite, and an evidence-backed completed-spec status.

- [ ] **Step 1: Run the static checks and preserve their RED evidence**

Run: `cd hanbova-app && dart format --output=none --set-exit-if-changed lib test`

Expected before cleanup: non-zero if any touched or pre-existing Dart file is not formatted.

Run: `cd hanbova-app && flutter analyze`

Expected before cleanup: the pre-existing `use_build_context_synchronously` finding remains until fixed; the two unused crypto variables must already be gone after Task 3.

- [ ] **Step 2: Fix the asynchronous navigation warning without changing behavior**

Capture the router before the asynchronous logout, then use it after the mounted check:

```dart
onPressed: () async {
  final router = GoRouter.of(context);
  await ref.read(authProvider.notifier).logout();
  if (!mounted) return;
  router.go('/login');
},
```

Run `dart format lib test`; do not edit the user's Claim-button sizing block.

- [ ] **Step 3: Run focused safety regression tests**

Run: `cd hanbova-app && flutter test test/wallet_context_test.dart test/wallet_identity_store_test.dart test/crypto_identity_context_test.dart test/cashu_wallet_context_test.dart test/biometric_service_test.dart test/wallet_backup_test.dart test/restore_wallet_controller_test.dart test/restore_seed_widget_test.dart`

Expected: PASS.

- [ ] **Step 4: Run the complete Flutter quality gate**

Run: `cd hanbova-app && dart format --output=none --set-exit-if-changed lib test`

Expected: exit 0 and no files reported as changed.

Run: `cd hanbova-app && flutter analyze`

Expected: `No issues found!`

Run: `cd hanbova-app && flutter test`

Expected: all tests pass with no failures. Record the exact final test count in the handoff; do not reuse an earlier count.

- [ ] **Step 5: Inspect the final diff for scope and secret safety**

Run: `git -C hanbova-app diff --check`

Expected: no whitespace errors.

Run: `git -C hanbova-app diff --stat d97adac..HEAD`

Expected: changes are limited to the wallet-context, identity, backup/restore, auth-intent, tests, and one analyzer cleanup described by this plan.

Run: `rg -n "mnemonic|walletSeedHex|protectedPaymentPrivkeyHex|accessToken" hanbova-app/lib/core hanbova-app/lib/features/security`

Expected: only field access, secure persistence, derivation, and API wiring appear; no new logging or interpolation of secret values appears.

- [ ] **Step 6: Mark the focused spec implemented only after every check passes**

Change the spec status to:

```markdown
**Status:** Implemented and verified on 2026-08-27
```

If any quality-gate command fails, leave the status as approved and report the exact remaining failure instead.

- [ ] **Step 7: Commit the final cleanup and verification status**

```bash
git -C hanbova-app add lib/features/protected_send/presentation/protected_send_screen.dart test/crypto_test.dart
git -C hanbova-app commit -m "chore: close wallet safety quality gate"
git -C hanbova-docs add docs/superpowers/specs/2026-08-27-wallet-context-recovery-safety-design.md
git -C hanbova-docs commit -m "docs: record wallet recovery verification"
```
