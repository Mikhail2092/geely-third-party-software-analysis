# Hey v1.2.4 Unlocked Build Static Analysis Report

## 1. Report Scope and Conclusions

This report is based only on static APK unpacking, Manifest/resource decoding, DEX decompilation, smali review, visible Native ELF exports/strings, and differential comparison with APKs in the same directory. The APK was not installed, launched, connected to a vehicle, allowed to access a server, or permitted to execute any of its code.

Core conclusions:

- **Code-confirmed**: Hey is an in-vehicle "external public-address/amplification" application. Its main pipeline consists of real-time microphone capture, 48 kHz PCM processing, software DSP/voice effects, and playback through a selected vehicle audio route.
- **Code-confirmed**: In addition to real-time public address, it implements text-to-speech, local WAV recording, local music playback, five voice effects, a draggable floating button, steering-wheel OK-key long-press triggering, log-keyword triggering, and external broadcast commands.
- **Code-confirmed**: Steering-wheel triggering connects to VHAL through a local plaintext gRPC connection. The default target is `localhost:40004`; it subscribes to a vehicle-property stream and listens to property `0x21407439` by default. A first `int32` value of `4` is treated as a long press of the OK key.
- **Code-confirmed**: The sample is a modified "unlocked build." In the license manager class `defpackage.match`, the initial activation check and activation-code acceptance method directly return `true`, bypassing the initial inactive branch in `onCreate`. However, `onResume` still invokes Native session validation, and the startup coroutine still performs a blacklist review; either path may enter the activation page. The original activation, migration, heartbeat, and remote-configuration code remains in the APK.
- **Code-confirmed**: The "Starray" variant in the same directory has identical DEX, images, layouts, Native libraries, and signing certificate. Its substantive differences are an additional system-window permission and a set of smaller interface-dimension resources.

## 2. APK Identity

| Field | Value |
|---|---|
| Original file | `Hey_v1.2.4_разблокированный_подписанный.apk` |
| File size | 4,462,006 bytes |
| SHA-256 | `A0E5995C40B4FDCD2D238795DE193C3390DB8EF2380C0ECC1FC0C837B83E5A8B` |
| Package name | `com.kooo.hey` |
| Application name | `Hey!` |
| Version | `1.2.4` / versionCode `26` |
| Android range | minSdk `28`, targetSdk `36`, compileSdk `36` |
| ABI | `arm64-v8a`, `x86_64` |
| Signing certificate SHA-256 | `BB:CA:73:98:98:13:6E:6F:84:77:AF:0D:32:53:AE:10:B5:B6:DE:C2:F6:3D:D4:6D:34:FB:5E:AF:83:6C:16:3F` |

This signing certificate matches the current signing certificates of `EVCC` and `EVCC mini` in the directory. This proves only that the current samples were signed with the same key; the certificate alone cannot establish that they came from the same packaging batch or identify the original developer.

## 3. Application Components and Startup Chain

### 3.1 Activities

| Component | Purpose | Static evidence |
|---|---|---|
| `MainActivity` | Main interface, public-address state machine, TTS/music/voice-effect panels, and external-command dispatch | Manifest, `activity_main.xml`, `MainActivity.java` |
| `ActivationActivity` | Device ID, manual/automatic activation, device migration, and purchase QR code | `activity_activation.xml`, `ActivationActivity.java`, `visit.java` |
| `DebugActivity` | Audio routing, speaker amplifier, DSP, triggers, logs, and upload configuration | `activity_debug.xml`, `include_debug_left.xml`, `DebugActivity.java` |

`MainActivity` also declares three Intent actions, `START_TALK`, `STOP_TALK`, and `TOGGLE_TALK`, so it serves both as the launch page and as the shortcut-command entry point.

### 3.2 Services and Receivers

| Component | Implementation responsibility |
|---|---|
| `FloatingWindowService` | Creates a draggable floating microphone button, sends a public-address toggle command to `MainActivity` when tapped, and saves X/Y position, size, and opacity. |
| `MusicPlaybackService` | Foreground notification, WakeLock, and stop action for local music playback. |
| `HeyMonitorService` | Maintains steering-wheel/VHAL gRPC monitoring and toggles public address after a long press of the OK key. |
| `LogcatTriggerService` | Continuously reads system logs using a user-configured keyword and starts public address on a match. |
| `BootReceiver` | Checks activation and `enable_hard_key` after boot, then starts `HeyMonitorService` when the conditions are met. |
| `HeyCommandReceiver` | Receives external broadcast commands and converts them into explicit `MainActivity` Intents. |

