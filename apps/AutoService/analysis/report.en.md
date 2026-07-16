# AutoService v0.1.0 Static Analysis Report

## 1. Report Scope

- This report analyzes only functional implementation, business processes, interface protocols, and software relationships. It does not assess security, privacy, or compliance.

### 1.1 Conclusion Labels

| Label | Meaning |
|---|---|
| `[Code-confirmed]` | Directly confirmed by Java, smali, Manifest, or interface code |
| `[Resource-confirmed]` | Directly confirmed by XML, assets, JSON, or static Native content |
| `[Reasonable inference]` | Multiple items of static evidence agree, but runtime observation has not validated them |
| `[Cannot be confirmed]` | Depends on a real head unit, server, shell decryption, or dynamic state and cannot be determined from the static sample |

## 2. Sample Identity

| Item | Value |
|---|---|
| Package name | `com.simplerbit.autoservice` |
| Application version | `0.1.0` |
| versionCode | `20` |
| Minimum Android SDK | 28 |
| target / compile SDK | 36 / 36 |
| APK size | 3,905,165 bytes |
| SHA-256 | `60249EF37B033EB3217348E8426B9E5DE907900288172C956443C8FFCBAD5D4E` |
| Current sample signature | Android Debug certificate |
| Certificate SHA-1 | `70:92:2D:33:E3:27:35:60:4B:51:19:FE:AE:FD:79:29:D2:EF:A2:B1` |
| Certificate SHA-256 | `F9:A0:79:A1:94:DE:39:3F:08:7B:62:06:F4:B6:90:8C:B7:03:78:34:23:95:9F:2E:1A:A8:19:4F:34:4E:E0:7C` |

Evidence: `apktool/apktool.yml`, `apktool/AndroidManifest.xml`, the APK hash, and the static certificate-extraction results.

## 3. Software Role

`[Code-confirmed]` AutoService is a unified head-unit property adaptation service. It converts three underlying data sources, Android Automotive VHAL, ECARX Car Signal, and an in-memory Mock, into a unified `CarPropertyValue` model, then exposes that model through AIDL/Binder to higher-level software such as AutoAudio and AutoDisplay.

Its responsibilities can be summarized as follows:

```text
VHAL HIDL --+
            +-> vehicle-profile mapping JSON -> unified property repository -> AIDL CarPropertyService
ECARX ------+                                                     +-> AutoAudio
Mock -------+                                                     +-> AutoDisplay
```

`[Code-confirmed]` AutoService does not itself produce alert sounds or instrument-cluster displays. It is responsible for lower-level signal access, conversion of vehicle-model differences, property caching, subscription dispatch, and unified read/write operations.

## 4. Overall Architecture

| Layer | Core classes/resources | Responsibility |
|---|---|---|
| Shell entry points | `com.SecShell.SecShell.AW/AP/CP`, `libSecShell*.so`, `assets/classes0.jar` | Application/ComponentFactory/Provider entry points and encapsulated-payload processing |
| Business Application | `AutoServiceApplication` | Restore data source/vehicle profile, build the property stack, start the foreground service |
| Foreground Binder service | `CarPropertyService`, `defpackage.x5` | Lifecycle, caller admission, AIDL methods, status overlay dot |
| Unified property definitions | `assets/car_properties.json`, `contract.*` | Metadata for 73 properties, types, states, access modes, Parcelable |
| Vehicle-profile mappings | 13 `*_signal_mapping.json` files | Unified propertyId to underlying signalId and transformation steps |
| VHAL source | `defpackage.vo`, VHAL callback classes | HIDL get/set/subscribe |
| ECARX source | `defpackage.r8` | `ecarxcar_service`, `car_signal`, SignalFilter |
| Mock source | In-memory property source and debugging-panel call code | Local read/write and callback simulation |
| Debug UI | `com.simplerbit.autoservice.debug.MainActivity` | Activation, data source/profile, property read/write/subscription, probing, updates |
| Licensing and updates | `defpackage.j5` and its callers | Licensing, code redemption, version query, APK download |

## 5. Manifest, Shell Entry Points, and Business Entry Point

Evidence: `apktool/AndroidManifest.xml`.

### 5.1 SecShell Component Replacement

- `[Code-confirmed]` The Application class in the Manifest is not `AutoServiceApplication`, but `com.SecShell.SecShell.AW`.
- `[Code-confirmed]` `appComponentFactory` is `com.SecShell.SecShell.AP`.
- `[Code-confirmed]` An additional `com.SecShell.SecShell.CP` Provider is registered with authority `com.simplerbit.autoservice.CP`; it is not exported and has `initOrder=2147483647`.
- `[Code-confirmed]` AW, AP, and CP are absent from the visible JADX Java and apktool smali.
- `[Reasonable inference]` SecShell restores or loads these entry points during the very early Provider/Application phase, then delegates to the actual business Application.

### 5.2 Business Application

- `[Code-confirmed]` The visible business-initialization class is `com.simplerbit.autoservice.AutoServiceApplication`.
- `[Code-confirmed]` The default data source is `VHAL`.
- `[Code-confirmed]` The default vehicle profile is `FS11_A2`.
- `[Code-confirmed]` `onCreate()` restores the data source and vehicle profile from `source_selection`, falling back to defaults on an exception.
- `[Code-confirmed]` The legacy vehicle-profile string `FX11_H` is migrated to `G636`.
- `[Code-confirmed]` If the vehicle profile is incompatible with the data source, the first visible profile for that source is selected.
- `[Reasonable inference]` SecShell ultimately triggers initialization of this business Application; otherwise the visible service code could not establish the property stack when it accesses `AutoServiceApplication`.

### 5.3 Activity

- `[Code-confirmed]` The Launcher Activity is `com.simplerbit.autoservice.debug.MainActivity`.
- `[Code-confirmed]` The Activity is exported, has `MAIN/LAUNCHER`, uses `excludeFromRecents=true`, and has an empty `taskAffinity`.
- `[Code-confirmed]` This Activity is not a simple settings page, but a complete vehicle-property debugging console.

### 5.4 CarPropertyService

- `[Code-confirmed]` Component name: `com.simplerbit.autoservice.service.CarPropertyService`.
- `[Code-confirmed]` `exported=true`; the foreground-service type is `connectedDevice`.
- `[Code-confirmed]` `onStartCommand()` returns value 1, corresponding to `START_STICKY`.
- `[Code-confirmed]` The foreground notification channel is `car_property_service`; the notification ID is 1001.
- `[Code-confirmed]` Service creation, start, binding, and unbinding all update the runtime heartbeat/overlay state.

Manifest actions:

