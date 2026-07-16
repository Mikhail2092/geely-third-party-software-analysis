# EVCC mini v2.3.8 Static Analysis Report

## 1. Report Scope

This report presents a purely static analysis of `com.kooo.evcc.mini` (EVCC mini v2.3.8). Its purpose is to recover the software's role, every confirmable function, key business flows, interprocess interface protocols, configuration formats, and relationship with the full EVCC application.

The APK was not installed, launched, or dynamically debugged during analysis, and no vehicle or backend service was connected. This report does not discuss security, privacy, or compliance risks.

The following labels are used consistently for evidence levels:

| Label | Meaning |
|---|---|
| **Code-confirmed** | Directly proven by decompiled Java/Smali, the Manifest, or definitive constants |
| **Resource-confirmed** | Directly proven by layouts, strings, JSON, icons, or configuration resources |
| **Reasoned inference** | Multiple pieces of static evidence agree, but executable verification is unavailable |
| **Unconfirmed** | Static evidence is insufficient because of obfuscation, Native encapsulation, version differences, or the restriction against dynamic analysis |

## 2. Sample Identity

| Item | Result | Evidence level |
|---|---|---|
| Software name | EVCC mini | Resource-confirmed |
| Original filename | `EVCC_mini_v2.3.8_разблокированный_подписанный.apk` | Code-confirmed (file system) |
| Package name | `com.kooo.evcc.mini` | Code-confirmed |
| Version | `2.3.8` | Code-confirmed |
| versionCode | `138` | Code-confirmed |
| APK size | 7,657,464 bytes | Code-confirmed |
| SHA-256 | `EAD8D7019C5D92E52D6BE16D250D44FF27DC993AFB72E917DE6542F35A957D4A` | Code-confirmed |
| minSdk | 28 | Code-confirmed |
| targetSdk | 30 | Code-confirmed |
| compileSdk | 36 | Code-confirmed |
| Current signing certificate SHA-256 | `BB:CA:73:98:98:13:6E:6F:84:77:AF:0D:32:53:AE:10:B5:B6:DE:C2:F6:3D:D4:6D:34:FB:5E:AF:83:6C:16:3F` | Code-confirmed |
| Application | `com.kooo.evcc.mini.MiniApp` | Code-confirmed |
| Main Activity | `com.kooo.evcc.mini.MiniDashboardActivity` | Code-confirmed |
| Vehicle platform declaration | Android Automotive optional, `required=false` | Code-confirmed |
| Native libraries | No `.so` included | Code-confirmed |

The Manifest declares only two business-related permissions:

- `com.kooo.evcc.permission.BIND_EVCC`
- `android.permission.INTERNET`

It also uses `<queries>` to query the full application package `com.kooo.evcc`. The application itself does not declare permissions for vehicle properties, Bluetooth, location, media notification listening, overlay windows, or file management.

## 3. Software Role and Overall Conclusion

**Code-confirmed: EVCC mini is a lightweight mini-window/dashboard frontend for the full EVCC application, not a standalone vehicle-control program.**

It is responsible for:

- Displaying dashboard pages and the music page supplied by the full application;
- Building a component grid from page JSON;
- Aggregating the vehicle properties and system metrics needed by pages;
- Subscribing to real-time data through Binder;
- Packaging button, toggle, slider, dropdown, and button-group operations for execution by the full application;
- Receiving media state aggregated by the full application and sending media-control commands;
- Cooperating with the head-unit system and full application to enter/leave mini-window mode;
- Reading theme, skin, scale, and immersive settings;
- Displaying demo pages when no configuration is available.

It is not responsible for:

- Directly connecting to Android VHAL, APVP, ECARX AIDL/CAPI;
- Directly running the automation engine;
- Directly establishing MQTT, gRPC, WebSocket, BLE, UDP, TCP, USB, or serial business connections;
- Directly executing RootService, Shell, ADB, or Frida;
- Directly maintaining the media-session bridge or vehicle-property writer.

The overall dependency relationship is:

```text
EVCC mini / MiniDashboardActivity
    |-- ContentResolver.query()
    |      `-- com.kooo.evcc.config / EvccConfigProvider
    |
    `-- bindService(IEvccBridge)
           `-- com.kooo.evcc.bridge.EvccBridgeService
                    |-- VHAL / APVP / AIDL / CAPI property sources
                    |-- System-metric collector
                    |-- Media-state bridge
                    |-- ActionRouter / automation execution layer
                    `-- Mini-window and display control layer
```

**Reasoned inference: mini is designed to maintain low resource usage inside a restricted head-unit mini-window while reusing the vehicle and system connections already established by the full application.**

## 4. Android Components and Startup Entry Point

### 4.1 Application

`MiniApp` initializes the global `BridgeClient` when the process is created. Business connectivity is centralized in a singleton client; the Activity only registers state listeners and callbacks.

### 4.2 MiniDashboardActivity

Manifest configuration:

| Attribute | Value |
|---|---|
| exported | `true` |
| launchMode | `singleTask` |
| clearTaskOnLaunch | `true` |
| taskAffinity | `com.kooo.evcc.mini` |
| configChanges | density, fontScale, layoutDirection, orientation, screenLayout, screenSize, smallestScreenSize, uiMode |

Head-unit window metadata:

| Metadata | Value |
|---|---|
| `com.android.app.start_windowmode` | `only_windowmode` |
| `com.android.app.windowmode.dpi` | `240` |
| `com.android.app.windowmode.resolution.width` | `800` |

**Code-confirmed: the main interface is built entirely in code. Its core structure consists of a scale container, ViewPager2, a page adapter, a status panel, and pagination state.**

### 4.3 Other Components

Except for the basic AndroidX Startup/Profile Installer components, there are no other business Activities, Services, Providers, or Receivers. In other words, mini has no resident background business service; its core lifecycle is jointly determined by the only Activity and its connection to the full application's service.

## 5. Startup and Page-Loading Flow

The main flow recovered from code is:

1. `MiniApp` creates the global `BridgeClient`.
2. `MiniDashboardActivity` builds the interface and registers a Bridge connection listener.
3. Query the full application's Provider for `main_info` to confirm the main application's state and bridge version.
4. Query `mini_pages`, `mini_grid`, `mini_settings`, and `current_skin`.
5. Calculate the page signature: page collection + grid columns/rows + DPI scale.
6. Rebuild `MiniPageAdapter` when the page signature changes.
7. Aggregate all vehicle-property subscription keys and system-metric types.
8. When Bridge is connected, submit subscriptions and register the music callback.
9. On receipt, callbacks first enter a pending map, after which the UI is refreshed in batches.
10. Start local timed refresh when a page contains a clock component.
11. When the Activity resumes, attempt to enter head-unit mini-window mode.
12. When the Activity is destroyed, cancel property, system-metric, and music subscriptions, and remove all Handler callbacks.

**Code-confirmed: when the Provider returns an empty page list, the Activity automatically enters demo mode instead of remaining blank.**

## 6. Configuration ContentProvider Protocol

### 6.1 Basic Information

| Item | Value |
|---|---|
| authority | `com.kooo.evcc.config` |
| Provider implementation | Full application `EvccConfigProvider` |
| Read permission | `com.kooo.evcc.permission.BIND_EVCC` |
| mini operation | Executes only `query()` |
| insert/update/delete | The full implementation does not process them and returns null or 0 |

### 6.2 URIs and Fields

| URI | Returned fields | Purpose |
|---|---|---|
| `content://com.kooo.evcc.config/main_info` | `version_code`, `version_name`, `bridge_version` | Full application version and bridge protocol number |
| `content://com.kooo.evcc.config/mini_grid` | `cols`, `rows`, `dpi_scale` | Grid and page scale |
| `content://com.kooo.evcc.config/mini_pages` | `page_id`, `page_title`, `page_type`, `page_json` | Page collection |
| `content://com.kooo.evcc.config/mini_settings` | `immersive`, `theme_mode`, `music_page_style` | Interface display options |
| `content://com.kooo.evcc.config/current_skin` | `skin_json` | Current skin JSON |