## 4. Complete Recovery of User Functions

### 4.1 Real-Time Public Address

- Tapping the main button starts a timed public-address session; resource text explicitly specifies 60 seconds.
- Holding the main button first enters a "charging" prompt. Releasing it enters continuous public address; tapping again while continuous mode is active ends it.
- The state machine includes at least idle, starting, timed public address, continuous public address, stopping/error, and related stages. `CountdownRingView`, `AudioGlowView`, and `GainSliderView` provide countdown, audio-glow, and gain interactions.
- On start, the application requests audio focus, starts an outside-speaker amplifier session, and creates `AudioRecord`/`AudioTrack`; on stop, it terminates the audio pipeline, abandons focus, and releases the amplifier session.
- After returning to idle, audio resources are released with a delay; the default delay in code is 10 seconds.

### 4.2 Real-Time Audio Processing

**Code-confirmed base format:**

- Sample rate: 48,000 Hz.
- Format: PCM 16-bit.
- Input/output: mono (`AudioRecord` channel `16`, `AudioTrack` channel mask `4`).
- Default audio source: `7`.
- Default AudioAttributes usage: `73`; the debug panel can change it to any value and test values `0..100` sequentially.
- Android input and output `AudioDeviceInfo` IDs can be selected, using `setPreferredDevice` to target a specific bus.

**Software DSP pipeline:**

- Volume gain defaults to `5.0`, with a default permitted range of `1.0..30.0`; the debug page can lower the minimum to `0.1` and raise the maximum to `100`.
- Band-pass filtering is enabled by default; default high-pass cutoff is `300 Hz`, and low-pass cutoff is `4000 Hz`. The code performs 48 kHz biquad filter calculations.
- Noise gate with configurable switch; default strength is `0.5`.
- AGC anti-feedback with configurable switch.
- Frequency-shift anti-feedback with a default offset of `5 Hz` and a configurable range of `1..10 Hz`; the implementation includes a 63-tap FIR processing structure.
- Reverb has three levels: none, moderate, strong.
- Ecarx hardware DSP `workmode`: off/noise suppression/echo cancellation/wake-up optimization. It first writes system settings and, on failure, can execute `settings put/get system workmode` through the bundled ADB client.

### 4.3 Voice Effects

The UI offers five modes:

1. Original voice
2. Mature male
3. Young girl
4. Robot
5. Electronic voice

`VoiceEffectBridge` loads `libvoiceeffect.so` and exposes `nativeCreate(48000)`, `nativeSetEffect`, `nativeProcess`, `nativeFlush`, and `nativeDestroy`. The Java layer sends effect numbers `0..4` to the Native processor in real time. Because the core algorithm is in a compiled C++ ELF, the effect entry points and processing flow are confirmed, but the complete mathematical parameters for each mode cannot be recovered from Java alone.

### 4.4 TTS and Custom Recording

- The user can enter text and save up to 20 items; the list is stored as a JSON array in `hey_config/tts_items`.
- Text entries are played through Android `TextToSpeech`; `tts_speech_rate`, `tts_pitch`, and `tts_volume` configure speech rate, pitch, and volume.
- The application records 48 kHz/16-bit/mono WAV files in the public download directory `Download/Hey/recordings/rec_yyyyMMdd_HHmmss.wav`.
- Recording entries are distinguished in the configuration list using the form `rec:<absolute-path>` and can be played, deleted, and renamed.
- TTS/WAV playback also uses configurable Audio usage and an amplifier session; it is not merely an ordinary phone-media route call.

### 4.5 Local Music

- The default directory is `Download/Hey/music`; `music_folder` can be changed on the debug page.
- Recognized formats are `mp3`, `wav`, `flac`, `ogg`, `m4a`, `aac`, and `wma`.
- The application implements a track list, selection, previous/next, progress seeking, play/pause, and playlist/single-track/off repeat modes.
- Android `MediaPlayer` is preferred; if hardware/system decoding fails, playback falls back to a `MediaExtractor` + `MediaCodec` software-decoding pipeline.
- `MusicPlaybackService` maintains a foreground notification and WakeLock; playback can be stopped directly from the notification.