| action | Expected invocation method |
|---|---|
| `com.simplerbit.autoservice.action.BIND_CAR_PROPERTY_SERVICE` | Higher-level application calls bindService |
| `com.simplerbit.autoservice.action.START_CAR_PROPERTY_SERVICE` | Application, boot broadcast, or higher-level application starts the service |
| `com.simplerbit.autoservice.action.UPDATE_STATUS_OVERLAY` | Request a refresh of the status overlay dot |

`[Code-confirmed]` The current `onStartCommand()` does not branch on the action. All three start actions ultimately only update the heartbeat, refresh the overlay dot, and keep the sticky service alive. The actual Binder object is returned by `onBind()`.

### 5.5 BootReceiver and FileProvider

- `[Code-confirmed]` `BootReceiver` listens only for `BOOT_COMPLETED`.
- `[Code-confirmed]` When the license branch is satisfied, it calls `startForegroundService()` with the START action.
- `[Code-confirmed]` `AutoServiceApplication` also starts the service under the same condition during initialization, so the service can be established outside boot startup.
- `[Code-confirmed]` The FileProvider authority is `com.simplerbit.autoservice.fileprovider`, used by the update-APK installation flow.

## 6. Startup and Runtime Flow

### 6.1 Normal Startup

1. `[Reasonable inference]` SecShell CP/AW reads the encapsulated payload and establishes the application entry point early in process startup.
2. `[Code-confirmed]` `AutoServiceApplication` reads `source_selection`.
3. `[Code-confirmed]` It creates the property metadata, mappings, and underlying data source for the selected vehicle profile.
4. `[Code-confirmed]` The default combination is `VHAL + FS11_A2`.
5. `[Code-confirmed]` It starts `CarPropertyService` when the license branch is satisfied.
6. `[Code-confirmed]` The service creates the foreground notification, writes runtime state, and creates the optional 1px overlay dot.
7. `[Code-confirmed]` Higher-level software obtains the `ICarPropertyService` Binder after binding.
8. `[Code-confirmed]` Every AIDL method first verifies the packages and certificates associated with the calling UID before entering the unified property layer.

### 6.2 Data-Source/Vehicle-Profile Switching

1. The debug interface selects VHAL, ECARX, or MOCK.
2. It selects a vehicle profile compatible with the data source; MOCK can display all vehicle-profile configurations.
3. `AutoServiceApplication.a()` closes the old underlying source.
4. It saves the new `data_source` and `vehicle_profile`.
5. It reads the new mapping JSON and rebuilds the property manager and underlying source.
6. It preserves the higher-level subscription registry.
7. It reads and redistributes current values for properties that already have subscriptions.

`[Code-confirmed]` Steps 6 and 7 are implemented by `AutoServiceApplication.a()` reusing the old `x` subscription dispatcher and iterating through supported properties.

## 7. Debug Main-Screen Functions

Key class: `debug.MainActivity`. Even after decompilation it exceeds 3,000 lines and contains the complete debugging workflow.

### 7.1 Activation and Basic Information

- `[Code-confirmed]` Displays license/activation state.
- `[Code-confirmed]` Displays the device code and product signature and can generate a QR code.
- `[Code-confirmed]` Supports redemption-code input.
- `[Code-confirmed]` Supports checking for updates, downloading an APK, and invoking system installation.

### 7.2 Data Source and Vehicle Profile

- `[Code-confirmed]` Three-way data-source selection: VHAL, ECARX, MOCK.
- `[Code-confirmed]` The 13 vehicle-profile configurations are filtered by data source.
- `[Code-confirmed]` Displays the current actual source and current vehicle profile.
- `[Code-confirmed]` The UI displays exception information if VHAL/ECARX source switching fails.

### 7.3 Service State

- `[Code-confirmed]` Displays whether the service is running, process state, start time, runtime duration, and most recent heartbeat.
- `[Code-confirmed]` Can enable a 1px status overlay dot in the lower-left corner.
- `[Code-confirmed]` The overlay-dot color changes according to the selected source, actual underlying source, and current number of Binder connections.

### 7.4 Property Browsing, Read/Write, and Subscriptions

- `[Code-confirmed]` Displays properties in `READ`, `WRITE`, and `READ_WRITE` sections.
- `[Code-confirmed]` Supports full refresh and per-property reads.
- `[Code-confirmed]` Provides input and write controls for writable entries.
- `[Code-confirmed]` Supports per-property subscribe/unsubscribe.
- `[Code-confirmed]` Supports subscribe-all and unsubscribe-all actions.
- `[Code-confirmed]` Callback values, states, and times can be displayed in the log/property row.

### 7.5 Mock Debugging

- `[Code-confirmed]` Provides Mock sliders for vehicle speed and engine speed.
- `[Code-confirmed]` Supports writing a value to any Mock property and triggering the unified callback.
- `[Reasonable inference]` Mock is primarily intended to validate AutoAudio/AutoDisplay higher-level behavior when no real vehicle bus is available.

### 7.6 Manual Lower-Level Debugging

- `[Code-confirmed]` Property ID, Area ID, value type, and value can be entered.
- `[Code-confirmed]` Real VHAL/ECARX sources support read, write, subscribe, and unsubscribe.
- `[Code-confirmed]` The interface explicitly disallows entering "manual lower-level debugging" with the MOCK source because MOCK has a separate property-simulation area.
- `[Code-confirmed]` Supports probing VHAL classes, HIDL services, and property configurations.
- `[Code-confirmed]` Supports probing the `ecarxcar_service` and `car_signal` Binders.

## 8. AIDL/Binder Protocol

### 8.1 Descriptors

| Interface | Descriptor |
|---|---|
| Service | `com.simplerbit.autoservice.contract.ICarPropertyService` |
| Callback | `com.simplerbit.autoservice.contract.ICarPropertyCallback` |

### 8.2 Service Transactions

Evidence: `contract/ICarPropertyService.java`.

| Transaction code | Method | Return value |
|---:|---|---|
| 1 | `getProperty(int propertyId)` | `CarPropertyValue` |
| 2 | `setProperty(CarPropertyValue value)` | `boolean` |
| 3 | `subscribe(int propertyId, ICarPropertyCallback cb)` | `void` |
| 4 | `unsubscribe(int propertyId, ICarPropertyCallback cb)` | `void` |
| 5 | `getSupportedPropertyIds()` | `int[]` |

Callback transaction code 1: `onPropertyChanged(CarPropertyValue value)`.

### 8.3 `CarPropertyValue` Parcelable

`[Code-confirmed]` The Parcel write and read order is strictly as follows and must remain consistent across APK contracts:

1. `int propertyId`
2. `int valueType`
3. `int status`
4. `long timestampMillis`
5. `long localUpdateTimestampMillis`
6. `int intValue`
7. `long longValue`
8. `float floatValue`
9. `double doubleValue`
10. `byte boolValue`
11. `String stringValue`
12. `byte[] bytesValue`