### 6.3 Actual Provider Return Logic

**Code-confirmed:**

- `main_info` reads the version from the full application's PackageInfo and always returns `bridge_version=11`.
- `mini_grid` returns the column count from full-application configuration, fixed `rows=10`, and the calculated DPI scale.
- The mini client's own fallback grid is `3 x 6` with `dpiScale=1.0`.
- The mini client limits columns to `1..12`, rows to `1..20`, and scale to `0.3..2.0`.
- `mini_settings` is currently returned by the Provider with fixed `immersive=1`; the theme comes from `evcc_theme/theme_mode`.
- Music-page style can be `HERO` or `CLASSIC`; the Provider default is `HERO`.
- `current_skin` is serialized to JSON by the full application's skin manager.
- If the full application has no separately saved mini pages, the Provider serializes the current dashboard components as one default `DASHBOARD` page.

### 6.4 Page Structure

mini 2.3.8 explicitly recognizes two page types:

| Type | Rendering method |
|---|---|
| `DASHBOARD` | Parse `page_json.widgets[]` and construct grid components |
| `MUSIC` | Construct a dedicated `MusicPageView` |

An unknown page type falls back to `DASHBOARD` through `MiniPageType.fromName()`.

The newer configuration code of the full application also contains an `APP_MENU` page type, but the mini 2.3.8 enum does not include it. **Code-confirmed: this mini version does not render `APP_MENU` as a separate application-menu page; statically, it falls back to DASHBOARD processing.**

### 6.5 DASHBOARD JSON

Top-level structure:

```json
{
  "widgets": [
    { "...": "Serialized fields of a full-application dashboard component" }
  ]
}
```

Components are deserialized through the data model shared with the full application and include:

- UUID/component ID;
- Component type;
- Title, unit, icon, and display text;
- Grid width/height, column, and row;
- `propertyId`, `areaId`;
- Mapping rules and expressions;
- Toggle on/off write values;
- Slider minimum, maximum, and step;
- Dropdown and button-group options;
- System-metric type;
- Chart range, colors, and time window;
- Action configuration and variable configuration.

Because model classes are obfuscated by R8, some JSON field names can only be recovered through the shared serializer. This report cannot guarantee that the conceptual names listed above equal all original key names. This is **Code-confirmed for semantics, but Unconfirmed for the original business names of all obfuscated fields**.

## 7. Binder Bridge Protocol

### 7.1 Binding Method

| Item | Value |
|---|---|
| action/descriptor | `com.kooo.evcc.bridge.IEvccBridge` |
| Explicit component | `com.kooo.evcc/com.kooo.evcc.bridge.EvccBridgeService` |
| Binding permission | `com.kooo.evcc.permission.BIND_EVCC` |
| Bridge version constant bundled in mini | `CURRENT_BRIDGE_VERSION=5` |
| Minimum mini requirement | `REQUIRED_BRIDGE_VERSION=1` |
| Version reported by the full Provider in this analysis | `11` |

### 7.2 mini v5 Interface Transaction Table

| Transaction | Method | Parameters/return value | Purpose |
|---:|---|---|---|
| 1 | `getBridgeVersion()` | `int` | Query service bridge version |
| 2 | `subscribeValues()` | `String[]`, `IValueCallback` | Subscribe to vehicle properties |
| 3 | `unsubscribeValues()` | callback | Unsubscribe from vehicle properties |
| 4 | `executeAction()` | `DashboardAction` | Execute a component action |
| 5 | `registerMusicCallback()` | `IMusicCallback` | Register music state |
| 6 | `unregisterMusicCallback()` | callback | Unregister music state |
| 7 | `sendMusicCommand()` | `int command`, `long value` | Media control |
| 8 | `setVariable()` | `String name`, `String value` | Write an automation variable |
| 9 | `subscribeSystemMetrics()` | `String[]`, callback | Subscribe to system metrics |
| 10 | `unsubscribeSystemMetrics()` | callback | Unsubscribe from system metrics |
| 11 | `enterMiniWindowMode()` | `int taskId` | Request entry into mini-window mode |
| 12 | `getMiniEntitlementState()` | `String` | Query mini state |
| 13 | `launchPackageFullscreen()` | `String packageName` | Launch a package fullscreen |

