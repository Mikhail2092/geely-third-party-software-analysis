# AutoDisplay v3.6.2 (versionCode 50) Static Analysis Report

## 1. Report Scope

This report is based solely on static analysis of the APK, decompiled Java/Smali, Manifest, resources, Native libraries, and signature containers. The APK was neither installed nor launched; no vehicle, QNX system, map, music application, or backend service was connected, and no network request was made. The report focuses on functional implementation, interface protocols, and relationships with other software in the project. It does not assess security, privacy, or compliance risks.

Evidence levels:

- **Code confirmed**: directly confirmed by the Manifest, Java/Smali, the resource table, or Native JNI declarations.
- **Resource confirmed**: directly confirmed by strings, assets, sample configuration, file paths, or package metadata, although the runtime branch was not executed.
- **Reasonable inference**: multiple code clues support a relatively reliable explanation, subject to obfuscation, decompilation failures, or Native black-box limitations.
- **Cannot be confirmed**: the static material is insufficient; confirmation requires the runtime environment, backend, vehicle, or original in-vehicle application.

Primary evidence roots:

- Manifest: `.analysis-work/autodisplay/apktool/AndroidManifest.xml`
- Business source: `.analysis-work/autodisplay/jadx/sources/com/simplerbit/autodisplay/`
- Obfuscated business source: `.analysis-work/autodisplay/jadx/sources/defpackage/`
- Smali: `.analysis-work/autodisplay/apktool/smali/`
- Resources and assets: `.analysis-work/autodisplay/apktool/res/`, `.analysis-work/autodisplay/apktool/assets/`

## 2. Exact APK Identity

| Item | Static result | Evidence level |
|---|---|---|
| Original filename | `AutoDisplay_v3.6.2_50_макет_интерфейса_подписанный.apk` | Code confirmed (filesystem) |
| File size | 102,641,428 bytes | Code confirmed |
| MD5 | `F8EACB1B60468FE934DA2660B3FBC6EA` | Code confirmed |
| SHA-1 | `B7244789825153905F030C450D6BB00B74E2E978` | Code confirmed |
| SHA-256 | `39C676B4F1943D2871C4558F4E8934901BADF56EC02B72A57D5FAFB5A000BC6E` | Code confirmed |
| Application name | `AutoDisplay` | Resource confirmed: `@string/app_name` |
| Package name | `com.simplerbit.autodisplay` | Code confirmed: Manifest |
| versionName / versionCode | `3.6.2` / `50` | Code confirmed: `apktool.yml` |
| minSdk / targetSdk / compileSdk | 28 / 36 / 36 | Code confirmed |
| DEX | One `classes.dex` | Code confirmed (ZIP directory) |
| ABI | `arm64-v8a`; the camera bridge also includes `armeabi-v7a` | Code confirmed |
| Android Gradle Plugin | 9.2.0 | Resource confirmed: `app-metadata.properties` |
| Source-version clue | Git revision `31f2d7acdadd5645e233e91cb8f9ff3fe505003e` | Resource confirmed: `version-control-info.textproto` |
| Signature subject/issuer | `C=US, O=Android, CN=Android Debug`, self-signed, RSA 2048, SHA256withRSA | Code confirmed: `META-INF/ANDROIDD.RSA` |
| Certificate SHA-256 | `F9A079A194DE393F087B6206F4B6908CB703783423959F2E1AA8194F344EE07C` | Code confirmed |
| Certificate validity | 2025-08-27 to 2055-08-20 | Code confirmed |

Considering the debug certificate, fixed device identifier, and multiple license checks directly modified to succeed, it is **code confirmed** that this file is not a conventional build that can be directly equated with the official original distribution. It is closer to a re-signed interface-edition/unlock-style modified build. This conclusion describes implementation differences and is not a security assessment.

## 3. Overall Software Role

**Code confirmed**: AutoDisplay is a configurable overlay display system for multi-display in-vehicle systems, HUDs, instrument clusters, and center displays. It maps the following data sources into freely positionable overlay widgets:

1. Vehicle properties provided by `AutoService`;
2. Standard navigation broadcasts from AMap Auto;
3. The built-in AMap Navigation SDK 10.1.600;
4. Route, position, lane, and traffic-light data emitted after injection into the Geely/Flyme stock map process;
5. Android MediaSession, stock media-center Binder, broadcasts from specific music applications, and online lyrics APIs;
6. Android views, external AMap floating maps, stock-map floating maps, Android camera frames in shared memory, and QNX native camera frames;
7. A QNX/HAB projection path that sends Android rendering results to cluster/HUD display endpoints.

It is not a simple fixed HUD. It combines a data-acquisition layer, state-normalization layer, multi-screen layout model, overlay-rendering layer, QNX bridge layer, and online-resource manager.

## 4. Manifest and System Entry Points

### 4.1 Permissions

**Code-confirmed** permissions:

- `SYSTEM_ALERT_WINDOW`: creates floating windows/HUD overlays.
- `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`: backend APIs, lyrics, the map SDK, and local/in-vehicle network paths.
- `RECEIVE_BOOT_COMPLETED`: starts the service after boot, quick boot, and package update.
- `ACCESS_COARSE_LOCATION`, `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`: built-in mapping and continuous navigation.
- `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_SPECIAL_USE`: persistent HUD foreground service; subtype declared as `vehicle_hud_overlay`.
- `REQUEST_INSTALL_PACKAGES`: invokes the system installer to update the APK after downloading it.
- External-storage read/write on Android 10 and below, plus `MANAGE_EXTERNAL_STORAGE`: configuration, fonts, wallpapers, Frida/Hook files, and QNX shared files.
- `com.kugou.android.auto.hafumedia`: integration with Kugou Auto media data.

The Manifest does not declare Bluetooth, USB Host, USB Accessory, or serial-port permissions.

### 4.2 Component Inventory

