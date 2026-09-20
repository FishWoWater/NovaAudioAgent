# Flutter mobile migration design

Date: 2026-09-20
Status: User approved the written design on 2026-09-20; implementation plan awaits review.

## Intent and baseline

Reproduce the existing iOS application in Flutter, preserving its interface,
interaction, server protocol and feature behavior as closely as possible. Deliver
working iOS and Android clients. The user selected Flutter UI/business logic plus
platform-native audio, with explicit AEC validation and honest reporting of gaps.

Public baseline: v0.2.0dev at a6a5b96bb1ea0f73c1cec2863d121e3dafcb3348.
Branch: feature/flutter-ios-v020. Source: clients/ios/Nova.
Uncommitted source-worktree changes are excluded; preserve them in place.

Build the public implementation first. Downstream private adaptations consume
public commits only; private changes must never flow back into public history.
This document and all public code contain no private deployment configuration.

## Scope and delivery structure

Add clients/flutter with lib/, test/, integration_test/, ios/ and android/.
Keep the Swift application available as the comparison baseline until acceptance.
Flutter owns all application screens and business state, not a WebView or embedded
SwiftUI wrapper. Retain native code only where platform integration warrants it.
No server redesign, model change, desktop migration, background recording, or
new product features are part of this migration.

Use the existing iOS deployment floor initially. Start Android at API 24 or the
selected stable Flutter toolchain minimum, whichever is higher. Record exact
Flutter, Dart, Xcode, Gradle and SDK versions in the implementation manifest.
Raise either floor only for a demonstrated dependency requirement and disclose it.
These platform floors are design choices, not tested compatibility claims.

## Functional parity

Port the current source behavior and protocol tests into an explicit parity ledger:
- Connection settings, manual connection, QR pairing confirmation and cancellation.
- Deployment-configured generic Feishu login, PKCE, callback validation and secure credentials.
- Real-time conversation, transcript streaming/finalization, voice visualization,
  mute, speaker control, start/stop and disconnected/connecting/error states.
- Host-negotiated text input and hold-to-dictate, editable drafts, cancel/timeouts,
  preserving drafts after failure and preventing duplicate sends.
- Approval cards, expiry, decision options, project/session context and results.
- Follow-host/relay/AOQ preferences and capability negotiation.
- Existing language behavior, accessibility semantics, reduced motion and text scaling.

For each item record source location, Flutter implementation, platform differences,
and verification evidence. Do not treat a hidden or unsupported control as parity.
Port timing/limits from the fixed baseline rather than inventing replacements.

## Architecture and ownership

Dart modules: protocol (wire validation and fixtures), connection (WebSocket and
connection generations), conversation (transcript/draft/approval state), UI, and
platform services. Keep dependencies one-way: UI -> state -> protocol/platform.
Use focused Dart state holders and Flutter's existing notification primitives;
introduce no global service framework solely for this migration.

A local audio plugin exposes a typed API to Swift and Kotlin. Control messages
use generated typed bindings where practical; PCM uses bounded binary transfer.
No Base64 per audio buffer and no unbounded Dart or native queue. A stalled Dart
consumer cannot accumulate microphone data indefinitely. Detect sustained overload,
stop the session and surface an actionable error rather than silently degrading.

Audio API: capabilities, startRelay(capture, threshold), enqueue(frame),
terminal(identity), clear(identity), mute, setSpeaker, stop and disconnect.
Events: captured PCM, level, speech onset, actual playback progress/terminal,
route/interruption changes and structured failures. Include session generation,
utterance identity and sequence where applicable; discard obsolete callbacks.

The native layer owns capture, conversion, playback scheduling, interruption,
local speech-onset response and playback accounting. Dart owns connection and
server event routing. Clearing playback invalidates callbacks before stopping the
player. Preserve the difference between accepted, rendered and audibly presented
samples. Never label enqueued audio as heard.

## iOS audio

Adapt existing VoiceAudio, PlaybackLedger and AOQAudio into the plugin. Preserve
AVAudioSession voiceChat, Voice Processing, 16 kHz PCM16 mono capture and 24 kHz
playback, actual played receipts, mute cadence and generation fencing. Resolve
Swift protocol dependencies explicitly instead of copying the entire Client.
Keep audio callbacks independent of Flutter rendering load.

## Android audio

Implement Kotlin capture/playback using AudioRecord and AudioTrack, communication
routing, native bounded queues and timestamps. Select VOICE_COMMUNICATION capture
and inspect AcousticEchoCanceler availability, creation, ownership and enable state.
A reported AEC capability is diagnostic evidence only, not acoustic acceptance.
Resample device-native audio to the unchanged wire format when required.

Track playback using native progress/timestamps; validate presentation semantics
on hardware. If accurate heard-time cannot be established, report the limitation
and block dependent acceptance instead of substituting queue length or wall time.
Handle audio focus, permissions, route changes and lifecycle termination explicitly.
Match existing foreground-only behavior; resume requires an explicit user action.
Do not simultaneously activate independent capture engines or stack AEC processors.
If system AEC fails on a target device, evaluate software AEC as a separate technical
decision including playback reference and delay handling; it is not an assumed fix.

## AOQ transport

