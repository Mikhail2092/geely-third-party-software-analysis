# EVCC v2.9.46 Static Analysis Report

## 1. Scope of the Report

This report conducts a purely static analysis of the Android Automotive software `com.kooo.evcc` (EVCC v2.9.46), aiming to recover its software architecture, all identifiable functional modules, vehicle data interfaces, key business processes, inter-process interfaces, network and custom protocols, native components, resource files, and related software as thoroughly as possible.

During the analysis, APK was neither installed nor launched, and no dynamic debugging was performed. No vehicles, servers, Bluetooth devices, or in-vehicle systems were connected. This report does not assess security, privacy, or compliance risks.

Evidence Levels:

| Tag | Meaning |
|---|---|
| **Code Confirmed** | Decompiled Java/Smali, Manifest, JNI declarations, or constants can directly prove |
| **Resource Confirmed** | Strings, layouts, JSON, Proto, scripts, DBC, or other resources can directly prove |
| **Reasonable Inference** | Multiple static pieces of evidence are consistent but have not been validated at runtime |
| **Cannot Confirm** | Obfuscation, native encapsulation, cloud configuration, vehicle model differences, or restrictions on dynamic analysis |

## 2. Sample Identity

| Item | Result | Evidence Level |
|---|---|---|
| Software Name | EVCC | Resource Confirmed |
| Original File Name | `EVCC_v2.9.46_разблокированный_подписанный.apk` | Code Confirmed (File System) |
| Package Name | `com.kooo.evcc` | Code Confirmation |
| Version | `2.9.46` | Code Confirmation |
| versionCode | `249` | Code Confirmation |
| APK Size | 34,420,627 bytes | Code Confirmation |
| SHA-256 | `5EE838B1940C1D9F20B4B4AC67B244218764ABE09856A6951A26D3B0AFD47A4C` | Code Confirmation |
| minSdk | 28 | Code Confirmation |
| targetSdk | 30 | Code Confirmation |
| compileSdk | 36 | Code Confirmation |
| ABI | `arm64-v8a` | Code Confirmation |
| Current Signing Certificate SHA-256 | `BB:CA:73:98:98:13:6E:6F:84:77:AF:0D:32:53:AE:10:B5:B6:DE:C2:F6:3D:D4:6D:34:FB:5E:AF:83:6C:16:3F` | Code Confirmation |
| Application | `com.kooo.evcc.EvccApplication` | Code Confirmation |
| Main Activity | `com.kooo.evcc.MainActivity` | Code Confirmation |
| Target Device | Android Automotive, `required=true` | Code Confirmation |
| Author Resources | `Author/Copyright@46` | Resource Confirmation |
| Bridge Protocol Version | `11` | Code Confirmation |

## 3. Product Positioning and Overall Architecture

**Code Confirmation: EVCC is an integrated control, monitoring, automation, projection, media, scripting, and system tools platform for Android Automotive car systems.** It is not just a vehicle attribute panel, but unifies multiple car system capabilities into the same application:

- Read/write vehicle attributes from sources such as VHAL, APVP, Android CarPropertyManager, ECARX, AIDL/CAPI;
- Render information cards, switches, buttons, sliders, charts, and music components on a configurable console;
- Achieve car system automation through event triggers and action orchestration;
- Provide HUD, instrument cluster, floating window, small window, multi-screen mirroring, and network secondary screen;
- Provide media session bridge, music forwarding, and desktop lyrics;
- Provide local/remote file management, APK installation, application management, ADB terminal;
- Provide Lua, Frida scripts, and script store;
- Provide MQTT, HTTP, Webhook, messaging bots, and remote control integration;
- Execute commands requiring system-side capabilities uniformly through RootService;
- Provide Binder and ContentProvider backend for standalone EVCC mini.

Overall Layering:

```text
UI: Console / Properties / Automation / Services / Scripts / Applications / Files / Terminal / Settings
  |
Business Layer: Dashboard / Automation / Media / Cast / File / Script / Remote
  |
Unified Bridging: EvccBridgeService / RootService Client / Vehicle Property Hub
  |
Access Layer: VHAL / APVP / Android Car / ECARX AIDL / CAPI / MQTT / HTTP / BLE
  |
System and Peripherals: In-vehicle system services / Vehicle bus mapping / Display / Audio / Network devices
```

## 4. APK Structure and Key Resources

### 4.1 DEX and Obfuscation

Business code is obfuscated with R8. Many inner classes are located in `defpackage`, but main package names, component names, protocol constants, resource texts, and data models can still be recovered. Some extremely large methods are marked by JADX as not fully decompiled; the report cross-verifies through call points, Smali, constants, and adjacent models.

### 4.2 Native Libraries

| File | Main Role |
|---|---|
| `libevcc_native.so` | License, device ID, signature requests, configuration unlocking, server/gRPC/RootService constants, theme/resource packages, signature verification, etc. |
| `libevcc_raw.so` | AVM Original camera initialization, open, frame capture, close |
| `libevcc_rootsqlite.so` | RootService Side SQLite query and execution |

### 4.3 Important assets

| Resource | Function |
|---|---|
| `default_dashboard.json` | Default console, only includes welcome section and setup wizard entry |
| `can_dbc.json` | Vehicle signal dictionary with 9 buses, 379 messages, 7621 signals |
| `iplm_ctrl.dex` | Independent `app_process` IPLM resource group command-line tool |
| `iplm_ctrl.java` | Complete Java source code corresponding to the above DEX |
| Built-in Frida scripts | Hook/assist functions for Activity, Toast, vehicle services, camera, HUD, etc. |
| APVP Proto | `apvp_proto/ApvpTransfer.proto` |
| placeholder APK resource | Resource for media app placeholder/navigation support |

## 5. Android Components

### 5.1 Activity

| Activity | Function |
|---|---|
| `MainActivity` | Main interface and nine navigation modules |
| `filemanager.ApkInstallActivity` | Receive content/file APK and install |
| `filemanager.MediaPreviewActivity` | File media preview |
| `store.ContentStoreActivity` | Content store |
| `scriptstore.ScriptStoreActivity` | Script store |
| `ui.RemindActivity` | Full-screen alert |
| `ui.SetupWizardActivity` | Setup wizard |
| `ui.PhoneBlockerActivity` | Immediately write the current reachable path to `dismissed=true`, return `RESULT_OK` and terminate; the prompt text and mini-game UI reserved in DEX are located after explicit `return` and are unreachable |
| `mediacenter.MusicRedirectActivity` | Current media app jump |
| `hud.HudTestActivity` | HUD test |
| `ui.DashcastActivity` | Instrument screen casting configuration and control |
| `ui.NetworkScreenActivity` | Network secondary screen configuration and control |
| `dashcast.MediaProjectionRequestActivity` | MediaProjection authorization relay |
| `cloudprobe.CloudProbeActivity` | Local Machine Cloud Message Bus Probe |

### 5.2 Service

