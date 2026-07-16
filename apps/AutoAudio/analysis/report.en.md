# AutoAudio v2.0.2 Static Analysis Report

## 1. Report Scope

- Analysis target: `AutoAudio_v2.0.x_макет_интерфейса_подписанный.apk`
- Analysis method: static analysis only, covering the APK, DEX decompiled code, smali, Manifest, resources, Native library exports, and printable strings.
- The APK was not executed; none of its Activities, Services, Receivers, Native libraries, or network requests were started.
- This report describes only the software's functions, implementation, business processes, interface protocols, and relationships with other software. It does not evaluate security, privacy, or compliance.
- Primary evidence directories: `.analysis-work/autoaudio/apktool/`, `.analysis-work/autoaudio/jadx/`.

### 1.1 Conclusion Markers

| Marker | Meaning |
|---|---|
| `[Code-confirmed]` | Directly confirmed by Java, smali, the Manifest, or protocol classes |
| `[Resource-confirmed]` | Directly confirmed by packaged resources such as XML, assets, audio, or images |
| `[Reasoned inference]` | Supported by multiple mutually consistent pieces of static evidence, but final behavior could not be observed because the application was not run |
| `[Unconfirmed]` | The static sample is insufficient, or the actual result depends on the head-unit environment, server, or runtime state |

## 2. Sample Identity

| Item | Value |
|---|---|
| Package name | `com.simplerbit.autoaudio` |
| Application version | `2.0.2` |
| versionCode | `211` |
| Minimum Android SDK | 28 |
| target / compile SDK | 36 / 36 |
| APK size | 27,601,338 bytes |
| SHA-256 | `E37A058D8CDB88D5E464DD08D291D1E51A80CE20A3B151BC0F7B6B5C6FF68104` |
| Current sample signature | Android Debug certificate |
| Certificate SHA-1 | `70:92:2D:33:E3:27:35:60:4B:51:19:FE:AE:FD:79:29:D2:EF:A2:B1` |
| Certificate SHA-256 | `F9:A0:79:A1:94:DE:39:3F:08:7B:62:06:F4:B6:90:8C:B7:03:78:34:23:95:9F:2E:1A:A8:19:4F:34:4E:E0:7C` |

Evidence: `apktool/apktool.yml`, `apktool/AndroidManifest.xml`, and static extraction results for the APK hash and certificate.

## 3. Software Purpose and Overall Architecture

`[Code-confirmed]` AutoAudio is an in-vehicle audio and vehicle-event notification manager. It converts the unified vehicle properties supplied by AutoService into audio business events and manages prompt sounds, in-cabin karaoke, external public-address speech, floating music, sound themes, audio routing, licensing, and updates.

The main modules are:

| Module | Entry point/core class | Responsibility |
|---|---|---|
| Main interface | `com.simplerbit.autoaudio.MainActivity`, `defpackage.bh` | Activation page, permissions page, six functional settings groups, and a custom-drawn main interface |
| Resident audio service | `com.simplerbit.autoaudio.AudioService` | Vehicle-event subscriptions, prompt playback, real-time microphone pipeline, and floating audio |
| Boot startup | `com.simplerbit.autoaudio.BootReceiver` | Checks availability after boot and starts AudioService |
| Vehicle interface client | `AudioService`, `defpackage.m3`, bundled `com.simplerbit.autoservice.contract.*` | Starts/binds AutoService and reads/subscribes to vehicle properties |
| Audio playback and routing | `defpackage.g3`, `defpackage.qq`, `defpackage.ig` | `MediaPlayer`, output-device selection, and ECARX internal/external media modes |
| Real-time speech processing | `AudioService`, `RnNoiseProcessor`, `libautoaudio_rnnoise.so` | Microphone capture, gain, AEC, RNNoise, and speaker output |
| Licensing and updates | `defpackage.d3` and its callers | Device identity, license, subscription, redemption codes, version checks, and APK downloads |
| LAN activation input | `defpackage.hg`, `defpackage.fg` | Temporary HTTP form and QR-code entry point |
| Accessibility component | `AutoAudioAccessibilityService` | Declares full capabilities, but its current Java callbacks are empty |

`[Reasoned inference]` The long-running component is `AudioService`, while `MainActivity` is primarily for configuration. This is consistent with the Activity's `excludeFromRecents=true` and `noHistory=true` settings and the service's sticky foreground execution mode.

## 4. Manifest and Components

Evidence: `apktool/AndroidManifest.xml`.

### 4.1 Activity