### 4.6 Shortcut Triggers

1. **Floating button**: size is adjustable within `32..120 dp`, default `56 dp`; opacity defaults to `0.7` and is adjustable within `0.1..1.0`; drag coordinates are saved as `floating_x/y`.
2. **Steering-wheel long press**: controlled by `enable_hard_key`, implemented through VHAL gRPC, and monitoring can be restored automatically after boot.
3. **Log keyword**: controlled by `enable_logcat_trigger`, with keyword `logcat_trigger_keyword`; a match launches the home page with the `START_TALK` action.
4. **Automatic public address in foreground**: `enable_auto_talk_on_foreground` starts a 60-second public-address session immediately when launched from the icon.
5. **Default function**: `launch_action=0/1/2` corresponds to public address/TTS/music; TTS or music can be configured to play a selected item automatically after launch.
6. **Broadcast/Intent**: see Section 6.1; other applications can explicitly trigger start, stop, and toggle.

### 4.7 Debugging and Head-Unit Adaptation

`DebugActivity` is not merely a log viewer, but a complete head-unit adaptation panel:

- Enumerates microphone/output devices and selects the audio source, usage, and input/output device IDs.
- Plays a 440 Hz/2-second test tone and displays the actual routing device; it can iterate through usage values `0..100`.
- Displays the microphone waveform and can directly perform a test public-address session.
- Queries, automatically matches, and sets the vehicle volume-group ID/volume.
- The external amplifier supports enable, disable, and local managed-state queries; HIDL transaction code is configurable and defaults to `55`. The status button reads the in-process amplifier flag, session table, and delayed-shutdown callback; it does not issue a HIDL query for the actual hardware state.
- Adjusts reverb, gain limits, band-pass filtering, noise gate, frequency shift, AGC, and Ecarx DSP workmode.
- Copies/clears the current log, uploads the current or previous log, and allows a device alias and problem description to be entered.
- The APK bundles a Java ADB TCP client. It attempts to connect to `127.0.0.1:5555` and local network-interface addresses, generates/saves its own RSA ADB key, and executes head-unit configuration commands over the `shell:` channel. This implementation is statically confirmed; the channel was not connected or executed during this analysis.

## 5. Key Business Flows

### 5.1 Ordinary Public Address

1. `MainActivity` checks microphone permission and the current state.
2. It reads usage, device IDs, gain, filtering, noise gate, AGC, reverb, frequency shift, and voice-effect parameters from `hey_config`.
3. It requests audio focus; when usage is `72`, `AmpManager` enables the external amplifier through `vendor.ecarx.xma.audiocontrol@1.0::IEcarxAudioControl` HIDL.
4. It creates 48 kHz mono `AudioRecord` and `AudioTrack` instances and binds the configured input/output devices whenever possible.
5. It repeatedly reads PCM, sequentially applies gain, filtering, noise gate/AGC, reverb/frequency shift, and Native voice effects, then writes to `AudioTrack`.
6. The short-press flow ends through a 60-second `CountDownTimer`; continuous mode ends when the user taps again.
7. It stops recording/playback, abandons audio focus, and releases the amplifier session; under usage `72`, the amplifier shuts down after a 60-second delay when no other session remains.

### 5.2 Steering-Wheel OK-Key Trigger

1. A boot broadcast or user configuration starts `HeyMonitorService` in the foreground.
2. The service reads `grpc_host=localhost`, `grpc_port=40004`, and `hard_key_vhal_prop_id=0x21407439`.
3. It creates a plaintext gRPC channel with a random UUID `session_id` and fixed `client_id=hey_app`.
4. It calls `StartPropertyValuesStream`, and calls `SendAllPropertyValuesToStream` in a separate thread to request all initial values.
5. The application parses the returned Protobuf bytes itself and processes only the target prop; a first `int32Values` value of `4` indicates a long press.
6. It explicitly starts `MainActivity` with `com.kooo.hey.TOGGLE_TALK`.
7. When the stream is interrupted, it closes the channel and reconnects after 3 seconds.