| Service | Function |
|---|---|
| `MiningService` | Internal Application Service Module |
| `EvccBridgeService` | EVCC mini Data and Action Bridge |
| `MidebaoLinkBridgeService` | Midbao Link AIDL Bridge |
| `KeepAliveService` | Frontend Keep-Alive, Car Machine Recovery, Auto-Start Submodule Coordination |
| `OverlayCastService` | Floating/Overlay Screen Casting |
| `AutomationService` | Automation Rule Engine |
| `DashcastProjectionService` | Instrument Screen Casting |
| `DashcastCardService` | Instrument Information Card |
| `NetworkScreenService` | Network Secondary Screen Encoding and Sending |
| `DashcastKeepAliveService` | Dashcast Keep-Alive |
| `FloatingWindowService` | Floating Window |
| `DesktopLyricOverlayService` | Desktop Lyrics |
| `MiniWindowService` | App small window / system small window |
| `DisplayMirrorService` | Multi-screen display |
| `EngineSoundService` | Simulated sound wave |
| `SystemDiagnosticUploadService` | System diagnostic material upload |
| `KeepAliveAccessibilityService` | Auxiliary function background keep-alive / interaction support |
| `MediaNotificationListener` | Media notification and status source |
| `StatusBarPluginService` | Flyme Auto native status bar plugin |
| `StatusBarPluginBridgeService` | Status bar plugin and EVCC data bridge |

### 5.3 Provider

| authority | Implementation | Function |
|---|---|---|
| `com.kooo.evcc.artwork` | AndroidX FileProvider | Provide cached cover to mini and others URI |
| `com.kooo.evcc.config` | `EvccConfigProvider` | Provide pages, grids, themes, and versions to mini |
| `com.kooo.evcc.displays` | `DashcastDisplayProvider` | Provide / register screen casting monitor information |

### 5.4 Receiver

| Receiver | Function |
|---|---|
| `MiniUpdateRequestReceiver` | Receives mini update check requests |
| `PlaceholderLaunchReceiver` | Launches the current bridged music application |
| `BootReceiver` | System/car system startup and wake-up recovery |
| `SystemEventReceiver` | Screen, power, network, Bluetooth, USB, time, and other events |
| `KeepAliveReceiver` | Keep-alive event handling |
| `MiniStateReceiver` | Mini status synchronization |
| `StatusBarPluginClickReceiver` | Status bar widget click distribution |

## 6. Startup, Keep-Alive, and Car System Recovery

### 6.1 Application Startup

`EvccApplication.onCreate()` first checks whether the Android user is unlocked:

- If not unlocked, registers the `USER_UNLOCKED` Receiver and delays business initialization;
- If unlocked, proceeds to the unified initialization method;
- Handles the scenario where system_server is restarting, skipping this round of background initialization;
- Preserve the original DisplayMetrics, and reapply EVCC DPI scaling when there is a configuration change.

The main initialization method has been fully expanded through JADX simple view and smali. Confirmable sequences include: establishing global Context and lifecycle callbacks; rotating current/last session logs and installing uncaught exception handling; migrating old debug configurations; identifying previous session crashes or black screen signals and scheduling diagnostic uploads according to frequency limits; initializing runtime configuration, media bridge, and theme status; starting vehicle diagnostics and optional modules such as EVA according to cloud configurations; generating startup scripts from ADB public key; starting KeepAliveService according to configuration; and finally restoring VHAL/APVP throttle parameters and background statuses such as floating windows.

### 6.2 MainActivity Startup

Recoverable key sequences:

1. Handle watchdog restarts and special jump Intents;
2. Identify window/display environment;
3. Call keep-alive service startup auxiliary functions;
4. Apply themes, backgrounds, and DPI;
5. Construct main navigation and pages;
6. Schedule mini update check after 90 seconds delay;
7. Automatically connect Root/`evcc_auto_connect`, automation, media bridge, and other modules according to ADB;
8. Handle deep links for content store, service pages, small windows, external actions, etc.

### 6.3 BootReceiver

Listen to Android and automaker broadcasts:

- `BOOT_COMPLETED`, `LOCKED_BOOT_COMPLETED`;
- Android/HTC QUICKBOOT;
- `MY_PACKAGE_REPLACED`, `REBOOT`;
- ECARX `SYSTEM_READY`, `ACC_ON`, `IGN_ON`, `WAKEUP`, `STR_RESUME`, `POWER_ON`;
- LynkCo/Geely `SYSTEM_READY`, `WAKEUP`, `STR_RESUME`;
- SystemUI ready and STR mode.

When the user is unlocked and auto-start configuration is enabled, Receiver starts `KeepAliveService`. When a media center broadcast arrives, it will first request the existing media bridge to re-register; the keep-alive service starts only if the bridge is not running.

### 6.4 KeepAliveService

This service runs with a foreground notification, mainly coordinating:

- RootService/ADB automatic connections;
- Automatic engine auto-start;
- Media center bridge and desktop lyrics restoration;
- Screen/ACC/sub-service recovery after deep sleep wake;
- DPI daemon injection;
- Automatic application of dashboard model colors;
- Small window, casting, remote services, etc. restored according to configuration;
- Alarm restart after task removal or service destruction.

`onStartCommand()` Identifies trigger broadcasts, deep sleep time intervals, recovery attachments, TLS self-healing flags, and returns the corresponding Service restart mode according to the auto-start configuration.

## 7. Main Interface and Navigation Module

Resources and adapters confirm that the main navigation includes:

1. Console
2. Properties
3. Automation
4. Services
5. Scripts
6. Applications
7. Files
8. Terminal
9. Settings

Visibility of different modules is determined by cloud configuration, license status, vehicle model capabilities, and local switches. **Code confirmation: `service.show_*` Type configuration controls service card display; unable to confirm the final set returned by the current server for a specific account.**

## 8. Vehicle Property Access and Unified Model

### 8.1 Unified Model

Upper layer uniformly uses:

```text
propertyId : int
areaId     : int
rawValue   : String / typed value
source     : VHAL / APVP / AIDL / CAPI
```

The console, automation, mini, simulated engine sound, MidBao, and remote services all reuse this model.

### 8.2 VHAL / Android Car

Two VHAL paths can be confirmed:

- Android `CarPropertyManager` read, subscribe, and write directly;
- gRPC VehicleServer path.

Proto methods:

| Service/Method | Function |
|---|---|
| `vhal_proto.VehicleServer/GetAllPropertyConfig` | Get all property configurations |
| `vhal_proto.VehicleServer/GetProperty` | Get specified property |

NativeBridge also provides encapsulated method names, semantics include:

- `GetAllConfig`
- `StartStream`
- `SendAll`
- `SetProperty`
- Derived `GetProperty`

gRPC metadata includes `client_id`. The actual host, port, clientId, and some full method names are unsealed via `libevcc_native.so`, **making it impossible to confirm their static plain values**.

### 8.3 APVP

Resource Proto: `apvp_proto/ApvpTransfer.proto`.

Confirmed service methods:

| Service | Method |
|---|---|
| `transfer_proto.TransferDebugServer` | `getAllTransfer` |
| Same as above | `getTransferSignalConfig` |
| Same as above | `setTransferMode` |
| Same as above | `setReady` |
| Same as above | `updateSignals` |
| `transfer_proto.TransferServer` | `readSignal` |
| Same as above | `writeSignal` |
| Same as above | `listenerSignalStream` |

APVP attributes can be filtered and subscribed by modules such as console, automation, and simulated sound based on source.

