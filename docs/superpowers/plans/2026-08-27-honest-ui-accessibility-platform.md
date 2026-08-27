# Honest UI, Accessibility, and Platform Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove deceptive capability signals, make core flows accessible and responsive, protect recovery material at the platform boundary, and align product documentation with verified behavior.

**Architecture:** A small capability model supplies truthful UI states while unsupported integrations remain disabled. Shared semantic controls and an English localization boundary replace hardcoded behavior in touched screens. A platform security channel activates native recovery-screen protection without adding a third-party security dependency.

**Tech Stack:** Flutter/Dart, Material, Flutter localization generation, Riverpod, Android Kotlin, iOS Swift, flutter_test, package_info_plus.

**Spec:** `hanbova-docs/docs/superpowers/specs/2026-08-27-release-hardening-design.md`

## Global Constraints

- QR camera scanning, push notifications, biometric login, live exchange rates, phone/email recipients, and non-English translations are not implemented in this release.
- Unsupported features must be hidden, disabled, or labeled “Coming soon”; no sample financial/security events remain.
- OS authentication remains enabled only for recovery-phrase reveal.
- Every actionable control has a minimum 48-by-48 logical-pixel hit target.
- Payment and recovery flows must render at 1.3x and 2.0x text scaling on a 360-by-640 logical-pixel viewport.
- Preserve Hanbova's current cream/green visual language and the user's Claim-button horizontal-layout intent.
- Every behavioral production change starts with a failing focused test.

---

### Task 1: Truthful capability model and Profile states

**Files:**
- Create: `hanbova-app/lib/core/capabilities/app_capabilities.dart`
- Modify: `hanbova-app/lib/features/profile/screens/profile_screen.dart`
- Modify: `hanbova-app/lib/features/notifications/screens/notifications_screen.dart`
- Modify: `hanbova-app/lib/app/router.dart`
- Test: `hanbova-app/test/capability_truth_test.dart`

**Interfaces:**
- Produces: immutable `AppCapabilities` and `appCapabilitiesProvider` with every unsupported integration set to false.
- Consumes: real authenticated user fields including email verification state.

- [ ] **Step 1: Write failing truthfulness widget tests**

```dart
testWidgets('profile never presents unsupported capabilities as enabled', (tester) async {
  await tester.pumpWidget(testProfileApp(capabilities: const AppCapabilities.release()));
  expect(find.text('Biometric Login / Face ID'), findsOneWidget);
  expect(find.text('Coming soon'), findsWidgets);
  expect(find.byType(Switch), findsNothing);
  expect(find.text('Enabled'), findsNothing);
});

testWidgets('notifications contain no fabricated financial events', (tester) async {
  await tester.pumpWidget(testNotificationsApp());
  expect(find.text('No notifications yet'), findsOneWidget);
  expect(find.textContaining('@amina'), findsNothing);
  expect(find.textContaining('@kofi'), findsNothing);
});

testWidgets('unverified account has no verified badge', (tester) async {
  await tester.pumpWidget(testProfileApp(user: unverifiedUser));
  expect(find.byIcon(Icons.verified), findsNothing);
});
```

- [ ] **Step 2: Run capability tests and verify RED**

Run: `cd hanbova-app && flutter test test/capability_truth_test.dart`

Expected: Profile has an enabled local biometric switch, hardcoded verified badge/notification state, and Notifications shows fake events.

- [ ] **Step 3: Implement capability-driven presentation**

```dart
@immutable
final class AppCapabilities {
  final bool cameraQrScanning;
  final bool pushNotifications;
  final bool biometricLogin;
  final bool liveExchangeRates;
  const AppCapabilities.release()
      : cameraQrScanning = false,
        pushNotifications = false,
        biometricLogin = false,
        liveExchangeRates = false;
}
```

Replace the biometric switch with disabled “Coming soon” text, replace Notifications “Enabled” with “Coming soon,” and show verified icon only when the user model's real `emailVerified` field is true. Replace fake fallback identity (`Jeremiah`, `@jeremiah`, sample email) with an authenticated loading/error state. Notifications becomes an honest empty-state screen and remains routable for future real data.

- [ ] **Step 4: Run capability and navigation tests**

Run: `cd hanbova-app && flutter test test/capability_truth_test.dart test/widget_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/capabilities lib/features/profile/screens/profile_screen.dart lib/features/notifications/screens/notifications_screen.dart lib/app/router.dart test/capability_truth_test.dart test/widget_test.dart
git -C hanbova-app commit -m "fix: show truthful capability states"
```