- `[Code-confirmed]` The launcher Activity is `com.simplerbit.autoaudio.MainActivity`.
- `[Code-confirmed]` The Activity is exported and has `MAIN/LAUNCHER`.
- `[Code-confirmed]` It uses `excludeFromRecents=true`, `noHistory=true`, and an empty `taskAffinity`; after the interface exits, no ordinary recent-task history is retained.

### 4.2 Service

- `[Code-confirmed]` `com.simplerbit.autoaudio.AudioService` is not exported.
- `[Code-confirmed]` Its foreground service types are `mediaPlayback|microphone`.
- `[Code-confirmed]` `AudioService.onStartCommand()` returns sticky semantics, allowing the system to request service recreation after reclaiming it.
- `[Code-confirmed]` `AutoAudioAccessibilityService` is exported to the system and requires `BIND_ACCESSIBILITY_SERVICE`.
- `[Resource-confirmed]` The accessibility configuration permits all event types, reading window content, and performing gestures.
- `[Code-confirmed]` In the currently decompiled Java, both `onAccessibilityEvent()` and `onInterrupt()` are empty.
- `[Unconfirmed]` The statically visible implementation does not prove that the service actually controls any third-party application interface; it only confirms that the capability entry point is declared in the Manifest.

### 4.3 Receiver and Provider

- `[Code-confirmed]` `BootReceiver` listens for `BOOT_COMPLETED`, `LOCKED_BOOT_COMPLETED`, and `QUICKBOOT_POWERON`, and supports direct boot.
- `[Code-confirmed]` The `FileProvider` authority is `com.simplerbit.autoaudio.fileprovider`.
- `[Resource-confirmed]` The FileProvider path points to the application's external-files root directory. Its actual call site hands a downloaded update APK to the system installation interface.
- `[Code-confirmed]` AndroidX Startup and Profile Installer components are framework initialization/baseline configuration, not AutoAudio business interfaces.

### 4.4 Permissions and Corresponding Functions

| Permission group | Functional purpose |
|---|---|
| `RECEIVE_BOOT_COMPLETED` and foreground-service permissions | Start at boot and keep the audio service running |
| `RECORD_AUDIO` and microphone foreground service | Real-time capture for karaoke/external public address |
| `SYSTEM_ALERT_WINDOW` | Floating audio button and list |
| External storage and media-audio read access | Scan floating music, import themes, and import custom sounds |
| `INTERNET`, `ACCESS_NETWORK_STATE` | Licensing, subscriptions, updates, and online themes |
| `REQUEST_INSTALL_PACKAGES` | Install an in-app update APK |
| `CAR_CONTROL_AUDIO_SETTINGS`, `CAR_CONTROL_AUDIO_VOLUME` | Head-unit audio routing and volume-group control |

## 5. Startup and Lifecycle Flows

### 5.1 User-Initiated Startup

1. `[Code-confirmed]` The launcher opens `MainActivity`.
2. `[Code-confirmed]` The Activity initializes the license state, required permissions, and configuration interface.
3. `[Code-confirmed]` After runtime conditions are met, it starts `AudioService`.
4. `[Code-confirmed]` `AudioService` creates a foreground notification and initializes the audio configuration.
5. `[Code-confirmed]` The service explicitly starts and binds AutoService's `CarPropertyService`.
6. `[Code-confirmed]` After the Binder connection is established, it first reads current values for key properties, then registers callbacks for 21 properties.
7. `[Reasoned inference]` Subsequent vehicle-state changes drive prompt and key-related behavior through Binder callbacks; the interface does not need to remain active.

### 5.2 Boot Startup

1. `[Code-confirmed]` `BootReceiver` receives one of the three boot broadcasts.
2. `[Code-confirmed]` The Receiver checks the software's current license/enabled state.
3. `[Code-confirmed]` When the conditions are met, it starts `AudioService` as a foreground service.
4. `[Code-confirmed]` The service reconnects to AutoService and restores audio and floating-overlay settings from SharedPreferences.

### 5.3 AutoService Disconnection Recovery

- `[Code-confirmed]` The service binds AutoService using an explicit component and action.
- `[Code-confirmed]` After connection, it performs initial-value reads and batch subscriptions.
- `[Code-confirmed]` During disconnection/destruction, it calls `unsubscribe()` for each item.
- `[Code-confirmed]` A delayed-retry Handler exists, showing that the design handles an unavailable or disconnected vehicle service.

Key evidence: `AudioService.java`, `defpackage.m3.java`.

## 6. User Interface and All Visible Functions