### 8.4 ECARX AIDL/CAPI

In the code, ECARX vehicle service AIDL and CAPI adaptation layer, `EvccBridgeService` collector are clearly divided into:

- `collectVhal`
- `collectApvp`
- `collectAidl`
- `collectCapi`

Data from each source is eventually converted to a unified propertyId/areaId/value update stream.

### 8.5 Attribute Dictionary

`assets/can_dbc.json`:

| Bus | message | signal |
|---|---:|---:|
| ChassisCAN1 | 11 | 91 |
| PropulsionCAN | 111 | 865 |
| ConnectivityCANFD | 32 | 996 |
| PrivateInfoCAN | 63 | 444 |
| DHU_LIN3 | 6 | 52 |
| IHU_LIN2 | 13 | 84 |
| ZCU_CANFD2 | 68 | 3434 |
| DIM_LIN1 | 5 | 29 |
| ZCU_CANFD1 | 70 | 1626 |
| **Total** | **379** | **7621** |

Resource source field: `SDB2243203_E245_DHU2_240812_VehicleParsingFile246.json`, version `1.0.3`. Signal fields include name, start, length, endianness, sign, scale, unit, etc.

**Resource confirmation: this file is a vehicle signal/attribute dictionary. Unable to confirm: currently APK has not found a complete service chain for directly sending and receiving frames, so the existence of DBC cannot be equated with direct application transmission and reception of CAN.**

### 8.6 Attribute Page and Connection Page

The main navigation “Attributes” is used to view and modify vehicle attribute values, with resource documentation clearly supporting real-time monitoring and manual writing. Confirmed functions include:

- Display attribute ID, area ID, current value, and data source;
- Read attributes according to VHAL, APVP, AIDL, CAPI connection status;
- Manually input propertyId, areaId, and the target value to perform writing;
- View the writable attribute list of AIDL;
- Configure the throttling mode for VHAL/APVP;
- Switch between on-demand subscription and full subscription of APVP;
- Display the current connection status for modules such as console, automation, and sound wave.

The unified listening/reading priority is explicitly written in the interface code as `VHAL -> APVP -> AIDL -> CAPI`. APVP only adds on-demand subscriptions for IDs identified as APVP signals, while other sources continue to use their respective real-time change streams.

## 9. Console

### 9.1 Pages and Layout

The console supports:

- Multiple tabs;
- Editing tab titles and order;
- Configurable grid column numbers, with a 10-column layout option in the code;
- Component drag-and-drop, scaling, and position saving;
- Themes, skins, icons, and backgrounds;
- Configuration import, export, sharing, and content store templates;
- Synchronizing pages to EVCC mini.

By default, `default_dashboard.json` does not preset real vehicle control, containing only a welcome section and setup wizard button.

### 9.2 Component Types

| Type | Function |
|---|---|
| INFO | Information card / numeric display |
| TOGGLE | Boolean switch |
| BUTTON | Single Action Button |
| SLIDER | Continuous Value Control |
| DROPDOWN | Dropdown Option |
| BUTTON_GROUP | Multi-Button Group Option |
| SECTION | Section Title |
| CHART | Property History Line Chart |
| MUSIC | Media Control Component |

### 9.3 Data Expressions and Mapping

Expressions support:

- `$variableName`
- `$prop(PropertyID)`
- `$prop(PropertyID, AreaID)`
- Mathematical and conditional operations

Value mapping conditions include:

- Equal/Not Equal;
- Greater Than/Less Than;
- Range;
- Direct Match;
- Fallback Rule.

Mapping results can change displayed text, color, icons, or other component states. Charts support historical persistence, time windows, color, range, and zoomed-in line charts.

### 9.4 Action Execution

Console interactions ultimately enter the component action engine or `ActionRouter`. Types that can be directly routed include buttons, switches, sliders, dropdowns, and button groups; complex actions enter the execution layer shared with automation.

## 10. Automation Engine

### 10.1 Rule Data

Main configurations:

- `evcc_automation_rules`
- `evcc_automation_variables`
- `automation_boot`
- `evcc_tts_config`

A rule consists of trigger conditions, constraints, action sequences, enable status, and execution records. `AutomationService` is responsible for subscribing to event sources and scheduling executions.

### 10.2 Triggers

Code/resource confirmed triggers:

| Category | Trigger |
|---|---|
| Manual/System | Manual, power on, screen on, central control power events |
| App | App enters the foreground |
| Bluetooth/BLE | BLE signal, Bluetooth switch, device connection, device disconnection |
| Input/System | Panel buttons, received broadcasts, log keywords |
| Property | Property changes, equal to, greater than, less than, range, contains, double-click |
| Variable | Variable judgment |
| Time | Cron, Cycle, Sunrise, Sunset |
| Location | Geofence Entry, Exit |
| Media | Music Events |

### 10.3 Actions

| Category | Actions |
|---|---|
| Vehicle/Data | Attribute Write, Loop Attribute, Set Variable, Trigger Other Automation |
| App/Task | Launch, Background, Exit App, Return to Home |
| Display | App Casting, Dual-Screen Display, App Floating Window, System Floating Window |
| Media/Audio | Previous, Next, Play/Pause, Volume, Play Sound, TTS |
| System | Shell, ADB, Settings, sysprop |
| Network | HTTP, Webhook, Remote Message |
| Android | Notification, Toast, Broadcast |
| Peripheral | BLE Broadcast, Hotspot, Midibao |
| Script | Lua/Frida Start and Stop |
| Flow | Delay, Conditional Interrupt, End Automation |

### 10.4 Typical Execution Flow

```text
Event Source
 -> AutomationService Match Enabled Rules
 -> Condition/Variable/Time Window Evaluation
 -> Sequential Action Execution
 -> Action calls property layer, RootService, media, network, screen casting, or script module
 -> Update running status and subsequent rules
```

## 11. Service Module Overview

The service page displays the following capabilities through cloud configuration and local feature switches:

| Module | Implementation / Function |
|---|---|
| Music Forwarding Bridge | Unified media session with third-party music apps |
| Media Center | Track information, control, current app navigation |
| Desktop Lyrics | Overlay display of lyrics |
| HUD Screen Casting | HUD Test, bind, start/stop screen casting |
| Main Screen Map Enhancement | Scripts or services to enhance map/navigation display |
| Dual-Screen Mirroring | Mirror the main interface to other displays |
| Dashcast | Dashboard screen casting and information cards |
| Floating Window | Vehicle data or application overlay |
| Native Status Bar Plugin | Flyme Auto plugin service |
| STR Deep Sleep Protection | Services before and after deep sleep and connection recovery |
| DPI Injection | Target App/Display DPI Adjust Guardian |
| Remote Control | MQTT / Message Bot / HTTP linkage |
| Broadcast Station | BLE, UDP, HTTP Data Broadcasting |
| Mini Program MQTT | Publish vehicle properties and variables |
| Midbot | BLE Moving Equipment and AIDL Bridge |
| EVA Linkage | Third-party linkage entry |
| AVM | Panoramic Camera, Raw Frames, Speed Limit Removal |
| Instrument Vehicle Model Color | Automatic/Manual modification of instrument vehicle model color |
| Dynamic Wallpaper | Import and Web Editor |
| Bottom Bar Debug | Vehicle system bottom bar visual/behavior debugging |
| Network Secondary Screen | H.264/JPEG Network Display |
| Hotspot Enhancement | 5GHz/AC Hotspot Mode |
| HUD Direct Cast | Attribute Activation, Bind CastService, Start Session |
| Simulated Engine Sound | Vehicle Attribute Driven Audio Synthesis |
| Lock Car Sound | Lock-related sound actions |
| CloudProbe | Local MQTT Cloud Message Mirroring |
| API Reflective Call | System Service/Method Browsing and Invocation |
| frpc | Intranet Penetration Deployment and Management |
| Bluetooth Scan Enhancement | Extended Bluetooth Discovery and Status |