The interface definition must be distinguished from the actual wrapper in this sample: transaction 12 remains in the AIDL/Stub, but the body of `BridgeClient.getMiniEntitlementState()` directly returns the fixed string `"active"` without accessing remote Binder. The Activity uses this local wrapper after connection and during state refresh every 30 seconds. Therefore, **Code-confirmed: the current business path in this sample does not obtain state through transaction 12 and always treats mini state as active.** This report records only the functional implementation and makes no other assessment of this behavior.

### 7.3 Transaction Differences from Full Application v11

Full application v11 inserted `getVariablesJson(String[])` at transaction 9, shifting all subsequent transaction numbers, and added further interfaces for the content store, toggle states, page saving, virtual displays, input injection, and other functions.

Therefore:

- Transactions 1..8 are identical in the two interface copies;
- mini v5 transactions 9..13 do not have the same method semantics as the same-numbered methods in full application v11;
- Static code in full application `EvccBridgeService.onBind()` directly returns the v11 Stub;
- No separate v5 compatibility Stub or implementation that selects an interface version by caller was found.

**Unconfirmed: the two APKs' static interface copies alone cannot prove that calls after transaction 9 from a v5 client will interoperate correctly with a v11 service.** This records a static fact about version compatibility, not a runtime result. The main property, action, music, and variable chain in transactions 1..8 retains matching numbers.

### 7.4 Fixed `active` Branch and the Full Application's Original State Chain

Client code location:

```text
.analysis-work/evcc_mini/jadx/sources/com/kooo/evcc/mini/BridgeClient.java:388
getMiniEntitlementState() -> return "active"
```

The mini UI still retains all display branches in `applyEntitlementState()`. It recognizes `none`, `active`, `stale`, `trial:<...>`, and `expired`, and retains the jump to the main application for activation/renewal. In other words, the interface layer's original state model has not been removed, but this sample's `BridgeClient` wrapper does not pass it remote state and instead always passes `active`.

By comparison, the full application's server implementation still retains the original state chain:

```text
.analysis-work/evcc/jadx/sources/com/kooo/evcc/bridge/EvccBridgeService$binder$1.java:252
```

The server can return `none`, `active`, `stale`, `trial:<remaining-information>`, or `expired`, and calls its own `isMiniEntitled()` at the entry points for variable reads, property subscriptions, music callbacks, system-metric subscriptions, and toggle-state subscriptions.

The static boundary is therefore:

- **Code-confirmed:** the state-query wrapper in this mini client build always returns `active`;
- **Code-confirmed:** the full application's original state storage, state response, subscription-entry checks, and code that clears subscriptions after state revocation still exist;
- **Code-confirmed:** the fixed return occurs only in the mini client's local state-display path and does not establish that the full application's server-side state check has been removed;
- **Unconfirmed:** the origin and creation method of this fixed branch, and the final state when the two APKs actually connect, because dynamic execution is prohibited in this report.

## 8. Binder Data Models

### 8.1 Property Subscription Key

Subscription-key format:

```text
<propertyId>:<areaId>
```

A component's own nonzero propertyId is added to the collection; property references found in expressions are also added. Expression patterns are:

```text
$prop(<propertyId>)
$prop(<propertyId>, <areaId>)
```

An omitted areaId is treated as `0`. The collection deduplicates entries while preserving insertion order.

### 8.2 ValueUpdate

| Field | Type | Meaning |
|---|---|---|
| `propertyId` | int | Vehicle property ID |
| `areaId` | int | Area ID |
| `rawValue` | String | Raw value in unified string form |

mini stores the latest value as `<propertyId>_<areaId> -> rawValue` for component refresh and `$prop()` expression substitution.

### 8.3 SystemMetricUpdate