`[Resource-confirmed]` The main navigation contains six entries: prompt sounds, floating audio, sound themes, volume settings, karaoke/public address, and About. Most interface construction logic is in the large `defpackage.bh` class.

### 6.1 Prompt Sounds

`[Resource-confirmed]` The prompt-item array in `res/values/arrays.xml` has 37 rows. "Outside vehicle" and "Inside vehicle" are group headers, leaving 35 actual event sound slots.

#### Outside-Vehicle Prompt Sounds: 13 Items

1. Vehicle-lock prompt
2. Left-turn prompt
3. Right-turn prompt
4. Hazard-light prompt
5. Reverse-gear prompt
6. Low-speed prompt
7. Charging connector inserted prompt
8. Charging connector removed prompt
9. Charging started prompt
10. Charging stopped prompt
11. Sentry mode enabled prompt
12. Sentry mode minor-trigger prompt
13. Sentry mode major-trigger prompt

#### Inside-Vehicle Prompt Sounds: 22 Items

1. Ignition-on prompt
2. Ignition-off prompt
3. P-gear prompt
4. D-gear prompt
5. N-gear prompt
6. R-gear prompt
7. Left front door open/closed prompts, 2 items
8. Right front door open/closed with the passenger seat occupied, 2 items
9. Right front door open/closed with the passenger seat unoccupied, 2 items
10. Left rear door open/closed prompts, 2 items
11. Right rear door open/closed prompts, 2 items
12. Trunk open/closed prompts, 2 items
13. Low-fuel prompt
14. Speeding prompt
15. Left-side door-opening warning prompt
16. Right-side door-opening warning prompt

`[Code-confirmed]` Each event sound slot can be independently enabled or disabled and is also controlled by the master prompt switch.

`[Code-confirmed]` The user can select a custom WAV/MP3 file for each item; the path is stored in the form `custom_prompt_audio_path_{inside|outside}/{prompt_key}`.

`[Code-confirmed]` The default audio for the current theme can be restored, and the current prompt configuration can be exported.

`[Code-confirmed]` Playback logic includes a repeat interval to prevent the same persistent state from triggering at an unlimited high rate; the interval is controlled by `prompt_repeat_interval_seconds`.

### 6.2 Floating Audio

- `[Code-confirmed]` After the user selects a local audio directory, the software scans playable files and saves the track list.
- `[Code-confirmed]` It can display a desktop floating button and a floating track list.
- `[Code-confirmed]` It supports track selection, starting playback, and stopping playback.
- `[Code-confirmed]` The floating window's `x/y` position is persisted.
- `[Code-confirmed]` The floating control size ranges from 60% to 150%.
- `[Code-confirmed]` Opacity ranges from 30% to 100%.
- `[Code-confirmed]` The number of displayed list entries is configurable.
- `[Code-confirmed]` Actual playback uses `MediaPlayer` and attempts to call `setPreferredDevice()` to select the audio output.

Key evidence: `defpackage.bh`, `defpackage.g3`, `defpackage.qq`.

### 6.3 Sound Themes

- `[Code-confirmed]` Themes can be imported from local ZIP packages.
- `[Code-confirmed]` The theme-package directory is `/sdcard/other/autoaudio/theme/packages`.
- `[Code-confirmed]` The extraction directory is `/sdcard/other/autoaudio/theme/extracted`.
- `[Code-confirmed]` Themes are organized into inside-vehicle/outside-vehicle subdirectories.
- `[Code-confirmed]` After removing the extension from an audio filename, the software also strips a trailing `_number` before matching it as a prompt key.
- `[Reasoned inference]` The `_number` suffix permits multiple candidate variants or versions for one business key; the exact selection strategy depends on the call site.
- `[Code-confirmed]` The online theme list comes from the content API, with the client filtering on `content_type=theme_pack`.
- `[Code-confirmed]` Parsed theme fields include `id`, `title`, `version`, `description`, `is_free`, and `file_size`.
- `[Code-confirmed]` After download, a theme enters the same unpack/apply flow as a local theme.

### 6.4 Volume and Audio Routing

- `[Code-confirmed]` Inside speakers, outside speakers, and the microphone input device can be selected separately.
- `[Code-confirmed]` Device enumeration uses Android `AudioDeviceInfo` and recognizes built-in devices, USB, Bluetooth telephony, Bluetooth media, and Audio Bus.
- `[Code-confirmed]` The software uses these devices through Android audio APIs; it does not implement underlying USB or Bluetooth transport protocols itself.
- `[Code-confirmed]` Separate settings exist for inside/outside prompt volume, inside karaoke volume, outside public-address volume, and microphone gain.
- `[Code-confirmed]` A low-speed prompt threshold can be set in km/h.
- `[Code-confirmed]` The prompt repeat interval is configurable.
- `[Code-confirmed]` The vehicle media playback location can be set to inside, outside, or both.