`[Code-confirmed]` The same object always contains all candidate fields; the caller reads the corresponding field according to `valueType`.

### 8.4 Types, States, and Access Modes

Type enumeration:

| Value | Type |
|---:|---|
| 1 | INT |
| 2 | LONG |
| 3 | FLOAT |
| 4 | DOUBLE |
| 5 | BOOLEAN |
| 6 | STRING |
| 7 | BYTES |
| 8 | CHAR |

State enumeration: 1=`AVAILABLE`, 2=`UNAVAILABLE`, 3=`ERROR`.

Access strings: `READ`, `WRITE`, `READ_WRITE`.

### 8.5 Service-Method Behavior

Core implementation: `defpackage.x5`.

- `getProperty`: reads the current cached/underlying value from the unified property manager.
- `getSupportedPropertyIds`: returns a copy of the IDs supported by the current vehicle-profile mapping.
- `subscribe`: stores callback Binders by propertyId; the same Binder is deduplicated with a wrapper key.
- `unsubscribe`: removes the callback and removes the property entry when its collection is empty.
- `setProperty`: rejects null values, nonexistent properties, read-only properties, and values with mismatched types.
- `setProperty`: after a successful lower-level write, a `READ_WRITE` property is read again from the lower layer and the unified repository is updated; a pure WRITE property is updated with the written value.

### 8.6 Caller Admission

`[Code-confirmed]` Every AIDL method first calls `CarPropertyService.a()` to check the Binder calling UID. The service's own UID is directly allowed; an external UID is matched by PackageManager package name and signature history.

| Package name | Allowed certificate SHA-1 |
|---|---|
| `com.simplerbit.autoaudio` | `10:47:96:CD:05:CC:CB:9A:C8:9B:8E:06:96:CC:A0:F9:96:5B:51:54` |
| `com.simplerbit.autoaudio` | `FA:AF:CD:D5:A2:A4:03:F2:EB:F1:AA:B3:B2:6D:8B:02:9B:F8:A2:91` |
| `com.simplerbit.autodisplay` | `82:58:0C:CA:43:9E:7F:90:67:B2:A8:FC:08:E2:9F:F7:C5:BE:88:09` |
| `com.simplerbit.autodisplay` | `FA:AF:CD:D5:A2:A4:03:F2:EB:F1:AA:B3:B2:6D:8B:02:9B:F8:A2:91` |

`[Code-confirmed]` The result is cached by calling UID. A mismatch throws `SecurityException("Caller is not allowed to access AutoService")`. This describes a protocol-access condition and is not a risk assessment.

## 9. Unified Vehicle-Property Dictionary

`[Resource-confirmed]` `assets/car_properties.json` defines 73 entries; ID 10048 is unused. All `100xx` properties are READ, while `30001-30011` are READ_WRITE.

### 9.1 Vehicle State and Powertrain

| ID | Type | Meaning |
|---:|---|---|
| 10000 | INT | Current vehicle-profile number |
| 10001 | INT | Vehicle speed, integer km/h |
| 10002 | INT | High-voltage battery SOC percentage |
| 10003 | CHAR | Gear character |
| 10004 | INT | Transmission gear |
| 10005 | FLOAT | Exterior temperature |
| 10006 | FLOAT | Interior temperature |
| 10007 | INT | Door-lock state |
| 10008 | INT | Brake-light state |
| 10009 | INT | Left turn-signal state |
| 10010 | INT | Right turn-signal state |
| 10011 | INT | Remaining fuel percentage |
| 10012 | INT | Remaining fuel volume, liters |
| 10013 | FLOAT | Front-left tire pressure, kPa |
| 10014 | FLOAT | Front-right tire pressure, kPa |
| 10015 | FLOAT | Rear-left tire pressure, kPa |
| 10016 | FLOAT | Rear-right tire pressure, kPa |
| 10017 | INT | Front-left door open/closed state |
| 10018 | INT | Front-right door open/closed state |
| 10019 | INT | Rear-left door open/closed state |
| 10020 | INT | Rear-right door open/closed state |
| 10021 | INT | Engine speed |
| 10022 | INT | Remaining driving range |

### 9.2 Lanes, Blind Spots, and Radar

| ID | Type | Meaning |
|---:|---|---|
| 10023 | INT | Left lane-line color |
| 10024 | INT | Left lane-line type |
| 10025 | INT | Right lane-line color |
| 10026 | INT | Right lane-line type |
| 10027 | INT | Left driving blind-spot alert |
| 10028 | INT | Right driving blind-spot alert |
| 10029 | INT | Left parking blind-spot alert |
| 10030 | INT | Right parking blind-spot alert |
| 10031 | INT | Front-left outer radar distance |
| 10032 | INT | Front-right outer radar distance |
| 10033 | INT | Rear-left outer radar distance |
| 10034 | INT | Rear-left inner radar distance |
| 10035 | INT | Rear-right inner radar distance |
| 10036 | INT | Rear-right outer radar distance |

### 9.3 Buttons, Lights, Power, and Body

| ID | Type | Meaning |
|---:|---|---|
| 10037 | STRING | Button event; underlying int list converted to a string |
| 10038 | INT | Low-beam state |
| 10039 | INT | High-beam state |
| 10040 | INT | Parking-brake state |
| 10041 | INT | Position-light state |
| 10042 | INT | Rear fog-light state |
| 10043 | STRING | Power-mode request; underlying int list converted to a string |
| 10044 | INT | Turn indication: 0 off, 1 left, 2 right, 3 hazard lights |
| 10045 | INT | Left mirror folding state |
| 10046 | INT | Right mirror folding state |
| 10047 | INT | Front-passenger seat occupancy state |
| 10049 | INT | Speed-limit value |
| 10050 | INT | Charging-connector state: 1 plugged in, 2 unplugged |
| 10051 | INT | Charging state: 1 charging, 2 stopped |
| 10052 | INT | `pavstdmodests` state |
| 10053 | INT | Alarm state |

### 9.4 Driving and Energy Extensions

| ID | Type | Meaning |
|---:|---|---|
| 10054 | FLOAT | Accelerator-pedal ratio, one decimal place |
| 10055 | INT | Brake-pedal ratio |
| 10056 | INT | Driver seat-belt state |
| 10057 | INT | Automated-driving controller type |
| 10058 | INT | Steering-wheel angle |
| 10059 | INT | Pure-electric range flag |
| 10060 | INT | High-voltage battery driving range |
| 10061 | FLOAT | Average fuel consumption per 100 km, one decimal place |
| 10062 | FLOAT | Average electricity consumption per 100 km, one decimal place |

### 9.5 Read/Write Control Properties