**Unable to confirm: Whether the service card is shown for a specific account/model depends on Native configuration, cloud configuration, and local status. The table only indicates that the code includes the relevant implementation.**

## 12. EVCC Mini Backend

### 12.1 ContentProvider

authority: `com.kooo.evcc.config`, read permission: `com.kooo.evcc.permission.BIND_EVCC`.

| Path | Return Content |
|---|---|
| `/main_info` | versionCode, versionName, bridgeVersion=11 |
| `/mini_grid` | cols, rows=10, dpiScale |
| `/mini_pages` | pageId, title, type, pageJson |
| `/mini_settings` | immersive=1, themeMode, musicPageStyle |
| `/current_skin` | Current skin JSON |

When there is no separate mini page, the Provider will serialize the current console component into the default DASHBOARD page.

### 12.2 IEvccBridge v11

descriptor: `com.kooo.evcc.bridge.IEvccBridge`.

| Transaction | Method | Function |
|---:|---|---|
| 1 | `getBridgeVersion` | Return bridge version |
| 2 | `subscribeValues` | Subscribe to attribute |
| 3 | `unsubscribeValues` | Unsubscribe from attribute |
| 4 | `executeAction` | Execute console action |
| 5 | `registerMusicCallback` | Register music callback |
| 6 | `unregisterMusicCallback` | Cancel music callback |
| 7 | `sendMusicCommand` | Play control/Seek |
| 8 | `setVariable` | Set automation variable |
| 9 | `getVariablesJson` | Batch read variables JSON |
| 10 | `subscribeSystemMetrics` | Subscribe to system metrics |
| 11 | `unsubscribeSystemMetrics` | Unsubscribe from system metrics |
| 12 | `enterMiniWindowMode` | Enter mini window |
| 13 | `getMiniEntitlementState` | Query mini status |
| 14 | `launchPackageFullscreen` | Full-screen startup package |
| 15 | `openContentStore` | Open specified store content |
| 16 | `subscribeToggleStates` | Subscribe to component switch status |
| 17 | `unsubscribeToggleStates` | Unsubscribe switch status |
| 18 | `saveMiniPageConfig` | Save mini page |
| 19 | `prepareSystemMiniWindowDpi` | Prepare target App/Display DPI |
| 20 | `launchPackageOnDisplay` | Launch package on specified display |
| 21 | `isDisplayIdle` | Query display idle state |
| 22 | `injectDisplayTouch` | Inject touch to display, oneway |
| 23 | `injectDisplayKey` | Inject key to display, oneway |
| 24 | `enableSystemMiniWindowFullscreenButton` | Enable system small window fullscreen button |

The server maintains four types of callback collections: properties, system metrics, music, and switch status, and cleans up invalid subscriptions via Binder death recipient.

### 12.3 mini status chain and the current fixed client branch

Location of full version server code:

```text
.analysis-work/evcc/jadx/sources/com/kooo/evcc/bridge/EvccBridgeService$binder$1.java:252
```

`getMiniEntitlementState()` reads data from the mini status item of AddonManager, return value model is:

| Return Value | Server Semantics |
|---|---|
| `none` | No mini state item |
| `active` | Current state is valid |
| `stale` | Update time difference of active state exceeds constant `2592000` |
| `trial:<...>` | Trial state and remaining information |
| `expired` | Trial/state has expired |

The full original state execution chain is still retained:

- `EvccBridgeService.isMiniEntitled()` calls AddonManager for judgment;
- `getVariablesJson()` returns empty JSON when the state is not satisfied;
- `subscribeValues()` does not establish property subscription when the state is not satisfied;
- `registerMusicCallback()` does not register music callback when the state is not satisfied;
- `subscribeSystemMetrics()` does not establish system metric subscription when the state is not satisfied;
- `subscribeToggleStates()` does not establish switch state subscription when the state is not satisfied;
- When Addon state detects qualification revocation, it clears existing property, music, system metric, and switch subscriptions.

The mini v2.3.8 client analyzed this time has different implementations:

```text
.analysis-work/evcc_mini/jadx/sources/com/kooo/evcc/mini/BridgeClient.java:388
getMiniEntitlementState() -> return "active"
```

The mini Activity still retains the interface branch of `none/active/stale/trial/expired` and the 30-second refresh logic, but actually calls the above client encapsulation, so in this sample, the local state display path is fixed to `active`, and the remote state query transaction defined in AIDL will not be executed.

Static conclusions:

- **Code confirmation:** This mini client build has a fixed `active` state branch;
- **Code confirmation:** The full version of EVCC original state storage, state calculation, server entry judgement, and post-cancellation subscription cleanup have not been removed;
- **Code confirmation:** The client's fixed state and server subscription judgement are two separate levels; the former cannot prove that the latter is bypassed or invalid;
- **Unable to confirm:** The source and formation method of this mini fixed branch, as well as the final behavior of the two APK when actually connected in the target car machine. This report does not run the sample nor make a risk assessment of this.

## 13. Media Center and Music Bridge

Confirmable capabilities:

- Listen to MediaSession / notify media status;
- Unify title, artist, album, duration, position, play status, lyrics, and package name;
- Cache cover art and provide content URI through `com.kooo.evcc.artwork`;
- Play/Pause, Previous, Next, Seek;
- Current music app jump;
- Music placeholder package works with `PlaceholderLaunchReceiver`;
- Desktop lyrics overlay;
- Forward media status to mini and MidBao modules;
- Re-register bridge after broadcasting in car media center.

Media command protocol is consistent with mini: 0 Play/Pause, 1 Next, 2 Previous, 3 Seek.

## 14. Screen Casting, Monitors, and Small Windows

### 14.1 Dashcast Instrument Screen Casting

Dashcast has multiple data paths at the same time and cannot be mixed as the same protocol.

#### H.264 RTP

- Encoder: Android MediaCodec `video/avc`;
- RTP version 2;
- Payload type 96;
- 12-byte standard RTP header;
- Fields include marker/PT, sequence, timestamp, SSRC;
- UDP transmission;
- I-frame interval is 1;
- Uses low latency configuration.

#### BGRA + LZ4 TCP

| Field | Value/Description |
|---|---|
| magic | `0x42475241`, ASCII, `BGRA` |
| Byte order | big-endian |
| v1 header | 36 bytes |
| v2/v3 header | 44 bytes |
| Pixel | BGRA |
| Compression | LZ4 |
| Metadata | version, width, height, stride, timestamp, rawLen, compressedLen, block information |
| Empty frame | v2 empty frame can be used as idle heartbeat |