#### ECARX Inside/Outside Media Mode

`[Code-confirmed]` The software accesses the following through reflection:

- `com.ecarx.xui.adaptapi.audio.audiofx.ExternalSpeakers`
- `com.ecarx.xui.adaptapi.audio.audiofx.IExternalSpeakers`
- `setExternalSpeakersMediaPlayMode(int)`

`[Code-confirmed]` The input value is restricted to 0, 1, or 2; the interface semantics are inside, outside, and both. Evidence: `defpackage.qq`, `defpackage.ig`.

#### Outside Media Volume

`[Code-confirmed]` The software reflectively accesses `android.car.media.CarAudioManager` and calls `setGroupVolume(6, volume, 0)`; volume group 6 is fixed. Evidence: `defpackage.bh`.

`[Unconfirmed]` The physical output mapping of volume group 6 is determined by the vendor system in each head-unit firmware; the static APK cannot guarantee identical semantics across all vehicle models.

### 6.5 Karaoke and Public Address

- `[Code-confirmed]` Inside-vehicle karaoke mode and outside-vehicle public-address mode are supported.
- `[Code-confirmed]` A long press on the `*` key can be bound as the push-to-talk key.
- `[Code-confirmed]` A long press on the menu key can be bound as the push-to-talk key.
- `[Code-confirmed]` The two key bindings are mutually exclusive in settings.
- `[Code-confirmed]` The `*` key event string is `"[0,17,0]"`.
- `[Code-confirmed]` The menu key event string is `"[0,6,0]"`.
- `[Code-confirmed]` The public-address/real-time audio pipeline starts only after the key has been held for 1000 ms and stops when the key is released.
- `[Code-confirmed]` Feedback suppression can be enabled.
- `[Code-confirmed]` Android AcousticEchoCanceler can be enabled in inside mode; actual availability depends on the device audio HAL.

### 6.6 About, License, Redemption, and Updates

- `[Code-confirmed]` The About page displays the version, device identity/product signature, and license state.
- `[Code-confirmed]` Online checks for license and subscription status are supported.
- `[Code-confirmed]` A membership/redemption code can be entered.
- `[Code-confirmed]` A QR code and LAN page can quickly return an activation code to the head unit.
- `[Code-confirmed]` The software can query the latest stable-channel version, download an APK, and enter the system installation flow through FileProvider.
- `[Code-confirmed]` The pending installation path is stored in `audio_update.pending_apk` during download.

### 6.7 License-Branch Behavior in the Current Build

- `[Code-confirmed]` The currently visible implementations of `defpackage.d3.l()`, `d3.n()`, and `d3.a()` return constant true results.
- `[Code-confirmed]` The current build returns a fixed hardware number/product signature instead of relying entirely on a real-time device calculation.
- `[Reasoned inference]` Therefore, the current sample's main business interface and service-start branches operate as if the license conditions have been met.
- This section records only the build implementation and does not evaluate the design.

## 7. Vehicle Event Protocol and Prompt Business Flows

### 7.1 Service Discovery and Binding

`[Code-confirmed]` The Manifest's `<queries>` explicitly declares the target package `com.simplerbit.autoservice`.

AutoAudio uses the following fixed values:

| Item | Value |
|---|---|
| Target package | `com.simplerbit.autoservice` |
| Target component | `com.simplerbit.autoservice.service.CarPropertyService` |
| Start action | `com.simplerbit.autoservice.action.START_CAR_PROPERTY_SERVICE` |
| Bind action | `com.simplerbit.autoservice.action.BIND_CAR_PROPERTY_SERVICE` |

Evidence: near `AudioService.java:841`.

### 7.2 AIDL Interfaces

AutoAudio bundles contract classes matching AutoService, so both Binder endpoints communicate using the same serialization format:

- `ICarPropertyService.getProperty(int)`
- `ICarPropertyService.setProperty(CarPropertyValue)`
- `ICarPropertyService.subscribe(int, ICarPropertyCallback)`
- `ICarPropertyService.unsubscribe(int, ICarPropertyCallback)`
- `ICarPropertyService.getSupportedPropertyIds()`
- `ICarPropertyCallback.onPropertyChanged(CarPropertyValue)`

In this sample, AutoAudio uses the interface only to read and subscribe to vehicle state; no vehicle-control write behavior was confirmed.

### 7.3 The 21 Actually Subscribed Properties