### Task 2: Manual-only payment request entry

**Files:**
- Modify: `hanbova-app/lib/features/scan/screens/scan_screen.dart`
- Modify: `hanbova-app/lib/features/send/domain/lightning_request_parser.dart`
- Modify: `hanbova-app/android/app/src/main/AndroidManifest.xml`
- Modify: `hanbova-app/ios/Runner/Info.plist`
- Test: `hanbova-app/test/manual_payment_entry_test.dart`

**Interfaces:**
- Consumes: `LightningRequestParser` from the wallet plan.
- Produces: manual paste/entry screen that returns supported BOLT11/claim inputs and rejects unsupported Cashu token/camera claims.

- [ ] **Step 1: Write failing manual-entry tests**

```dart
testWidgets('scan route clearly states camera is unavailable', (tester) async {
  await tester.pumpWidget(testScanApp());
  expect(find.text('Camera scanning coming soon'), findsOneWidget);
  expect(find.textContaining('scan automatically'), findsNothing);
  expect(find.text('Paste payment request'), findsOneWidget);
});

testWidgets('unsupported Cashu token is rejected without routing to Claim', (tester) async {
  await tester.pumpWidget(testScanApp());
  await tester.enterText(find.byType(TextField), 'cashuBtoken');
  await tester.tap(find.text('Continue'));
  await tester.pump();
  expect(find.text('Cashu token import is not available yet.'), findsOneWidget);
});
```

- [ ] **Step 2: Run manual-entry tests and verify RED**

Run: `cd hanbova-app && flutter test test/manual_payment_entry_test.dart`

Expected: screen claims automatic camera scanning and routes unknown input to Claim.

- [ ] **Step 3: Replace fake scanner with explicit manual entry**

Remove the painted viewfinder and show an informational camera-coming-soon card above a labeled multiline input, Paste button, and Continue button. Parse claim codes explicitly; delegate BOLT11 parsing to `LightningRequestParser`; show a specific unsupported message for `cashuA`/`cashuB`; reject unknown formats. Remove Android CAMERA permission and iOS `NSCameraUsageDescription` because this release does not access the camera.

- [ ] **Step 4: Run manual-entry, instant-send, and route tests**

Run: `cd hanbova-app && flutter test test/manual_payment_entry_test.dart test/instant_send_flow_test.dart test/widget_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/scan/screens/scan_screen.dart lib/features/send/domain/lightning_request_parser.dart android/app/src/main/AndroidManifest.xml ios/Runner/Info.plist test/manual_payment_entry_test.dart
git -C hanbova-app commit -m "fix: replace fake QR scanner with manual entry"
```

### Task 3: Estimated fiat labeling

**Files:**
- Modify: `hanbova-app/lib/core/currency/currency_provider.dart`
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/instant_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_review.dart`
- Modify: `hanbova-app/lib/features/transactions/presentation/transactions_screen.dart`
- Modify: `hanbova-app/lib/features/transactions/presentation/transaction_details_screen.dart`
- Test: `hanbova-app/test/currency_test.dart`
- Test: `hanbova-app/test/capability_truth_test.dart`

**Interfaces:**
- Produces: `FiatEstimate(formatted, sourceLabel, isLive)` through `ExchangeRateProvider.estimate`.

- [ ] **Step 1: Write failing estimate-label tests**

```dart
test('development rate identifies itself as an estimate', () {
  final estimate = const DevelopmentExchangeRateProvider().estimate(FiatCurrency.ngn, 1000);
  expect(estimate.formatted, startsWith('₦'));
  expect(estimate.sourceLabel, 'Estimated • Development rate');
  expect(estimate.isLive, isFalse);
});

