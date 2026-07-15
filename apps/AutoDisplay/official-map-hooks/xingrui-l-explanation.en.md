# How the Geely Xingrui L Map Hook Works

[`xingrui_l.js`](./xingrui_l.js) is a Frida agent that adapts the navigation data inside the Geely map application to the protocol consumed by AutoDisplay. It is intended for the `com.geely.map` process and identifies itself as the `geely_xingrui_l` provider and vehicle profile.

The script has the same broad purpose as [`l7.js`](./l7.js): observe the official map without replacing its route calculation, collect the navigation state that is already available inside the application, and forward that state to another process on the head unit. The similarity ends at the output protocol, however. The two map applications expose their internal data through different classes and callbacks, so the Xingrui L implementation follows a substantially different route through the application.

The L7 hook mostly reads high-level Flyme Auto Java objects. The Xingrui L hook combines a native interception point in `libGbl.so`, Java presenter callbacks, dead-reckoning location data, an internal JSON protocol, route caches, and periodic navigation-state polling. That mixed design is the main idea behind the file.

## File layout and runtime

The bundled file contains 16,166 lines. As with `l7.js`, the first 13,652 lines are generated runtime code rather than vehicle logic. They include Frida Java Bridge, Android ART and JNI support, Node.js compatibility helpers, and module-loader wrappers.