Evidence: `defpackage.m3.java:58-78`, `AudioService.java:1026-1046`.

| ID | Unified meaning | AutoAudio use |
|---:|---|---|
| 10000 | Current vehicle model | Initialize model-specific logic |
| 10001 | Vehicle speed | Low-speed prompt and speeding detection |
| 10003 | Gear character | P/D/N/R prompts and reverse state |
| 10007 | Door-lock state | Vehicle-lock prompt |
| 10017 | Left front door | Open/closed prompts |
| 10018 | Right front door | Refined prompt based on passenger-seat occupancy |
| 10019 | Left rear door | Open/closed prompts |
| 10020 | Right rear door | Open/closed prompts |
| 10043 | Power mode request | Ignition-on/off state transition |
| 10044 | Turn indication | Left turn, right turn, and hazard-light prompts |
| 10047 | Passenger-seat occupancy | Right front door prompt branch |
| 10011 | Remaining fuel percentage | Low-fuel prompt |
| 10029 | Left parking blind spot | Left-side door-opening warning |
| 10030 | Right parking blind spot | Right-side door-opening warning |
| 10037 | Key-event string | `*` key/menu key public-address trigger |
| 30011 | Trunk state | Open/closed prompts |
| 10049 | Speed limit | Speeding threshold reference |
| 10050 | Charging connector state | Connector insertion/removal prompts |
| 10051 | Charging state | Charging start/stop prompts |
| 10052 | `pavstdmodests` | Sentry mode state |
| 10053 | Alarm state | Sentry minor/major trigger |

### 7.4 Unified Event Processing

`[Code-confirmed]` After the first connection, the service calls `getProperty()` in sequence to obtain current values and passes them into the unified handler, avoiding the need to wait for the next change before state becomes available.

`[Code-confirmed]` Subsequent `ICarPropertyCallback.onPropertyChanged()` calls also enter `AudioService.a(CarPropertyValue, boolean)`.

`[Code-confirmed]` This method has been fully expanded in JADX's simple view and cross-checked branch by branch against smali; the 21 property-ID branches, state cache, prompt keys, master/per-item switches, and repeat-interval decisions are all confirmed.

`[Code-confirmed]` The business state machine follows this flow: "save previous value -> compare new value -> check master switch/per-item switch/repeat interval -> select inside or outside device -> play the corresponding prompt."

### 7.5 Recovered Key Business Flows

#### Doors and Passenger-Seat Occupancy

1. Subscribe to 10017-10020 and 10047.
2. When a door state changes, determine whether it opened or closed.
3. For the left front, left rear, and right rear doors, directly select the corresponding open/closed sound slot.
4. For the right front door, additionally read passenger-seat occupancy and select the "occupied" or "unoccupied" sound slot.
5. Play to the inside output according to the enabled state and repeat interval.

#### Turn Signals and Hazard Lights

1. Subscribe to 10044.
2. Value 0 means off; 1/2/3 correspond to left turn/right turn/hazard lights.
3. When the state is active, select an outside prompt.
4. For a persistent state, use the configured repeat interval to control subsequent playback.

#### Vehicle Speed, Low Speed, and Speeding

1. 10001 provides the current integer vehicle speed.
2. When speed is below or within the configured `low_speed_threshold_kmh` range, the low-speed scenario can trigger an outside prompt.
3. 10049 provides the speed-limit value.
4. When current speed exceeds a valid speed limit, enter the inside speeding-prompt branch.
5. Repeated prompts are constrained by the common repeat interval.

#### Charging Flow

1. Values 1/2 of 10050 mean connector insertion/removal and map to two outside sound slots.
2. Values 1/2 of 10051 mean charging start/stop and map to two outside sound slots.
3. Each property change is played after unified deduplication and switch checks.

#### Sentry Flow

1. 10052 represents the `pavstdmodests`/sentry-mode state.
2. On entering the enabled state, play "Sentry mode enabled."
3. Alarm state 10053 distinguishes minor and major trigger prompts.
4. `[Unconfirmed]` Raw enum values for 10052/10053 across all vehicle models are determined by the AutoService mapping JSON and underlying vehicle signals; complete enum consistency cannot be guaranteed independently of a specific model.

#### Push to Talk

1. The 10037 callback provides a key-event string.
2. Depending on settings, accept only one of `"[0,17,0]"` or `"[0,6,0]"`.
3. After the key has been held for 1000 ms, create the real-time audio pipeline.
4. Send microphone audio to the inside or outside output according to mode.
5. On release, stop capture and playback and release the real-time audio objects.

