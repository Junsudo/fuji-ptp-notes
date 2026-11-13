# GFX100RF tethering: the FUJIFILM SDK and PTP

Shooting, liveview and settings over a cable or Wi-Fi, through Fujifilm's
Camera Control SDK (XSDK) and raw PTP.

Body: GFX100RF, firmware 1.12 · SDK 1.3.4 (macOS binaries) · Hosts: macOS 27,
iOS 27.3 (iPhone Air) · Notes current as of 2026-11-13

Marks as in the [README](README.md#1-confidence): **measured** on a GFX100RF,
**community**, **weak**. **unconfirmed** is a reading not yet checked.

## Contents

1. [Connection](#1-connection)
2. [SDK setup](#2-sdk-setup)
3. [SDK on iOS](#3-sdk-on-ios)
4. [Session behaviour](#4-session-behaviour)
5. [Liveview and shooting](#5-liveview-and-shooting)
6. [Exposure, focus and the AF grid](#6-exposure-focus-and-the-af-grid)
7. [Movie](#7-movie)
8. [Tether-mode PTP, read directly](#8-tether-mode-ptp-read-directly)
9. [The SDK's own tables](#9-the-sdks-own-tables)
10. [Wi-Fi](#10-wi-fi)
11. [Not available on this body](#11-not-available-on-this-body)
12. [Data files](#12-data-files)

---

## 1. Connection

| Item | Value | |
|---|---|---|
| USB | `04CB:030F`, class 06/01/01, PTP. ImageCaptureCore opens it on macOS and iOS | measured |
| Connection mode for the SDK | USB TETHER SHOOTING AUTO | measured |
| Transaction id | Fuji wants OpenSession numbered 1, not 0 | measured |

### Sleep over USB

With AUTO POWER OFF on, the body leaves the USB bus entirely when it sleeps (a
15 s setting: gone within 20 s, back only after a button press). Asleep it shows
no USB device at all, not even by vendor id. Pulling and replugging the cable
does not wake it, so a hub with per-port power would not help. Nothing over USB
wakes it. measured

A host can only keep it awake: set AUTO POWER OFF to Off while connected
(`0x411B` = 3). Over USB, leave it Off after the session too; putting the body's
value back lets it sleep off the bus. measured

## 2. SDK setup

| Item | Value | |
|---|---|---|
| Model table | `XSDK.DAT` must be in the main bundle's resources (`Contents/Resources` on macOS, the bundle root on iOS). Without it `OpenEx` falls back to the generic module FF0000 (51 calls) and most settings answer `API_NOTFOUND` 0x1013. The GFX100RF is FF0021 | measured |
| Command-line tools | Need `XSDK.DAT` beside the binary | measured |
| Variadic calls | `XSDK_SetProp`, `GetProp`, `CapProp` must be called from C on Apple arm64; Swift's variadic import passes them wrongly | measured |
| Threads | Calls belong on a background thread with the main run loop free. `FTLPTP.dylib` goes through ImageCaptureCore, whose device list fills on the main run loop: a blocked main thread detects nothing, and blocking it during a session (even in a quit handler) turns every call into `COMMUNICATION` 0x2001 | measured |
| Locking | The SDK is not safe for two calls at once. Every call, including small ones such as `CapZoomPos` inside an aperture list, goes under one lock | measured |
| Buffers | Getters write further than their documented size. `SDK_GetCustomName` memsets tens of kilobytes into a 256-byte buffer (found with Guard Malloc; it corrupted the heap). Give every getter that fills an array, string or struct 64 KB, then copy out what is documented | measured |
| Raw PTP through the SDK | `XSDK_GetPropertyValue(code, type)` works once given the PTP data type itself: `0xD18D` with `0xFFFF` reads the bank name, `0xD36B` the battery string. The type from `GetPropertyCaps` (0 for these) makes it fail. The SDK returns success for any code, so only a nonzero type means present | measured |

## 3. SDK on iOS

No iOS SDK is published, but the macOS binaries link nothing macOS-only
(CoreFoundation, Foundation, ImageCaptureCore, libc++, libobjc). measured

1. Thin each binary to arm64 (`lipo`), retag the platform as iOS (`vtool
   -set-build-version`), point the framework paths at iOS's flat frameworks
   (`install_name_tool`).
2. Sign each binary as a plain Mach-O. Signed inside its macOS bundle, the
   signature binds `Info.plist` and iOS answers code signature invalid.
3. Lay the model modules out flat: `FF0021API.bundle/FF0021API` and
   `FF0021API.bundle/Info.plist`. XAPI reaches the model through CFBundle, and
   iOS's CFBundle finds no executable in a `Contents/MacOS` bundle. With the
   macOS layout, connecting and simple reads work but every `SetProp`
   (liveview included) answers `NO_MODEL_MODULE` 0x1012. measured
4. Keep `XAPI.bundle` in the macOS layout: it finds the model modules at
   `../../../FFxxxxAPI.bundle` from its own executable.

On the iPhone Air (iOS 27.3, USB-C to the camera): Detect 1, OpenEx ok (mode 5),
GetSensitivity ISO 160, liveview frames at about 20 fps. Whether those frames
reach the screen is not yet checked. measured

ImageCaptureCore on iOS:

| Item | Value | |
|---|---|---|
| Authorization | The app needs control authorization (`requestControlAuthorization`). iOS titles the prompt "Camera access" and shows `NSCameraUsageDescription` | measured |
| Device mask | Leave `browsedDeviceTypeMask` at the iOS default 257. Runs that set `0xf01` saw no camera | weak |
| Unadvertised properties | iOS refuses them with `0x200A`; macOS forwards them | measured (README §3) |

A device that iOS never puts in its USB registry stays invisible to both
ImageCaptureCore and the SDK; this happened between runs with identical code.
Clearing and re-granting ImageCaptureCore authorization coincided with it
coming back once. weak

## 4. Session behaviour

| Item | Value | |
|---|---|---|
| `FORCEMODE_BUSY` 0x1016 | `OpenEx` answers it for a moment while the dial is being changed. After a test run the body refused every write, priority included, with it until `SetForceMode` cleared it at once | measured |
| Priority | Camera priority (1) and PC priority (2) each keep their own value of some settings: DRIVE MODE set to CH in 1 still reads Single in 2 and comes back as CH in 1; focus mode read MF in 1 and AF-S in 2. PC priority offers DRIVE Single only | measured |
| Busy detail bits | `XAPI.H` 0x1 S1 to 0x200. The GFX100RF answers 0x80 (RECORDING in the header) for stills liveview, never 0x100; 0x82 with AE-L on; 0x20 for a moment after a shot | measured |
| Lock buttons | With LOCK BUTTON on Switch, AE-L stays on after release and a second press clears it. AF-L clears on release | measured |
| Side effects | Turning subject detection on turns face detection and eye AF off, and they stay off when it goes back | measured |
| Camera clock | `GetDateTime` reads seconds as 0 | measured |
| Media record | `SetMediaRecord` 1 (computer and card) and 4 (computer only) both deliver to the host | measured |

## 5. Liveview and shooting

| Item | Value | |
|---|---|---|
| Liveview rate | Tops out at 29 fps whatever its size or quality. Median 22 fps with polling, 27 to 32 in quiet stretches | measured |
| Call cost with liveview on | Value reads 6 to 10 ms, focus mode 44, focus position 24, `CapShutterSpeed` 42, `CapAperture` 130 | measured |
| Transfer | 198 MB RAF in 0.6 s; preview 1.6 MB | measured |
| Receiving | One thread reads the buffer (`ReadImageInfo`, then `ReadImage`) for the whole session; liveview frames and shots come through the same buffer. Release and settings go on another thread; the SDK's lock keeps them apart | measured |
| Release | `ReleaseEx` 0x10000 \| 0x20000 \| 0x2 \| 0x1 | measured |
| Bulb with AF | In AF-S with focus priority a failed AF leaves the shutter shut and `ReleaseEx` returns AF status 0. Bulb frames want MF | measured |
| Aspect ratio | Taken from the body's dial in camera priority; PC priority freezes whatever was there. Liveview, the JPEG or HEIF and RAF tag 0x115 all follow it. With RAW alone everything stays 4:3 and image size reads a fixed L 4:3 | measured |

## 6. Exposure, focus and the AF grid

| Item | Value | |
|---|---|---|
| Exposure mode against writes | Shutter in A or P and aperture in S or P answer `COMBINATION` 0x2003. Each mode keeps its own shutter and aperture | measured |
| Focus position | Reported range 545 to 1343 with over-travel 60, yet the lens runs 485 to 1452 and clamps by itself. Do not clamp on the host | measured |
| Focus indicator | `GetFocusLimiterIndicator` is linear in 1/distance: metres ≈ 0.2 / (1 − scale). DOF width 21/1024 at f/4, 72 at f/16 | measured |
| `SDK_FocusPosToDistance` | Exported by the model module, in no API table, no camera handle. Answers only zoom positions 53/54 (tables for other lenses) | measured |
| AF frame | `CapFocusArea` 0x2235: h −6 to 6, v −4 to 4 at 4:3, narrower under a crop (65:24 is v −2 to 2). Pitch 812 of the sensor's 11648 × 8736 pixels, v positive upwards | measured |
| AF frame size by mode | Single 1 to 6, zone 17 to 20 (3×3, 5×3, 5×5, 7×7), wide 0, tracking 3 | community (SDK manual pp. 238-245, printed as images) |
| ZONE CUSTOM | Three slots, h 1 to 13 and v 1 to 9; read 5×3, 7×3, 7×1 | measured |
| DEPTH-OF-FIELD SCALE | SDK 0x2276 to 0x2278, in no header. `SDK_SetDOFScale` writes PTP `0xD028` (UINT16) unchanged: 1 pixel basis, 2 film format basis. libgphoto2 calls `0xD028` CommandDialMode | measured |
| Star focus | Half-flux diameter in the middle quarter of liveview, fitted to the lowest point of a focus sweep. Measured indoors only, never against a sky | weak |

## 7. Movie

On this body movie is DRIVE MODE 8, offered only in camera priority. PC
priority lists Single alone and refuses 8 with `COMBINATION`; with 8 set, PC
priority is refused too. measured

| Item | Value | |
|---|---|---|
| REC | `ReleaseEx` 0x40000 is not in `CapReleaseEx` and answers `COMBINATION`. A full press (`ReleaseEx` 0x30000 then 0x3) starts recording and the next one stops it | measured |
| 0x20000000 / 0x2000 | Appear only in movie and answer 0x1008 | measured |
| Mic levels, timecode, heat | Read fine | measured |
| Movie exposure calls | `GetMovieShutterSpeed` in camera priority read 429496732000. The rest not yet read or written in movie mode | unconfirmed |

## 8. Tether-mode PTP, read directly

ImageCaptureCore's PTP passthrough reaches the camera on macOS without the SDK.
In USB TETHER SHOOTING AUTO, body firmware 1.12: measured

| Item | Value |
|---|---|
| Operations | 24: 20 standard and `900C` `900D` `901D` `9018` |
| Events | 0 |
| Properties | 286, exactly the firmware's 286-code array ([FIRMWARE.md](FIRMWARE.md)) |
| Storages | 2 |
| Descriptors | 189 full (79 for codes no one names), 89 refused with `0x2002` in both priorities (the `D229`-`D2B3` block), 8 UNDEF. All in [`data/gfx100rf_tether_propdesc.tsv`](data/gfx100rf_tether_propdesc.tsv) |

Through the SDK's raw reads in the same mode, 156 properties answer with a real
type, all inside the 286. `D18C` answers (UINT16 enum 1 to 7, was 7). `D18D`
reads with type `0xFFFF`. `D18F`-`D1A8` and `D212` give no type through the SDK
path. measured. The [README](README.md#3-environment) found the slot block
present but unadvertised in this mode over ImageCaptureCore directly on macOS;
the SDK path does not reach it.

`D212` (CurrentState in libgphoto2) reads but stayed 0 through ISO changes made
from the computer. Whether it records only changes made on the body, or the SDK
drains it, is open.

### From libfuji's notes, not yet checked here

- A `GetDevicePropValue` must follow each `SetDevicePropValue`.
- With a menu or quick menu open, `SetDevicePropValue` answers DeviceBusy.
- DeviceInfo seems to carry a record of which properties changed (a bitmap?).
- DeviceInfo's FunctionalMode is `0x02a8` in that mode.

## 9. The SDK's own tables

The model module answers `XSDK_APICode2APIInfo(code, name, count, shape)`
without a camera. measured

| Item | Count |
|---|---|
| API codes in FF0021 | 681 |
| In no header | 110 (shutter type, CH/CL fps, program shift, ISO auto, custom slot names and writes, flash, camera status and rotation, a second DRIVE MODE at 0x4051) |
| Refused | 64 |
| Module functions with no code | 182: 39 are bodies of exported `XSDK_` functions, 143 have no route at all (photometry level, liveview exposure, BKT, flash modes, MB-D200 and non-CPU lens names, which read like a shared code base) |

Parameter names come from the SDK's own trace strings. All in
[`data/gfx100rf_api_codes.tsv`](data/gfx100rf_api_codes.tsv) and
[`data/gfx100rf_unmapped.tsv`](data/gfx100rf_unmapped.tsv). Settings the owner's
manual describes but the SDK headers do not expose are in
[`data/gfx100rf_undocumented_menu.tsv`](data/gfx100rf_undocumented_menu.tsv).

The SDK manual (pages 231-245 focus area by ratio, 275 focus position
over-travel, 432 frame guide directions, the movie menu pages) prints its
tables as images; `pdftotext` returns only the headings.

`0xD043` is INT16 while every other property in the flash block is UINT16. Sign
has to come from a table, not a rule. measured

## 10. Wi-Fi

| Item | Value | |
|---|---|---|
| Camera SoftAP | `192.168.0.1`, ports 55740 command, 55741 event, 55742 liveview | measured |
| Wireless tethering | A different shape: the computer listens on TCP 51560 and broadcasts on UDP 51562; the camera connects in | measured |
| libfuji's broadcast address | Hardcoded to `192.168.1.255`, which reaches nothing on another subnet. Derive it from the interface | measured |
| Subnet collision | The camera's network is `192.168.0.x`, often the house LAN too; the routing table then sends the camera's address to the router. `IP_BOUND_IF` pins the socket | measured |
| SDK discovery handshake | Lines `NOTIFY * HTTP/1.1`, `DSC: <ip>`, `CAMERANAME: <name>`, any line, `MX: 5`, `SERVICE: PCSS/1.0`, otherwise 403. The address comes from `DSC`, not the peer; the SDK then connects to it on 15740 with no local bind | measured |
| SDK security argument | Undocumented: 3 and 4 are refused, 2 is open | measured |
| Advertised properties, wireless tether | 286 | measured |
| Still capture over XApp (`D207`/`D208`) | ACKed but inert on this body; BLE fires the shutter | measured |

### A Mac as the access point

Internet Sharing no longer works for this on macOS 27
(`com.apple.InternetSharing` is gone). `CWInterface
startHostAPModeWithSSID:...` starts an access point on `ap1` (not `en0`, which
goes inactive); channel 36 works. The SDK only uses interfaces named `en*` and
binds TCP 51560 to that address, so `en0` gets `192.168.78.1/32`, and the host
sends the 51562 broadcast on `ap1` and relays 51560. measured

## 11. Not available on this body

Over USB in tether mode: measured

- `GetMode` / `SetMode` / `CapMode`: UNSUPPORTED 0x1005.
- `SetFocusOperation`: UNSUPPORTED. Held arrows repeat single steps instead.
- `GetChangedProperty`: never returns.
- Card playback APIs: −1. No card browsing.
- Custom AUTO POWER OFF: reads 0 with no range.
- Continuous drive and bracketing from the host: PC priority offers Single only.

## 12. Data files

| File | Holds |
|---|---|
| [`data/gfx100rf_tether_propdesc.tsv`](data/gfx100rf_tether_propdesc.tsv) | All 286 tether-mode property descriptors: type, access, current, factory default, allowed values |
| [`data/gfx100rf_firmware_ptp.txt`](data/gfx100rf_firmware_ptp.txt) | The 286-code and 61-code arrays found in firmware 1.13 |
| [`data/gfx100rf_api_codes.tsv`](data/gfx100rf_api_codes.tsv) | The model module's 681 SDK API codes, shapes and trace parameter names |
| [`data/gfx100rf_unmapped.tsv`](data/gfx100rf_unmapped.tsv) | Model module functions with no API code |
| [`data/gfx100rf_undocumented_menu.tsv`](data/gfx100rf_undocumented_menu.tsv) | Menu items in the manual that the SDK headers leave out |
| [`data/gfx100rf_firmware_names.tsv`](data/gfx100rf_firmware_names.tsv) | SDK codes matched to firmware 1.13 setting names, by name only (guesses) |