Dashcast also collaborates through DisplayProvider, ProjectionService, CardService, and KeepAliveService to manage virtual displays, projection authorization, information cards, and background recovery.

### 14.2 Network Secondary Screen

`NetworkScreenService` renders vehicle data/custom screens and sends them to LAN devices, supporting:

- Target IP and port;
- Output size and rotation;
- Frame rate and image quality;
- Background, theme, and signal expressions;
- Simulated data;
- LAN discovery/scan;
- H.264 RTP mode;
- JPEG fragment mode.

H.264 RTP uses the same standard 12-byte RTP structure as mentioned above. JPEG First compress according to the configured quality, then split into payload blocks of up to 1400 bytes. Each UDP fragment uses a 16-byte little-endian header:

| Offset | Length | Field |
|---:|---:|---|
| 0 | 2 | magic `0x5354`, online byte is `54 53` |
| 2 | 2 | reserved field, fixed at 0 |
| 4 | 4 | total JPEG bytes |
| 8 | 4 | fragment sequence number, starting from 0 |
| 12 | 4 | total number of fragments, calculated according to `(length + 1399) / 1400` |

After the header, directly append this fragment's JPEG data. The receiver can reassemble according to the fragment sequence number and verify the complete frame with the total length.

### 14.3 Cross-screen display, small windows, and input injection

`DisplayMirrorService`, `MiniWindowService` work with Bridge transactions 19..24 to accomplish:

- Launch the App on the specified display;
- Check if the virtual display is idle;
- Inject touch events and key presses;
- Prepare DPI for the system small window;
- Allow the system small window full-screen button;
- Switch between normal fullscreen, small window, and secondary screen tasks.

### 14.4 HUD and AVM

The resource definition for HUD direct cast shows the typical sequence:

1. Write to `SETTING_FUNC_HUD_ACTIVE=true`;
2. Asynchronously bind CastService;
3. After waiting for connection, initiate HUD projection;
4. The current session can be stopped.

AVM obtains raw camera frames through `libevcc_raw.so`'s `nativeInit/open/grabFrame/close` and stores them in the panoramic camera display and service entrance related to speed limit lifting.

## 15. File Manager

### 15.1 Local Files

Supports:

- Directory browsing;
- Copying, moving, deleting, renaming;
- Compression and decompression;
- Text editing;
- Image/audio-video preview;
- APK recognition and installation;
- content/file URI handling.

### 15.2 Remote Connection

| Type | Protocol/Capability |
|---|---|
| 123 Cloud Drive | Account password, QR code login, directory and file operations, multi-domain compatibility |
| WebDAV | PROPFIND, MKCOL, PUT, DELETE, GET |
| FTP | FTP Directory and file operations |
| Flying Cow NAS Share | Share link parsing and download |
| EVCC Download site | Public file browsing and download |

**Code confirmation: File manager `RemoteType` does not have SFTP.** JSch appears in other remote deployment logic, which cannot be used to conclude that the file manager supports SFTP.

123 Cloud Drive related domains:

- `yun.123pan.cn`
- `login.123pan.com`
- `www.123pan.cn`
- `www.123865.com`
- `www.123912.com`
- `www.123684.com`

## 16. Script System

### 16.1 Frida

Built-in / downloadable Runtime versions include:

- Frida 16.5.9
- Frida 17.9.11

Functions:

- Select target App/process;
- attach or spawn inject;
- Start/stop Frida daemon;
- Manage sessions and outputs;
- Edit, favorite, and run scripts;
- Download/upload from the script store.

Built-in script topics:

- Activity lifecycle;
- Toast interception;
- SSL pinning bypass;
- VHAL/AIDL/CarService tracing;
- Switch from main camera to front camera;
- Clipboard;
- HUD;
- Main screen map;
- Instrument cluster model color;
- Other car infotainment specific hooks.

### 16.2 Lua

Includes Lua editor, syntax highlighting, reference to API, and automation action integration. The editor uses the Sora code editing component. Lua can read variables/properties and call the action interfaces exposed by EVCC. Specific availability API is defined by built-in reference resources.

### 16.3 Script Store

Supports script list, details, download, upload, and version retrieval. Frida resource endpoints:

- `/frida/scripts.json`
- `/frida/scripts/<file>`
- `/frida/<variant>`

## 17. Application Management and Terminal

### 17.1 Application Management

Confirmable functions:

- Enumerate installed apps;
- Launch apps;
- Install and uninstall;
- Freeze/enable packages;
- View and adjust permissions/component status;
- End process or remove task;
- Move tasks to specified monitor;
- Coordinate with mini-window, casting, and DPI injection.

Operations requiring system-level privileges are performed through RootService or ADB.

### 17.2 Terminal

Functions:

- Built-in ADB Shell;
- Command history;
- Command favorites;
- Log capture;
- Direct connection ADB and wireless debugging configuration;
- Automatic connection and startup RootService.

Main configuration files: `evcc_adb_config` and `evcc_auto_connect`.

## 18. RootService Custom Protocol

### 18.1 Startup Method

RootService is started via `app_process` under the root environment:

```text
com.kooo.evcc.rootservice.RootServiceMain
```

Preferred `LocalServerSocket` name:

```text
evcc_rootsvc
```

TCP fallback only listens on loopback and tries in order:

- `127.0.0.1:40006`
- `127.0.0.1:40007`
- `127.0.0.1:40008`

### 18.2 Handshake

Protocol handshake:

1. Server sends big-endian `int32=32`;
2. Followed immediately by a 32-byte random challenge;
3. Client returns `int32=32`;
4. Immediately follow with `HMAC-SHA256(challenge, APK signing certificate hash)`;
5. The server compares the result using constant time MessageDigest.

This is the code fact of the protocol identity verification process; this report does not make a risk assessment on it.

### 18.3 Message Frame

```text
int32 big-endian payloadLength
UTF-8 JSON payload
```

- Maximum request 1 MiB;
- Request common fields: `id`, `cmd`;
- Response reserved `id`;
- Both success results and error messages are carried in the JSON field.

### 18.4 Command Words

| cmd | Semantics |
|---:|---|
| 17 | ping |
| 33 | settingsGet |
| 34 | settingsPut |
| 49 | syspropGet |
| 50 | syspropSet |
| 65 | Binder/Reflective invoke |
| 81 | serviceList |
| 82 | serviceMethods |
| 97 | toast |
| 113 | shellExec |
| 129 | startWatchdog |
| 130 | stopWatchdog |
| 131 | watchdogStatus |
| 145 | EAS mount-bind command has been removed |
| 146 | Removed EAS mount-bind command |
| 161 | Frida inject |
| 162 | Frida stop |
| 163 | Frida list |
| 164 | Frida output |
| 165 | Frida spawnInject |
| 166 | Frida daemon |
| 177 | inputInject |
| 193 | Car API |
| 209 | SQLite query |
| 210 | AdaptAPI bridge |
| 211 | removeTaskByPackage |
| 225 | bindFile |
| 226 | unbindFile |
| 227 | checkBindStatus |
| 241 | moveTaskToDisplay |
| 242 | rootAutomation |
| 243 | removeTaskById |
| 257 | audioKtvMode |
| 273 | carAudioVolume |
| 274 | SQLite exec |
| 289 | carPowerEvents |
| 305 | stockNaviSnapshot |

