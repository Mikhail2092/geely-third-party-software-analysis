# Hey v1.2.4 Xingyuan Unlocked Edition Static Analysis Report

## 1. Role and Conclusions

The `синъюань` text in the sample filename is an approximate Cyrillic transliteration of "Xingyuan." In view of the head-unit context, this report uses "Xingyuan edition" as a short name. The association with that vehicle model is a semantic inference from the filename; the APK code contains no plaintext vehicle-model constant.

Core conclusions:

- **Confirmed by per-file hashes**: except for three groups of `dimens.xml` files, which are compiled into `resources.arsc`, the Xingyuan edition and standard unlocked edition have identical `classes.dex`, all Native `.so` files, layout XML, images, and other decoded resources. Their business functions, protocols, and licensing modifications are therefore identical.
- **Confirmed by Manifest**: the Xingyuan edition additionally declares `android.permission.INTERNAL_SYSTEM_WINDOW`; all other application components and permissions are identical.
- **Confirmed by structured resource differences**: only the application's own dimensions in `values/dimens.xml`, `values-sw1000dp/dimens.xml`, and `values-sw1400dp/dimens.xml` differ. Main-screen, panel, dialog, and text dimensions are mostly reduced to approximately 70% of those in the standard edition; default activation-page dimensions are mostly approximately 50%.
- **Reasonable inference**: this variant is intended for a head unit with different DPI/window dimensions or system-window requirements. It is a display-adaptation edition, not a separate functional branch.

## 2. APK Identity

| Field | Value |
|---|---|
| Original file | `Hey_v1.2.4_синъюань_разблокированный_подписанный.apk` |
| File size | 4,462,006 bytes |
| APK SHA-256 | `F62606E1A24D5BFCBACBBE1716CE7B053F959F9A17B7126D74A3044120E1FB72` |
| Package name | `com.kooo.hey` |
| Application name | `Hey!` |
| Version | `1.2.4` / versionCode `26` |
| Android range | minSdk `28`, targetSdk `36`, compileSdk `36` |
| ABI | `arm64-v8a`, `x86_64` |
| Signing-certificate SHA-256 | `BB:CA:73:98:98:13:6E:6F:84:77:AF:0D:32:53:AE:10:B5:B6:DE:C2:F6:3D:D4:6D:34:FB:5E:AF:83:6C:16:3F` |
| `classes.dex` SHA-256 | `02FD00E112C49E65622AB0874F94BAF2711001CA33D0C782B6D7BFE55027625D` |
| ARM64 `libheylicense.so` | `AAEBBCA7C2EDA6EF3747CEBCFE52FA45B3F8A4BDF9DC4D1F8D993EDB051306DE` |
| ARM64 `libvoiceeffect.so` | `E49BDC531CAADC7DC93108F61214081EDFD02AA523F5028822F52C04BD25B771` |

The DEX and Native hashes above are exactly the same as those in the standard unlocked edition. The two APKs also have the same package name and versionCode, so they are replacement installation variants and cannot coexist as two independent applications.

## 3. Exact Differences From the Standard Unlocked Edition

### 3.1 Manifest

The only decoded Manifest difference is that the Xingyuan edition adds:

```xml
<uses-permission android:name="android.permission.INTERNAL_SYSTEM_WINDOW"/>
```

No Activity, Service, Receiver, Provider, or Intent Filter is added.

### 3.2 Interface Dimensions

Comparison of representative default resources:

| Resource | Standard edition | Xingyuan edition | Ratio |
|---|---:|---:|---:|
| `talk_button_size` | 196 dp | 137 dp | 69.9% |
| `audio_glow_size` | 420 dp | 294 dp | 70.0% |
| `countdown_ring_size` | 224 dp | 157 dp | 70.1% |
| `gain_slider_height` | 280 dp | 196 dp | 70.0% |
| `panel_width` | 400 dp | 280 dp | 70.0% |
| `ve_panel_width` | 480 dp | 336 dp | 70.0% |
| `func_button_size` | 56 dp | 39 dp | 69.6% |
| `music_play_button_size` | 56 dp | 39 dp | 69.6% |
| `status_text_size` | 15 sp | 11 sp | 73.3% |
| `activation_card_width` | 780 dp | 390 dp | 50.0% |
| `activation_title_text_size` | 54 sp | 30 sp | 55.6% |