| ID | Type | Meaning | Access |
|---:|---|---|---|
| 30001 | BOOLEAN | Air-conditioning switch | READ_WRITE |
| 30002 | INT | Front-left window position | READ_WRITE |
| 30003 | INT | Front-right window position | READ_WRITE |
| 30004 | INT | Rear-left window position | READ_WRITE |
| 30005 | INT | Rear-right window position | READ_WRITE |
| 30006 | INT | Front-left reading light | READ_WRITE |
| 30007 | INT | Front-right reading light | READ_WRITE |
| 30008 | INT | Rear-left reading light | READ_WRITE |
| 30009 | INT | Rear-right reading light | READ_WRITE |
| 30010 | INT | Instrument-cluster state | READ_WRITE |
| 30011 | INT | Trunk open/closed state | READ_WRITE |

`[Cannot be confirmed]` The complete semantics of every enumeration value on every vehicle model cannot be derived from the unified dictionary alone. Raw values and conversion rules are governed by the corresponding vehicle-profile mapping JSON.

## 10. Vehicle-Profile Mapping Protocol

### 10.1 JSON Structure

`[Code-confirmed]` Every vehicle-profile mapping entry supports the following fields:

| Field | Purpose |
|---|---|
| `propertyId` | Unified property ID |
| `readSignalId` | Underlying read-signal ID |
| `readSignalName` | Optional underlying read-signal name |
| `writeSignalId` | Underlying write-signal ID |
| `writeSignalName` | Optional underlying write-signal name |
| `readTransform` | Raw value to unified value |
| `writeTransform` | Unified value to underlying value |

Parser class: `defpackage.po`; per-entry model: `defpackage.oo`.

### 10.2 Transformation Steps

`[Code-confirmed]` A transform is an ordered `steps` array supporting:

- `mapping`: looks up the value's canonical string as a key in `map`; `default` can be used on a miss.
- `expression`: parses only one arithmetic expression beginning with `x`.

`[Code-confirmed]` The expression implementation supports `x / n`, `x * n`, `x + n`, and `x - n`, converting to the target INT/LONG/FLOAT/DOUBLE type. Evidence: `defpackage.u8.j()`.

### 10.3 Thirteen Vehicle-Profile Configurations

| Configuration | Model name | Backend | Mapping file | Property count | Writable mapping count |
|---|---|---|---|---:|---:|
| FS11_A2 | Xingrui L Hybrid | VHAL | `vhal_fs11_a2_signal_mapping.json` | 69 | 11 |
| KX11_A2 | Xingyue L Hybrid | VHAL | `vhal_kx11_a2_signal_mapping.json` | 69 | 11 |
| KX11_22_LSHD | MY2022 Xingyue L Leishen Hybrid | VHAL | `vhal_22kx11lshd_signal_mapping.json` | 22 | 5 |
| KX11_24 | MY2024 Xingyue L | VHAL | `vhal_24kx11_signal_mapping.json` | 69 | 11 |
| KX11_24_TJ | MY2024 Xingyue L Tianji | VHAL | `vhal_24kx11tj_signal_mapping.json` | 69 | 11 |
| KX11_25 | MY2025 Xingyue L | VHAL | `vhal_25kx11_signal_mapping.json` | 69 | 11 |
| FX12 | Galaxy L7 | VHAL | `vhal_fx12_signal_mapping.json` | 72 | 11 |
| FX121_8 | Galaxy L7, system 1.8 | VHAL | `vhal_fx121_8_signal_mapping.json` | 69 | 11 |
| FS12 | Galaxy L6 | VHAL | `vhal_fs12_signal_mapping.json` | 72 | 11 |
| FX11 | Boyue L | VHAL | `vhal_fx11_signal_mapping.json` | 69 | 11 |
| G636 | Boyue L overseas edition | VHAL | `vhal_g636_signal_mapping.json` | 69 | 11 |
| G426 | Boyue Cool | VHAL | `vhal_g426_signal_mapping.json` | 65 | 11 |
| KX11_21 | MY2021 Xingyue L | ECARX | `ecarx_21kx11_signal_mapping.json` | 21 | 10 |

`[Resource-confirmed]` The 72 mappings for FS12 and FX12 cover additional states including 10050, 10051, and 10052. Differences in property counts between vehicle profiles reflect differences in underlying signal capabilities.

`[Code-confirmed]` MOCK mode allows any vehicle-profile configuration to be used, but does not access the real VHAL/ECARX service declared by that profile.

## 11. VHAL Data Source

Core class: `defpackage.vo`.

### 11.1 Connection

- `[Code-confirmed]` Directly calls `android.hardware.automotive.vehicle.V2_0.IVehicle.getService()`.
- `[Code-confirmed]` This is the Android Automotive Vehicle HAL 2.0 HIDL/Binder interface.
- `[Code-confirmed]` If the service is null or the connection fails, an initialization exception is thrown and displayed by the debug interface.

### 11.2 Read

1. Converts the unified propertyId to `readSignalId` according to the vehicle-profile mapping.
2. Creates `VehiclePropValue`, setting prop and fixed `areaId=0`.
3. Calls `IVehicle.get(value, callback)`.
4. Waits up to 2000 ms using `CountDownLatch`.
5. Converts the underlying value to the unified type through `readTransform`.
6. Divides the VHAL nanosecond timestamp by 1,000,000 to obtain milliseconds; uses local time if no valid timestamp is available.

### 11.3 Subscription

- `[Code-confirmed]` Uses `IVehicle.subscribe(IVehicleCallback, SubscribeOptions[])`.
- `[Code-confirmed]` Every `SubscribeOptions.propId` comes from the vehicle-profile mapping.
- `[Code-confirmed]` `sampleRate=0.0f` and `flags=1`.
- `[Code-confirmed]` The underlying callback is converted, cached, and dispatched to the corresponding AIDL callback.

### 11.4 Write

1. Checks the unified property's access mode and type.
2. Obtains the underlying value through `writeTransform`.
3. Creates a VHAL `VehiclePropValue` with areaId fixed at 0.
4. Calls `IVehicle.set()`.
5. Return status 0 is treated as success.
6. A READ_WRITE property is then read again from the lower layer so that the actual vehicle state prevails.

### 11.5 Special Values

`[Code-confirmed]` For unified properties 10037 (button event) and 10043 (power-mode request), the VHAL `int32Values` list is formatted as a string, such as `"[0,17,0]"` used by AutoAudio.

## 12. ECARX Data Source

Core class: `defpackage.r8`.

### 12.1 Connection Chain

1. `[Code-confirmed]` Obtains `ecarxcar_service` from Android `ServiceManager`.
2. `[Code-confirmed]` Converts it to the `IECarXCar` interface.
3. `[Code-confirmed]` Calls `getCarService("car_signal")`.
4. `[Code-confirmed]` Constructs `ECarXCarPropertyManagerBase`.
5. `[Code-confirmed]` KX11_21 is currently the only vehicle-profile configuration declared to use ECARX.