## 8. Audio Implementation

### 8.1 Prompt and Floating-Music Playback

- `[Code-confirmed]` The core player uses Android `MediaPlayer`.
- `[Code-confirmed]` `setPreferredDevice(AudioDeviceInfo)` can be called after creating the player.
- `[Code-confirmed]` Inside/outside prompts have independent output-device and volume configurations.
- `[Code-confirmed]` A custom audio path takes priority over the theme's default resource; if it is missing, playback falls back to an available theme/bundled audio file.
- `[Reasoned inference]` Media mode and preferred device are two routing layers: the former controls the vendor's inside/outside media channel, while the latter controls an Android-enumerated device.

### 8.2 Real-Time Microphone Pipeline

`[Code-confirmed]` The real-time pipeline has these basic parameters:

| Item | Value |
|---|---|
| Sample rate | 48,000 Hz |
| PCM format | 16 bit |
| Channels | Mono input, mono output |
| Input | `AudioRecord` |
| Output | `AudioTrack` |
| RNNoise frame | 480 samples, or 10 ms |

The processing order can be recovered as follows:

1. `AudioRecord.read()` reads PCM16 microphone samples.
2. Apply the corresponding microphone gain according to inside/outside mode.
3. Enable Android AEC in inside mode when supported by the device.
4. When feedback suppression is enabled, send 480-sample frames to RNNoise.
5. Apply smooth attenuation/gain transitions and peak limiting to the output to prevent short overflow.
6. `AudioTrack.write()` outputs to the selected inside or outside device.
7. When speaking stops, stop and release capture, playback, and RNNoise state.

Key evidence: `AudioService.java`, `RnNoiseProcessor.java`, `defpackage.v3.java`.

### 8.3 RNNoise Native Library

All four ABIs include `libautoaudio_rnnoise.so`:

| ABI | Size (bytes) |
|---|---:|
| arm64-v8a | 5,749,560 |
| armeabi-v7a | 5,728,700 |
| x86 | 5,762,628 |
| x86_64 | 5,763,104 |

`[Code-confirmed]` The JNI interfaces are create, destroy, frameSize, and process; the Java side requires the native frame size to equal 480.

`[Code-confirmed]` `nativeProcess()` returns a float speech probability/processing result. On an exception, the Java side closes the processor and continues in degraded mode.

`[Reasoned inference]` The library wraps RNNoise denoising to suppress environmental sound and feedback in the real-time amplification pipeline; no evidence shows it handling vehicle communication.

## 9. Public-Network HTTP Protocol

### 9.1 Base Address and Authentication Fields

`[Code-confirmed]` Base URL: `https://automanager.simplerbit.com`.

Common request fields:

| Field | Calculation/value |
|---|---|
| `hardware_id_hash` | `SHA-256(hardwareId)` |
| `product_signature` | Product-signature string |
| `platform` | `android` |
| `timestamp` | Current millisecond timestamp |
| `requestSignature` | `SHA-256(hardware_id_hash + ":" + product_signature + ":" + timestamp)` |
| `channel` | `stable` for update queries |

Key evidence: `defpackage.d3`.

### 9.2 Endpoints

| Method | Path | Purpose/confirmed fields |
|---|---|---|
| POST | `/api/client/licenses/status` | Query license state |
| POST | `/api/client/subscriptions/status` | Query subscription state |
| POST | `/api/client/codes/redeem` | Redeem, with additional `redeem_code` |
| GET | `/api/client/releases/latest?...&t={time}` | Query the latest stable-channel version |
| GET | `/api/client/releases/{id}/download?...` | Download an update APK |
| POST | `/api/client/contents` | Query content; client filters for `theme_pack` |
| POST | `/api/client/contents/{id}/download` | Download a theme package; Accept is `application/octet-stream` |

### 9.3 Response Structure

- `[Code-confirmed]` Common top-level fields are `ok`, `data`, and `error`.
- `[Code-confirmed]` Error text is read from `error.message`.
- `[Code-confirmed]` License data includes `expires_at`.
- `[Code-confirmed]` At least `id` and `version` are read for a release.
- `[Code-confirmed]` Theme content reads `id/title/version/description/is_free/file_size`.
- `[Unconfirmed]` Other fields and business constraints that the server may return cannot be fully recovered from client parsing code alone.

`[Code-confirmed]` No WebSocket, MQTT, or custom persistent-connection protocol was found; public-network functions use ordinary HTTP(S) requests.

## 10. Temporary LAN HTTP Activation Protocol

Key classes: `defpackage.hg`, `defpackage.fg`.