### 5.3 Default TTS/Music Startup

1. All shortcut entry points eventually enter `MainActivity.m487import()`.
2. With `launch_action=1`, it opens the TTS panel; if `launch_auto_play=true`, it plays the text or recording selected by `launch_tts_index`.
3. With `launch_action=2`, it first checks media-read permission and then opens the music panel; automatic playback selects `launch_music_index`.
4. If a toggle is received while TTS or music is playing, the application stops current playback and returns to the background.

### 5.4 Activation and Startup Review

1. The current implementation of `match.m1148final()` directly sets the activation result to true and session value to `1`, so the initial inactive redirect in `MainActivity.onCreate()` is bypassed.
2. `MainActivity.onResume()` still calls `NativeBridge.validateSession(1)`; a return value of `0` starts `ActivationActivity`.
3. The startup coroutine also calls `POST <getServerUrl>/api/check-blacklist` with request body `{"device_id":"..."}`.
4. The response field `blacklisted` is parsed only for HTTP 200; request exceptions are treated as "not blacklisted."
5. Only two consecutive `blacklisted=true` results, separated by 2 seconds, clear the local activation state and enter the activation page.

Static code therefore confirms only that `onCreate`'s initial activation decision is bypassed by a fixed-true path; it cannot establish that the application will never enter the activation page during all ordinary startup sequences.

## 6. Interfaces and Protocols

### 6.1 Android Intents/Broadcasts

| External Action | Internal Action | Effect |
|---|---|---|
| `com.kooo.hey.action.START_TALK` | `com.kooo.hey.START_TALK` | Start default function/public address |
| `com.kooo.hey.action.STOP_TALK` | `com.kooo.hey.STOP_TALK` | Stop public address |
| `com.kooo.hey.action.TOGGLE_TALK` | `com.kooo.hey.TOGGLE_TALK` | Toggle start/stop |
| `com.kooo.hey.music.STOP` | Service-internal | Stop music and terminate foreground service |

### 6.2 VHAL gRPC

| Field | Value/behavior |
|---|---|
| Default address | `localhost:40004` |
| Transport | gRPC over plaintext HTTP/2 |
| Metadata | `session_id=<random UUID>`, `client_id=hey_app` |
| Service | `vhal_proto.VehicleServer` |
| Methods | `SetProperty` (BIDI_STREAMING; descriptor is built but not used by the current monitoring stream); `StartPropertyValuesStream` (SERVER_STREAMING); `SendAllPropertyValuesToStream` (UNARY) |
| Serialization | Raw Protobuf byte arrays; the application encodes/decodes required fields itself |
| Monitored property | Default `0x21407439`, replaceable through `hard_key_vhal_prop_id` |
| Trigger value | First `int32Values == 4` of the target property |

### 6.3 Ecarx Amplifier HIDL/Binder

| Item | Value |
|---|---|
| Service | `vendor.ecarx.xma.audiocontrol@1.0::IEcarxAudioControl` |
| instance | `default` |
| interface token | Same as service name |
| transaction code | Default `55`; configurable through `amp_transaction_code` |
| Request parameter | `int32 1` to enable, `int32 0` to disable |
| Session references | Shared by keys such as `engine`, `tts:*`, `music`, `sweep`, and `foreground`; shutdown is delayed by 60 seconds after no active session remains |

### 6.4 HTTP/JSON

The following interface code remains in the sample. The unlock modification skips the initial inactive branch in `onCreate`, but Native session validation and startup blacklist review remain in the startup chain. `NativeBridge` returns the service address from `libheylicense.so`; the Java layer confirms interface paths and fields. URL data is encoded/split in the Native library and cannot be completely recovered from visible strings. Network configuration explicitly permits plaintext HTTP for subdomains of `suyunkai.top`; the purchase QR code uses the hard-coded HTTPS address `https://coauto.cc/`.