RootService is an important common execution backend for application management, terminals, Frida, monitor tasks, system settings, in-vehicle API, and automated system actions.

## 19. MQTT Protocol

### 19.1 CloudProbe

Local Broker: `127.0.0.1:1883`, implements a read-only probe interface for NanoMQ/in-vehicle cloud message bus.

Functions:

- Subscribe to the local message bus;
- Classify by CAN, HMI, Navigation, Media, Charging, Accounts, Diagnostics, Remote Control, etc.;
- Display shadow, bindings, topics;
- Count message categories and quantities;
- View raw messages.

"Read-only" is confirmed by product copy and interface resources.

### 19.2 Mini Program MQTT

Configuration: `evcc_mini_program_mqtt`.

| Parameter | Behavior |
|---|---|
| scheme | `ssl` / `mqtts` / `tcp` / `mqtt` |
| Default Port | 8883 |
| clientId | `evcc_mini_<username>_<deviceSuffix>`, up to 80 characters after cleaning |
| clean session | true |
| connect timeout | 15 seconds |
| keepalive | 30 seconds |
| reconnect | automatic |

LWT:

```text
topic: evcc/<username>/status
body: offline
QoS: 1
retain:true
```

Status topic:

```text
Vehicle attributes: evcc/<username>/<propertyId>/state
Variable: evcc/<username>/var/<name>/state
```

Status publishing QoS 0, non-retain. Default attribute IDs:

```text
561024570, 561025079, 561024448, 561024549, 557849089,
561023639, 557874774, 561024172, 561025056, 561025057,
559947198, 561024651, 561024565
```

Users can also add `$variable_name`, `gps_lng`, `gps_lat`.

### 19.3 Home Assistant MQTT

Configuration: `evcc_remote_bot`.

Behaviors that can be confirmed:

- Supports tcp/ssl;
- Default port 1883;
- LWT is `evcc/<deviceId>/status`;
- Map console components to Home Assistant entities;
- Publish auto-discovery configuration;
- Receive control commands and route them to console actions.

The auto-discovery publishing method has been fully restored from JADX simple view and smali. Component type mapping is: `INFO/CHART -> sensor`, `TOGGLE -> switch`, `BUTTON -> button`, `SLIDER -> number`, `DROPDOWN/BUTTON_GROUP -> select`, `SECTION_HEADER/MUSIC` does not publish entities.

Discovery topic and control topic:

```text
homeassistant/<entityType>/<deviceId>/<componentSlug>/config
evcc/<deviceId>/<componentSlug>/state
evcc/<deviceId>/<componentSlug>/set
evcc/<deviceId>/status
```

`componentSlug` Replace non-`[a-zA-Z0-9_-]` characters with `_`, convert to lowercase and truncate to 64 characters. Discovery message QoS 0, retain=true. Common payload fields are `name`, `unique_id`, `object_id`, `availability_topic`, and `device`; `device` contains `identifiers`, `name`, `manufacturer=kooo`, `model`, `sw_version`. Entity-specific fields are as follows:

- `switch`: `state_topic`, `command_topic`, `payload_on=ON`, `payload_off=OFF`;
- `sensor`: `state_topic`, with `unit_of_measurement` and `state_class=measurement` added according to component configuration;
- `select`: `state_topic`, `command_topic`, `options`;
- `number`: `state_topic`, `command_topic`, `min`, `max`, and optional `step`, `unit_of_measurement`;
- `button`: `command_topic`, `payload_press=PRESS`.

## 20. HTTP, Webhook and Message Platform

### 20.1 EVCC Server and Download Site

Confirmed base address: `https://download.coauto.cc:9568/`.

Endpoints:

- `/api/update/stable`
- `/api/update/beta`
- `/download/changelog`
- `/api/public`
- `/frida/scripts.json`
- `/frida/scripts/<file>`
- `/frida/<variant>`

Other fixed resources:

- `https://evcc.coauto.cc/static/engine_sounds/manifest.json`
- `https://evcc.coauto.cc/static/dynamic-wallpaper-editor/index.html`
- `https://evcc.coauto.cc/static/skin-editor/index.html`
- `https://api.sunrise-sunset.org/json?lat=...&lng=...&date=...&formatted=0`

`NativeBridge.getServerUrl()` will also provide other server base addresses, but the plaintext is in the Native configuration, **unable to confirm its final value**.

### 20.2 General HTTP/Webhook Actions

Automation supports configuring HTTP method, URL, header, body, and other requests, and using the results for notifications or subsequent actions. The broadcast station can also send data as a HTTP request.

### 20.3 DingTalk

- `POST https://api.dingtalk.com/v1.0/oauth2/accessToken`
- `POST https://api.dingtalk.com/v1.0/robot/oToMessages/batchSend`

Used to obtain access token and robot batch messages.

### 20.4 Feishu

- `POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal`
- `POST https://open.feishu.cn/callback/ws/endpoint`
- `POST /open-apis/im/v1/messages?receive_id_type=...`
- `POST /open-apis/im/v1/messages/<id>/reply`

Feishu long connection uses WebSocket and custom Protobuf wire frame `Pbbp2Frame`.

Frame fields:

| field | Type/Meaning |
|---:|---|
| 1 | seqID uint64 |
| 2 | logID uint64 |
| 3 | service int32 |
| 4 | method int32, 0 control / 1 data |
| 5 | repeated header sub-message |
| 6 | payloadEncoding |
| 7 | payloadType |
| 8 | payload bytes |
| 9 | logIDNew |

header sub-message: field 1 key, field 2 value. Confirmed keys include `type`, `message_id`, `trace_id`, `seq`, `sum`, `biz_rt`; `type` supports event, ping, pong.

### 20.5 Telegram

Basic API: `https://api.telegram.org`, used for bot messages / remote control integration.

## 21. BLE, UDP and Midbao Protocol

### 21.1 Broadcast Station

The broadcast station supports three transports: BLE, UDP, HTTP.

BLE:

- manufacturer data;
- Optional service UUID and device name;
- legacy, non-connectable, scannable;
- high power, low latency;
- payload can be optional structured protocol or raw HEX;
- Android BLE API fallback exists when unavailable HCI `hcitool cmd`.

UDP:

- target host/port;
- can be fixed sourcePort;
- can receive return data when source port is fixed.

### 21.2 Midea Direct Transmission BLE

Action codes:

| code | action |
|---:|---|
| `F0` / 240 | MUSIC_START |
| `F2` / 242 | MUSIC_STOP |
| `E0` / 224 | SLOW |
| `E1` / 225 | SPEED_UP |
| `F3`..`F8` | ACTION1..5 / RANDOM |

Action frame:

```text
C6 01 <seq>  FE 55 10 <actionCode> 55 FE
```

Speed frame:

```text
C6 02 <seq> <speed> 78 00 FE 55 10 <actionCode> 55 FE
```

Default broadcast duration 500 ms, interval 80 ms. Supports speed mapping, lighting, target MAC, and test commands from BPM.

### 21.3 Media Treasure AIDL