The `sw1000dp` and `sw1400dp` resources retain an approximately 70% reduction, for example:

- `sw1000dp/talk_button_size`: `294 dp -> 206 dp`.
- `sw1000dp/panel_width`: `560 dp -> 392 dp`.
- `sw1400dp/talk_button_size`: `420 dp -> 294 dp`.
- `sw1400dp/panel_width`: `800 dp -> 560 dp`.

This systematic scaling shows that the variant adjusts overall UI density at the resource level rather than changing only one page.

### 3.3 APK Container Differences

A per-entry comparison of the original ZIP files found hash differences only in the following entries:

- `AndroidManifest.xml`: additional permission.
- `resources.arsc`: dimension-table differences.
- `META-INF/MANIFEST.MF`, `ANDROIDD.SF`, `ANDROIDD.RSA`: signature results after package content changed; the certificate itself is the same.

## 4. Functional Implementation

All functions below are implemented by the same DEX/Native code as in the standard edition. They are actual functions of the Xingyuan edition, not inferences from the filename.

### 4.1 Real-Time Public Address

- A short press starts a 60-second countdown public-address session; another press can end it early.
- A long press enters continuous public-address mode; another press ends it.
- Uses real-time microphone capture, software DSP/voice changing, and playback through a specified vehicle-audio route.
- The interface displays a countdown ring, audio glow effect, status text, and draggable gain slider.

### 4.2 Audio Parameters and DSP

| Item | Implementation/default value |
|---|---|
| PCM | 48 kHz, 16-bit, mono |
| Recording source | Default `7`, configurable |
| Audio usage | Default `73`; can iterate over `0..100` for test playback |
| Device routing | Can specify input/output `AudioDeviceInfo` ID |
| Gain | Default `5.0`, configurable lower/upper limits |
| Band-pass | Default `300..4000 Hz`, enabled by default |
| Noise gate | Can be enabled; default strength `0.5` |
| AGC | Can be enabled |
| Frequency shift | Can be enabled; default `5 Hz` |
| Reverb | None/moderate/strong |
| Voice changing | Original/deep male/high-pitched female/robot/electronic |
| Hardware DSP | Ecarx workmode: off/noise suppression/echo cancellation/wake-up optimization |

`libvoiceeffect.so` implements five Native voice-changing levels. The Java layer assembles gain, filtering, noise gate, AGC, reverb, and frequency shift.

### 4.3 TTS, Recording, and Music

- Stores up to 20 text or WAV recording shortcut entries.
- Android TTS supports speech-rate, pitch, and volume adjustment.
- Recordings are saved as `Download/Hey/recordings/rec_yyyyMMdd_HHmmss.wav` and can be played, renamed, and deleted.
- The default music directory is `Download/Hey/music`; MP3/WAV/FLAC/OGG/M4A/AAC/WMA are supported.
- Music supports play/pause, previous/next track, progress, playlist repeat, and single-track repeat. If `MediaPlayer` fails, it falls back to `MediaExtractor`/`MediaCodec`.
- TTS, recordings, and music all reuse the configurable vehicle-output usage/device and amplifier session.

### 4.4 Quick Activation and Debugging