| Component | exported | Purpose | Evidence level |
|---|---:|---|---|
| `PermissionCheckActivity` | true | Launcher; runtime-environment, activation, overlay, storage, location, notification-listener, and AutoService availability checks | Code confirmed |
| `MainActivity` | false | Main configuration UI; `singleTask`, no task affinity, excluded from recents | Code confirmed |
| `OverlayDisplayService` | false | Core foreground HUD service, `specialUse=vehicle_hud_overlay` | Code confirmed |
| `StartupBroadcastReceiver` | true | Receives `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, and `QUICKBOOT_POWERON` | Code confirmed |
| `HabRawFrameStreamService` | false | Captures frames from an Android virtual display and sends them to QNX through Native HAB | Code confirmed |
| `MusicNotificationListener` | true (protected by system binding permission) | NotificationListener/MediaSession authorization entry point; also ensures that the HUD service starts | Code confirmed |
| `com.amap.api.location.APSService` | false | AMap location SDK service | Code confirmed |
| `FileProvider` | false | Exposes only `external-files/updates/` so the system installer can read update packages | Code confirmed |
| AndroidX Startup/Profile components | Library defaults | Emoji, Lifecycle, and Profile Installer initialization | Code confirmed |

### 4.3 Startup Chains

**Code-confirmed** foreground startup chain:

`Launcher -> PermissionCheckActivity.onCreate()`:

1. Calls `t2.I(context)` for a runtime-environment check; in this APK the method was modified to always return `true`.
2. Enables the `MusicNotificationListener` component.
3. Checks overlay, file-access, foreground/background location, notification-listener, and license states.
4. Explicitly binds `com.simplerbit.autoservice.action.BIND_CAR_PROPERTY_SERVICE`, constrained to package `com.simplerbit.autoservice`, with a 1.5-second timeout to confirm service availability.
5. Starts `MainActivity` when all conditions are met; the main UI validates again and starts `OverlayDisplayService`.

**Code-confirmed** background startup chain:

- After boot, quick boot, or this package being updated, `StartupBroadcastReceiver` calls `OverlayDisplayService.i()` when all permissions are present.
- `MusicNotificationListener.onListenerConnected()` also calls the same entry point.
- `OverlayDisplayService.i()` uses `startForegroundService()`; the service creates the `overlay_display` notification channel and a persistent notification with ID `1001`, while `onStartCommand()` returns `START_STICKY` (value 1).
- After starting, the service loads configuration, creates the rendering controller, starts the music bridge, binds AutoService, registers the AMap navigation receiver, starts the local TCP service on port 23791, prepares the stock-map Hook controller, and registers the screen-wake broadcast.

## 5. Complete User-Visible Functionality

### 5.1 Main UI Modes

`MainActivity.U()` provides seven main actions at the top, **code confirmed**:

1. **My Uploads**: view submitted layouts/resources, status, review notes, previews, and updated submissions.
2. **Simple Settings / Resource Downloads**: browse center-display layouts, cluster layouts, HUD layouts, static wallpapers, dynamic wallpapers, and fonts by vehicle model and type; preview, download, delete, and apply them.
3. **Built-in Map Settings**: choose either the internal navigation SDK or the stock-map Hook SDK and configure the AMap key, HUD map, route, style, TTS, and automatic linkage.
4. **Global Settings**: cross-widget settings for screens, theme, camera linkage, button bindings, lyrics source, import, and export.
5. **Advanced Settings**: edit position, size, layer, visibility, and detailed style per screen and per widget.
6. **About and Activation**: license/membership, device and signature QR codes, redemption codes, version checks, update downloads, and installer invocation.
7. **One-Tap Shutdown**: turns off display/service-related functions.

### 5.2 Multi-Screen and Layout System

**Code confirmed**: `DisplayConfiguration` manages any number of `ScreenConfig` objects. Each screen contains:

- A stable ID, name, and Android `displayId`;
- Available area `x/y/width/height`;
- A visibility switch for all widgets on the screen;
- Rendering-layer type: regular floating window or Presentation layer.

The default screen size is 1440×1920. The code also handles an 800×480 cluster, a 1920×532 HUD, and `virtual_cluster`/`virtual_hud` as special cases. Every `OverlayWidget` has an independent `OverlayPlacement` on every screen, so the same speed, navigation, or lyrics widget can have a different position and style on different physical displays.

Rendering is centrally scheduled through `OverlayDisplayService -> tn`. A vehicle-value change normally redraws only the affected `OverlayKind`; gear and left/right turn-signal changes cause immediate refreshes. The time widget refreshes on minute boundaries. External-map and video/map windows are managed separately to avoid rebuilding them when text widgets update.

### 5.3 All Widget Types

The following list comes from `OverlayKind.values()` and is the **code-confirmed** complete feature catalog:

- **Reference/background**: reference grid, background image, and edge mask.
- **Speed/RPM**: speed, speed unit, speed gauge, speed needle; RPM, RPM unit, RPM gauge, and RPM needle.
- **Temperature/gear**: outside temperature and icon/unit, cabin temperature and icon/unit, gear, actual transmission gear, and steering-wheel angle.
- **Energy**: high-voltage battery, icon, percentage, and unit; fuel quantity, icon, percentage, volume, and unit; electric-range icon, type, value, and unit; average fuel consumption and average energy consumption; combined range and total range with icons/units.
- **Lights/driving state**: left turn, right turn, position light, low beam, high beam, rear fog light, brake light, Auto Hold, door light, multifunction/tire pressure, left/right blind-spot warning, left/right lane line, and time.
- **Cameras**: left, right, front, and rear channels.
- **Music**: artwork, track name, artist, album, playback progress, playback time, and lyrics.
- **External navigation**: external AMap floating map, next-maneuver icon, lane guidance, next road and icon, remaining route distance/time, remaining current-segment distance, ETA, traffic lights, and cruise traffic lights.
- **Built-in/Hook navigation**: HUD map, next maneuver, lane guidance, next road, remaining distance/time, remaining current-road distance, ETA, traffic lights, average-speed zone, current road, camera information, and junction enlargement.
- **Stock map**: floating-map window from the official AMap/Geely map.

### 5.4 Widget Styling Capabilities

`OverlayPlacement` supports far more editing dimensions than position and size, **code confirmed**. The principal fields include:

- Enable state, layer, foreground/background color, font name/weight, and image name;
- Color change below a low-value threshold; rounded corners for album artwork;
- Progress-bar direction, thickness, segmentation, gap, left/right or overall corner radii, and curvature;
- Vertical/diagonal lane-line modes, default/assist/pressure colors, thickness, and dashed-line spacing;
- Tint/brightening for external maneuver icons, high contrast for the lane image, and hidden background/divider;
- 1/3/5/7 lyrics lines, current/non-current colors, and three animation modes;
- Rectangular or elliptical background/content masks, four-side depth, feathering, and transparent regions;
- Needle/progress speed modes and complete gauge styling including start/end angles, center, length, width, needle tail, dual needles, glow, shadow, trail, and animation speed;
- Camera ID, rotation, horizontal/vertical mirroring, cropping on all four sides, corner radius, and delayed display in D gear;
- Stock-map DPI.

### 5.5 Vehicle-Data Display and Button Functions

**Code confirmed**: `vehicle.b` subscribes to and normalizes vehicle data from AutoService, then updates the vehicle widgets described above. The main direct property mapping is provided in Section 8.1. It also implements:

- Four-door/multi-door state aggregation, blind-spot warnings, lane-line state, and multi-wheel tire-pressure aggregation;
- Four camera triggers based on gear and turn signals; hazard lights can trigger the left and right cameras; after the trigger clears, shutdown is delayed according to configuration;
- Auto Hold detection based on a combination of speed, D gear, and brake-related properties;
- Long-press of the Menu/Trip key to switch between the stock and custom instrument cluster;
- Configurable continuous writes to the cluster switch value for "cluster keep-alive";
- Steering-wheel media keys mapped to previous track, next track, and play/pause under the `thirdPartyMusicSteeringControlEnabled` switch;
- Menu or Trip key switching of the multifunction/tire-pressure page.

### 5.6 Music, Artwork, and Lyrics

**Code-confirmed** collection order and capabilities:

1. The authorized `MusicNotificationListener` obtains permission for `MediaSessionManager.getActiveSessions()`.
2. It reads `MediaMetadata` and `PlaybackState` from active `MediaController` objects and preferentially selects the currently playing or most recently updated session.
3. It extracts package name, track, artist, album, duration, position, and artwork; the position advances according to playback state.
4. It registers Geely/ECARX media-center broadcasts and handwritten Binder callbacks to supplement stock-music metadata, progress, and original lyrics.
5. It registers lyrics broadcasts from QQ Music Auto and Luna and recognizes many common field aliases, including `title/song/songName/musicName/trackName`, `artist/singer/author`, `duration/length/totalTime`, and `lyric/lrc/qrc/ttml/xml/content`.
6. If local/stock lyrics are insufficient, it queries NetEase Cloud Music, QQ, Kugou, Kuwo, LRCLIB, or LrcAPI according to configuration, then scores candidates by title, artist, and duration.
7. Artwork may come from a MediaMetadata Bitmap, content URI, or HTTP(S) URL. HTTP artwork reads are limited to 6 MiB, and the decoded target is no larger than approximately 1024×1024.

The lyrics source can be set to `auto/netease/qq/kugou/kuwo/lrclib/lrcapi`. A separate title-compatibility switch cleans the title to improve the match rate.

### 5.7 Navigation Functions

AutoDisplay has three parallel navigation sources:

#### A. External AMap Auto Broadcasts

**Code confirmed**: listens for `AUTONAVI_STANDARD_BROADCAST_SEND` and reads current position, origin/destination, road names, maneuver icons, remaining distance/time, ETA, lanes, traffic lights, and other fields. It can synchronize origin and destination coordinates from external AMap to the built-in navigation engine, calculate multiple routes, and automatically match the route closest to the external one.

#### B. Built-in AMap Navigation SDK

**Code confirmed**: includes the complete AMap mapping/navigation/search/location SDK, with version string `10.1.600`. Users can:

- Save their own AMap API key;
- Search POIs by place name or directly enter latitude and longitude;
- Configure congestion avoidance, avoid highways, avoid tolls, prefer highways, and multiple routes;
- Calculate and switch routes;
- Start simulated navigation, start GPS navigation, or stop navigation;
- Configure license-plate restrictions;
- Use built-in TTS or Geely in-vehicle TTS;
- Adjust map DPI, vehicle-marker position, and day/night/snow/normal custom styles;
- Enable "show HUD map only during navigation";
- Display maneuvers, lanes, road names, distance, time, ETA, current road, cameras, average-speed zones, traffic lights, and junction enlargements.

#### C. Stock-Map Hook SDK

**Code confirmed**: supports targets `com.geely.map` (Preface) and `com.flyme.auto.map` (L7). Workflow:

1. Executes management scripts through an Android-local telnet/root helper;
2. Deploys `frida-server`, `frida-inject`, and `hook_geely_map_overlay_widget.js` from `/sdcard/other/autodisplay/frida/` or by backend download;
3. Starts Frida on `0.0.0.0:27042` and injects the script into the target map process;
4. The script writes route, location, navigation-widget, lane, and traffic-light data to AutoDisplay at local `127.0.0.1:23791`;
5. AutoDisplay draws the HUD route and widgets from this data and can control stock-map floating-map position and visibility through `/sdcard/other/autodisplay/official_map_control.txt`.

The built-in map and Hook map are mutually exclusive data-source options, but external broadcasts can still assist with origin/destination and route matching.

### 5.8 Cameras and Linkage

**Code confirmed**: there are two camera-display implementations:

- **Android helper/HAB path**: Native `libautodisplaycamera.so` receives or sends frames through shared memory/HAB, after which the Android overlay displays them.
- **QNX native-service path**: deploys an encrypted payload on QNX and directly controls the native camera service; reverse, turn, and front-view linkage sends control commands without routing the image through the Android helper.

The default mapping is left=0, right=1, front=2, and rear=3, although each widget can change its low-level ID. Manual display/clear, four-channel linkage, mirroring, rotation, cropping, rounded corners, and automatic shutdown after a trigger clears are supported. Global functions enable/disable the camera backend and start, stop, test, or keep it running.

### 5.9 Projection and Virtual Displays

**Code confirmed**: `HabRawFrameStreamService` renders Android virtual displays into raw frames and sends them to QNX through `libautodisplaycamera.so`/`libuhab.so`. It supports up to three target screens. The start Intent can specify FPS 1–60, 2–8 buffer slots, per-screen dimensions, displayId, and z-order. Native opening uses channel/service identifier `5767769` and a 10-second timeout.

The system also supports the QNX receiver `qnx_hab_screen_receiver`: Android maps an encrypted QNX runtime loader through `/data/vendor/nfs/nfs_ota` to QNX `/otaupdate`, with startup arguments including display, width/height, 30000, and z-order. `projection_keep_alive_enabled` controls keep-alive behavior.

### 5.10 Configuration and Online Resources

**Code confirmed**:

- Global configuration and single-screen layouts can be imported, exported, and reset; missing fonts/backgrounds produce a prompt and can be downloaded online.
- Supports TTF/OTF/TTC fonts, PNG static wallpapers, MP4 dynamic wallpapers, and `.config` layouts.
- Online content is filtered by vehicle model and by center-display/cluster/HUD/static-wallpaper/dynamic-wallpaper/font categories; content may be free or VIP.
- Users can upload configuration files and PNG previews, associate them with a vehicle model and matching wallpaper, and view review status, review notes, and the content code after approval.
- The About page supports activation/redemption, license and resource-membership status, version checks, update downloads, unknown-source installation authorization, and the system installer.

## 6. Key Business Workflows

### 6.1 Vehicle Data to HUD

**Code confirmed**:

`OverlayDisplayService` binds AutoService -> `ICarPropertyService.subscribe(propertyId, callback)` -> `VehicleDataController$1.onPropertyChanged()` -> `vehicle.b.c()` normalizes by type/status -> `fn.w(value, OverlayKind)` writes the unified state table -> `fn.n()` marks only affected widgets -> `OverlayDisplayService.g()` -> `tn` updates the View/Presentation/map/camera window on the corresponding screen.

### 6.2 Automatic Linkage from External AMap to Built-in Navigation

**Code confirmed**:

The AMap standard broadcast provides origin, destination, and total route distance/time -> `o1/k1` caches the external route summary -> the built-in SDK calculates candidate routes using the same origin/destination and multiple preferences -> scores distance difference, time difference, strategy/direction, and other factors -> selects the best route within a threshold -> starts built-in HUD navigation when automatic linkage is enabled.

### 6.3 Music and Lyrics

**Code confirmed**:

MediaSession/stock Binder/broadcast produces track information -> `jl` normalizes title, artist, package, duration, position, and artwork -> stock lyrics take priority when available -> otherwise `qk` builds candidate sources and search requests according to configuration -> scores title/artist/duration -> parses LRC/KRC/QRC/TTML or plain lyrics -> playback progress drives the current line -> `fn` updates track, artist, album, artwork, progress, time, and lyrics widgets.

### 6.4 QNX Cameras

**Code confirmed**:

Generate and save `auth_token` on first start -> download an encrypted QNX payload from the backend or read the built-in encrypted loader -> decrypt with Native code -> write to the Android NFS shared directory -> copy it to a temporary path through SSH/QNX shell and start the control service on port 50020 -> Android sends `hello` and receives a `challenge` -> each command includes `v=2 challenge=<value> nonce=<random value> mac=<64 hex>` -> QNX returns status -> changes in gear/turn signals trigger commands such as `show/showasync/showmulti/showmultiasync/stop/status`.

## 7. Network and Backend HTTP Protocols

### 7.1 AutoManager API

The base URL is **code confirmed** as `https://automanager.simplerbit.com`. Proprietary APIs mainly use `HttpURLConnection`, with POST for JSON requests. Some version-query and file-download entry points use GET.