The Xingrui-specific module begins at [`xingrui_l.js:13653`](./xingrui_l.js#L13653) under the original source name `hook_geely_map_overlay_widget.js`. Its own implementation is about 2,514 lines long, compared with roughly 1,375 lines for the L7 module.

The main constants are defined at [`xingrui_l.js:13661`](./xingrui_l.js#L13661):

| Setting | Value |
| --- | --- |
| Target map package | `com.geely.map` |
| Provider | `geely_xingrui_l` |
| Vehicle profile | `geely_xingrui_l` |
| Socket address | `127.0.0.1:23791` |
| Protocol version | `1` |
| Coordinate system | `gcj02` |
| Hook version | `2026.07.07.auto-heartbeat` |
| Native route offset | `40063332` / `0x2635164` |

The whole Xingrui module runs inside `Java.performNow()`. A small empty interval keeps the agent alive, while the actual 25 ms processing loop is created later by the location and socket installer.

## What is actually enabled

The file contains more hook implementations than the current entry point starts. This distinction matters because several functions are complete fallback designs but are not part of the active runtime path.

The startup block at [`xingrui_l.js:16127`](./xingrui_l.js#L16127) enables these components:

1. `installRoutePointExportWhenReady(1)` for the native route hook.
2. `installLocationSocketExport()` for location, socket transport, heartbeats, and task processing.
3. `installGuidanceWidgetExport()` for lane show and hide events.
4. `installRouteFocusExport()` for route-selection changes.
5. `installPresenterGuidanceExport()` for maneuvers and the final displayed distance.
6. `installExternalInteractionProtocolExport()` for traffic-light protocol messages.
7. `startNavigationStatePolling()` for navigation lifecycle and route refreshes.

The file also defines installers for `NaviController`, `DrivingLayer`, `NaviComponent`, and several Geely external-screen route managers. Those installers are not called by the current startup block. They document alternative extraction paths and share the same conversion helpers, but they do not install hooks in this build unless another caller invokes them.

The same applies to the JSON guidance parser and the object-based traffic-light countdown parser: both are implemented, but their installers are not active in the current entry path.

## End-to-end data flow

The active runtime can be summarized as follows:

```text
libGbl.so setPathInfosNative        -> full route and alternatives
NmeaManager$1.onDrInfoUpdate        -> vehicle position
BaseNaviPresenter.onUpdateNaviInfo  -> maneuver and road information
DriveGuideInfoHolder                -> final distance shown by the map UI
NaviObserverRouter                  -> lane show/hide information
ExternalInteractionManager 80189    -> traffic-light JSON
NaviController.isNaving()           -> navigation lifecycle
                                      |
                                      v
                       conversion and deduplication
                                      |
                                      v
                      pending task / socket queues
                                      |
                                      v
                           TCP 127.0.0.1:23791
```

Most Java hooks call the original map method first, retain any Java object that must survive beyond the callback, and then enqueue a short task. The task queue is limited to 120 entries, and the 25 ms processing loop executes up to eight tasks per pass.

## Socket transport and message framing

The socket implementation begins at [`xingrui_l.js:15909`](./xingrui_l.js#L15909). It creates a Java `Socket`, enables `TCP_NODELAY`, and writes UTF-8 through a `BufferedWriter`.

Messages are stored as objects in `pendingSocketLines`. Each object contains the line, a diagnostic label, and the enqueue time. The queue keeps at most 100 entries. If the socket is unavailable, the queued lines stay in the array and the loop attempts to flush them again later.

The transport recognizes the same top-level message families as the L7 hook:

| Prefix | Payload |
| --- | --- |
| `HELLO` | Process, map, provider, vehicle, and hook identity |
| `PING` | Periodic liveness message |
| `LOC` | Compact pipe-separated position |
| `ROUTE` | JSON route state |
| `GUIDE` | JSON maneuver or distance update |
| `LANE` | JSON lane state |
| `TL` | JSON traffic-light state |

`HELLO` is queued after a connection is established. `PING` is generated no more than once every three seconds. The same 25 ms loop processes tasks, sends the newest location, generates heartbeats, and flushes the socket queue.

## Vehicle location

The active location source is installed on [`NmeaManager$1.onDrInfoUpdate()`](./xingrui_l.js#L16105):

```java
com.autosdk.common.location.NmeaManager$1
    .onDrInfoUpdate(com.autonavi.gbl.pos.model.DrInfo)
```

This callback supplies dead-reckoning data. The hook reads:

- `drMatchPos` as the preferred road-matched position;
- `drRawPos` when no matched position is available;
- `drMatchAzi` as the preferred course;
- `drAzi` as the course fallback;
- `spd` as the speed value.

Longitude and latitude are stored as integers and divided by `1,000,000`. The resulting coordinates are ordinary GCJ-02 degrees.

Unlike the L7 `LocInfo` source, `DrInfo` does not provide the current route segment, link, and point indexes in this hook. Xingrui therefore sends `-1` for those three fields.

The callback does not immediately create a line for every update. Instead, it overwrites `pendingCarLocation` with the newest `DrInfo`. The 25 ms loop sends the latest unsent value, so bursts of location callbacks are naturally coalesced while the newest position is preserved.

The Xingrui location format is:

```text
LOC|timestamp|latitude|longitude|course|speed|-1|-1|-1|1
```

The L7 location line continues after the fixed `1` field with the protocol version, provider, source, and coordinate system. The Xingrui line is therefore the shorter, legacy-compatible form of the same position message.

## Native route extraction

The main route hook starts at [`xingrui_l.js:14857`](./xingrui_l.js#L14857). Instead of waiting for a high-level route manager callback, it locates `libGbl.so` and attaches an interceptor at:

```text
libGbl.so base + 0x2635164
```

The script names this target `setPathInfosNative`. It retries the module lookup once per second, up to 30 attempts, so the native library can finish loading after the Frida agent starts.

When the native function is entered, the interceptor receives the JNI environment pointer, a list of path attributes, and the selected-route index. It manually builds a small JNI function table for operations such as:

- `GetObjectClass`
- `GetMethodID`
- `CallObjectMethodA`
- `CallIntMethodA`
- `CallLongMethodA`
- `GetFieldID`
- `GetObjectField`
- `GetIntField`

Using those JNI calls, the hook walks through the Java list without relying on Frida's high-level Java wrappers at that interception point.

For every path it reads:

- `getPathID()`
- `getLength()`
- `getTravelTime()`
- `getTrafficLightCount()`
- `getTollCost()`
- `getSegmentCount()`
- every Segment and Link point list

Each route becomes an object with `index`, `selected`, `pathId`, `length`, `travelTime`, `trafficLightCount`, `tollCost`, `signature`, and `points`.

Route geometry follows the familiar hierarchy:

```text
PathInfo
  -> SegmentInfo
       -> LinkInfo
            -> Coord2DInt32 [lon, lat]
```

The hook first reads points from every Link. If a segment produces no Link points, it falls back to `SegmentInfo.getPoints()`.

The active native payload contains:

```json
{
  "ts": 0,
  "source": "native:setPathInfosNative:selected-live",
  "navigating": true,
  "selectIndex": 0,
  "routeCount": 3,
  "pointCount": 1234,
  "routes": []
}
```

The actual `ts`, counts, routes, and navigation state are filled at runtime. The payload contains all paths from the native list, not only the selected one.

The native route coordinates are appended directly from the integer `lon` and `lat` fields. In contrast with L7's `normalizeCoordinate()`, this route builder does not divide or normalize them before putting them into `points`. The separate location path does divide its coordinates by `1,000,000`.

## Route cache and selection changes

Every usable route payload is saved in `lastRoutePayload`. Native payloads are also copied into `nativeRoutePayload`. The cache allows the script to respond to a route-selection event without walking the entire route tree again.

Three active hooks update the selected route:

| Hook | Selection method |
| --- | --- |
| `RouteCarResultData.setFocusIndex(int)` | Select by list index |
| `NaviPresenter.suggestChangePathID(long)` | Find a cached route by `pathId` or `naviId` |
| `NaviPresenter.switchAlternateRoute(..., int)` | Select by alternative-route index |

`cloneNativeRoutePayloadWithSelection()` deep-copies the cached JSON, updates `selected` on every route, changes `selectIndex`, sets `navigating=true`, recalculates `pointCount`, and sends the result as a fresh route state.

The route fingerprint is based on the selected route's identifiers, length, travel time, traffic-light count, toll cost, total point count, route signature, and the first, middle, and last coordinate pairs. Navigation polling uses this fingerprint to recognize when the cached active route has changed.

## Java route implementations kept in the file

The module includes a second, high-level Java route reader beginning at [`xingrui_l.js:15049`](./xingrui_l.js#L15049). It understands `PathInfo`, `NaviPath`, `RouteCarResultData`, and the different path-list containers used by Geely's map UI.

Its helper functions can obtain a current route from several places:

1. `ModuleServiceUtils.getDriveService().getRouteCarResultData()`.
2. `RouteRequestController.getInstance().getmCarRouteResult()`.
3. `driveService.getNaviPathInfo()`.
4. `NaviController.getInstance().getNaviPath()`.

The file also contains complete hook installers for:

- both `DrivingLayer.drawRoute()` overloads;
- `NaviController.onMainNaviPath()`;
- `NaviController.setNaviPath()` and `startNavi()`;
- `NaviComponent.drawRoute()`, `setNaviPath()`, and `setRoute()`;
- `ExtraScreenManager`, `WidgetExtScreenManager`, and `HudExtScreenManager` route methods.

Those installers are not invoked by the current startup block. They form a ready-made Java fallback layer and explain much of the size difference between `xingrui_l.js` and `l7.js`.

## Navigation guidance

The active full-guidance hook is installed at [`xingrui_l.js:14655`](./xingrui_l.js#L14655):

```java
BaseNaviPresenter.onUpdateNaviInfo(ArrayList)
```

The callback retains both the navigation list and the Presenter instance. In the queued task, it tries to read `getmDirectionInfo()` from the Presenter and uses that text as an override for the next-road name shown by the lower-level navigation objects.

Like L7, Xingrui may receive several `NaviInfo` objects, and each object may contain several panels in `NaviInfoData`. The script scores each panel:

- preferred `NaviInfoFlag` panel: +2;
- usable maneuver ID: +8;
- non-empty next-road name: +6;
- available segment distance: +4;
- positive segment distance: +1.

Route remaining distance and time add another point each to the overall `NaviInfo` score. The highest-scoring context is used for the outgoing message.

Xingrui reads more fallback fields around the upcoming cross than L7. It examines the first entry in `nextCrossInfo` and can use:

- `maneuverID`
- `crossManeuverID`
- `outCnt`
- `curToSegmentDist`
- `nextRoadName`

It also falls back to `naviInfo.crossManeuverID` and `naviInfo.linkRemainDist`. This gives the presenter path several ways to reconstruct a maneuver even when the primary panel is sparse.

The full `GUIDE` payload includes the maneuver, roundabout exit, current and next road names, route distance and time, remaining traffic-light count, route position indexes, selected panel information, and `rawSegmentDistanceMeters`.

Full guidance messages use a 500 ms duplicate window.

## Final displayed maneuver distance

Xingrui has a second guidance channel that does not exist in the L7 implementation. It hooks:

```java
DriveGuideInfoHolder.updateNextRoadDistanceView(
    int distance,
    String[] textArray,
    int totalDistance)
```

This callback is close to the final user-interface value. It sends a small partial `GUIDE` message containing:

```json
{
  "segmentDistanceMeters": 320,
  "totalSegmentDistanceMeters": 500,
  "finalSegmentDistance": true
}
```

When this hook is installed, the regular full-guidance message sets `segmentDistanceMeters` to `-1` and keeps the panel's original value in `rawSegmentDistanceMeters`. The receiver can therefore use the dedicated final-distance message as the authoritative changing distance while still retaining the raw navigation value for context.

These partial updates use a 250 ms duplicate window, allowing the displayed countdown distance to change more frequently than the rest of the maneuver payload.

## Reserved JSON guidance path

The file also implements `sendGuideFromProtocolJson()`. It can parse fields such as:

- `icon` and `newIcon`
- `roundAboutNum`
- `nextRoadName`
- `segRemainDis` and `nextSegRemainDis`
- `routeRemainDis` and `routeRemainTime`
- `routeRemainTrafficLightNum`
- `curRoadName`, `curSegNum`, and `curPointNum`
- `etaText`

It marks those messages with `guideProtocol: true`. No active startup hook calls this parser in the current file, so it represents an additional protocol adapter retained alongside the presenter-based path.

## Lane guidance

The active lane hooks are installed on [`NaviObserverRouter`](./xingrui_l.js#L14702):

```java
onShowNaviLaneInfo(LaneInfo)
onHideNaviLaneInfo()
```

The show callback retains the `LaneInfo` object and processes it asynchronously. The converter reads up to 24 values from:

- `frontLane`
- `backLane`
- `extensionLane`, falling back to `backExtenLane`
- `optimalLane`

The outgoing lane count is the greater of the front and back array lengths. If `optimalLane` contains usable values, it defines the `recommend` flags. Otherwise, non-`255` entries in `frontLane` are treated as recommendations.

The payload includes `show=true`, all lane arrays, the Boolean `recommend` array, `extensionLane`, and empty compatibility arrays for `extFlags` and `tollGate`. Identical lane states are suppressed for 500 ms.

The hide callback sends `show=false` and `clear=true`, resetting the lane signature at the same time.

The conversion logic is essentially the same as L7. The difference is the event source: Xingrui uses `NaviObserverRouter`, while L7 uses Flyme's `NaviObserverProxy`.

## Active traffic-light protocol

The traffic-light hook is installed at [`xingrui_l.js:14620`](./xingrui_l.js#L14620) on two overloads of:

```java
ExternalInteractionManager.buildDataAndDispatch(...)
```

Only protocol ID `80189` is processed. The associated string is parsed as JSON and converted into a `TL` message.

The active parser reads:

- `trafficLightStatus`
- `redLightCountDownSeconds`
- `greenLightLastSecond`
- `dir`
- `waitRound`

The status mapping is:

| Raw protocol status | Shared status |
| --- | --- |
| `1` or `2` | `1` - red |
| `4` or `5` | `4` - green |
| `8` | `2` - transition state |
| another non-negative value | `2` - transition state |

Direction values `1`, `2`, and `4` are kept. Raw direction `8` becomes `4`, raw direction `7` becomes `1`, and other values become `-1`.

The countdown uses `redLightCountDownSeconds`, falling back to `greenLightLastSecond`. The resulting payload includes normalized and raw status values, direction, waiting round, a generated traffic-light ID, and `trafficLightProtocol=true`. Its `distanceMeters` value is `-1` because protocol `80189` does not supply the separate ticker distance used by L7.

Traffic-light messages use a 500 ms duplicate window.

## Object-based traffic-light implementation

Another complete traffic-light converter is present at [`xingrui_l.js:14395`](./xingrui_l.js#L14395). It can process `NaviController.onUpdateTrafficLightCountdown(ArrayList)` objects by:

1. matching the countdown entry to the current navigation `pathID`;
2. reading the `lightStates` list;
3. selecting the state whose `stime` and `etime` cover the current time;
4. falling back to `lastSeconds` when needed;
5. reading direction, wait count, link identity, phase, and display type.

That path distinguishes raw `lightType` values `2`, `3`, and `4` as red, green, and transition. Its installer belongs to `installNaviControllerGuidanceExport()`, which is defined but not started by the current entry block. The active source remains protocol `80189`.

## Navigation lifecycle

Xingrui determines official navigation state by calling:

```java
NaviController.getInstance().isNaving()
```

The polling loop begins at [`xingrui_l.js:14825`](./xingrui_l.js#L14825) and runs once per second.

When the state changes from inactive to active, the hook clones the latest cached route, sets `navigating=true`, and sends it as the current navigation route.

While navigation remains active, the poller compares the route fingerprint with the last navigation fingerprint. It refreshes the cached active route when the fingerprint changes or when more than two seconds have passed since the previous refresh.

When navigation changes to inactive, it clears the route caches and sends a route payload with `navigating=false`, zero counts, and an empty `routes` array.

Lane state is cleared by the corresponding lane hide callback. Guidance and traffic-light clear helpers also exist in the module and are used when their own data paths report empty or hidden content.

## Detailed comparison with the L7 hook

Both scripts speak to the same AutoDisplay endpoint, but their internal strategies reflect two different map applications.

| Area | Xingrui L | Galaxy L7 |
| --- | --- | --- |
| Target package | `com.geely.map` | `com.flyme.auto.map` |
| Provider/profile | `geely_xingrui_l` | `flyme_l7` / `geely_l7` |
| Vehicle module size | About 2,514 lines | About 1,375 lines |
| Main route source | Native `libGbl.so + 0x2635164` | Java `RouteRequestManager` |
| Route object access | Manual JNI calls | Frida Java wrappers |
| Route alternatives | Native hook exports the full list | Route result exports the full usable list |
| Route selection | Rewrites a cached native payload | Reads the current route result again |
| Route point values | Raw integer `lon/lat` from the SDK | Normalized degree coordinates |
| Route metadata | Compact route payload | Adds protocol, provider, vehicle, package, coordinate, point-order, and point-unit fields |
| Position source | Dead-reckoning `DrInfo` | Road-matched `LocInfo` |
| Position scaling | Explicit `/ 1,000,000` | Multi-format coordinate normalization |
| Segment/link/point | Sent as `-1` | Read from `matchInfo` when available |
| Location buffering | Keeps only the newest pending position | Enqueues each accepted throttled line |
| Full guidance source | `BaseNaviPresenter` | `NaviObserverProxy` |
| Next-road fallback | Presenter text plus `nextCrossInfo` | Panel and `maneuverInfo` data |
| Displayed distance | Separate partial `GUIDE` channel | Included in the full guide payload |
| Guidance fallback | JSON parser is present but inactive | `GuidanceTbtData(...)` log parser is active |
| Lane event source | `NaviObserverRouter` | `NaviObserverProxy` |
| Lane conversion | `front/back/optimal/extension` | Same conversion model |
| Active traffic-light source | Internal JSON protocol `80189` | `GuidanceNsHelper` log messages |
| Traffic-light distance | `-1` in the active protocol path | Taken from the ticker log |
| Navigation lifecycle | `isNaving()` poll every second | Stop hooks, lifecycle logs, and route-result polling |
| Active-route refresh | Cached route every two seconds or on change | Event-driven export with route dirty state |
| Hook installation | Immediate startup; native route retries for 30 seconds | Dedicated 25 ms installation retry loop |
| Socket line capacity | 100 queued line objects | 200 queued strings |
| Task processing | Up to 8 tasks every 25 ms | Up to 8 tasks every 25 ms |
| Heartbeat | About every 3 seconds | About every 3 seconds |

## Practical event sequences

### Starting navigation on Xingrui L

```text
Map prepares PathInfo objects
  -> native setPathInfosNative hook builds and caches all routes
  -> NaviController.isNaving() becomes true
  -> poller clones the cached route with navigating=true
  -> ROUTE is sent to AutoDisplay
  -> BaseNaviPresenter starts producing GUIDE messages
  -> lane and traffic-light sources update their own states
```

### Switching an alternative route

```text
RouteCarResultData / NaviPresenter reports a new selection
  -> cached native payload is deep-copied
  -> selected flags and selectIndex are updated
  -> pointCount and timestamp are refreshed
  -> the revised ROUTE payload is sent
```

### Updating the next-turn distance

```text
BaseNaviPresenter supplies maneuver and road context
  -> full GUIDE message is sent
DriveGuideInfoHolder updates the final UI distance
  -> small GUIDE message with finalSegmentDistance=true is sent
  -> AutoDisplay combines the stable maneuver context with the changing distance
```

### Ending navigation

```text
NaviController.isNaving() changes to false
  -> route caches are cleared
  -> ROUTE with navigating=false and an empty route list is sent
  -> lane hide and other content-specific events clear their displays
```

## Summary

`xingrui_l.js` is best understood as a multi-source adapter for Geely's own map stack. The complete route is captured close to the native GBL boundary, location comes from the dead-reckoning observer, maneuver context comes from the navigation Presenter, the final countdown distance comes from the UI holder, lanes come from the guide router, and traffic lights come from an internal JSON protocol.

`l7.js` reaches the same AutoDisplay protocol through a more centralized Flyme Auto object model. The output categories are deliberately similar, but the acquisition paths, route coordinate representation, lifecycle handling, and guidance composition are tailored to the two different map applications.