| Function | Method/path | Request JSON | Response |
|---|---|---|---|
| Automatic activation | `POST NativeBridge.getAutoActivateUrl()` | `device_id`, `device_model`, `app_version`, optional `cert_sha256`, `android_version`, optional `serial_no` | `success`, `code`, `config`; or `status=migration_available` + `migration` |
| Migration confirmation | `POST <getServerUrl>/api/migration/confirm` | `device_id`, `device_model`, `app_version`, optional `cert_sha256`, `android_version`, optional `serial_no`, `migration_token` | `success`, `error`, `code`, `config` |
| Heartbeat | `POST NativeBridge.getHeartbeatUrl()` | `device_id`, `activated=true`, `app_version`, `device_model`, `android_version`, optional `cert_sha256`, optional `serial_no`, `timestamp` | Reads only HTTP status; locally sent at most once every 12 hours |
| Blacklist review | `POST <getServerUrl>/api/check-blacklist` | `device_id` | Reads `blacklisted` for HTTP 200; activation state is cleared only after two consecutive true results |
| Log upload | `POST <getServerUrl>/api/upload-log` | `device_id`, `device_model`, `app_version`, `nickname`, `description`, `content`; the last two use empty strings when empty | HTTP `200/201` is success; other statuses display response text |

The Content-Type for all recovered requests is `application/json` or `application/json; charset=UTF-8`. Business HTTP code reuses the `GrpcUtil.HTTP_METHOD` constant, whose value is `POST`; this does not mean that the requests are sent over gRPC.

### 6.5 Local ADB TCP

- Protocol: native Android Debug Bridge packet header/authentication flow, not an invocation of an external `adb` binary.
- Default port: `5555`.
- Targets: `127.0.0.1` first, followed by the head unit's local IPv4 addresses.
- Authentication: self-generated RSA key pair stored as `adb_private_key` / `adb_public_key` in the application's private directory.
- Commands: sent through `shell:<command>\0`; confirmed uses include head-unit debugging/configuration flows such as `settings get/put system workmode`.

## 7. Local Data and Configuration

| Storage | Key content |
|---|---|
| `SharedPreferences: hey_config` | Main configuration for audio usage/source/device ID, gain, filtering, noise gate, AGC, frequency shift, reverb, voice effects, TTS/music, floating window, steering wheel, log trigger, startup action, and related options |
| `SharedPreferences: hey_hb` | `last_ts`, heartbeat throttle time |
| `SharedPreferences: log_upload` | Log-feedback interface values such as `device_nickname` |
| Local license record | `match` retains AES/GCM-wrapped activation-record logic containing `fp`, normalized `code`, `ts`, and `v=1`; the activated decision in the current unlocked sample has been directly rewritten |
| Logs | `current_session.log` and `previous_session.log`, rotated when a new session starts |
| Music | `Download/Hey/music` or a user-selected directory |
| Recordings | `Download/Hey/recordings/*.wav` |

The `config` in a remote activation response can be mapped into local configuration. Statically recovered remotely configurable keys include `grpcPort`, `grpcHost`, `hardKeyVhalPropId`, `audioUsage`, `audioSource`, `inputDeviceId`, `outputDeviceId`, `launchAction`, `enableHardKey`, `enableFloating`, `enableAutoTalkOnForeground`, `enableBandpass`, `enableNoiseGate`, `enableAgc`, `volumeGain`, `minGainLimit`, `maxGainLimit`, `highPassCutoff`, `lowPassCutoff`, `freqShiftHz`, `rippleThreshold`, `reverbMode`, `voiceEffectType`, `floatingSize`, `floatingAlpha`, `ampTransactionCode`, and others.

## 8. Native Libraries and Main Dependencies

| File/component | Functional role |
|---|---|
| `libvoiceeffect.so` | 48 kHz voice-effect DSP implementing creation, effect selection, block processing, flush, and destruction |
| `libheylicense.so` | Provides device ID, license validation, session checks, service address, and log token; Java's initial activation check follows a fixed-true path, but `onResume` still calls `validateSession()` |
| gRPC Java + OkHttp transport | Connects to local VHAL gRPC |
| AndroidX/Material | Activities, lifecycle, foreground notifications, and interface controls |
| ZXing | Generates the purchase-link QR code |
| Kotlin/coroutines | Asynchronous startup, licensing, and UI task scheduling |

## 9. Relationships with Other APKs

### 9.1 Relationship with the Hey Starray Variant