| Item | Value |
|---|---|
| action | `com.kooo.evcc.midebao.BIND` |
| Related call package | `com.galaxy.lyricsnext` |

Transaction:

| Transaction | Function |
|---:|---|
| 1 | protocolVersion |
| 2 | status |
| 3 | snapshotJson |
| 4 | registerCallback |
| 5 | unregisterCallback |

Callback provides snapshot/status. Snapshot configuration items include fields such as key, name, propertyId or variable, areaId, map, realtime, etc.

## 22. Simulated Sound Waves and Audio

`EngineSoundService` is a foreground/background audio generation service, configuration file is `engine_sound_prefs`.

Processing chain confirmed by code:

1. Read the enabled `PropertyBinding`;
2. Subscribe to vehicle attributes such as VHAL/APVP according to source;
3. Save role values such as RPM, vehicle speed, and load according to role;
4. Normalize the attribute range;
5. Calculate sound effect parameters based on profile, curve type, and curve parameters;
6. Select the local sound package `active_pack_id/pack.json`;
7. Output audio according to master volume and audio usage;
8. Re-subscribe/load when attribute binding or configuration changes.

External manifest: `https://evcc.coauto.cc/static/engine_sounds/manifest.json`, used for discovering downloadable sound packages.

## 23. `assets/iplm_ctrl.dex` Special Analysis

### 23.1 File Identity

| Item | Value |
|---|---|
| File | `assets/iplm_ctrl.dex` |
| Size | 5300 bytes |
| SHA-256 | `3773790C09CDD430D4B3E678ECCF96D6E7771BA0C97A474FAD8D40A3EC1A2F39` |
| Main Class | Default package `IplmCtrl` |
| Corresponding Source Code | `assets/iplm_ctrl.java` |

### 23.2 Expected Execution Method

Check the source code comments and main parameters to confirm its intended usage:

```text
app_process -Djava.class.path=/data/local/tmp/iplm_ctrl.dex "
  /system/bin IplmCtrl <request|release|state|hold> <rg_id> [priority]
```

**Code confirmed: it is a CLI DEX intended to be launched independently by root shell/app_process, not a plugin loaded by DexClassLoader inside the Android application process.**

### 23.3 HIDL/HwBinder Protocol

Descriptor:

```text
vendor.ecarx.xma.iplm@1.0::IIplm
```

Transactions:

| Transaction | Method |
|---:|---|
| 1 | subscribe |
| 2 | unsubscribe |
| 3 | stopIpActivity |
| 4 | releaseRG |
| 5 | getRGState |
| 6 | requestRG |

Commands exposed by the CLI:

- `request`: requests a resource group;
- `release`: releases a resource group;
- `state`: queries resource-group state;
- `hold`: keeps the process alive after requesting a group.

Resource groups:

| rgId | Name/combination |
|---:|---|
| 2 | RG1: DHU + TCAM + ASDM |
| 4 | RG2: DHU + TCAM |
| 8 | RG3: DHU + ASDM |

priority: 0 LOW, 1 HIGH. After a successful `hold` request, the process sleeps every 60 seconds and continuously retains the resource group by keeping the process and HwBinder connection alive.

### 23.4 Call Relationship Inside the APK

Results of a global search across Java, Smali, and resource references:

- No string reference to `iplm_ctrl.dex` was found outside the asset itself;
- No reference to the `IplmCtrl` main class was found;
- No reference to this HIDL descriptor was found in the main DEX;
- No definite call chain was found that copies the asset to `/data/local/tmp`;
- No definite call chain was found that assembles the `app_process` command above;
- The located DexClassLoader instances belong to Frida Runtime or general-purpose libraries and are unrelated to this file;
- Android does not automatically load DEX files from assets.

Final assessment:

- **Code confirmed:** `iplm_ctrl.dex` is an independent command-line tool for requesting, releasing, querying, and continuously retaining IPLM resource groups.
- **Code confirmed:** the current APK's main DEX contains no reference that directly loads or executes this asset.
- **Cannot be confirmed:** whether the UI/backend invokes it indirectly through runtime command assembly, cloud-delivered scripts, or another external deployment process. Existing static evidence does not support the claim that the APK loads this DEX at startup.

## 24. Diagnostics and Static SOME/IP/DoIP Probing

The diagnostic code contains static probing parameters for the in-vehicle network:

- `198.18.44.15:30511` UDP;
- SOME/IP SD port 30490;
- DoIP TCP 13400;
- service `0xC0A8`;
- instance `1`;
- routine `0x091A`.

The code text explicitly states that no active payload is sent. **Code confirmed: this is diagnostic/forensic probing logic and should not be described as a production vehicle-control protocol.**

## 25. USB, Serial Ports, and Other Hardware Interfaces

- USB is primarily used as a source of system broadcasts, ADB/device connection state, and automation triggers;
- No definite USB bulk business-data transfer implementation was found;
- No serial-port/UART business implementation was found;
- No direct SocketCAN send/receive chain was found;
- BLE advertising and scanning implementations are clearly present;
- The AVM raw camera is implemented through a Native library.

## 26. Content Store, Updates, and Resource Editors

### 26.1 Content Store

Content categories supported by `ContentStoreActivity`:

- Dashboard templates;
- Widgets;
- Skins;
- Automations;
- Scripts;
- Configuration packages;
- Share codes;
- Uploads and downloads.

Bridge transaction `openContentStore(type,id,code)` allows mini or another internal module to open specified content directly.

### 26.2 Updates

Stable/beta channels, changelogs, APK downloads, and coordination of mini updates are supported. mini delegates `com.kooo.evcc.action.CHECK_MINI_UPDATE` to `MiniUpdateRequestReceiver`.

### 26.3 Web Editors

- Dynamic wallpaper editor: `https://evcc.coauto.cc/static/dynamic-wallpaper-editor/index.html`
- Skin editor: `https://evcc.coauto.cc/static/skin-editor/index.html`

## 27. Local Configuration and Data

Main SharedPreferences:

| Configuration name | Module |
|---|---|
| `evcc_dashboard` | Dashboard pages/widgets |
| `evcc_automation_rules` | Automation rules |
| `evcc_automation_variables` | Automation variables |
| `evcc_auto_connect` | Automatic Root/ADB/service connection |
| `evcc_cloud_config` | Cloud-configuration cache |
| `evcc_cast` | Overlay projection |
| `evcc_display_mirror` | Cross-display mirroring |
| `dashcast_master` | General Dashcast configuration |
| `dashcast_prefs` | Dashcast parameters |
| `dashcast_autostart` | Dashcast autostart |
| `evcc_floating_window` | Floating window |
| `mini_window_layouts` | Mini-window layouts |
| `evcc_ble_beacon` | Beacon/BLE |
| `evcc_mini_program_mqtt` | Mini-program MQTT |
| `evcc_remote_bot` | Home Assistant/bots |
| `evcc_midebao_linkage` | Midebao linkage |
| `engine_sound_prefs` | Simulated engine sound |
| `evcc_frida` | Frida |
| `evcc_frpc` | frpc |
| `evcc_adb_config` | ADB terminal |
| `evcc_license` | Local license state |
| `evcc_security_password` | Application feature-password configuration |
| `remote_connections` | Remote file connections |
| `pan123_auth` | 123 Cloud Drive authentication state |
| `evcc_update` | Update channel/state |
| `evcc_theme` | Theme |
| `evcc_skin` | Skin |