### 12.2 Read and Write

- `[Code-confirmed]` Read calls `getProperty(cls, signalId, 1)`.
- `[Code-confirmed]` Write calls `setProperty(cls, signalId, 1, value)`.
- `[Code-confirmed]` A write is considered successful only if it returns `ApiResult.SUCCEED`.
- `[Code-confirmed]` Java type `cls` is selected according to the unified property type.
- `[Code-confirmed]` Both read and write values pass through the corresponding transform in the vehicle-profile mapping.

### 12.3 Subscription

- `[Code-confirmed]` Uses `SignalFilter` to collect the required ECARX signalId values.
- `[Code-confirmed]` Calls `registerCallback()` and `registerSignals()`.
- `[Code-confirmed]` During cleanup it correspondingly calls `unregisterSignals()` and `unregisterCallback()`.
- `[Code-confirmed]` KX11_21 additionally creates `HandlerThread("EcarxPropertyPolling")`, indicating that some signals are supplemented by polling.

## 13. Mock Data Source

- `[Code-confirmed]` Mock is an in-memory property source and does not connect to VHAL or an ECARX Binder.
- `[Code-confirmed]` The unified get/set/subscribe interface remains valid.
- `[Code-confirmed]` Changing a property updates its in-memory value and triggers a callback in the same form as a real source.
- `[Code-confirmed]` The debug UI provides vehicle-speed and engine-speed sliders and an entry point for writing any property.
- `[Reasonable inference]` Because the higher layer still receives the same `CarPropertyValue`, Mock can be used to recover and exercise AutoAudio/AutoDisplay business processes without changing the clients.

## 14. Key End-to-End Flows

### 14.1 Property Read

```text
client getProperty(10001)
  -> x5 caller check
  -> unified property-definition check
  -> current VHAL/ECARX/Mock source read
  -> readTransform
  -> CarPropertyValue(status/type/timestamps/value)
  -> Binder Parcel return
```

### 14.2 Property Subscription

1. The client calls `subscribe(propertyId, callback)`.
2. The service stores the callback Binder by propertyId.
3. The first subscription triggers a subscription to the corresponding underlying signal or adds it to the filter.
4. VHAL/ECARX/Mock produces a change.
5. The mapping converts the underlying value into a unified value.
6. The unified repository updates the timestamp and state.
7. `onPropertyChanged()` is called on every callback for that propertyId.
8. When the final callback unsubscribes, the corresponding lower-level subscription can be removed.

### 14.3 Property Write

1. The client constructs `CarPropertyValue`, setting propertyId, valueType, and the corresponding value field.
2. The service checks that the property exists, is not READ, and has a matching type.
3. `writeTransform` converts it into the lower-level format.
4. VHAL `set`, ECARX `setProperty`, or an in-memory Mock update is performed.
5. Failure returns false.
6. On success, a READ_WRITE property is read again from the lower layer and distributed, avoiding substitution of the requested value for the vehicle's final state.

### 14.4 AutoAudio Events

1. AutoAudio explicitly starts/binds CarPropertyService.
2. It calls `getProperty()` to initialize current speed, gear, doors, and other states.
3. It calls `subscribe()` for 21 propertyId values.
4. AutoService normalizes underlying signalId values to fixed propertyId values according to the vehicle profile.
5. AutoAudio does not need to understand vehicle-model signal differences and directly converts callbacks into alert-sound and push-to-talk button behavior.

## 15. Public HTTP Protocol

### 15.1 Base Address and Authentication

`[Code-confirmed]` Base URL: `https://automanager.simplerbit.com`.

| Field | Calculation/value |
|---|---|
| `hardware_id_hash` | `SHA-256(hardwareId)` |
| `product_signature` | Product-signature string |
| `platform` | `android` |
| `timestamp` | Current millisecond timestamp |
| `requestSignature` | `SHA-256(hardware_id_hash + ":" + product_signature + ":" + timestamp)` |
| `channel` | `stable` for update queries |

Core implementation: `defpackage.j5`.

### 15.2 Endpoints

| Method | Path | Function |
|---|---|---|
| POST | `/api/client/licenses/status` | Query license state |
| POST | `/api/client/codes/redeem` | Redeem a code, adding `redeem_code` |
| GET | `/api/client/releases/latest?...&t={time}` | Query the latest version on the stable channel |
| GET | `/api/client/releases/{id}/download?...` | Download the update APK |

- `[Code-confirmed]` The returned structure reads `ok`, `data`, and `error.message`.
- `[Code-confirmed]` A release reads at least `id` and `version` and generates a download URL.
- `[Code-confirmed]` The update file is passed to the system installation flow through FileProvider.
- `[Code-confirmed]` No WebSocket, MQTT, or custom persistent-connection protocol was found.

### 15.3 License Behavior in the Current Build

- `[Code-confirmed]` The currently visible implementations of `defpackage.j5.h()` and `j5.j()` always return true.
- `[Code-confirmed]` Fixed hardware ID: `SERVICE-DE434BF85819ED0F`.
- `[Code-confirmed]` Fixed product signature: `C9:F1:BC:D6:84:DC:63:12:D6:6B:61:5D:B8:BA:2F:68:52:93:8A:29`.
- `[Reasonable inference]` The Application, BootReceiver, and Service license branches in the current sample operate as if their conditions are satisfied.
- This section describes only the actual code branches and does not assess them.

## 16. Local Configuration and Files

### 16.1 SharedPreferences

| File | Main keys/purpose |
|---|---|
| `source_selection` | `data_source`, `vehicle_profile` |
| `autoservice_license` | License state including activation, heartbeat, offline grace period, expiration, device number, and product signature |
| `car_property_service_runtime` | `running`, `start_time`, `last_heartbeat` |
| `autoservice_status_overlay` | `enabled`, controls the 1px status dot |
| `autoservice_update` | `pending_apk`, update file pending installation |

### 16.2 Assets

- `car_properties.json`: unified definitions of 73 properties.
- 12 VHAL vehicle-profile mapping JSON files.
- One ECARX vehicle-profile mapping JSON file.
- `classes0.jar`: high-entropy SecShell encapsulated payload, detailed in the next section.
- `meta-data/rsa.pub`, `rsa.sig`, `manifest.mf`: shell/payload package metadata.
- `dexopt/baseline.prof`, `baseline.profm`: Android baseline-profile resources.

`[Code-confirmed]` No business SQLite/Room database was found. Long-term state primarily uses SharedPreferences and asset configuration.

## 17. Dedicated Static Analysis of `assets/classes0.jar`