testWidgets('home does not present development fiat as live value', (tester) async {
  await tester.pumpWidget(testHomeApp());
  expect(find.text('Estimated • Development rate'), findsOneWidget);
});
```

- [ ] **Step 2: Run currency tests and verify RED**

Run: `cd hanbova-app && flutter test test/currency_test.dart test/capability_truth_test.dart`

Expected: no `FiatEstimate` API or source label exists.

- [ ] **Step 3: Implement estimate model and label all touched financial views**

```dart
final class FiatEstimate {
  final String formatted;
  final String sourceLabel;
  final bool isLive;
  const FiatEstimate({required this.formatted, required this.sourceLabel, required this.isLive});
}
```

Retain the fixed development math, but render the source label adjacent to every fiat amount touched by this plan. Do not add timestamps because the values are calibrated constants, not fetched data.

- [ ] **Step 4: Run currency and payment widget tests**

Run: `cd hanbova-app && flutter test test/currency_test.dart test/capability_truth_test.dart test/instant_send_flow_test.dart test/protected_send_review_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/currency/currency_provider.dart lib/features/home/presentation/home_screen.dart lib/features/send/presentation/instant_send_review.dart lib/features/protected_send/presentation/protected_send_review.dart lib/features/transactions/presentation test/currency_test.dart test/capability_truth_test.dart
git -C hanbova-app commit -m "fix: label fiat conversions as estimates"
```

### Task 4: Semantic controls and 48dp targets

**Files:**
- Modify: `hanbova-app/lib/app/shell/app_shell.dart`
- Modify: `hanbova-app/lib/features/protected/presentation/protected_screen.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/wallet/presentation/unified_deposit_sheet.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/instant_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/claim_payment_screen.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Modify: `hanbova-app/lib/features/security/presentation/restore_seed_screen.dart`
- Test: `hanbova-app/test/accessibility_test.dart`

**Interfaces:**
- Produces: semantic “Pay” center action and minimum 48dp actions throughout payment/recovery flows.

- [ ] **Step 1: Write failing semantics and hit-target tests**

```dart
testWidgets('center action is exposed as a Pay button', (tester) async {
  await tester.pumpWidget(testShellApp());
  final semantics = tester.getSemantics(find.byKey(const Key('center-pay-button')));
  expect(semantics.label, 'Pay');
  expect(semantics.hasFlag(SemanticsFlag.isButton), isTrue);
});

testWidgets('Claim action has a 48dp hit target', (tester) async {
  await tester.pumpWidget(testIncomingProtectedPayment());
  final size = tester.getSize(find.widgetWithText(ElevatedButton, 'Claim'));
  expect(size.height, greaterThanOrEqualTo(48));
  expect(size.width, greaterThanOrEqualTo(48));
});
```

- [ ] **Step 2: Run accessibility tests and verify RED**

Run: `cd hanbova-app && flutter test test/accessibility_test.dart`

Expected: center GestureDetector lacks a semantic button/label and Claim height is 36.

- [ ] **Step 3: Implement semantic Material actions**

Replace `_CenterPayButton`'s raw `GestureDetector` with `Semantics(button: true, label: 'Pay')` plus `InkResponse`/`IconButton` using `key: Key('center-pay-button')`, `tooltip: 'Pay'`, and `constraints: BoxConstraints.tightFor(width: 56, height: 56)`. Keep the animation via `AnimatedScale` driven by Material pressed state.

Change the user's Claim style to `minimumSize: Size(72, 48)` while retaining 12px horizontal padding. Audit every button/IconButton in payment and recovery screens and set a 48dp constraint when custom sizing bypasses Material defaults. Add text/icon status alongside color-only warning states.

- [ ] **Step 4: Run semantics and navigation tests**

Run: `cd hanbova-app && flutter test test/accessibility_test.dart test/widget_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/app/shell/app_shell.dart lib/features/protected/presentation/protected_screen.dart lib/features/receive lib/features/wallet/presentation/unified_deposit_sheet.dart lib/features/send lib/features/protected_send lib/features/security test/accessibility_test.dart test/widget_test.dart
git -C hanbova-app commit -m "fix: add semantic wallet controls and touch targets"
```

### Task 5: Large-text and small-screen resilience

**Files:**
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/instant_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_review.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Test: `hanbova-app/test/responsive_wallet_flows_test.dart`

**Interfaces:**
- Produces: overflow-free core wallet flows at 360×640 and text scales 1.3/2.0.

- [ ] **Step 1: Write failing device-matrix widget tests**

```dart
for (final scale in [1.3, 2.0]) {
  testWidgets('payment and recovery flows fit small screen at ${scale}x', (tester) async {
    tester.view.physicalSize = const Size(360, 640);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.resetPhysicalSize);
    await tester.pumpWidget(MediaQuery(
      data: MediaQueryData(textScaler: TextScaler.linear(scale)),
      child: testWalletFlowApp(),
    ));
    await exerciseHomeReceiveSendProtectedAndBackup(tester);
    expect(tester.takeException(), isNull);
  });
}
```

- [ ] **Step 2: Run responsive tests and verify RED**

Run: `cd hanbova-app && flutter test test/responsive_wallet_flows_test.dart`

Expected: at least one RenderFlex overflow or clipped action at 2.0x.