The common JSON authentication body is generated by `d9.b()`:

```text
hardware_id_hash = SHA-256(device_id)
product_signature = application signature string
platform = "android"
timestamp = current time in milliseconds
requestSignature = SHA-256(hardware_id_hash + ":" + product_signature + ":" + timestamp)
```

The common server JSON envelope is `{ "ok": boolean, "data": ..., "error": { "message": ... } }`. This is not a Bearer/Cookie session; authentication data is recalculated in each request body. The download query string likewise includes `platform=android&channel=stable&hardware_id_hash=...&product_signature=...&timestamp=...&requestSignature=...`.

**Code-confirmed** endpoints:

| Method | Path | Request/response purpose |
|---|---|---|
| POST | `/api/client/licenses/status` | License and resource-membership status; `data.license` / `data.subscription`, with `expires_at` in the license |
| POST | `/api/client/codes/redeem` | Activation/redemption; adds `redeem_code`, returns license and subscription models |
| GET | `/api/client/releases/latest?<authentication-query>` | Latest stable release; reads `data.release.id/version/changelog/force_update/size_bytes` |
| GET | `/api/client/releases/{id}/download?<authentication-query>` | APK update download |
| POST | `/api/client/contents` | Resource list; adds `vehicle_scope` and optional `vehicle_model_id` |
| POST | `/api/client/contents/{id}/preview` | Resource-preview binary |
| POST | `/api/client/contents/{id}/download` | Resource-file binary with progress support |
| POST | `/api/client/wallpaper-assets/{id}/download` | Matching wallpaper download for a layout |
| POST | `/api/client/vehicle-models` | Vehicle-model list |
| POST | `/api/client/content-submissions/mine` | Upload records for the current device/signature |
| POST multipart | `/api/client/content-submissions` | Creates a resource submission |
| POST multipart | `/api/client/content-submissions/{id}/update` | Updates a resource submission |
| POST | `/api/client/content-submissions/{id}/preview` | Submission preview |
| POST | `/api/client/official-map-hooks/{target}/download` | Stock-map Hook package download |
| POST | `/api/client/qnx-payloads/{name}/download` | QNX camera/projection payload download |