- A draggable floating button is available, with persisted position, size, and opacity.
- A long press on the steering-wheel OK button is triggered through local VHAL gRPC.
- Automatic triggering by log keywords is supported.
- Automatic public address after tapping the icon can be configured, as can opening/playing a specified TTS or music entry by default.
- The debug page can enumerate audio devices, play a 440 Hz test tone, iterate usage values, display the microphone waveform, test public address, query/set volume groups, and control the external amplifier. The displayed amplifier state reads in-process management state rather than querying actual HIDL hardware state.
- The built-in ADB TCP client can connect to local port `:5555`, mainly as a fallback channel for head-unit configuration such as Ecarx DSP workmode.

## 5. Application Components

| Type | Component | Purpose |
|---|---|---|
| Activity | `MainActivity` | Main public-address interface and state machine |
| Activity | `ActivationActivity` | Original activation/migration interface |
| Activity | `DebugActivity` | Head-unit audio and trigger adaptation |
| Service | `FloatingWindowService` | Floating public-address button |
| Service | `MusicPlaybackService` | Foreground music playback |
| Service | `HeyMonitorService` | VHAL steering-wheel listener |
| Service | `LogcatTriggerService` | Log-keyword listener |
| Receiver | `BootReceiver` | Restores steering-wheel listening at boot |
| Receiver | `HeyCommandReceiver` | External start/stop/toggle commands |

## 6. Key Protocols

### 6.1 Intent/Broadcast

- `com.kooo.hey.action.START_TALK`
- `com.kooo.hey.action.STOP_TALK`
- `com.kooo.hey.action.TOGGLE_TALK`
- Internal equivalents: `com.kooo.hey.START_TALK`, `STOP_TALK`, `TOGGLE_TALK`
- Stop music: `com.kooo.hey.music.STOP`

### 6.2 VHAL gRPC

| Field | Value |
|---|---|
| Address | Default `localhost:40004`, plaintext |
| Metadata | `session_id=<UUID>`, `client_id=hey_app` |
| Service | `vhal_proto.VehicleServer` |
| RPC | `StartPropertyValuesStream`, `SendAllPropertyValuesToStream`; a `SetProperty` descriptor is also constructed |
| Target property | Default `0x21407439` |
| Trigger condition | First `int32Values == 4` |
| Disconnection handling | Close the channel and reconnect after 3 seconds |

### 6.3 ECARX External Amplifier

- HIDL service: `vendor.ecarx.xma.audiocontrol@1.0::IEcarxAudioControl/default`.
- Default transaction code: `55`, configurable.
- `int32=1` enables it; `int32=0` disables it.
- This dedicated management path is entered only for usage `72`; it shuts down 60 seconds after all sessions are released.

### 6.4 HTTP/JSON

| Function | Method | Main fields |
|---|---|---|
| Automatic activation | `POST getAutoActivateUrl()` | `device_id`, `device_model`, `app_version`, optional `cert_sha256`, `android_version`, optional `serial_no` |
| Migration confirmation | `POST <server>/api/migration/confirm` | Device fields above, with certificate/SN conditional, plus `migration_token` |
| Heartbeat | `POST getHeartbeatUrl()` | Device/version/time and conditional certificate/SN; locally throttled to once every 12 hours |
| Blacklist recheck | `POST <server>/api/check-blacklist` | `device_id`; on HTTP 200 reads `blacklisted`; activation state is cleared only after two true results 2 seconds apart |
| Log upload | `POST <server>/api/upload-log` | `device_id`, `device_model`, `app_version`, `nickname`, `description`, `content`; empty strings are written when nickname/description is absent |

Service URLs are supplied by `libheylicense.so` and are split/encoded. The network configuration explicitly permits plaintext HTTP for subdomains of `suyunkai.top`; the purchase QR code uses `https://coauto.cc/`.

## 7. Unlock Modifications

The unlock logic in the Xingyuan sample is exactly the same as in the standard sample:

- `defpackage.match.m1148final()` directly sets the static activation result to true and the session value to `1`, then returns `true`.
- `defpackage.match.m1143case(code)` also directly returns `true`.
- The initial inactive branch of `MainActivity.onCreate()` is skipped because of the fixed-true path above.
- `MainActivity.onResume()` still calls `NativeBridge.validateSession(1)` and opens `ActivationActivity` when it returns `0`.
- The startup coroutine still performs two blacklist rechecks; only two `blacklisted=true` results clear activation state and open the activation page.
- Automatic activation, migration, heartbeat, blacklist, and remote-configuration code all remains present.

Therefore, "Xingyuan" and "unlocked" are two independent dimensions: the former is a UI/window adaptation, while the latter is the same licensing-control-flow modification as in the standard variant. Statically fixing the initial check to true cannot be further described as "the activation page will never be entered during any normal startup."

## 8. Local Data

- `hey_config`: audio, DSP, voice changing, TTS/music, floating window, triggers, and startup actions.
- `hey_hb/last_ts`: heartbeat throttling.
- `log_upload/device_nickname`: information for the log-upload interface.
- `current_session.log` / `previous_session.log`: application-session log rotation.
- `Download/Hey/music`: default music directory.
- `Download/Hey/recordings`: WAV recordings.
- `adb_private_key` / `adb_public_key`: authentication keys for the built-in ADB TCP client.

The interface-dimension differences are stored in the APK resource table and are not a runtime SharedPreferences switch.

## 9. Relationships With Other Software

- **Standard Hey unlocked edition**: same package name, version, DEX, Native code, and certificate; direct replacement relationship.
- **EVCC/EVCC mini**: the current samples share a re-signing certificate and the `com.kooo.*` namespace, while Hey and EVCC share the `coauto.cc` product-service domain. Hey does not bind an EVCC service and can run independently.
- **AutoAudio/AutoDisplay/AutoService**: no package query, AIDL, Intent, or Provider call relationship; it only shares the in-vehicle audio functional domain with AutoAudio.

## 10. Static-Analysis Boundaries

- No test was performed on a Xingyuan head unit, so the exact effect of `INTERNAL_SYSTEM_WINDOW` on the target firmware cannot be confirmed from the APK alone.
- Native session validation and the startup blacklist recheck can still open the activation page; the final startup branch for a particular device and server state cannot be confirmed statically.
- Resource scaling ratios can be confirmed, but the physical resolution and Android density of the target head unit cannot be determined from static resources alone.
- The mathematical parameters of Native voice changing and the complete split URL in the Native licensing library could not be fully recovered from the Java layer.
- Large R8-obfuscated method placeholders in default JADX have been covered by the simple view and smali. The key gRPC property/value, difference table, licensing branches, and talk state machine have all been reviewed.

## 11. Evidence Index

- Variant Manifest: `.analysis-work/hey_variant/apktool/AndroidManifest.xml`
- Variant dimensions: `.analysis-work/hey_variant/apktool/res/values*/dimens.xml`
- Per-file comparison: `.analysis-work/hey_standard/raw` and `.analysis-work/hey_variant/raw`
- Main business code: `.analysis-work/hey_variant/jadx/sources/com/kooo/hey`
- Audio/TTS/music/protocols: `.analysis-work/hey_variant/jadx/sources/defpackage`
- smali review: `.analysis-work/hey_variant/apktool/smali`
- Native: `.analysis-work/hey_variant/apktool/lib`

## 12. Review of JADX Gap Recovery

The Xingyuan edition's default structured output contains 165 `Method not decompiled` placeholders across 125 Java files. The complete simple output contains 1,530 Java files, with only four framework/helper method placeholders covered by smali remaining.

The seven directly failed business methods and corresponding smali files in the Xingyuan and standard editions have identical per-file SHA-256 values. Recovered content includes debug-console and configuration persistence, startup-section switching, activation/session rechecks on foreground restoration, the continuous/60-second talk state machine, Audio usage scanning, and the Android Car volume-group command tool.

The conclusion about variant differences therefore remains unchanged: both editions share the same business DEX, and the Xingyuan edition's differences remain concentrated in a Manifest permission and resource dimension calibration.