Primary business data is stored as SharedPreferences JSON, local configuration files, scripts, sound packs, cached artwork, and downloaded files. **Cannot be confirmed:** no Room database was found as core business storage in this analysis. RootService's SQLite capability queries or executes against target databases and does not mean that EVCC itself uses Room.

## 28. Associated Software and External Systems

| Software/system | Relationship |
|---|---|
| EVCC mini `com.kooo.evcc.mini` | Binder + ContentProvider; lightweight mini-window frontend |
| `com.galaxy.lyricsnext` | Midebao AIDL caller/lyrics linkage |
| Android Automotive / `android.car` | CarProperty, Car Power, audio, displays |
| ECARX/Flyme Auto | System broadcasts, status-bar plugin, media center, HIDL, window modes |
| Home Assistant | MQTT auto-discovery and entity control |
| NanoMQ | Local port 1883 in-vehicle cloud message bus |
| DingTalk | OAuth + bot messages |
| Feishu | OAuth, IM HTTP, WebSocket |
| Telegram | Bot API |
| 123 Cloud Drive | Remote file management |
| WebDAV/FTP | Remote file management |
| FNOS NAS | Shared-file parsing/download |
| Frida | Process injection and in-vehicle scripts |
| frpc | Intranet-tunneling deployment |
| sunrise-sunset.org | Sunrise/sunset calculation for automation |
| GitHub/mirror sites | Downloading tools such as frpc |
| Midebao hardware | BLE advertising frames and AIDL linkage |
| Network secondary-display receiver | RTP H.264/JPEG fragmentation |
| Dashcast CP/cluster endpoint | H.264 RTP or BGRA+LZ4 |

## 29. Key Business Workflows

### 29.1 Vehicle-Property Display and Control

```text
VHAL/APVP/AIDL/CAPI
 -> unified property layer (propertyId, areaId, value)
 -> dashboard/mini/automation/engine-sound/remote modules subscribe
 -> mapping and expressions
 -> UI display

User action
 -> dashboard action model
 -> ActionRouter/automation action layer
 -> vehicle-property writer
 -> corresponding vehicle interface
```

### 29.2 Automation

```text
Boot/property/App/BLE/time/location/media event
 -> AutomationService
 -> rule matching and condition evaluation
 -> action sequence
 -> property/RootService/App/mini-window/projection/script/remote message/BLE/media/TTS/sound/volume/Shell
```

### 29.3 mini

```text
EvccConfigProvider -> mini pages and theme
EvccBridgeService -> properties/system metrics/music
mini user action -> DashboardAction -> EVCC execution layer
```

### 29.4 RootService

```text
EVCC Client
 -> start root app_process
 -> LocalServerSocket/TCP loopback
 -> challenge/HMAC handshake
 -> length + JSON command
 -> SystemApiHandler
 -> JSON response
```

### 29.5 Dashcast

```text
Virtual display/MediaProjection/rendered image
 -> H.264 RTP over UDP
    or BGRA frame -> LZ4 -> TCP
 -> cluster/CP receiver
```

### 29.6 Remote Vehicle State

```text
Properties/variables/GPS
 -> mini-program MQTT or HA MQTT
 -> Broker/HA
 -> state topic / discovery entity
 -> control message returns to the widget action layer
```

## 30. Static-Analysis Boundaries

1. **Cannot be confirmed:** the server URL, gRPC host/port/clientId, license-request fields, and plaintext of some configuration resources encapsulated by `libevcc_native.so`.
2. **Cannot be confirmed:** final service cards, feature quotas, and vehicle-model switches determined by cloud configuration and account state.
3. **Cannot be confirmed:** whether `iplm_ctrl.dex` is invoked indirectly by a cloud-delivered script or an external deployment process; the main DEX has no direct reference.
4. **Code confirmed:** DBC is present, but no direct SocketCAN business implementation was confirmed.
5. **Code confirmed:** no USB bulk or serial-port/UART business implementation was confirmed.
6. **Cannot be confirmed:** actual compatibility of private system services and reflection APIs across different ECARX/Flyme/Geely/LynkCo firmware versions.

## 31. Final Conclusion

EVCC v2.9.46 is an Android Automotive platform application that combines vehicle-property access, a visual dashboard, automation orchestration, system-level execution, a media bridge, projection, mini-windows, remote protocols, scripts, and file/application management.

Its central architectural characteristic is a unified property model plus a shared action-execution layer: vehicle sources such as VHAL, APVP, AIDL, and CAPI are normalized into propertyId/areaId/value, while the dashboard, mini, automation, engine sound, MQTT, and Midebao all work around the same data model. Operations requiring system capabilities are centralized in RootService, ADB, Android Car, and private automaker interfaces.

The protocol layer exposes multiple independent paths, including Binder v11, Provider, RootService length-prefixed JSON, VHAL/APVP gRPC, three MQTT groups, Feishu Protobuf WebSocket, BLE/UDP advertising, Midebao binary frames, and Dashcast RTP/BGRA+LZ4. `assets/iplm_ctrl.dex` is confirmed as an independent app_process IPLM CLI tool; there is no evidence that the current APK's main code directly loads it.

## 32. JADX Gap-Recovery Review

The default structured output contained 1,594 `Method not decompiled` placeholders across 847 Java files, including both EVCC business code and dependencies such as Fastjson2, Netty, BouncyCastle, LuaJ, and the DingTalk SDK. Full simple mode reduced the placeholder count to 22; full fallback output produced 9,350 Java files with zero method placeholders and exited with code 0.

The final simple-mode report contained 25 error nodes: 13 in EVCC business code and 12 in third-party libraries. Three error nodes still generated low-level code, while fallback and smali cover the other 22. Remaining business items include default parameters for file-management connections, the RootService Handler, PhoneBlocker, media-source normalization, system diagnostics, automation-action editing and execution, H.264 RTP, the RootService client, the content store, and skin parsing.

Conclusions added or tightened by this recovery pass:

- `PhoneBlockerActivity.onCreate()` now immediately marks dismissed, sets a successful result, and finishes after `super.onCreate()`; the subsequent prompt and mini-game UI are unreachable.
- `RootServiceMain.startResilientHandler()` uses a dedicated Looper thread with a startup wait limit of 5 seconds. If a Looper callback throws, it records the first 10 frames and continues looping.
- H.264 RTP supports both Annex-B and AVCC NAL input. A single NAL is limited to 1,188 bytes; oversized NAL units use FU-A with FU data chunks of at most 1,186 bytes. Timestamps are converted to 90 kHz by multiplying milliseconds by 90.
- The complete register-level state machine of the core automation executor `rk.llI1(...)` is preserved, covering property, App, mini-window/projection, script, remote-message, BLE, media, TTS, sound, volume, Shell, and looped property actions.
- Home Assistant Discovery and the JPEG fragment header were fully recovered; exact fields are documented in Sections 19.3 and 14.2.

Fallback is an auditable register-level static view, not a reconstruction of the pre-obfuscation Kotlin/Java source project. Original variable names, comments, and some high-level coroutine structures remain irrecoverable.