Resource model `e3` has the following **code-confirmed** fields: `id/title/content_type/description/preview_key/file_key/file_size/version/is_free/vehicle_model_id/vehicle_model_name/vehicle_brand/vehicle_series/vehicle_model_year`. Supported types:

`autodisplay_layout_cluster`, `autodisplay_layout_hud`, `autodisplay_layout_center`, `autodisplay_wallpaper_static`, `autodisplay_wallpaper_dynamic`, `autodisplay_font`.

Vehicle-model fields: `id/name/brand/series/model_year`. Upload-record fields: `id/code/title/content_type/status/review_note/vehicle_model_name/approved_content_code/created_at/reviewed_at`.

The multipart upload has these **code-confirmed** fields: the five common authentication fields, `title`, `content_type`, optional `vehicle_model_id`, optional `wallpaper_asset_id/wallpaper_original_name`, and file parts `config_file` (`application/octet-stream`) and `preview_file` (`image/png`).

### 7.2 Online Lyrics APIs

All are **code-confirmed** GET requests:

- Kugou mobile search: `https://mobileservice.kugou.com/api/v3/lyric/search?...&keyword=<encoded-keyword>`; reads `data.info[].filename/singername/hash/album_audio_id/duration/trans_param.union_cover/album_img`.
- Kugou KRC candidates: `https://krcs.kugou.com/search?ver=1&man=yes&client=mobi&...&hash=...`; also includes compatible paths through `lyrics.kugou.com/search` and `lyrics.kugou.com/download`.
- Kuwo: `https://www.kuwo.cn/openapi/v1/www/lyric/getlyric?musicId=...&httpsStatus=1&reqId=...&plat=web_www&from=lrc`; reads `data.lrclist[].time/lineLyric`.
- NetEase Cloud Music details: `https://music.163.com/api/song/detail?ids=[...]`; lyrics: `https://music.163.com/api/song/lyric?id=...&lv=1&kv=1&tv=-1`, reading `lrc/klyric/tlyric.lyric`.
- QQ: `https://c.y.qq.com/lyric/fcgi-bin/fcg_query_lyric_new.fcg?songmid=...&format=json&nobase64=1`.
- LrcAPI: `https://api.lrc.cx/lyrics?title=...`.
- LRCLIB: `https://lrclib.net/api/search?track_name=...`, reading `syncedLyrics/plainLyrics`.