This section strictly follows the no-execution requirement. No APK, DEX, JAR, or ELF code was loaded or executed during the analysis. Only ELF/ZIP structure parsing, Rizin disassembly, and custom byte-transformation scripts were used to recover static data.

### 17.1 File Facts

| Item | Value |
|---|---|
| Path | `apktool/assets/classes0.jar` |
| Size | 440,825 bytes |
| SHA-256 | `806214333CEE1538CEC8103A4CB81C1DEF583A58278DBE90454930B3929D1A6E` |
| First 4 bytes | `07 63 29 F3` |
| Little-endian UInt32 | `0xF3296307` |
| Start of first 64 bytes | `07-63-29-F3-7B-68-6A-DB-...` |
| Shannon entropy | 7.9980 bits/byte |
| Byte-value coverage | All values 0-255 occur |

- `[Code-confirmed]` The file header is not ZIP/JAR `PK`.
- `[Code-confirmed]` The file header is not Android DEX `dex\n`.
- `[Code-confirmed]` The file header is not ELF.
- `[Code-confirmed]` When JADX opens it as a JAR, it reports `zip END header not found`.
- `[Code-confirmed]` Almost all printable strings are short random fragments; there is no ordinary class/JAR directory structure.

The original file is not an ordinary damaged JAR, but a SecShell payload produced by segmented transformation of a ZIP/JAR.

### 17.2 Static x86 Native Unpacking

Original x86 library:

| Item | Value |
|---|---|
| Path | `apktool/lib/armeabi-v7a/libSecShell-x86.so` |
| Size | 1,374,293 bytes |
| SHA-256 | `0CC8218164F5C391ABCD073804A2B94D3BCEBE20A8E03F346F73699F0EFDB751` |

`[Code-confirmed]` The self-unpacking entry point is at VA `0x165A80`; its stub is at VA `0x165A70`, file offset `0x14EA70`. Its main recovery function at `0x165E47` reads `0x804B0` bytes from file offset `0xEB78`, decompresses them with NRV2D-8 to `0x135E58` bytes, and covers `0xEB78..0x1449CF`; its end exactly matches the first `PT_LOAD`.

`[Code-confirmed]` After unpacking, 153 custom relocations must also be applied. The target table is at VA `0x14F5A8`, the value table at VA `0x14F80C`, and every entry executes `*(imageBase + target[i]) = value[i]`.

Reproducible scripts and outputs:

| File | Size | SHA-256 |
|---|---:|---|
| `recovered/static_unpack.js` | 4,997 bytes | Script source file |
| `libSecShell-x86.main.unpacked.bin` | 1,269,336 bytes | `A7D4D63716FFE730E1500DB0DAD7C6C3E9EC5F74BB8431744C51418F74E532DF` |
| `libSecShell-x86.statically-unpacked.so` | 1,374,293 bytes | `135E6D5399A3C6492C943C36F47FB1B4D0558C570F678054792503409417E747` |

`[Code-confirmed]` The following key symbols can be resolved from the recovered ELF:

| Symbol | VA | Static role |
|---|---:|---|
| `JNI_OnLoad` | `0x3B920` | Main Native entry point and subsequent registration/loading dispatcher |
| `find_dexfile` | `0x4E620` | DEX lookup chain |
| `loadDgg` | `0x55F30` | SecShell DGG/DEX loading chain |
| `doRegistern2` | `0x577A0` | Dynamic JNI registration |
| `add_assets` | `0x66340` | Adds an Asset to loading resources |
| `setup_zipres` | `0x665C0` | ZIP resource setup |
| `p85F9...` | `0x88E20` | JAR-buffer processing dispatcher |
| `p4B0...` | `0x41C00` | Separate dual-mode RC4/SM4 function; current call uses a fixed invalid mode and is effectively a no-op |

A separate static unpacking path also completed recovery of the ARM library: compressed input offset `0x1022C`, compressed length `0x634F7`, decompressed length `0x8F5BF`, with output replacing `0x1022C..0x9F7EA`. The recovered `forensics/classes0-static-recovery/libSecShell-arm.static-unpacked.so` is 698,405 bytes and has SHA-256 `1AE8EF1762CA4CAF102BB2A75FED9FCC692BBE49335967B808E4EFE2166F5250`.

### 17.3 `gConfig` Configuration Read Chain

`[Code-confirmed]` The recovered x86 library exports a 4,096-byte `gConfig` at VA `0x1470C0`, file offset `0x1460C0`. `JNI_OnLoad` initializes the reader through `0x87A80`; callback `0x11B20` copies the requested number of bytes, XORs every result byte with `0xAC`, and advances the cursor.

Four reader types were identified statically:

| Address | Function |
|---:|---|
| `0x88620` | Decode MessagePack string/binary value |
| `0x88700` | Decode Boolean value |
| `0x888B0` | Decode integer |
| `0x87AA0` | Bypass MessagePack and read a caller-specified number of raw bytes |

After the same `XOR 0xAC` is applied to the entire `gConfig`, 22 consecutive objects can be parsed from `0x00..0x43`, including the following confirmed fields:

| Offset | Type/value | Meaning |
|---:|---|---|
| `0x00` | `true` | First mode field; exact name awaits closure at its consumer |
| `0x01` | bin12 `classes0.jar` | Payload Asset name |
| `0x0F` | uint32 `440825` | Payload length, exactly matching the actual file |
| `0x19` | bin12 `SecShell.dgc` | Shell cache name |
| `0x28` | bin11 `classes.dgh` | Payload cache name |

Starting at `0x44`, the data cannot be parsed as continuous MessagePack: the next raw-read branch consumes fixed-length slices through `0x87AA0`. The remaining 4,028 bytes have SHA-256 `75C63E45059D6F08B20DB655FC08D99C4A83661EA91ACEDF3BB44E84CF46E320`; this entire segment is not mislabeled here as one key or one object.

Reviewable artifacts are located in `forensics/classes0-static-recovery/`: `recover_gconfig.js`, `gConfig.fields.json`, `gConfig.raw.bin`, `gConfig.xor-ac.bin`, `gConfig.tail-from-0x44.bin`, and `classes0-native-static-trace.md`.

### 17.4 ZIP Tail and Entry Recovery

`[Code-confirmed]` The actual file-offset transformation function for this sample is x86 VA `0x5F460`. Boundary checks are at `0x5F7B2` and `0x5FB78`. When the file offset is at least `0x20000`, `0x5FCB8..0x5FCC3` applies `XOR 0xAC` byte by byte to the buffer. The direct call site is `0x60011`.

Applying this transformation to the current `classes0.jar` starting at file offset `0x20000` directly restores a valid ZIP tail:

| ZIP field | Recovered value |
|---|---|
| EOCD offset | `0x6B9E3` |
| Central-directory offset / size | `0x6B992` / `0x51` |
| Entry count | 1 |
| Entry name | `classes.dex` |
| Compression method | Deflate (8) |
| CRC-32 | `0x46EDC797` |
| Compressed size | `0x6B94D` (440,653 bytes) |
| Uncompressed size | `0x1E40F8` (1,982,712 bytes) |
| Local-file-header offset | `0` |
| Local-header extra-field length | `0` |
| Deflate data start | `0x29` (30-byte local header + 11-byte filename) |

The central directory and EOCD provide complete structural constraints for restoring the prefix: the local header must be `PK 03 04`, the entry name must be `classes.dex`, and the compressed stream's CRC32 must be `46EDC797`.

### 17.5 First 128 KiB of RC4 and Key Initialization

The complete key-initialization chain for the current sample is as follows:

1. `JNI_OnLoad` calls `setup_zipres` (`0x665C0`) at `0x3C25C`.
2. `0x3C412` obtains the static field `PKGNAME` through JNIEnv `+0x240` (signature `Ljava/lang/String;`); `0x3C425` reads the field object, `0x3C442` obtains the UTF-8 string through JNIEnv `+0x2A4`, and `0x3C455` copies it into the 128-byte global buffer `p403E47B6692945B1A4A3FA0625E24068` (VA `0x14FA40`).
3. `0x3C24D..0x3C253` passes that buffer as the third argument to `setup_zipres`; `0x66640..0x66650` then copies it to VA `0x150740`.
4. `0x6666C..0x666B7` constructs the fixed 16-byte seed `66 97 6C E8 6D 46 38 B0 09 5A A5 D7 0F CB 9A A0` on the stack.
5. `0x666F0..0x6672F` limits the participating length to 16; `0x667C8..0x667D0` performs `seed[i] XOR source[i]`.
6. `0x66746..0x66764` writes the result to VA `0x150510..0x15051F`.
7. The KSA of decryption function `0x5F460` reads the key from VA `0x150510` at `0x5F60B`; KSA is at `0x5F608..0x5F644`, and PRGA is at `0x5F660..0x5F6E5`.

The value of `PKGNAME` matches the Manifest package name: `com.simplerbit.autoservice`. XORing its first 16 bytes with the seed produces the effective RC4 key for the current payload:

`05 F8 01 C6 1E 2F 55 C0 65 3F D7 B5 66 BF B4 C1`

The algorithm order is established: `[0, 0x20000)` receives RC4 only; `[0x20000, EOF)` receives only `XOR 0xAC`, with no additional XOR applied to the prefix.

The previously located `p4B0...` is not the effective decryption path for this payload. Its mode is fixed before execution to big-endian `0x81DF6521`, while the function accepts only `0/1`, so the current call definitively follows default/no-op. `mthkey.sig` enters only a separate named-blob RC4 derivation chain and does not override the payload key above. The previous report's conclusion that treated the MD5/Fibonacci candidate values from `p4B0...` as a still-unresolved payload key is therefore withdrawn.

### 17.6 Complete JAR, DEX, and JADX Recovery

After recovery with the two segmented transformations described above, all internal ZIP, Deflate, and DEX checks close successfully:

| Item | Result |
|---|---|
| Recovered JAR | `recovered/classes0_recovered.jar` |
| JAR size / SHA-256 | 440,825 / `B63E4FA84B14A6510548664432F7A662A07E493D845CD091A610F984D9A768C2` |
| ZIP entry | Single `classes.dex`, Deflate (8) |
| Compressed / uncompressed size | 440,653 / 1,982,712 bytes |
| CRC32 | Directory value and actual decompressed result both equal `46EDC797` |
| Recovered DEX | `recovered/classes0_recovered.dex` |
| DEX SHA-256 | `CFD67B8E7B92F65A6AD8F85DDB044DF89DAAA5DD15252553E353D8064C757CDA` |
| DEX magic | `dex\n039\0` |
| DEX SHA-1 | Header value and recalculated value both equal `38A0F5E848B18E533B863C169405183B6156FF01` |
| DEX Adler32 | Header value and recalculated value both equal `B7F88AD3` |
| DEX class_defs | 839 |

The reproducible script is `recovered/recover_classes0.js`; the complete validation record is `classes0_recovery_verification.json`.

JADX 1.5.1 generated two static-source sets:

- `classes0_jadx/`: standard output, 802 Java files;
- `classes0_jadx_badcode/`: audit-aid output with `--show-bad-code`, also 802 Java files; the 65 method placeholders in standard mode are retained here as lower-level statements.

The recovered DEX is not an additional collection of SecShell Java entry points, but the AutoService business payload. It contains `AutoServiceApplication`, `CarPropertyService`, `BootReceiver`, the AIDL contract, vehicle-profile/data-source enumerations, the debug interface, and AndroidX/Material dependencies. Compared with the APK's visible DEX, both have 839 `class_defs`; all 802 relative Java paths from the recovered output exist in the visible DEX output, while the visible output contains only one additional path, `kotlin/coroutines/jvm/internal/DebugProbesKt.java`. The two DEX files have different string/method-table counts and SHA-256 values and therefore are not byte-for-byte copies, but no additional business-class path present only in the recovered payload was found.

### 17.7 Native Loading and Registration Closure

`[Code-confirmed]` The following call sequence has now been recovered:

1. `JNI_OnLoad` initializes the `gConfig` reader and obtains fields including `classes0.jar`, its length, `SecShell.dgc`, and `classes.dgh`.
2. At `0x3C25C`, `JNI_OnLoad` calls `setup_zipres(char*, char*, char*, int)` to establish the ZIP-resource context.
3. `open_zip_infile` has explicit call sites at `0x64C45` and `0x64E38`; the library also exports `open_zip_buffer`, covering both file and memory ZIP input.
4. A subsequent control block consecutively calls `loadDgg` and `doRegistern2` at `0x3FB52` and `0x3FB5A`.
5. `doRegistern2` recovers class name `com/SecShell/SecShell/H1` and Native method `replace`; it calls `FindClass` through JNIEnv function table `+0x18` and `RegisterNatives` through `+0x35C`, with a registration count of 1.
6. `find_dexfile`, ART 5.0/9.0 `DexFile` construction-compatibility functions, `dvmJarFileOpen_stub`, ClassLoader operations, and memory-permission adjustments together form the ART/Dalvik version-adaptation layer.

This chain mutually validates the recovered JAR/DEX: the `classes0.jar` name, length, offset-based decryption, ZIP context, DGG loading, and JNI takeover belong to the same Native initialization process. The Manifest AW/AP/CP classes remain absent from both the visible and recovered DEX, showing that these early proxy entry points are supplied by the SecShell Native/injection layer rather than directly defined by the business DEX.

## 18. Native and Platform Dependencies