### 10.1 Service Creation

- `[Code-confirmed]` Creates `ServerSocket(0)`, allowing the system to allocate a free TCP port.
- `[Code-confirmed]` Obtains the head unit's current IPv4 address.
- `[Code-confirmed]` Generates a UUID token with hyphens removed.
- `[Code-confirmed]` QR/display URL format: `http://{head-unit-IPv4}:{port}/?t={token}`.
- `[Code-confirmed]` The temporary service closes with the activation Dialog/Activity lifecycle.

### 10.2 Request Protocol

| Request | Parameters | Result |
|---|---|---|
| `GET /?t={token}` | Query parameter `t` | Returns an HTML form for entering the activation code |
| `POST /submit` | `t` and `code` as `application/x-www-form-urlencoded` | Returns a nonempty code to the head-unit input field |

- `[Code-confirmed]` A token mismatch returns HTTP 403.
- `[Code-confirmed]` If code is empty, the form is shown again with a prompt to continue entering it.
- `[Code-confirmed]` After successful submission, a main-thread Handler updates the head-unit interface.
- `[Code-confirmed]` There is no WebSocket; this is a minimal, short-lived HTTP/1.x form server.

## 11. Local Files and Configuration

### 11.1 SharedPreferences: `audio_settings`

| Configuration group | Main keys |
|---|---|
| Audio devices | `inside_speaker(_id)`, `outside_speaker(_id)`, `microphone(_id)` |
| Prompt sounds | `inside_prompt_volume`, `outside_prompt_volume`, `prompt_repeat_interval_seconds`, `prompt_master_enabled`, `disabled_prompt_keys` |
| Vehicle threshold | `low_speed_threshold_kmh` |
| Microphone | `inside_mic_gain`, `outside_mic_gain`, `talk_feedback_suppression_enabled` |
| Karaoke/public address | `inside_karaoke_volume`, `outside_talk_volume`, `inside_karaoke_mode` |
| Keys | `talk_key_binding_enabled`, `menu_talk_key_binding_enabled` |
| Vehicle media | `vehicle_media_play_mode`, `outside_media_volume` |
| Theme/UI | `theme_name`, `ui_day_theme` |
| Custom sounds | `custom_prompt_audio_path_{inside-or-outside/prompt-name}` |
| Floating audio source | `floating_audio_dir`, `floating_audio_saved` |
| Floating overlay | `floating_overlay_enabled/x/y/size_percent/opacity_percent` |
| Floating list | `floating_audio_display_count` |
| File picker | `prompt_audio_picker_dir` |

### 11.2 Other Preferences

- `[Code-confirmed]` `audio_license` stores the license, device identity, and related state.
- `[Code-confirmed]` `pending_apk` in `audio_update` stores the path of the downloaded APK awaiting installation.

### 11.3 External Files

- `[Code-confirmed]` Theme packages: `/sdcard/other/autoaudio/theme/packages`.
- `[Code-confirmed]` Theme extraction: `/sdcard/other/autoaudio/theme/extracted`.
- `[Code-confirmed]` Floating audio and custom prompt sounds are read from external files/directories selected by the user.
- `[Code-confirmed]` No business database was found; core state is stored in SharedPreferences and ordinary files.

## 12. Third-Party and Platform Dependencies

| Dependency | Purpose |
|---|---|
| Android Media API | `MediaPlayer`, `AudioRecord`, `AudioTrack`, device enumeration |
| Android AcousticEchoCanceler | AEC for real-time in-cabin amplification |
| Android Car API | Reflectively adjust the outside volume group |
| ECARX XUI AdaptAPI | Reflectively select inside/outside media playback mode |
| AutoService contract | Read and subscribe to unified vehicle properties |
| RNNoise Native | Real-time audio denoising/feedback suppression |
| AndroidX / Material | UI, Lifecycle, FileProvider, Startup |

`[Code-confirmed]` The APK itself does not directly access VHAL or `ecarxcar_service`; AutoService handles vehicle-bus adaptation.

## 13. Relationships with Other Project Software

### 13.1 AutoService: Direct, Strong Relationship

`[Code-confirmed]` AutoAudio's vehicle-state functions directly depend on `com.simplerbit.autoservice`:

```text
Vehicle VHAL / ECARX signals
          down
AutoService model mapping and unified CarPropertyValue
          down AIDL Binder
AutoAudio AudioService
          down
Prompt sounds / karaoke-public-address keys / inside-outside audio routing
```

- AutoAudio is responsible for business semantics and sound output.
- AutoService is responsible for vehicle-model, underlying-source, and signal-ID differences.
- Both share the contract package name and Parcel field order.