Requests use a browser-like User-Agent; QQ and Kuwo set the corresponding Referer/Origin. No WebSocket or MQTT client was found in the static code.

## 8. Cross-Process, Broadcast, and Local Protocols

### 8.1 AutoService AIDL

This is the clearest software dependency inside the project, **code confirmed**:

- Bind Action: `com.simplerbit.autoservice.action.BIND_CAR_PROPERTY_SERVICE`
- Target package: `com.simplerbit.autoservice`
- Descriptor: `com.simplerbit.autoservice.contract.ICarPropertyService`
- Callback descriptor: `com.simplerbit.autoservice.contract.ICarPropertyCallback`

AIDL transactions:

| Transaction code | Method | Arguments/result |
|---:|---|---|
| 1 | `getProperty(int propertyId)` | Returns `CarPropertyValue` |
| 2 | `setProperty(CarPropertyValue)` | Returns boolean |
| 3 | `subscribe(int, ICarPropertyCallback)` | Subscribes |
| 4 | `unsubscribe(int, ICarPropertyCallback)` | Unsubscribes |
| 5 | `getSupportedPropertyIds()` | Returns `int[]` |
| callback 1 | `onPropertyChanged(CarPropertyValue)` | Property-change callback |

`CarPropertyValue` Parcel order: `propertyId, valueType, status, timestampMillis, localUpdateTimestampMillis, intValue, longValue, floatValue, doubleValue, boolValue, stringValue, bytesValue`. Status: 1 available, 2 unavailable, 3 error. Type: 1 int, 2 long, 3 float, 4 double, 5 boolean, 6 string, 7 bytes, 8 char.

Main property mapping, **code confirmed**:

- 10001 speed; 10021 RPM; 10005 outside temperature; 10006 cabin temperature; 10003 gear; 10004 actual gear; 10058 steering-wheel angle;
- 10002 high-voltage battery; 10011 fuel; 10012 fuel volume; 10022 range; 10059 electric-range type; 10060 electric range; 10061 average fuel consumption; 10062 average energy consumption;
- 10038 low beam; 10039 high beam; 10040 brake light; 10041 position light; 10042 rear fog light;
- 10023–10030, 10013–10020, 10031–10036, 10054–10056, and others are used for tires, doors, blind spots, lane lines, and multi-value aggregation;
- 10044 is the combined left/right turn state; 10037 is a physical-button event and is also subscribed separately.

### 8.2 External AMap Floating-Map Control Broadcasts

**Code confirmed**: sent to `com.autonavi.amapautp` or `com.autonavi.amapauto`:

- Show: `com.autonavi.plus.showmap`
- Close: `com.autonavi.plus.closemap`
- Show extras: `x/y/w/h/displayId/displayIndex`

The stock-map floating map does not use an outgoing broadcast. It writes `/sdcard/other/autodisplay/official_map_control.txt`:

```text
visible=0|1
x=<left>
y=<top>
w=<right>
h=<bottom>
displayId=<id>
displayIndex=<index>
dpi=<dpi>
```

AutoDisplay receives `AMAP_EXTERNAL_MAP_READY/SHOWN/HEARTBEAT/HIDDEN` (each prefixed with `com.simplerbit.autodisplay.`). `SHOWN/HEARTBEAT/HIDDEN` carry `displayId`; they track whether the external map is actually visible and support timeout cleanup.