| Dependency | Purpose |
|---|---|
| Android Automotive VHAL 2.0 | Real vehicle-property HIDL get/set/subscribe |
| ECARX Car API | `ecarxcar_service`, `car_signal` property access |
| Android Binder/AIDL | Exposes the unified property service to higher-level software |
| Android WindowManager | 1px service-status overlay dot |
| AndroidX / Material | UI, Lifecycle, FileProvider, Startup |
| SecShell Native | Shell entry points and encapsulated-payload processing |

`[Code-confirmed]` No independently defined vehicle protocol over Bluetooth, USB, serial, or similar transports was found. VHAL and ECARX are both system Binder/HIDL interfaces.

## 19. Relationships With Other Project Software

### 19.1 AutoAudio

`[Code-confirmed]` AutoAudio explicitly queries, starts, and binds this service and carries identical contract classes. It subscribes to 21 properties:

`10000, 10001, 10003, 10007, 10017, 10018, 10019, 10020, 10043, 10044, 10047, 10011, 10029, 10030, 10037, 30011, 10049, 10050, 10051, 10052, 10053`

AutoService is responsible for vehicle-model differences, lower-level signal reads, value conversion, and callbacks.

AutoAudio is responsible for gear/door/turn-signal/charging/sentry alert sounds and the long-press public-address behavior of property 10037.

### 19.2 AutoDisplay

- `[Code-confirmed]` The `CarPropertyService` caller allowlist explicitly contains `com.simplerbit.autodisplay`.
- `[Reasonable inference]` AutoDisplay is another primary AIDL consumer and uses unified properties to drive vehicle-information displays and writable controls.
- `[Cannot be confirmed]` AutoDisplay's complete set of subscribed properties and display flows must be taken from its separate report and cannot be fully recovered from the service allowlist alone.

### 19.3 Actual Reachability of the Current Re-Signed Samples

- `[Code-confirmed]` The current AutoService and AutoAudio sample certificate SHA-1 values are both `70:92:...:A2:B1`.
- `[Code-confirmed]` This value is not in the visible service allowlist.
- `[Code-confirmed]` Under ordinary PackageManager signature results, external AIDL methods reject this AutoAudio sample.
- `[Code-confirmed]` The recovered `classes0.jar/classes.dex` adds no new signature-query proxy class path; its `CarPropertyService` uses the same allowlist logic as the visible implementation.
- `[Cannot be confirmed]` Whether the SecShell Native/early-injection layer rewrites PackageManager signature-query results at actual runtime.
- Therefore, both "the protocol relationship is confirmed" and "actual reachability of the current re-signed samples cannot be confirmed" must be retained; they must not be conflated into one conclusion.

## 20. Recovered Implementation Boundaries

### 20.1 Confirmed

- The relationship between SecShell Manifest entry points and the visible business Application/Service.
- Three data-source types, 13 vehicle profiles, and the coverage count of each mapping file.
- 73 unified vehicle properties, their types, states, and access modes.
- The AIDL descriptor, five service transactions, one callback transaction, and Parcelable field order.
- VHAL 2.0 get/set/subscribe, two-second read wait, and timestamp conversion.
- ECARX `ecarxcar_service -> car_signal` connection, property read/write, and SignalFilter subscription.
- The mapping/default/expression conversion protocol.
- Debug-interface functions for the service, properties, Mock, lower-level probing, licensing, and updates.
- Public license, redemption, and update endpoints and the request-signature algorithm.
- NRV2D-8 self-unpacking of the SecShell x86 library, 153 custom relocations, and key Native loading symbols.
- The complete segmented algorithm for `classes0.jar`: RC4 for the first `0x20000` bytes, then bytewise `XOR 0xAC`.
- The `setup_zipres` seed, package-name XOR key initialization, and final RC4 key `05F801C61E2F55C0653FD7B566BFB4C1`.
- Fully recovered JAR and DEX, ZIP CRC32, DEX SHA-1/Adler32, and both standard and auxiliary JADX source sets.
- The recovered DEX's 839 class_defs and 802 Java outputs, and their class-path relationship with the visible DEX.

### 20.2 Cannot Be Confirmed

- Whether every target head unit actually registers the VHAL 2.0 or ECARX service and the corresponding system access policy.
- Real-time values and side effects of every lower-level signalId on each vehicle profile's actual firmware.
- Current server responses for license, redemption, and updates.
- SecShell's effect on signature recognition and cross-package Binder reachability for the currently re-signed samples.
- Property timing, callback frequency, and vehicle execution results; these require a dynamic environment, which this report did not run.

## 21. Core Conclusions

`[Code-confirmed]` AutoService is the vehicle-data middleware between the in-vehicle applications in this project. Its 73 unified properties and stable AIDL contract isolate vehicle-model differences, allowing clients such as AutoAudio and AutoDisplay to understand only fixed propertyId values without directly handling VHAL or ECARX signalId values.

`[Code-confirmed]` The software's key implementation consists of four parts: vehicle-profile JSON mappings with lightweight value transformations; three data-source types, VHAL/ECARX/Mock; a unified property cache and subscription dispatcher; and an exported Binder service with calling-package/certificate matching conditions. The debug Activity covers source switching, property read/write/subscription, Mock injection, and lower-level probing.

`[Code-confirmed]` SecShell's AW/AP/CP entries in the Manifest, the statically recovered Native loading chain, and the fully recovered `classes0.jar/classes.dex` together prove that a shell-payload recovery stage exists before application startup. The recovered DEX contains 839 class_defs and JADX outputs 802 Java files. Its class paths almost completely overlap with the visible business DEX, and no additional business-class path present only in the payload was found.

`[Cannot be confirmed]` The current re-signed certificate does not match the client allowlist. The recovered DEX adds no signature proxy class, but the runtime influence of the SecShell Native/early-injection layer still cannot be excluded from purely static results. This report can therefore confirm AutoAudio/AutoDisplay relationships at the design and protocol levels, but cannot establish the final Binder connectivity of this APK set on a real head unit from the static samples alone.

## 22. Review of JADX Gap Recovery

Four views of the recovered DEX, default, show-bad-code, simple, and fallback, have been generated, each with 802 Java files. default has 65 method placeholders; show-bad-code and fallback both have 0; simple fails on only one third-party Fragment reflection-helper method, which is covered by the other two views.

Key business methods `o4.<init>(...)`, `u8.e(...)`, and `j5.k(...)` have respectively recovered the vehicle-profile mapping/data-source construction, debug property editor, and version-number comparison logic. Every default placeholder has at least one auditable alternative method body; no business method remains visible only as a placeholder `throw`.

Here, complete recovery means that DEX instructions and control flow can be statically audited. It does not mean that pre-obfuscation class names, variable names, comments, or a directly compilable original project have been restored.