### 13.2 Signature Allowlist and Reachability of the Current Sample

- `[Code-confirmed]` Visible AutoService code checks the calling package and certificate SHA-1 against an allowlist.
- `[Code-confirmed]` The re-signing SHA-1 of the current AutoAudio sample is `70:92:...:A2:B1`.
- `[Code-confirmed]` This value is not among the AutoAudio allowlisted certificates listed in the visible AutoService code.
- `[Unconfirmed]` AutoService contains a SecShell wrapper; the no-execution constraint prevents confirming whether it restores or proxies the original signature result at runtime.
- The precise conclusion is therefore: the protocol, package names, components, and business relationship are confirmed; Binder reachability between these two currently re-signed samples on a real head unit cannot be confirmed from static code alone.

### 13.3 Other APKs

- `[Code-confirmed]` The AutoAudio Manifest explicitly queries only the AutoService package.
- `[Code-confirmed]` No direct binding to components of AutoDisplay, EVCC, or Hey was found.
- `[Reasoned inference]` If these applications share vehicle data, the likely relationship is that they consume AutoService in common, rather than communicating directly with AutoAudio.

## 14. Recovered Business Boundary

### 14.1 Confirmed

- Six main functional areas and their settings.
- 35 actual vehicle-event prompt sound slots.
- 21 subscribed AutoService properties and their business uses.
- AutoService explicit start/bind actions, AIDL methods, and callback model.
- Prompt sounds, themes, floating music, output devices, and inside/outside media routing.
- 48 kHz real-time speech pipeline, AEC, and RNNoise 480-sample processing.
- Public-network license, subscription, redemption, update, and theme endpoints and the signature algorithm.
- LAN QR-code activation form protocol.
- SharedPreferences and external theme directories.

### 14.2 Still Unconfirmable from the Static Sample Alone

- Final physical mapping of Android audio devices, Audio Bus, and volume group 6 in each vehicle model's firmware.
- Whether the head-unit platform provides and permits access to ECARX XUI AdaptAPI and Android Car API.
- Actual server responses, theme inventory, and license policy on the analysis date.
- SecShell's effect on AutoService runtime signature recognition and, consequently, whether the current re-signed sample can bind successfully.
- Acoustic results, echo-cancellation availability, latency, and stability; these require a dynamic audio environment and were not tested in this report.

## 15. Core Conclusions

`[Code-confirmed]` AutoAudio's core is not a simple player, but a business layer that maps vehicle state to audio behavior. Through AutoService's unified AIDL interface, it subscribes to 21 property classes, including speed, gear, door lock, doors, turn indication, passenger-seat occupancy, keys, trunk, charging, and sentry mode, then converts them into 35 inside/outside vehicle prompt slots and a push-to-talk action.

`[Code-confirmed]` The software also provides complete user-facing audio management: theme packages, online themes, custom sounds, floating tracks, device selection, volume control, and a real-time karaoke/public-address pipeline based on `AudioRecord -> AEC/gain/RNNoise -> AudioTrack`.

`[Code-confirmed]` External interfaces have three layers: local Binder/AIDL with AutoService, HTTPS JSON/binary downloads with the management site, and a short-lived LAN HTTP form for returning an activation code. No MQTT, WebSocket, serial, or self-implemented Bluetooth/USB business protocol was found.

`[Unconfirmed]` Actual cross-package Binder reachability between the two re-signed samples depends on the AutoService allowlist and SecShell runtime behavior; this is the principal boundary of the strictly static analysis.

## 16. Review of Recovered JADX Gaps

The default structured output contained 118 `Method not decompiled` placeholders across 76 Java files. The complete simple output contains 890 Java files, with zero placeholders and zero `JADX ERROR` entries.

All direct business-logic gaps have been recovered:

- `MainActivity.g()` precisely checks permissions for recording, overlay windows, external files, media audio, notifications, and installation from unknown sources, and returns a list of descriptions for missing permissions.
- `AudioService.a(CarPropertyValue, boolean)` fully covers branches for model, gear, door lock, four doors, power, speed, turn indication, passenger-seat occupancy, fuel, blind spots, keys, trunk, speed limit, charging connector, charging state, and sentry state.
- Prompt playback and delay branches in the service's internal asynchronous tasks are covered by simple/smali output.

The remaining failures in the default output primarily concern AndroidX, Material, layout, font, collection, and network helper code. The recovered results do not overturn the original functional conclusions, but elevate the unified event processing in Section 7.4 from inference to code-confirmed status.