### 8.3 AMap Standard Navigation Broadcast

**Code confirmed** Action: `AUTONAVI_STANDARD_BROADCAST_SEND`. Statically recoverable fields include:

- Origin/destination/location: `FromPoiLatitude/Longitude`, `ToPoiLatitude/Longitude`, `arrivePOILatitude/Longitude`, `CAR_LATITUDE/LONGITUDE`, `CAR_DIRECTION`, `endPOILatitude/Longitude`;
- Route: `EXTRA_ROAD_INFO`, `ROUTE_ALL_DIS`, `ROUTE_ALL_TIME`, `ROUTE_REMAIN_DIS`, `ROUTE_REMAIN_TIME`, `SEG_REMAIN_DIS`, `ETA_TEXT`;
- Road and maneuver: `CUR_ROAD_NAME`, `NEXT_ROAD_NAME`, `NEW_ICON`, `ICON`, `ROUNG_ABOUT_NUM`;
- Traffic lights: `routeRemainTrafficLightNum`, `TRAFFIC_LIGHT_NUM`.

Extension broadcasts:

- `com.simplerbit.autodisplay.AMAP_LANE_INFO`: `show/frontLane/backLane/extFlags/extensionLane/tollGate/recommend`; normalized to a six-segment `|`-delimited array.
- `com.simplerbit.autodisplay.AMAP_CRUISE_TRAFFIC_LIGHT`: `lights`, `KEY_TYPE`, `trafficLightStatus`, `redLightCountDownSeconds`, `greenLightLastSecond`, `dir`, `waitRound`.

Field types may be int, String, array, or Bundle across AMap Auto versions; `k1` contains multiple compatibility-reading branches. Obfuscation prevents full recovery of the semantics of every numeric `KEY_TYPE`, so all enumeration names **cannot be confirmed**.

### 8.4 Stock-Map Hook Local TCP Protocol

**Code confirmed**: the service listens on `127.0.0.1:23791`, backlog 1, UTF-8, line-delimited frames, TCP_NODELAY, with no separate application-layer login handshake. Recoverable frames:

- `HELLO|...` / `PING|...`: reads at least `[2]=source/identifier`, `[3]=version/state`, and `[4]=integer state`, serving as a heartbeat.
- `ROUTE|<JSON>`: route state; parses the navigating flag and route-point list for subsequent HUD-route drawing.
- `LOC|<reserved>|lat|lon|bearing|speed|segment|link|point|matched|...|...|source`: position, heading, speed, route indices, match state, and source.
- `LANE|<JSON>`: fields `clear/show/frontLane/backLane/recommend/extFlags/extensionLane/tollGate/source`.
- `GUIDE|<JSON>`: fields `clear`, `maneuverId`, `roundaboutExit`, `legacyIcon`, `nextRoadName`, `routeDistanceMeters`, `routeTimeSeconds`, `segmentDistanceMeters`, `source`. The next maneuver is normalized as `maneuverId|roundaboutExit|legacyIcon`; the other fields update the next road, remaining route distance, remaining route time, remaining current-road distance, and ETA. Data is cleared when `clear=true` or after approximately 1.8 seconds without an update.
- `TL|<JSON>`: `source/clear/status/seconds/direction/waitRound/rawLightType/rawLightStatus/distanceMeters/trafficLightId`; normalized for display as `status|seconds|direction|0|waitRound`.

Lane/guidance data is cleared after approximately 1.8 seconds without an update, and traffic-light data after approximately 3 seconds. Heartbeat time determines whether the Hook is online.

### 8.5 Music Broadcasts and Stock Binder

**Code confirmed** receivers:

- `ecarx.xsf.mediacenter.action.BROADCAST_MEDIA_CENTER`
- `ecarx.xsf.mediacenter.action.BROADCAST_MEDIA_CENTER.SEMANTIC`
- `mediacenter_action`
- `com.tencent.qqmusiccar.LYRIC_TTML_BROADCAST`
- `com.tencent.qqmusiccar.COLLECT_BROADCAST`
- `com.luna.music.LYRIC_BROADCAST`
- `com.luna.music.LYRIC_TTML_BROADCAST`
- `com.luna.music.COLLECT_BROADCAST`

Handwritten Binder compatibility protocols:

- `com.ecarx.eas.xsf.mediacenter.IExCallback`: transaction 1 is action and transaction 2 is callback; reads action code, method, and payload. Events 201/203/204/514 or events related to lyrics/playback/metadata trigger a refresh.
- `ecarx.xsf.mediacenter.IMediaCenterSvc`: transaction 19 registers the client Binder.
- `com.geely.lib.oneosapi.mediacenter.IMediaCenter`: transaction 5 gets the music manager; also reads current/focused audio source and application.
- `com.geely.lib.oneosapi.mediacenter.listener.IMediaStateListener`: transaction 1 is media data, 2 is progress, and 6 is LRC. Media data includes media ID, title, artist, artwork/extension fields, and duration.

The code also queries/integrates `com.ecarx.sdk.openapi`, `com.geely.mediacenterservice`, `com.flyme.auto.music`, `com.tencent.qqmusiccar`, `com.luna.music`, `cn.kuwo.kwmusiccar`, and `com.kugou.android.auto`. The Manifest queries Kuwo and Kugou ContentProviders, but the related read branches are obfuscated, so the complete column-name contract of each Provider **cannot be confirmed**.

### 8.6 QNX Camera Control Protocol

**Code confirmed**: the default display-end host is `192.168.118.2`, overridable in `autodisplay_stream.host`; control port is 50020. QNX loader arguments include both `192.168.118.2` and `192.168.118.1`. ASCII line protocol:

1. The client sends `hello\n`;
2. The server returns a line beginning with `OK` and containing `challenge=<value>`;
3. Command format: `v=2 challenge=<challenge> nonce=<random hex> mac=<64 hex> <command>\n`;
4. `mac` is generated by `QnxNativeSecurity.nativeControlMac(auth_token, challenge, nonce, command)`;
5. The server returns a normal line or `ERR...`.