| Field | Type | Meaning |
|---|---|---|
| `type` | String | `SYSTEM_*` metric type |
| `value` | double | Aggregated value |
| `breakdown` | `Map<String, Double>` | Itemized data |

The adapter generates a system-metric subscription only when a component's system type starts with `SYSTEM_`.

### 8.4 MusicState

| Field | Meaning |
|---|---|
| `isPlaying` | Whether media is playing |
| `title` | Title |
| `artist` | Artist |
| `album` | Album |
| `durationMs` | Total duration |
| `positionMs` | Playback position |
| `artworkUri` | Artwork URI |
| `packageName` | Current media package name |
| `lyric` | Lyrics |

### 8.5 DashboardAction

| Field | Meaning |
|---|---|
| `actionKind` | Action type number |
| `widgetId` | Component ID |
| `widgetJson` | Complete component configuration JSON |
| `payload` | Current interaction value |

Action types:

| Number | Type | payload |
|---:|---|---|
| 0 | ACTION | Empty string |
| 1 | TOGGLE | `"true"` / `"false"` |
| 2 | SLIDER | Numeric string formatted with no decimal places |
| 3 | DROPDOWN | Selected item value |
| 4 | BUTTON_GROUP | Selected button value |

## 9. Vehicle-Property Display Flow

```text
Provider page_json
  -> MiniPageAdapter.parseWidgets()
  -> allSubscriptionKeys()
  -> IEvccBridge.subscribeValues()
  -> Full application aggregates VHAL/APVP/AIDL/CAPI
  -> IValueCallback(ValueUpdate)
  -> pending value map
  -> batch flush
  -> grid.dispatchValue()
  -> numeric mapping/expression/text/chart refresh
```

**Code-confirmed: mini sees only the propertyId, areaId, and rawValue unified by the full application; it is unaware of which underlying head-unit interface supplied each property.**

Callbacks merge updates in a pending map, after which the main thread performs one consolidated refresh. This avoids a separate redraw for every high-frequency vehicle-signal callback.

## 10. Component Operation Flow

```text
User click/drag/selection
  -> MiniPageAdapter identifies ActionKind
  -> Package DashboardAction(widgetId, widgetJson, payload)
  -> IEvccBridge.executeAction()
  -> Full application deserializes the shared component model
  -> ActionRouter
       |-- Directly write a vehicle property
       |-- Set a variable
       `-- Delegate to the action/automation execution engine
