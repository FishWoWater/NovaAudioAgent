# Flutter iOS and Android Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the public Nova iOS client behavior in a Flutter app that works on iOS and Android, then make its commits reusable by the private adaptation.

**Architecture:** Dart owns product UI, protocol and session state. A local plugin owns Swift/Kotlin real-time audio and AOQ SDK integration; native playback receipts remain authoritative. Preserve the current server contract and the native app as a comparison reference.

**Tech Stack:** Existing Flutter 3.44.6 / Dart 3.12.2 initially; Swift/AVFoundation; Kotlin/AudioRecord/AudioTrack; AOQ platform SDKs; flutter_test and integration_test; existing Node/TypeScript protocol mock. Use Dart SDK networking and ChangeNotifier; no new global state framework. Evaluate maintained packages only for narrow camera/auth/storage integration, pin selected versions and inspect their native behavior before adoption.

**Spec:** ../specs/2026-09-20-flutter-mobile-design.md

## Global Constraints

- Public baseline: v0.2.0dev at a6a5b96bb1ea0f73c1cec2863d121e3dafcb3348.
- Branch: feature/flutter-ios-v020; worktree: .worktrees/flutter-ios-v020.
- Start Android at API 24 or the selected stable Flutter toolchain minimum, whichever is higher.
- Keep the existing iOS deployment floor, verified as iOS 17.0.
- Flutter owns all application screens and business state, not a WebView or embedded SwiftUI wrapper.
- No server redesign, model change, desktop migration, background recording, or new product features are part of this migration.
- No Base64 per audio buffer and no unbounded Dart or native queue.
- Never label enqueued audio as heard.
- Private changes must never flow back into public history; publish neither branch without authorization.
- Newly configured environments live on the external drive with symlinks from conventional local paths; preserve existing data before migration.
- No source file may exceed 4,000 physical lines; prefer focused modules well below that ceiling.
- Physical-device, service credentials and acoustic acceptance are separate gates from builds/tests. Missing gates remain pending.

## Review Focus

- Permission result arrives after disconnect/background: do not start recording; covered by Task 3.
- Old playback completion arrives after clear/reconnect: do not emit a receipt for the new utterance; covered by Tasks 4 and 5.
- Login callback is duplicated, redirected or delivered after cancellation: never exchange/save stale credentials; covered by Task 6.
- Text acknowledgement arrives after the draft changed: preserve the newer draft and never resend automatically; covered by Task 7.
- Android route change occurs during simultaneous speech and playback: release the old capture/player, show a stopped state and require explicit voice resume; covered by Tasks 5 and 9.

## File map and task dependencies

All paths below are relative to this worktree unless prefixed by $NOVA_EXT.
The machine-local value of NOVA_EXT is /Volumes/sqxh-junjie/DeveloperStorage.
Do not commit that absolute path into application configuration.

- clients/flutter/lib/protocol/{wire,models,pairing,aoq_bridge}.dart: validated server contracts.
- clients/flutter/lib/connection/{session,recovery,transport}.dart: lifecycle, bounded sends and generations.
- clients/flutter/lib/conversation/{conversation_state,input_state,approvals}.dart: user-visible state and commands.
- clients/flutter/lib/services/{credentials,enterprise_login,preferences}.dart: platform integrations.
- clients/flutter/lib/ui/{app,conversation_screen,settings_sheet,approval_card,voice_orb,transcript}.dart: Flutter widgets.
- clients/flutter/lib/l10n/{app_en,app_zh}.arb: source-derived translations.
- clients/flutter/packages/nova_audio/lib/nova_audio.dart: public audio types and adapter interface.
- clients/flutter/packages/nova_audio/{ios,android}: native capture/playback and AOQ implementations.
- clients/flutter/test and integration_test: protocol/state/widget and device checks.
- clients/flutter/docs/{parity,environment,acceptance}.md: evidence and limitations.
- clients/flutter/tool/validate_mobile.sh: reproducible checks; no secrets, deployment or publishing.

Sequence: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> private plan.
Tasks 4/5 precede full UI polish to expose audio infeasibility early.
Each numbered deliverable is independently reviewable; checkboxes are individual actions.
Commit only named task files, never `git add .` across the repository.

## Task 1: External environment and a runnable Flutter shell