Recognized command families include `status`, `preload`, `show`, `showasync`, `showmulti`, `showmultiasync`, and `stop/quit/exit`. Each image-path parameter is organized by `ep`: cameraId, screen type/target, x, y, width, height, mirroring, rotation, four-side cropping, and corner radius.

The Android root helper separately listens on local `127.0.0.1:50030`; telnet fallback tries port 23 on `::1` and `127.0.0.1`. Frida status detection connects to `127.0.0.1:27042`.

### 8.7 HabRawFrameStreamService Intent Protocol

**Code confirmed**:

- `com.simplerbit.autodisplay.stream.START_PROJECTION_RAW_FRAME`
  - `fps`: default 30, range 1–60;
  - `slots`: default 6, range 2–8;
  - Single target: `screen_id/width/height/display_id/zorder`, default 800×480;
  - Multiple targets: `screen_specs`, format `screenId,displayId,width,height,zorder;...`, maximum 3 entries.
- `com.simplerbit.autodisplay.stream.SET_PROJECTION_ZORDER`
  - `screen_id`, `zorder`; calls Native `nativeSetZOrder` while running.
- `com.simplerbit.autodisplay.stream.STOP_PROJECTION_RAW_FRAME`
- Failure report to `OverlayDisplayService`: `com.simplerbit.autodisplay.PROJECTION_STREAM_FAILED`, extra `reason=<ExceptionClass:message>`.

### 8.8 Protocols Not Found

Within AutoDisplay's own package and the obfuscated business package, it is **code confirmed that no** Android Bluetooth API, USB API, serial-port library, MQTT client, or WebSocket client was found. Hardware paths are implemented by AutoService AIDL, TCP, SSH/telnet, NFS shared directories, and HAB Native. The AMap SDK may internally use additional HTTP/Socket protocols, but those belong to a third-party SDK implementation and should not be classified as custom AutoDisplay protocols.

## 9. Local Data, Configuration, and Files

### 9.1 Main Configuration

**Code confirmed**: the main file is `files/display_overlay_config.json` in the application's private directory. The export format is `formatVersion=2`; core fields:

- Global: `selectedWidgetId`, physical-button bindings/write values/keep-alive, long-press milliseconds, tire-pressure key, third-party music control;
- Cameras: automatic shutdown, left/right turn, D/R gear, and hazard-light linkage;
- Lyrics: `lyricSourceId`, title compatibility;
- `screens[]`: `id/name/displayArea/presentationLayer`; runtime visibility can be saved separately;
- Per-screen `overlays[]`: widget ID, enable state, position/size, layer, and the style fields from Section 5.4.

A single-screen export is `formatVersion=2, exportType="screenLayout", screen={...}`. Assets also include the old `display_overlay_config.sample.json` (version 1), proving that the importer must support the historical format.

### 9.2 External Directories

Directories **code confirmed** through `ym`:

- `/sdcard/other/autodisplay/config`
- `/sdcard/other/autodisplay/config/layouts/{hud|center|cluster}`
- `/sdcard/other/autodisplay/backsources/{static|dynamic}`
- `/sdcard/other/autodisplay/gauges`
- `/sdcard/other/autodisplay/fonts`
- `/sdcard/other/autodisplay/frida/`
- `/sdcard/other/autodisplay/official_map_control.txt`

Resource downloads also write `.meta.json`, containing `id/content_type/version/preview_key/vehicle_model_id/vehicle_model_name/file_size/actual_size/saved_at`. List caches are vehicle-model/resource JSON files under the application cache and contain `base_url` and the save time.

### 9.3 SharedPreferences

Important tables, **code confirmed**:

- `display_license`: `device_id/signature/is_activated/last_heartbeat_success/offline_grace_started/license_expires_at/license_seal`;
- `display_update`: `pending_apk`;
- `builtin_map_settings`: API key, navigation data source, map DPI/style, vehicle-marker position, TTS, license plate, route preferences, external automatic navigation, and prompts;
- `official_map_hook`: target, automatic injection, automatic-injection state, and time;
- `autodisplay_stream`: QNX host, active virtual screens, `projection_keep_alive_enabled`;
- `qnx_camera_server`: `auth_token`, camera-service keep-alive switch, and related state.

Backup rules exclude the entire file/database/sharedpref/external/root scope; FileProvider permits only the update directory.

## 10. Native Libraries and Resource Roles

| ABI / library | Size | Functional role | Evidence level |
|---|---:|---|---|
| arm64 `libAMapSDK_NAVI_v10_1_600.so` | 60,642,936 | AMap map rendering, routing, navigation, and location core | Code confirmed |
| arm64 `libautodisplaycamera.so` | 268,096 | HAB frame sending, camera bridge, QNX payload decryption, control MAC, license seal, runtime-environment check | Code confirmed (JNI declarations) |
| armv7 `libautodisplaycamera.so` | 157,996 | Same as above, 32-bit compatibility | Code confirmed |
| arm64 `libuhab.so` | 19,592 | Low-level Qualcomm/QNX HAB communication | Reasonable inference (name and call chain) |
| arm64 `libc++.so` | 710,824 | Native C++ runtime | Code confirmed |

Java-side capabilities exported by `libautodisplaycamera.so`: `nativeOpen/nativeSendFrame/nativeSetZOrder/nativeStats/nativeBoostThread/nativeClose`, plus `nativeControlMac/nativeDecryptQnxAsset/nativeDecryptQnxAssetToBytes/nativeLicenseSeal/nativeRuntimeClean`. Algorithm details and the HAB binary-frame format reside in Native code and **cannot be fully recovered from Java alone**.

Asset roles:

- Extensive AMap styles, shaders, road/lane/junction SVGs, navigation resources, and offline resources;
- AMap TTS speech models and the `xiaoyun` voice;
- 2,079 Bootstrap SVG icons for the custom icon selector;
- 159 AMap/navigation SVGs;
- InterDisplay font;
- Sample layout configuration.