```

Confirmed basic routing in the full application's `ActionRouter`:

- ACTION: execute the action bound to the component;
- TOGGLE: select the on/off value or action according to the Boolean payload;
- SLIDER: parse a floating-point number and write it according to component configuration;
- DROPDOWN/BUTTON_GROUP: execute an action or write a property according to the selected option value.

mini does not include the underlying implementation of these actions. Therefore, even with an identical UI, actual capabilities depend jointly on the full application version, vehicle-property source, Root/ADB state, and component configuration.

## 11. Music Page and Media Control

Normal mode uses `BridgeBackedMusicSource` with data from `IMusicCallback`; demo mode uses `DemoMusicSource`.

Media commands:

| command | Purpose | long parameter |
|---:|---|---|
| 0 | Play/pause | Unused |
| 1 | Next track | Unused |
| 2 | Previous track | Unused |
| 3 | Seek | Target milliseconds |

The music page is responsible for:

- Displaying artwork, title, artist, album, and lyrics;
- Displaying play/pause state;
- Displaying duration and playback progress;
- Sending previous, next, play/pause, and Seek commands;
- Running the corresponding page-refresh logic only while the current music page is active;
- Selecting HERO or CLASSIC style according to the Provider setting.

The full application caches the artwork URI and exposes it through the `com.kooo.evcc.artwork` FileProvider; mini does not fetch network artwork itself.

## 12. Mini-Window and Fullscreen Switching

mini attempts two paths for entering a mini-window:

1. `FlymeMiniWindowHelper` requests a mini-window through head-unit window reflection/APIs;
2. It calls the full application's `enterMiniWindowMode(taskId)`, allowing the full application to help adjust the task window.

Confirmed behavior:

- Recognizes head-unit mini-window windowingMode `11`;
- Uses the task ID to request mini-window handling by the full application;
- Constructs ActivityOptions with windowingMode `1` for fullscreen launch;
- After detecting that it has been maximized from a mini-window, switches back to the full application and finishes the mini Activity;
- Uses different in-window padding and title-area masks for music and ordinary dashboard pages;
- Supports immersive mode, day/night themes, skins, and overall scaling.

**Reasoned inference: the two mini-window paths provide compatibility with Flyme Auto private capabilities and the full application's system-level task-control capabilities.**

## 13. Connection State and Recovery

`BridgeClient` maintains bind state, the service reference, and listeners.

Reconnection backoff sequence after disconnection:

```text
500 ms -> 1 s -> 2 s -> 4 s -> subsequent attempts 10 s
```

After a successful connection:

- Obtain `getBridgeVersion()`;
- Notify listeners that the connection has recovered;
- The Activity reestablishes property, system-metric, and music subscriptions;
- Refresh the main-application version/state panel;
- Reapply mini state every 30 seconds, although the client method in this sample always returns `active`.

When connection fails or the full application is unavailable, resources contain the following state semantics:

- Main application not installed;
- Main application version too old;
- Connecting;
- Connection failed;
- Check for updates;
- mini state unavailable.

## 14. Update Check

No independent HTTP download implementation was recovered in mini. Update checks are delegated to the full application through a broadcast:

```text
action: com.kooo.evcc.action.CHECK_MINI_UPDATE
receiver: com.kooo.evcc.update.MiniUpdateRequestReceiver
permission: com.kooo.evcc.permission.BIND_EVCC
```

**Code-confirmed: mini has the INTERNET permission, but current static business evidence does not show it directly accessing the EVCC update server; the full application handles the update chain.**

## 15. Demo Mode

Entry conditions:

- Provider pages are empty; or
- The startup Intent specifies demo-related parameters.

ADB/Intent extras:

| Parameter | Meaning |
|---|---|
| `demo=true` | Request demo mode |
| `demo_variant` | Select a simulated music-state variant |
| `demo_pages=widget` | Show only the component demo page |
| `demo_pages=music` | Show only the music demo page; default value |
| `demo_pages=both` | Show both component and music pages |

The component demo uses a `3 x 6` grid and constructs three music-type components starting at rows 0, 2, and 4 respectively. The music demo generates playback state from a local timed source and does not depend on Binder media callbacks.

## 16. Local State, Network, and Native Capabilities

### 16.1 Local State

The full application stores most long-term business configuration, which mini reads through the Provider. mini itself retains only a small amount of theme/process state, with no evidence of an independent business database.

### 16.2 Network

Static search results:

- No OkHttp/Retrofit business interface call chain was found;
- No MQTT client instance was found;
- No WebSocket, gRPC, UDP, or TCP Socket business implementation was found;
- No BLE scanning or advertising implementation was found;
- No USB bulk, serial/UART, or SocketCAN implementation was found.

Therefore, **Code-confirmed: mini's main interprocess communication protocols are Android Binder and ContentProvider.**

### 16.3 Native

The APK contains no Native `.so`, and no business call to `System.loadLibrary()` was recovered. Therefore, vehicle data, system control, and network protocols are not implemented inside mini through JNI.

## 17. Functional Relationship with the Full EVCC Application

| mini function | Full-application provider |
|---|---|
| Page and component configuration | `EvccConfigProvider`, dashboard configuration manager |
| Theme/skin/scale | Theme and skin managers, mini configuration manager |
| Vehicle properties | `EvccBridgeService` + VHAL/APVP/AIDL/CAPI |
| System metrics | Full-application system-metric collector |
| Component actions | `ActionRouter` + automation/system-control layer |
| Automation variables | Full-application variable repository |
| Music state | Full-application media-session bridge |
| Artwork files | Full-application artwork FileProvider |
| Media commands | Full-application media-center control layer |
| Mini-window task control | Full-application task/display control layer |
| Update check | `MiniUpdateRequestReceiver` |
| Open application fullscreen | Full-application package/task launch layer |

The full application is a hard dependency. Without it, mini can only show a status page or enter a local demo; it cannot provide real vehicle display and control.

## 18. Summary of Key Business Flows

### 18.1 Cold Start

```text
Launcher
 -> MiniApp
 -> MiniDashboardActivity
 -> Query com.kooo.evcc.config
 -> Bind EvccBridgeService
 -> Build pages
 -> Subscribe to properties/system metrics/music
 -> Request mini-window