**Files:** Create clients/flutter/{pubspec.yaml,lib/main.dart,test/smoke_test.dart,docs/environment.md,docs/parity.md,.gitignore}; generated ios/ and android/ hosts.
**Interfaces:** Produces `NovaApp` as a Widget, package name `nova_mobile`, and a development app ID distinct from the original so both can coexist. No live microphone in the initial shell.

- [ ] Record `git status --porcelain`, installed tool versions, symlink destinations and free space; inspect active Dart/Gradle/emulator processes before migrating a cache. Verify the external mount exists.
- [ ] Reuse current Flutter/Android SDK/Gradle links. Copy the existing Pub cache to `$NOVA_EXT/Caches/pub`, preserving files and links; compare relative paths, SHA-256 hashes and link targets before replacing the local path with a symlink. Keep the verified old tree as a rollback backup outside the active path. If the target already exists, inspect and reconcile instead of overwriting it. Request sandbox access for the authorized external writes when needed.
- [ ] Set a task-local environment with `PUB_CACHE`, `GRADLE_USER_HOME`, `ANDROID_SDK_ROOT`, and `ANDROID_AVD_HOME` pointing to external locations; keep AVD data separate from the user's adb keys. Put build output and Xcode DerivedData in external per-worktree directories. Do not move shared active DerivedData. Check Flutter's project build-directory support before choosing an ignored `build` symlink; verify resolution before the first build.
- [ ] Run `flutter doctor -v`, `java -version`, `xcodebuild -version`, `adb devices -l`, and `xcrun simctl list devices available`; write exact versions and missing gates to docs/environment.md. Install only missing dependencies onto the external drive. Do not record device serials in committed public docs.
- [ ] Scaffold with `flutter create --platforms=ios,android --org com.nova --project-name nova_mobile clients/flutter`. Set iOS 17.0 and Android minSdk 24, or explain a higher actual Flutter requirement. Use a separate development bundle/application ID; keep the original install available.
- [ ] Replace the generated counter test with this initial failing test, and run `flutter test test/smoke_test.dart` inside clients/flutter:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:nova_mobile/main.dart';
void main() {
  testWidgets('starts disconnected without microphone activation', (tester) async {
    await tester.pumpWidget(const NovaApp());
    expect(find.text('NOVA'), findsOneWidget);
    expect(find.text('Not connected'), findsOneWidget);
  });
}
```

- [ ] Implement only a disconnected shell to make that test pass:

```dart
import 'package:flutter/material.dart';
void main() => runApp(const NovaApp());
class NovaApp extends StatelessWidget {
  const NovaApp({super.key});
  @override
  Widget build(BuildContext context) => const MaterialApp(
    home: Scaffold(body: Column(children: [Text('NOVA'), Text('Not connected')])),
  );
}
```

- [ ] Run `flutter analyze`, the smoke test and debug builds using the external output paths. Create parity.md rows for each source view and each `Client` event/action, including status and evidence columns. Commit `feat(mobile): add isolated Flutter application shell` with the files above and required generated host files, excluding caches/local.properties/signing data.

## Task 2: Exact Dart protocol and transcript contracts

**Files:** Create lib/protocol/{wire,models,pairing,aoq_bridge}.dart; lib/conversation/conversation_state.dart; test/{wire,transcript,pairing,aoq_bridge}_test.dart.
**Source:** clients/ios/Nova/Nova/Protocol/{Wire,PairingCode}.swift; fixtures/client-protocol/v1/vectors.json; NovaTests/ProtocolTests.swift.
**Interfaces:** `Wire.audio(Uint8List) -> AudioFrame`, `Wire.json(Uint8List, {int limit = 16384}) -> Map<String,dynamic>`, `Wire.endpoint(String, {bool debugLocalhost = false}) -> Uri`; `AudioFrame` fields `identity: PlaybackIdentity`, `sequence: int`, `pcm: Uint8List`; identity fields `utteranceId: String`, `epoch: int`. `ConversationState.receive({required String role, required String text, required bool finalText, String? id}) -> void`, `messages` exposes immutable message views.

- [ ] Add this fixture-driven failing test; read the existing fixture, do not duplicate its vectors:

```dart
import 'dart:convert';
import 'dart:io';
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:nova_mobile/protocol/wire.dart';
Uint8List unhex(String s) => Uint8List.fromList([
  for (var i = 0; i < s.length; i += 2) int.parse(s.substring(i, i + 2), radix: 16),
]);
void main() {
  final vectors = jsonDecode(File('../../fixtures/client-protocol/v1/vectors.json').readAsStringSync()) as List;
  for (final v in vectors) {
    test(v['name'] as String, () {
      final bytes = unhex(v['hex'] as String);
      if (v['valid'] != true) {
        expect(() => Wire.audio(bytes), throwsFormatException);
      } else {
        final frame = Wire.audio(bytes);
        expect(frame.identity.utteranceId, v['expected']['utterance_id']);
        expect(frame.identity.epoch, v['expected']['generation_epoch']);
        expect(frame.sequence, v['expected']['sequence']);
        expect(frame.pcm, orderedEquals(unhex(v['expected']['pcm_hex'])));
      }
    });
  }
  test('floating point epoch is rejected', () {
    expect(() => Wire.json(Uint8List.fromList(utf8.encode('{"generation_epoch":1.0}'))), throwsFormatException);
  });
}
```

- [ ] Run `flutter test test/wire_test.dart` and observe the missing implementation failure.
- [ ] Port validation constants and exact source behavior: magic NOVA, 2-byte big-endian header length 2..2048, nonempty even PCM <=65536 bytes, integer limit 9007199254740991, epoch >=1, identifier scalar limits, handshake capabilities and format checks. Dart distinguishes int from double; additionally reject negative-zero integer tokens as the source does. Cover escaped/duplicate JSON keys and last-member behavior before replacing the Swift lexical check with a simpler parser.

```dart
final headerSize = ByteData.sublistView(bytes).getUint16(4, Endian.big);
// Bounds are validated before taking either sublist.
final header = Wire.json(Uint8List.sublistView(bytes, 6, 6 + headerSize), limit: 2048);
final pcm = Uint8List.sublistView(bytes, 6 + headerSize);
```

- [ ] Add source-equivalent transcript tests: a final message cannot be overwritten by a stale partial; same ID updates in place; no-ID partials combine only with the last nonfinal same-role message. Port pairing size/expiry/code checks, AOQ sequence/allowlist/envelope limits and pending-command byte limits.
- [ ] Run all four test files; update parity evidence. Commit `feat(mobile): port wire and conversation contracts`.

## Task 3: Session lifecycle, bounded transport and permission races

**Files:** Create lib/connection/{session,recovery,transport}.dart; packages/nova_audio/{pubspec.yaml,lib/nova_audio.dart}; test/{session,recovery}_test.dart; test/support/{fake_audio,fake_transport}.dart.
**Source:** Client.swift `open`, `receive`, `command`, `enqueue`, `failed`, `clearConnection`, `end`, `startVoice`; Wire.swift ConnectionRecovery.
**Interfaces:** `Recovery.markReady(int nowMs)`, `Recovery.nextDelay(int nowMs) -> Duration?`; `Session.connect(Uri,String)`, `Session.end()`, `Session.startVoice()`, `Session.background()`, `Session.receive(Object,int generation)`; `Session` exposes `connected`, `voice`, `generation`. Inject `AudioPort`, `TransportPort`, a monotonic `int Function()` clock and `Future<bool> Function()` microphone permission request. TransportPort exposes incoming stream, bounded send and close. Fakes implement these same interfaces and expose sent messages/start counts without opening sockets or microphone.

Audio contract in nova_audio.dart:

```dart
abstract interface class AudioPort {
  Stream<Map<String, Object?>> get events;
  Future<Map<String, Object?>> capabilities();
  Future<void> startRelay({required int generation, required bool capture, required double threshold});
  Future<void> enqueue(Uint8List frame);
  Future<void> terminal(String utteranceId, int epoch);
  Future<void> clear(String utteranceId, int epoch);
  Future<void> mute(bool muted);
  Future<void> setSpeaker(bool enabled);
  Future<void> stop();
  Future<void> disconnect();
}
```

The enqueue bytes are the validated NOVA frame so native code receives identity,
sequence and PCM together. Events carry generation and typed payload fields; publish
an exhaustive event schema alongside this interface. PCM crosses a BinaryMessenger
channel with one in-flight packet and acknowledgement, never an unbounded EventChannel.

- [ ] Write recovery tests with concrete expected values:

```dart
test('repeated ready-malformed cycles exhaust recovery', () {
  final r = Recovery();
  expect(r.nextDelay(0), const Duration(seconds: 1));
  r.markReady(1000);
  expect(r.nextDelay(1001), const Duration(seconds: 2));
  expect(r.nextDelay(1002), const Duration(seconds: 4));
  expect(r.nextDelay(1003), isNull);
});
```

- [ ] Run `flutter test test/recovery_test.dart`; implement the 1/2/4 second budget and 30-second stable-ready reset. Refusal codes 4003/4006/4009 bypass retries. A recovered transport never automatically calls startRelay.
- [ ] Write a Session test with a Completer<bool> permission future: mark transport ready, invoke startVoice without awaiting, call end, complete permission true, then await startVoice. Assert fakeAudio.startCount == 0, session.voice == false. Repeat with background and with an old generation's ready/PCM events.
- [ ] Port session lifecycle and wire routing. Preserve original queue limits and watchdog durations from Client.swift; reject overflow with a session failure. After every asynchronous boundary recheck generation and active foreground state:

```dart
final attempt = generation;
final granted = await requestMicrophone();
if (!granted || generation != attempt || !connected || !foreground) return;
await audio.startRelay(generation: attempt, capture: true, threshold: speechThreshold);
```

- [ ] Add fake transport tests for queue overflow, malformed handshake, explicit end cancelling retries, refusal codes, AOQ host heartbeat loss and UI state after native start failure. Run `flutter test test/session_test.dart test/recovery_test.dart` and `flutter analyze`. Commit `feat(mobile): preserve session lifecycle and recovery`.

## Task 4: iOS relay audio bridge with real playback accounting

**Files:** Create packages/nova_audio/ios/Classes/{NovaAudioPlugin,VoiceAudio,PlaybackLedger,AudioWire}.swift and native test target/files; lib implementation of AudioPort channel adapter; test/audio_channel_test.dart.
**Source:** Native VoiceAudio.swift, PlaybackLedger.swift, Wire.swift identity/frame helpers. Copy only cohesive audio/protocol helpers, not Swift Client or SwiftUI.
**Interfaces:** Implements AudioPort and event schema from Task 3. AudioWire decodes the same NOVA bytes. PCM messages are binary; send one then await acknowledgement. Timer/clear/played calculations remain native.

- [ ] Port existing Swift ledger tests into the plugin's native test target. Add the stale-ticket regression:

```swift
func testOldCompletionCannotCreditNewPlayback() {
    var ledger = PlaybackLedger()
    let first = PlaybackIdentity(utteranceID: "first", epoch: 1)
    XCTAssertTrue(ledger.accept(AudioFrame(identity: first, sequence: 0, pcm: Data(repeating: 0, count: 480))))
    let oldTicket = ledger.ticket
    ledger.clear(first)
    let second = PlaybackIdentity(utteranceID: "second", epoch: 2)
    XCTAssertTrue(ledger.accept(AudioFrame(identity: second, sequence: 0, pcm: Data(repeating: 0, count: 480))))
    ledger.rendered(240, ticket: oldTicket)
    XCTAssertEqual(ledger.renderedSamples, 0)
}
```

- [ ] Run the native test target and observe failure before adding the port. Retain fixture tests for native frame decoding.
- [ ] Bridge start/stop/clear/mute/speaker with Swift async completion, structured errors and session generations. Preserve these engine settings and the source capture guard:

```swift
try session.setCategory(.playAndRecord, mode: .voiceChat, options: [.defaultToSpeaker, .allowBluetooth])
try engine.inputNode.setVoiceProcessingEnabled(true)
```

- [ ] Preserve source played-back completion callbacks and downstream latency subtraction; unknown latency does not invent progress. Clear invalidates tickets before player.stop. Native speech onset clears playback before notifying Dart. Preserve synthetic muted PCM pacing and skipped timer ticks.
- [ ] Add a fake BinaryMessenger test that withholds PCM acknowledgement and verifies bounded native handoff; integration-run it with debug diagnostics. Verify disconnect cancels outstanding delivery and cannot send a late PCM packet into a new session.
- [ ] Run native unit tests, Dart channel tests and an unsigned iOS build. Run simulator lifecycle smoke without claiming AEC acceptance. Commit `feat(mobile): bridge iOS relay audio`.

## Task 5: Android relay audio, AEC probe and route lifecycle

**Files:** Create packages/nova_audio/android/build.gradle; src/main/kotlin/com/nova/audio/{NovaAudioPlugin,RelayAudio,PlaybackLedger,AudioWire,AudioRoute}.kt; src/test/kotlin/com/nova/audio/{PlaybackLedgerTest,AudioWireTest}.kt; src/androidTest/kotlin/com/nova/audio/RelayLifecycleTest.kt.
**Interfaces:** Same AudioPort channel names/event schema as Task 4. Kotlin ledger mirrors Swift accept/render/clear/disconnect semantics with Long sample counters; no signed 32-bit epoch narrowing. Use AudioTimestamp/playback head with wrap handling and an explicit presentation-valid flag.

- [ ] Write Kotlin tests for old completion generation, queue bound, negative/unsafe identity, sequence discontinuity and 32-bit playback-head wrap. Use the same ten NOVA fixture vectors through test resources generated from the repository fixture.

```kotlin
@Test fun clearedEpochCannotBeAcceptedAgain() {
    val ledger = PlaybackLedger()
    val first = PlaybackIdentity("first", 1L)
    ledger.clear(first)
    assertFalse(ledger.accept(AudioFrame(first, 0L, ByteArray(480))))
}
```

- [ ] Run `./gradlew :nova_audio:testDebugUnitTest` from clients/flutter/android after Flutter plugin registration; verify failure, then port ledger/parser constants and implement native serialized ownership.
- [ ] Build capture with permission already granted, VOICE_COMMUNICATION input and device-supported rates; convert to 16 kHz PCM16 mono. Build playback for 24 kHz mono or a validated output conversion path. Size buffers from native minimums and preserve bounded handoff.

```kotlin
val effect = AcousticEchoCanceler.create(recorder.audioSessionId)
val available = AcousticEchoCanceler.isAvailable()
if (effect != null && effect.hasControl()) effect.enabled = true
// Publish availability, control and actual enabled status; these are not an AEC pass.
```

- [ ] Set communication audio mode and implement speaker/earpiece/Bluetooth routing with API-version guards. On route/focus/lifecycle loss invalidate the session token, stop capture/player, release effect/resources and report stopped; never auto-resume microphone. Do not run two AEC processors concurrently.
- [ ] Implement muted zero packets, native speech-onset detection and conservative playback reports. If timestamps cannot prove actual presentation, expose the missing evidence and leave heard-time parity open; do not fabricate it.
- [ ] Run JVM tests, instrumented lifecycle tests and `flutter build apk --debug`. On an available physical Android device run early relay/AEC smoke before proceeding to polish. Log missing hardware as pending and continue independent work. Commit `feat(mobile): add Android relay audio adapter`.

## Task 6: Pairing, secure credentials and deployment-configured login

**Files:** Create lib/services/{credentials,enterprise_login,preferences}.dart; lib/ui/pairing_scanner.dart; test/{credentials,enterprise_login,pairing_flow}_test.dart; native camera/auth/storage adapter files or pinned narrow packages; AndroidManifest.xml and iOS Info.plist callback/permission configuration.
**Interfaces:** `Credential` stores server/token/expiry; `CredentialStore.read()`, `write(Credential)`, `clear()` return Futures; `EnterpriseLogin.start() -> Future<Credential?>`, `cancel() -> void`; `EnterpriseSSO.callback(Uri,{required String state}) -> String`, `origin(String) -> Uri`. Session consumes validated credentials, not raw callback URLs.

- [ ] Add tests for duplicate state, wrong host/scheme, expired exchange, redirect, oversize body and late callback:

```dart
test('duplicate callback state is rejected', () {
  final code = List.filled(43, 'a').join();
  expect(() => EnterpriseSSO.callback(
    Uri.parse('nova-sso://callback?state=s&state=s&code=$code'), state: 's'),
    throwsFormatException);
});
```

- [ ] Run the new tests and confirm failure. Port source constraints: HTTPS bare login origin, random 32-byte state/verifier, SHA-256 PKCE, exactly two callback query items, 43-character code, same-origin wss credential, bounded bodies, 5-minute login expiry and no redirects on exchange.

```dart
final request = await client.postUrl(exchangeUri);
request.followRedirects = false;
request.headers.contentType = ContentType.json;
request.write(jsonEncode({'code': code, 'code_verifier': verifier, 'device_name': deviceName}));
```

- [ ] Implement Keychain and Keystore-backed encrypted persistence; test lock/write failures leave no successful connection claim. Test editing the server never forwards a saved bearer to another origin. Keep settings without secrets in platform preferences.
- [ ] Implement native web authentication on iOS and browser callback flow on Android, plus camera-only QR scan with preview confirmation. Cancel/timeout/background increments attempt identity; late results cannot save or connect. System callback entry points forward validated data only.
- [ ] Run unit/widget tests and real simulator/emulator browser/camera-permission lifecycle checks where available. Record actual camera scan and deployment login as physical/service gates. Commit `feat(mobile): port pairing and secure login`.

## Task 7: Full Flutter screens, input and approval behavior

**Files:** Create lib/ui/{app,conversation_screen,settings_sheet,approval_card,voice_orb,transcript}.dart; lib/conversation/{input_state,approvals}.dart; l10n/app_en.arb and app_zh.arb; test/{input_state,approvals,conversation_widget,settings_widget}_test.dart; integration_test/client_flow_test.dart; tool/mobile_mock.mjs.
**Source:** ContentView.swift and localized strings, Client.swift receiveHost and input methods. Reuse the production ClientServer in a separate synthetic test harness; do not change the original mock's behavior or production protocol.
**Interfaces:** `InputState.draft`, `beginDictation`, `finishDictation`, `cancelDictation`, `sendDraft`, and request-ID-correlated receive handlers; `ApprovalCard.actionable(DateTime)` and `decide` preserve source decisions. NovaApp accepts an injected Session/state for tests; production creates real adapters.

- [ ] Write InputState test: draft A is sent, user edits to B, applied acknowledgement for A arrives; assert draft remains B. Test transcription after cancel/disconnect cannot edit draft. Preserve 4000 UTF-16-unit limit and source 45-second capture/35-second transcription/15-second acknowledgement timers.

```dart
final pendingText = draft.trim();
// When the matching applied acknowledgement arrives:
if (draft.trim() == pendingText) draft = '';
// On timeout keep the draft and explain uncertainty; never automatically resend.
```

- [ ] Write approval tests for missing/expired deadline, unknown decision, repeated tap, replaced approval and acceptForSession mapping. Run these failing tests before the state port.
- [ ] Implement state reducers and all source-supported screen states. Retain dark ink/mint appearance, voice visualization, scrolling behavior, mode switch, hold/release dictation, settings and accessible labels. Layout uses safe areas and keyboard insets; Android system back dismisses the current modal first. Translate exact existing public strings; system permission instructions name the running platform.

```dart
final ink = const Color.fromRGBO(9, 14, 20, 1);
final mint = const Color.fromRGBO(148, 235, 217, 1);
// Fixed brand colors, flexible layout; text scaling must not hide actions.
```

- [ ] Add widget tests at phone portrait/landscape sizes and 2x text scale; assert primary controls remain reachable, expired approvals disabled, voice stops on mode switch, reduce-motion disables decorative animation and semantic labels distinguish mute from unmute.
- [ ] Extend a new synthetic mock harness using existing runtime exports for cascaded text/dictation, approval outcomes and failure injection. Configure host capabilities truthfully. Never emit mock transcription in production. Android emulator uses `adb reverse tcp:18787 tcp:18787` so the app can keep the loopback-only debug policy.
- [ ] Run `flutter test`, `flutter analyze`, and integration_test/client_flow_test.dart on each available simulator/emulator. Compare original and Flutter screen captures at matching dimensions; store synthetic visual evidence externally and commit only parity.md findings. Commit `feat(mobile): reproduce conversation and settings flows`.

## Task 8: AOQ SDK bridges and strict transport ownership

**Files:** Create packages/nova_audio/ios/Classes/AOQAudio.swift; android/src/main/kotlin/com/nova/audio/AoqAudio.kt; lib/protocol/aoq_bridge.dart updates; packages/nova_audio/lib/aoq_port.dart; tool/fetch_aoq.py; test/aoq_lifecycle_test.dart; native SDK adapter tests.
**Interfaces:** `AoqPort.start(Map<String,Object?> payload,{required int generation,required bool runtime})`, `command(Map<String,Object?>)`, `mute(bool)`, `setSpeaker(bool)`, `clear()`, `stop()` return Futures; events contain generation and validated nonaudio SDK JSON. SDK availability is a capability, not a hardcoded true.

- [ ] Download official SDKs only during execution into external ignored storage. Inspect headers/AAR classes, checksums, supported ABI, license/distribution terms and source compatibility. Record exact versions and SHA-256 in the fetch manifest. Preserve the existing pinned iOS SDK if it satisfies the adapter; do not upgrade a model or server setting to fit the latest sample.
- [ ] Write lifecycle tests: relay stopped before AOQ start, no media before session.updated, late ready after stop ignored, sequence gap rejected, oversized command fails, host heartbeat loss releases SDK. Mock the SDK adapter only in tests.

```dart
final before = generation;
await relay.stop();
if (before != generation) return;
await aoq.start(payload, generation: before, runtime: runtimeMode);
```

- [ ] Port the existing Swift AOQ behavior and Kotlin equivalents using verified SDK signatures. Keep fresh credentials per connection, separate chat/runtime negotiation and command allowlist; SDK owns capture/AEC/playback exclusively.
- [ ] Preserve clear semantics without inventing played_ms or speech-heard receipts. Simulator builds advertise only compiled/usable transports. SDK initialization failures produce explicit errors and truthful capability state; no silent success or fake fallback.
- [ ] Build both device targets, run native/Dart tests and validate live data/control/voice on physical devices with host-issued credentials if available. Report SDK incompatibilities immediately with exact API/ABI evidence. Commit `feat(mobile): bridge AOQ on iOS and Android`.

## Task 9: Reproducible acceptance and reusable public commits

**Files:** Create/update docs/{acceptance,parity,environment}.md; tool/validate_mobile.sh; integration_test/audio_lifecycle_test.dart; local external evidence directory excluded from Git.
**Interfaces:** validate_mobile.sh exits nonzero on a failed runnable check; unavailable hardware is recorded as pending in acceptance.md and never silently counted as passed.

- [ ] Build runtime serially with the repository's documented build command before starting `node runtime/scripts/client-protocol-mock.mjs --port=18787`; keep runtime/dist changes out of commits. Run protocol/state/widget/native tests once, then integration tests against the synthetic harness.
- [ ] Implement the validation runner with explicit commands and fail-fast status:

```sh
#!/bin/sh
set -eu
flutter analyze
flutter test
flutter build apk --debug
flutter build ios --debug --no-codesign
```

Native and device commands are separate named gates so absent devices are visible.
Use external output paths verified in Task 1, and run iOS/Android builds serially.

- [ ] Execute the spec's AEC matrix on one iPhone and at least two Android vendors: near/far/double speech, silence, 25/50/75/100% volume, earpiece/Bluetooth, route switches and reconnect. Three trials per case, >=30 seconds each. Capture synchronized audio only with explicit test-recording consent; otherwise record observable events and do not claim ERLE.
- [ ] Add an instrumented test for route change during playback/capture: before change one engine is active; after change capture inactive, obsolete callbacks discarded, Flutter shows stopped; explicit Start creates exactly one new engine. Manually verify on real Bluetooth hardware as a separate gate.
- [ ] Record false turns, false/missed interruptions, onset-to-stop latency and transcript contamination by device and route. Compare original iOS and Flutter under the same conditions. No self-triggered far-end turns, all deliberate interruptions handled, no stale audio or stuck capture are required for these controlled trials to pass.
- [ ] Install development builds when signing/devices permit; report OS/device evidence and actual relay/AOQ sessions. Produce a limitations table distinguishing unavailable hardware, unverified service access and reproduced incompatibility. Do not call either mobile platform usable solely from a build.
- [ ] Audit the public diff and reachable migration commits for private material, secrets and generated caches. Commit `test(mobile): record dual-platform migration acceptance`. Produce an explicit ordered list of public Flutter commit SHAs for the private plan; omit unrelated repository changes. Leave all changes local.

## Plan self-review and execution handoff

Coverage: parity/UI -> 2/6/7; session/errors -> 3; iOS -> 4; Android/AEC -> 5/9;
AOQ -> 8; storage/environment -> 1; privacy/reuse -> 9 and the private plan.
Review-focus cases have owners above. No task may convert a pending physical gate
into a passed automated test. Source fixes needed to preserve parity must be documented.

Recommended execution method: Native, sequential implementation in this session,
with one independent final review after the branch is ready. Shared audio/session
interfaces make serial integration efficient; early native gates limit rework.
The user must review this written plan and select the execution method before code
implementation, as required by the explicitly invoked brainstorming workflow.