## 11. Third-Party Components and Their Functional Roles

- **AMap Map/Navigation SDK 10.1.600**: maps, location, POI, routing, multi-route selection, navigation, junction images, TTS, and map styles; it accounts for most of the APK size.
- **AndroidX AppCompat/Core/Lifecycle/RecyclerView/Startup/ProfileInstaller**: UI compatibility, lifecycle, and performance initialization. Visible versions include AppCompat 1.6.1, Core 1.9.0, and RecyclerView 1.1.0.
- **Material Components 1.10.0**: UI controls and themes.
- **Kotlin Coroutines 1.6.4**: part of the asynchronous infrastructure; the main business code also uses Java Executor/Handler extensively.
- **JSch/JZlib/jBCrypt**: related to SSH, compression, and authentication on the QNX side; SSH sessions and QNX command execution are visible in the code.
- **Alibaba NUI classes/resources**: TTS/speech dependencies bundled with the AMap SDK; AutoDisplay's business logic does not present itself as a standalone Alibaba Cloud speech-product client.
- **ECARX/Geely/Flyme media interfaces**: not present as complete SDK Manifest components; compatibility with stock media centers is implemented by package queries, reflection/generic EAS Binder, and handwritten Parcel protocols.
- **Bootstrap Icons**: custom-widget icon library.

## 12. Relationship with Other APKs in the Project

### 12.1 AutoService: Strong Dependency

**Code confirmed**: this is the only in-project APK explicitly declared in Manifest `<queries>` and directly bound through its own package name. AutoDisplay contains a copy of the `com.simplerbit.autoservice.contract` AIDL contract and must verify that the service can be bound before startup; at runtime it continuously subscribes to vehicle properties. Without AutoService, the startup check shows "Vehicle data service unavailable," and the core vehicle widgets, camera linkage, and physical-button functions have no data source.

The relationship can therefore be determined as:

`AutoService (vehicle-bus/property adaptation layer) -> AIDL -> AutoDisplay (layout, state aggregation, and display layer)`.

### 12.2 AutoAudio: Complementary Functionality, No Direct IPC Found

**Code confirmed**: AutoDisplay does not reference the `AutoAudio` package name, Action, Provider, or AIDL. The two may target related in-vehicle audio/display scenarios, but AutoDisplay gets music data from the system MediaSession, stock media center, and third-party music applications; it does not depend on AutoAudio. At project level, their relationship can only be described as the **reasonable inference** that they belong to the same product family and complement the same usage scenario; no runtime call can be asserted.

### 12.3 External In-Vehicle Software

**Code-confirmed** strong associations include:

- AMap Auto: `com.autonavi.amapautp`, `com.autonavi.amapauto`;
- Geely/Flyme maps: `com.geely.map`, `com.flyme.auto.map`;
- ECARX/Geely media: `com.ecarx.sdk.openapi`, `com.geely.mediacenterservice`, `com.flyme.auto.music`;
- QQ Music Auto, Luna, Kuwo, and Kugou.

## 13. Static-Analysis Limitations and Items Requiring Dynamic Confirmation

1. Default JADX output reported 5 errors and left 296 method placeholders. After a full simple-mode recovery, only three internal AMap methods and `pb.onDraw(Canvas)` remained; all `com.simplerbit` business methods were recovered. The remaining four items have fallback/smali instruction views and do not affect this report's business-protocol conclusions.
2. `libautodisplaycamera.so`, `libuhab.so`, and the 60 MiB AMap Native library were not reverse-engineered at instruction level. Only JNI boundaries can be confirmed for the HAB frame header, shared-memory layout, QNX payload encryption algorithm, and control-MAC algorithm.
3. Stock-map Hook JavaScript/Frida resources are downloaded from the backend on demand by target. The APK does not contain every final script, so internal Hook points in the map process and other possible extension-object fields cannot be fully confirmed. The ROUTE/GUIDE JSON schema on the current APK's local receiver has been recovered.
4. AutoService property-ID names are available only from the AutoDisplay consumer-side mapping. Exact vehicle definitions for combined properties such as 10013–10036 should follow the AutoService report.
5. The ECARX/Geely media interface is a handwritten Binder compatibility layer. Different vehicle firmware may use different transaction codes or Parcel order; static analysis can describe only the current compatibility implementation.
6. AMap standard-broadcast fields differ in type between head-unit versions. The current code includes compatible readers, but the actually transmitted subset requires confirmation in a vehicle environment.
7. No backend API was requested, so current server responses, membership policies, resource counts, the complete set of Hook payload names, and update availability cannot be confirmed.
8. Other project APKs were not installed, so a shared directory is not treated as evidence of a software relationship unless an explicit package name/AIDL/Action provides such evidence.

## 14. Conclusion

**Code confirmed**: AutoDisplay v3.6.2 is the project's main display and interaction-orchestration layer. Its core value is not a single HUD page, but a unified, editable multi-screen widget model that combines AutoService vehicle properties, built-in/external/Hook AMap navigation, stock and third-party music, online lyrics, Android/QNX cameras, and virtual-display/HAB projection.

## 15. JADX Gap-Recovery Review

The default structured output contained 296 `Method not decompiled` placeholders across 182 Java files. Full simple mode recovered 292; the remaining four are three internal AMap SDK methods and the custom drawing method `pb.onDraw(Canvas)`, each of which has fallback or smali instruction-level content.

The methods that failed in the default output within AutoDisplay's own package, including Binder media callbacks, screen-configuration import/generation, HAB Raw Frame Service, and vehicle-property cache updates, now all have method bodies. The obfuscated class method `dm.q(String)` was also fully recovered, confirming the `GUIDE` fields `clear`, maneuver, roundabout exit, compatibility icon, next road, route distance/time, segment distance, and source, as well as the 1.8-second expiry cleanup flow.

Consequently, no AutoDisplay business-protocol method is currently missing solely because of a JADX placeholder. The remaining limitations come from Native black boxes, on-demand Hook resources, and differences in real in-vehicle firmware.