Bridge platform SDKs behind the same capability interface. The SDK exclusively owns
capture/AEC/playback while AOQ is active. Preserve ordered control envelopes,
connection identity checks, short-lived credentials and host authority over tools.
Start sending microphone media only at the existing negotiated readiness point.

Official documentation currently lists Android AAR and iOS framework integration.
This establishes availability, not compatibility with the existing pinned SDK,
PCM configuration or Nova control semantics. Verify SDK headers, version, ABI,
licensing/distribution terms and artifacts before choosing pins. Do not upgrade
the server's model or transport configuration just to match an online sample.
AOQ clear/terminal and played-time limitations remain explicit. An unavailable
AOQ adapter must not advertise support. Relay success alone does not close AOQ parity.

## Platform services and failure behavior

Use Keychain on iOS and Keystore-backed encrypted storage on Android. Bind saved
credentials to the intended origin; reject credential forwarding on redirects or
address edits. Keep API keys server-side. Validate QR and login callbacks, cancel
late results after backgrounding or a connection generation change.

Preserve source recovery behavior: eligible transport failures retry after 1, 2 and 4
seconds; refusal codes 4003/4006/4009 do not retry. Reset the retry budget only after
30 seconds of stable readiness. Transport recovery never reopens the microphone
automatically; explicit user action is required to resume voice. Permission denial, native start failure,
SDK timeout, network loss and malformed protocol data release resources and leave
consistent UI state. Errors must be user-readable and must not expose credentials.
Only the existing pairing flow uses the camera; no camera streaming is introduced.

## Visual acceptance

Capture original and Flutter screens at matching logical dimensions, text scale,
language and state. Compare layout, colors, spacing, typography, transcript scrolling,
input gestures, dialogs and animation behavior. Android uses the same product
appearance with working system back/keyboard/insets/accessibility behavior.
System permission dialogs and keyboard rendering are platform-specific exceptions.
Maintain a screen-by-screen difference ledger; no claim of pixel identity across OSes.

## Verification and acceptance

1. Port protocol fixture vectors and assert malformed, duplicate, out-of-order and
   late messages, approval expiry, cancellation, and stale playback callbacks.
2. Exercise real widgets against the existing mock server: pair/connect, input,
   dictation, approval, transcript, clear, disconnect and reconnect.
3. Build iOS and Android; run simulator/emulator flows. These prove neither acoustic
   behavior nor physical SDK transport operation.
4. On physical iPhone and at least two Android devices from different vendors,
   verify relay and AOQ independently with the same host and controlled audio.
5. AEC cases: far-end-only speech, near-end-only, double talk, silence; speaker at
   25/50/75/100 percent, earpiece and Bluetooth; route changes and repeated reconnect.
   Repeat each active case three times for at least 30 seconds. Log device/OS/route,
   volume, false user turns, false interruptions, missed intentional interruptions,
   onset-to-stop latency and transcript contamination. Audio capture needs explicit
   test-consent and must remain local. ERLE only when valid synchronized references
   exist; do not fabricate acoustic measurements from app logs.
6. Initial acceptance: no self-triggered turns during far-end-only trials, all
   deliberate interruptions handled, no stale playback after clear, no crash or
   stuck capture after route/lifecycle changes. Compare iOS against the original
   under identical conditions; report raw Android results by device, not an average
   that conceals a failing model. These trial criteria do not guarantee all devices.
7. Package signed development builds when signing/device access is available;
   otherwise mark install and hardware gates pending, not complete.

## Risks and completion definition

Known unresolved evidence: target-device availability, AOQ build/runtime compatibility,
Android AEC quality, Android audible progress precision, signing and real service
credentials. Report any hard incompatibility as soon as reproduced with alternatives
and impact. No unsupported capability may silently become a mock success.

Completion requires functional and visual parity evidence plus actual working
sessions on both mobile platforms; unit tests and builds are separate milestones.
No push, store publication or deployment is part of this design approval.

## Local environment storage

The user requires newly configured development environments to live on the external
drive with symlinks from conventional local paths. Reuse existing external Flutter,
Android SDK and Gradle cache links. Place new Pub cache, emulator images, build caches
and Xcode derived data on the same external DeveloperStorage volume where supported.
Keep machine-specific absolute paths out of committed application configuration.
Inspect existing data before migration; preserve it, verify the copied tree, then
switch the link. Do not remove an old copy until verification succeeds. Do not move
an environment in active use. If the external volume is absent, fail clearly rather
than silently filling the internal disk. No environment has been migrated in this
specification stage.

## Execution order

Public protocol/state and UI -> iOS adapter -> Android adapter and early AEC gate ->
AOQ parity -> dual-platform acceptance -> downstream private adaptation. Audio
feasibility gates should run early enough to avoid completing UI around an unusable
transport. Detailed commits and commands belong in the subsequent implementation plan.

## References checked 2026-09-20

- https://docs.flutter.dev/platform-integration/platform-channels
- https://developer.android.com/reference/android/media/audiofx/AcousticEchoCanceler
- https://developer.android.com/reference/android/media/MediaRecorder.AudioSource
- https://help.aliyun.com/zh/model-studio/real-time-voice-conversation-using-aoq-access-qwen-audio-3-0-realtime-plus