- [ ] **Step 3: Fix only reproduced layout failures**

Replace rigid horizontal rows with `Wrap` or `Expanded/Flexible`, allow status text to wrap, keep primary actions in the scrollable body, and avoid fixed card heights where text content is dynamic. Do not reduce fonts below theme sizes to make tests pass.

- [ ] **Step 4: Rerun responsive and existing widget tests**

Run: `cd hanbova-app && flutter test test/responsive_wallet_flows_test.dart test/widget_test.dart test/auth_widget_test.dart`

Expected: PASS with `tester.takeException()` null at both scales.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/features/home/presentation/home_screen.dart lib/features/receive lib/features/send lib/features/protected_send lib/features/security/presentation/backup_seed_screen.dart test/responsive_wallet_flows_test.dart
git -C hanbova-app commit -m "fix: support large text on small wallet screens"
```

### Task 6: English localization boundary

**Files:**
- Create: `hanbova-app/l10n.yaml`
- Create: `hanbova-app/lib/l10n/app_en.arb`
- Modify: `hanbova-app/pubspec.yaml`
- Modify: `hanbova-app/lib/app/app.dart`
- Modify: `hanbova-app/lib/features/profile/screens/profile_screen.dart`
- Modify: `hanbova-app/lib/features/notifications/screens/notifications_screen.dart`
- Modify: `hanbova-app/lib/features/scan/screens/scan_screen.dart`
- Modify: `hanbova-app/lib/features/home/presentation/home_screen.dart`
- Modify: `hanbova-app/lib/features/receive/presentation/receive_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/send_screen.dart`
- Modify: `hanbova-app/lib/features/send/presentation/instant_send_review.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_review.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Modify: `hanbova-app/lib/features/security/presentation/restore_seed_screen.dart`
- Test: `hanbova-app/test/localization_test.dart`

**Interfaces:**
- Produces: generated `AppLocalizations`, English `supportedLocales`, and localized strings for all screens touched by this hardening phase.

- [ ] **Step 1: Write failing localization test**

```dart
test('app exposes English localization and capability copy', () async {
  final l10n = await AppLocalizations.delegate.load(const Locale('en'));
  expect(l10n.cameraScanningComingSoon, 'Camera scanning coming soon');
  expect(AppLocalizations.supportedLocales, contains(const Locale('en')));
});
```

- [ ] **Step 2: Run localization test and verify RED**

Run: `cd hanbova-app && flutter test test/localization_test.dart`

Expected: `AppLocalizations` and generated delegates do not exist.

- [ ] **Step 3: Configure generation and migrate touched strings**

