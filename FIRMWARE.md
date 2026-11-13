# GFX100RF firmware 1.13, static analysis

What the update file gives without a camera. Marks as in the
[README](README.md#1-confidence); **unconfirmed** is a reading not yet checked.

## Extraction

The 1.13 update's main code streams are LZ-compressed (header `0x4000`, v4) at
flash offsets `0x860000` and `0xfa0000`. The X-T5 emulator's `fwpatch.py`
decompresses them. The GFX100RF uses the same processor family as the X-T5.

## Confirmed by direct evidence

| Item | Value |
|---|---|
| USB device controller | DesignWare USB3 (DWC3) at base `0xffc0c000`. Register fingerprint present: DSTS `0xc70c`, DEPCMD0 `0xc80c`, DCTL `0xc704`, GUSB2PHYCFG `0xc200`. The X-T5 main application uses the same base |
| USB descriptor classes | 06/01/01 (still image, PTP) and FF/21/FF (vendor) |
| PTP property arrays | Two u16 arrays in the data section: 286 codes and 61 codes ([`data/gfx100rf_firmware_ptp.txt`](data/gfx100rf_firmware_ptp.txt)) |
| Vendor operations | 137 codes `0x9xxx`, and four null-terminated OperationsSupported lists |
| Unnamed codes | 167 of the 286 are named in neither libgphoto2 nor the SDK (bulk `0xD250`-`0xD2AC`) |
| Setting names | 3942 `UI_SETP_*` strings; no flat code-to-setting table found |

## Measured on the camera since

The 286-code array is exactly the property list the camera advertises in USB
TETHER SHOOTING AUTO (GetDeviceInfo, firmware 1.12), and the operations it
advertises there (20 standard and `900C` `900D` `901D` `9018`) are the second of
the four per-mode lists. measured. See [TETHER.md §8](TETHER.md#8-tether-mode-ptp-read-directly).

## Not confirmed

- That the 61-code array is USB RAW CONV./BACKUP RESTORE's list. It holds the
  RAW conversion codes `D183`-`D187`, the bank contents `D18F`-`D1A8`, battery
  and the change list `D212`, which fits, and the [README](README.md#3-environment)
  counts 61 properties in that mode. Its handler has not been cross-referenced.
- What `D183`-`D187` do on this body. On X-Trans IV/V bodies (fujimod#259,
  libfuji): `900C`/`900D` upload a RAF, `D185` is the conversion profile, `D183`
  = 1 starts a full-size conversion, `D184` IOP code, `D186`/`D187` tether RAW
  condition and compatibility codes. community

## Setting names

fujimod's `catalog/firmware/gfx100rf/1.13/ui-setting-names` lists the body's
3866 internal settings by name and number, with no SDK or PTP codes.
[`data/gfx100rf_firmware_names.tsv`](data/gfx100rf_firmware_names.tsv) matches
SDK codes to them by name only; these are guesses.

Numbers 2905 to 2933 are a PC block (priority, liveview exposure and white
balance, card writing, standby, function lock password, backup, sensitivity gain
low and high), which the undocumented and unrouted SDK calls line up with.
`CAMERA_OPE_LOCK` 3074 and C2PA settings exist in 1.13. fujimod's FF80 service
protocol has no GFX100RF body definition.