- The package name, version, DEX, Native libraries, layouts, and images are identical, so both variants cannot be installed together.
- The Starray build additionally declares `android.permission.INTERNAL_SYSTEM_WINDOW`.
- Three groups of `dimens.xml` in the Starray build generally reduce the application's own controls to about 70% of the standard build; many default activation-page dimensions are 50%.
- Therefore, it is a vehicle/display calibration build of the same business code, not a functional branch.

### 9.2 Relationship with EVCC/EVCC mini

- The three current APKs use the same re-signing certificate, and their package names belong to `com.kooo.*`.
- Hey's purchase page uses `coauto.cc`, while EVCC's bundled download site uses `download.coauto.cc:9568`, indicating that both belong to the same service/product domain.
- No evidence shows Hey binding EVCC AIDL/ContentProvider or explicitly starting EVCC components. They share a release/service ecosystem and the same head-unit technology domain, but there is no mandatory runtime dependency.

### 9.3 Relationship with AutoAudio/AutoDisplay/AutoService

- Hey does not query, bind, or call any `com.simplerbit.*` package.
- Both software groups target in-vehicle audio/vehicle-property scenarios, but use different signatures, package names, and IPC protocols.
- They can therefore be considered related only by functional domain; no static evidence proves that Hey is a client or required component of that three-application suite.

## 10. Static Analysis Boundaries

- The parameters of each voice effect in `libvoiceeffect.so` and the complete split/encoded URL in `libheylicense.so` cannot be recovered precisely from Java bytecode.
- Native session validation and startup blacklist review can still enter the activation page; **Unconfirmed**: because the sample was not executed, the final startup branch for a particular device and server state cannot be determined.
- Actual availability of `AudioRecord`/`AudioTrack` usage, Ecarx HIDL, and VHAL prop depends on the specific head-unit firmware and was not verified on real hardware.
- The DEX is obfuscated with R8. Large-method placeholders in default JADX output were covered by the simple view and smali; gRPC property decoding, the default prop, trigger value, and the main-interface talk state machine were all reviewed.
- The report's term "unlocked" describes a code fact in the current sample's control flow; this sample alone cannot recover the complete license execution result of an unmodified official APK.

## 11. Key Evidence Index

- Manifest: `.analysis-work/hey_standard/apktool/AndroidManifest.xml`
- Main flow: `.analysis-work/hey_standard/jadx/sources/com/kooo/hey/MainActivity.java`
- Steering-wheel service: `HeyMonitorService.java`, `defpackage/ct.java`, `defpackage/clear.java`, `apktool/smali/Boolean.smali`
- Audio engine: `defpackage/union.java`, `defpackage/of.java`, `defpackage/eb.java`, `defpackage/l9.java`
- TTS/recording: `defpackage/ur.java`, `defpackage/nr.java`
- Music: `defpackage/ai.java`, `defpackage/ei.java`, `MusicPlaybackService.java`
- Configuration: `defpackage/sigma.java`, `res/layout/include_debug_left.xml`
- Activation/migration: `ActivationActivity.java`, `defpackage/match.java`, `defpackage/visit.java`, `license/NativeBridge.java`
- Amplifier: `defpackage/token.java`
- ADB: `defpackage/parse.java`
- Native: `apktool/lib/*/libheylicense.so`, `apktool/lib/*/libvoiceeffect.so`

## 12. Review of Recovered JADX Gaps

The default structured output contained 165 `Method not decompiled` placeholders across 125 Java files. The complete simple output contains 1,530 Java files and only four remaining placeholders, located in AndroidX, gRPC, and two obfuscated helper classes; all retain their original smali.

Seven directly relevant business methods that failed to decompile have been fully recovered: saving debug configuration, building the debug console, startup-item visibility, activation/session review in `MainActivity.onResume()`, the public-address toggle state machine, the Audio usage scanning tool, and the Android Car volume-group command-line tool. The corresponding smali files in both Hey builds have identical SHA-256 values, so these recovery conclusions also apply to the Starray build.

The recovered results confirm that when the main interface returns to the foreground, it rechecks activation and the Native session and cancels delayed audio-engine release. The public-address toggle updates the floating window, microphone icon/animation, state text, and either the continuous or 60-second asynchronous public-address flow.