Use:

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
synthetic-package: false
```

Set `flutter: generate: true` in `pubspec.yaml`. Add `localizationsDelegates: AppLocalizations.localizationsDelegates` and `supportedLocales: AppLocalizations.supportedLocales` to `MaterialApp.router`. Populate ARB keys for the capability, payment review, policy error, offline, backup, and accessibility labels introduced by these plans; replace corresponding hardcoded strings with `context.l10n` accessors.

- [ ] **Step 4: Generate and test localization**

Run: `cd hanbova-app && flutter gen-l10n && flutter test test/localization_test.dart test/capability_truth_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add l10n.yaml lib/l10n pubspec.yaml lib/app/app.dart lib/features test/localization_test.dart
git -C hanbova-app commit -m "feat: establish English localization boundary"
```

### Task 7: Native recovery-screen protection and backup exclusion

**Files:**
- Create: `hanbova-app/lib/core/security/sensitive_screen_service.dart`
- Modify: `hanbova-app/lib/features/security/presentation/backup_seed_screen.dart`
- Modify: `hanbova-app/android/app/src/main/kotlin/org/hanbova/hanbova/MainActivity.kt`
- Modify: `hanbova-app/android/app/src/main/AndroidManifest.xml`
- Create: `hanbova-app/android/app/src/debug/AndroidManifest.xml`
- Create: `hanbova-app/android/app/src/main/res/xml/data_extraction_rules.xml`
- Modify: `hanbova-app/ios/Runner/AppDelegate.swift`
- Create: `hanbova-app/scripts/validate-mobile-security.sh`
- Test: `hanbova-app/test/sensitive_screen_test.dart`

**Interfaces:**
- Produces: `SensitiveScreenService.setProtected(bool)` over method channel `org.hanbova/security`; Android toggles `FLAG_SECURE`; iOS overlays sensitive content during active capture; Dart hides phrase on inactive lifecycle.

- [ ] **Step 1: Write failing Dart lifecycle/channel tests**

```dart
testWidgets('revealed phrase is hidden when app becomes inactive', (tester) async {
  final channel = FakeSensitiveScreenService();
  await tester.pumpWidget(testBackupApp(sensitiveScreen: channel));
  await revealPhrase(tester);
  expect(channel.protected, isTrue);
  tester.binding.handleAppLifecycleStateChanged(AppLifecycleState.inactive);
  await tester.pump();
  expect(find.byKey(const Key('recovery-phrase-words')), findsNothing);
  expect(channel.protected, isFalse);
});
```

- [ ] **Step 2: Run sensitive-screen test and verify RED**

Run: `cd hanbova-app && flutter test test/sensitive_screen_test.dart`

Expected: service is missing and backup screen does not observe lifecycle.

- [ ] **Step 3: Implement Dart and native platform protection**

```dart
final class SensitiveScreenService {
  static const _channel = MethodChannel('org.hanbova/security');
  Future<void> setProtected(bool value) =>
      _channel.invokeMethod<void>('setSensitiveScreenProtected', {'enabled': value});
}
```

Android handles the method on the UI thread and adds/clears `WindowManager.LayoutParams.FLAG_SECURE`. Set `android:allowBackup="false"`, `android:fullBackupContent="false"`, `android:dataExtractionRules="@xml/data_extraction_rules"`, and `android:usesCleartextTraffic="false"`; the rules exclude all cloud backup and device transfer data. The debug manifest alone overrides cleartext traffic to true for local HTTP development.

iOS registers the method channel during engine initialization. When enabled, observe `UIScreen.capturedDidChangeNotification`; add an opaque app-branded overlay window while `UIScreen.main.isCaptured`, remove it when capture ends, and remove observers when disabled. Dart's backup screen implements `WidgetsBindingObserver`, clears `_isRevealed`, and disables protection on inactive/paused/detach and dispose.

Create `scripts/validate-mobile-security.sh` that fails unless the main Android manifest contains `allowBackup="false"` and `usesCleartextTraffic="false"`, fails if the main iOS plist contains `NSAllowsArbitraryLoads` followed by `<true/>`, and fails if CAMERA permission or `NSCameraUsageDescription` remains.

- [ ] **Step 4: Run Dart and native build verification**

Run: `cd hanbova-app && flutter test test/sensitive_screen_test.dart && bash scripts/validate-mobile-security.sh && flutter build apk --debug && xcodebuild -workspace ios/Runner.xcworkspace -scheme Runner -sdk iphonesimulator -configuration Debug CODE_SIGNING_ALLOWED=NO build`

Expected: Dart test passes and both platform builds compile.

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/security/sensitive_screen_service.dart lib/features/security/presentation/backup_seed_screen.dart android/app/src/main android/app/src/debug/AndroidManifest.xml ios/Runner/AppDelegate.swift scripts/validate-mobile-security.sh test/sensitive_screen_test.dart
git -C hanbova-app commit -m "fix: protect recovery phrase at platform boundary"
```

### Task 8: Analyzer cleanup, generated version, and dependency cleanup

**Files:**
- Modify: `hanbova-app/lib/core/crypto/crypto_identity_service.dart`
- Modify: `hanbova-app/lib/features/protected_send/presentation/protected_send_screen.dart`
- Create: `hanbova-app/lib/core/app_info/app_info_provider.dart`
- Modify: `hanbova-app/lib/features/profile/screens/profile_screen.dart`
- Modify: `hanbova-app/pubspec.yaml`
- Modify: `hanbova-app/pubspec.lock`
- Test: `hanbova-app/test/app_info_test.dart`

**Interfaces:**
- Produces: `appInfoProvider` backed by `PackageInfo.fromPlatform`; Profile renders package version instead of hardcoded `v0.5.0-beta`.

- [ ] **Step 1: Write failing version test**

```dart
testWidgets('profile renders injected package version', (tester) async {
  await tester.pumpWidget(testProfileApp(packageInfo: PackageInfo(
    appName: 'Hanbova', packageName: 'org.hanbova.hanbova', version: '0.1.0', buildNumber: '1')));
  expect(find.textContaining('v0.1.0 (1)'), findsOneWidget);
  expect(find.textContaining('v0.5.0-beta'), findsNothing);
});
```

- [ ] **Step 2: Run app-info test and baseline analyzer**

Run: `cd hanbova-app && flutter test test/app_info_test.dart && flutter analyze`