```

### 18.2 Real-Time Property Refresh

```text
Vehicle property source -> Unified collection by full application -> Binder callback
 -> mini pending map -> Main-thread batch refresh -> Component mapping/expression/chart
```

### 18.3 User Control

```text
Component interaction -> DashboardAction -> Binder
 -> Full-application ActionRouter -> Property write or action engine -> Vehicle/system/App
```

### 18.4 Music Control

```text
Full-application media session -> MusicState -> mini music page
mini controls -> command 0/1/2/3 -> Full-application media-control layer
```

### 18.5 Page Configuration Update

```text
Full application saves pages/theme/skin
 -> Provider returns new configuration
 -> mini detects signature change
 -> Rebuild pages and subscription collection
```

## 19. Unconfirmed Items in Static Analysis

1. **Unconfirmed:** With APK execution prohibited, actual support for windowingMode 11 and Flyme mini-window reflection interfaces in the specified head-unit firmware cannot be verified.
2. **Unconfirmed:** The mini v5 interface copy and full application v11 differ in transaction numbering after transaction 9. No compatibility Stub was found, so static analysis cannot prove the runtime outcome of these extended calls.
3. **Unconfirmed:** Some component data classes are obfuscated by R8. Although their semantics and serialization chain can be recovered, the original business names of all JSON keys cannot be guaranteed.
4. **Unconfirmed:** Native/cloud configuration is not in mini, but the full application may change the final visible functions according to licensing, vehicle model, or cloud switches.
5. **Code-confirmed:** No implementation was found in this sample for direct mini connections to VHAL, MQTT, BLE, UDP, USB, serial, or an external HTTP business service.

## 20. Final Conclusion

EVCC mini v2.3.8 is a rendering and interaction client for an in-vehicle mini-window. Its core purpose is not to implement vehicle protocols, but to compress the full application's dashboard page model into a mini-window and reuse the full application's vehicle properties, system metrics, action execution, music session, artwork cache, and task-window control through permission-protected Provider/Binder contracts.

Functionally, mini's business loop is "read configuration -> build pages -> aggregate subscriptions -> receive real-time values -> render components -> return interactions to the full application." It has no independent vehicle access layer, automation engine, network service layer, or Native driver layer; the full application `com.kooo.evcc` is the sole core provider of its real business capabilities.

## 21. Review of Recovered JADX Gaps

The default structured output contained 149 `Method not decompiled` placeholders across 98 Java files. Full simple mode reduced the count to 2; full fallback output produced 1,226 Java files with zero method placeholders and exited with code 0.

All 12 methods that failed by default in the mini business package, including the main Activity, Bridge data models, music page, and update check, were recovered in simple mode. The two remaining simple failures were the mini skin JSON parser `a.a.n(JSONObject)` and an AndroidX Fragment helper method; both were covered by fallback and smali.

The skin parser's currently confirmable top-level structure is `colors`, `widgetOverrides`, `chart`, `shapes`, and `pageBackground`. It includes card background, border, corner radius, font scale, chart colors, gauge ticks, page gradient/image, top-bar light/dark mode, blur, and other fields. The default page background is `solid`, angle 135, scope `console`, with image fit `cover`. These are JSON keys confirmable in the current DEX, but they do not constitute recovery of the data-class field names before R8 obfuscation.

This recovery did not change mini's core role: it remains a mini-window client dependent on the full application's Provider/Binder, with no newly identified independent vehicle protocol, automation executor, or network service.