Expected: test fails because app-info provider is missing; analyzer reports the two unused crypto locals and async-context lint.

- [ ] **Step 3: Add package info and remove verified dead code/dependencies**

Run: `cd hanbova-app && flutter pub add package_info_plus`

Implement a FutureProvider wrapping `PackageInfo.fromPlatform`, with an overrideable provider for tests. Remove the hardcoded Profile version. Remove unused `savedTransportPriv` and `savedProtectedPriv` reads after confirming deterministic derivation is authoritative. Replace the async-gap navigation check with `if (!context.mounted) return;`. Remove `google_fonts` from `pubspec.yaml` after `rg -n "GoogleFonts|google_fonts" lib test` confirms no uses.

- [ ] **Step 4: Run focused test and analyzer**

Run: `cd hanbova-app && flutter test test/app_info_test.dart && flutter analyze`

Expected: PASS and `No issues found!`

- [ ] **Step 5: Commit**

```bash
git -C hanbova-app add lib/core/app_info lib/core/crypto/crypto_identity_service.dart lib/features/profile/screens/profile_screen.dart lib/features/protected_send/presentation/protected_send_screen.dart pubspec.yaml pubspec.lock test/app_info_test.dart
git -C hanbova-app commit -m "chore: align app metadata and analyzer state"
```

### Task 9: Documentation truth and final cross-system verification

**Files:**
- Modify: workspace `README.md` (the workspace root is not a Git repository)
- Modify: `hanbova-docs/README.md`
- Modify: `hanbova-docs/roadmap.md`
- Modify: `hanbova-docs/product.md`
- Modify: `hanbova-docs/architecture.md`
- Modify: `hanbova-docs/threat-model.md`
- Modify: `hanbova-docs/PROJECT_CONTEXT_CHATGPT_VERIFIED.md`
- Create: `hanbova-docs/verification/RELEASE_HARDENING_20260827.md`

**Interfaces:**
- Consumes: final outputs and exact verification counts from all three plans.
- Produces: one consistent statement of implemented, disabled, experimental, and future capabilities.

- [ ] **Step 1: Capture fresh Flutter evidence**

Run: `cd hanbova-app && dart format --set-exit-if-changed lib test && flutter analyze && flutter test`

Expected: formatting check passes, analyzer has zero issues, all tests pass. Record the exact final test count in the verification document.

- [ ] **Step 2: Capture fresh Rust evidence**

Run: `cd hanbova-backend && cargo fmt --all -- --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace`

Expected: all commands pass without external services. Record exact crate/test counts and the existing SQLx future-compatibility warning.

- [ ] **Step 3: Capture platform and Compose evidence**

Run: `cd hanbova-app && flutter build apk --debug`

Run: `cd hanbova-app && bash scripts/validate-mobile-security.sh`

Run: `cd hanbova-backend && POSTGRES_PASSWORD=test-only JWT_SECRET=test-only-secret-at-least-32-bytes docker compose config >/tmp/hanbova-compose.yml && bash scripts/validate-compose-config.sh /tmp/hanbova-compose.yml`

Expected: debug APK builds and both mobile-security and Compose validations pass.

- [ ] **Step 4: Update documentation from evidence**

Set the Flutter package/profile version to the package-derived value, replace historical test counts with the Step 1/2 counts, mark camera notifications/biometric login/live rates as disabled future work, retain NUT-13 recovery limitations, and document WalletContext, ledger reconciliation, domain policy, production fail-closed rules, and explicit CDK integration command.

`RELEASE_HARDENING_20260827.md` must list commit IDs, commands, exit results, test counts, device/build scope, remaining experimental limitations, and a release decision of “controlled pilot only.”

- [ ] **Step 5: Run documentation consistency checks and commit**

Run: `rg -n "55 tests|69 Flutter|v0\.5\.0-beta|camera scan automatically|Biometric Login.*Enabled|Milestones 1–5.*complete" README.md hanbova-docs hanbova-app/lib`

Expected: no stale claims remain outside explicitly labeled historical records.

```bash
git -C hanbova-docs add README.md roadmap.md product.md architecture.md threat-model.md PROJECT_CONTEXT_CHATGPT_VERIFIED.md verification/RELEASE_HARDENING_20260827.md
git -C hanbova-docs commit -m "docs: record release hardening evidence"
```

The workspace-root `README.md` remains an intentional workspace-level edit because its parent directory has no Git metadata; report that fact explicitly in the final handoff.
