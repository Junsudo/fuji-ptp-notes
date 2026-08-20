# Fujifilm custom setting slots over USB PTP

Writing film recipes into a camera's custom setting slots (C1–C7) over a cable,
and reading them back.

Body: GFX100RF · Hosts: macOS 26, iOS 26 (ImageCaptureCore) · Notes
current as of 2026-08

## Contents

1. [Confidence](#1-confidence)
2. [Model](#2-model)
3. [Environment](#3-environment)
4. [Operations](#4-operations)
5. [Register reference](#5-register-reference)
6. [Value encodings](#6-value-encodings)
7. [Write constraints](#7-write-constraints)
8. [Failure reference](#8-failure-reference)
9. [Limits](#9-limits)
10. [Provenance](#10-provenance)

---

## 1. Confidence

| Mark | Means |
|---|---|
| measured | Verified on a GFX100RF |
| community | From other projects, credited in §10; `×4` is the number agreeing |
| weak | Single source |
| disputed | Sources disagree; both readings given |

---

## 2. Model

### Two separate sets of settings

The camera holds the picture settings twice over:

- Live settings — what the camera is set to right now. One set.
- Stored slots — seven independent copies, C1 through C7, each a full mirror
  of the same settings.

They are reached through different property codes. Writing a live register
while the body is showing C4 does not edit C4.

### The cursor

Stored slots are not addressed directly. A cursor register (`0xD18C`) selects
which slot the block at `0xD18E`–`0xD1A5` refers to:

```
write 0xD18C = 4        # point at C4
write 0xD192 = 17       # C4's film simulation is now Classic Neg.
write 0xD18C = 6        # point at C6
read  0xD192            # C6's film simulation, not C4's
```

The cursor is stateful and sticky: its value survives closing the session
and unplugging the cable, and once written the register reports that value
rather than the slot the body is actually on. Changing the C slot on the body
resets it. measured

### The other path to a slot

With `CUSTOM SETTING AUTO UPDATE` on, the camera files the live settings into
whichever slot it is showing. So a slot can be written two ways:

| Path | Reaches | Needs |
|---|---|---|
| Slot block | Any slot, regardless of what the body shows | `USB RAW CONV.` on iOS |
| Live + auto update | Only the slot the body is showing | `USB TETHER SHOOTING`, auto update on |

---

## 3. Environment

### Connection mode changes what exists

`CONNECTION MODE` in the camera menu changes the advertised property list
completely. measured

| Mode | Properties advertised | Live settings | Slot block |
|---|---|---|---|
| USB CARD READER | 5 | no | no |
| USB TETHER SHOOTING | 286 | yes | present but not advertised |
| USB RAW CONV. | 61 | no | yes |

### Host changes what gets through

macOS and iOS both use ImageCaptureCore and `requestSendPTPCommand`, and differ
on properties the camera never advertised. measured

| | Unadvertised property |
|---|---|
| macOS | forwarded to the camera |
| iOS | refused with `0x200A` |

This applies to the advertised list as a whole, not to the slot block
specifically: `0xD18D` is advertised in tethered mode and writes fine from an
iPhone, while `0xD192` in the same block returns `0x200A`.

### Which mode to use

| Host | Goal | Mode |
|---|---|---|
| iOS | write a stored slot | `USB RAW CONV.` |
| iOS | write live settings | `USB TETHER SHOOTING` |
| macOS | either | `USB TETHER SHOOTING` |

---

## 4. Operations

### Read a slot

```
set 0xD18C = <1-7>
for code in 0xD18E ... 0xD1A5:
    get code                     # decode per §6
```

### Write a slot

```
set 0xD18C = <1-7>
set 0xD192 = <film simulation>   # first, see §7
set 0xD191 = 0                   # if writing tone curves
... remaining registers
set 0xD18D = <name>              # last; goes to the cursor's slot
```

### Write the live settings

Same order, using the live codes in §5.2. The camera stores them into the slot
it is showing if `CUSTOM SETTING AUTO UPDATE` is on.

### Name a slot

`0xD18D` is a PTP string and writes to wherever the cursor points. Naming
without having just set the cursor puts the name on an unpredictable slot,
because of the stickiness described in §2. fujicli declares a limit of 25
characters; that is unmeasured here, and the camera truncates silently, so read
the name back to see what it kept.

---

## 5. Register reference

### 5.1 Slot block

| Code | Setting | Basis |
|---|---|---|
| `0xD16E` | USB mode — `6` is RAW conversion | community |
| `0xD18C` | Slot cursor (C1–C7) | measured |
| `0xD18D` | Slot name | measured |
| `0xD18E` | Image size | community |
| `0xD18F` | Image quality | community |
| `0xD190` | Dynamic range | measured |
| `0xD191` | D-Range priority | measured + community |
| `0xD192` | Film simulation | measured |
| `0xD193` | Monochromatic colour, warm–cool | community ×4 |
| `0xD194` | Monochromatic colour, magenta–green | community ×4 |
| `0xD195` | Grain effect | measured |
| `0xD196` | Colour chrome effect | measured |
| `0xD197` | Colour chrome FX blue | community ×2 |
| `0xD198` | Smooth skin effect | community ×2 |
| `0xD199` | White balance | measured |
| `0xD19A` | WB shift, red–cyan | measured |
| `0xD19B` | WB shift, blue–yellow | measured |
| `0xD19C` | Colour temperature | measured |
| `0xD19D` | Highlight tone | measured |
| `0xD19E` | Shadow tone | measured |
| `0xD19F` | Colour | measured |
| `0xD1A0` | Sharpness | measured |
| `0xD1A1` | High ISO noise reduction | measured + community |
| `0xD1A2` | Clarity | measured |
| `0xD1A3` | Lens modulation optimiser | community ×2 |
| `0xD1A4` | Colour space | community ×2, disputed |
| `0xD1A5` | — unidentified, reads 7 | — |

### 5.2 Live equivalents

The same settings on the live path. Note they are not adjacent and not in the
same order, which is why a mapping table is needed rather than an offset.

| Setting | Slot | Live |
|---|---|---|
| Film simulation | `0xD192` | `0xD001` |
| Dynamic range | `0xD190` | `0xD007` |
| D-Range priority | `0xD191` | `0xD02E` |
| Colour | `0xD19F` | `0xD008` |
| White balance | `0xD199` | `0x5005` |
| Colour temperature | `0xD19C` | `0xD017` |
| WB shift red–cyan | `0xD19A` | `0xD00B` |
| WB shift blue–yellow | `0xD19B` | `0xD00C` |
| Highlight tone | `0xD19D` | `0xD320` |
| Shadow tone | `0xD19E` | `0xD321` |
| Sharpness | `0xD1A0` | `0x5015` |
| Clarity | `0xD1A2` | `0xD032` |
| High ISO NR | `0xD1A1` | `0xD01C` |
| Grain effect | `0xD195` | `0xD023` |
| Colour chrome | `0xD196` | `0xD029` |
| Colour chrome FX blue | `0xD197` | `0xD030` |
| Smooth skin | `0xD198` | `0xD189` |
| Mono warm–cool | `0xD193` | `0xD104` |
| Mono magenta–green | `0xD194` | `0xD031` |
| Colour space | `0xD1A4` | `0xD00A` |
| Lens modulation optimiser | `0xD1A3` | — |
| Image format | — | `0xD1B2` |

---

## 6. Value encodings

### 6.1 Scales

| Kind | Registers | Encoding |
|---|---|---|
| Tenths | colour, sharpness, clarity, highlight, shadow, mono axes | menu value × 10; `+2` is `20` |
| Direct | dynamic range | 100 / 200 / 400, or 65535 for auto |
| Enumerated | film simulation, WB, grain, chrome, smooth skin | see 6.2 |
| Non-linear | high ISO noise reduction | see 6.4 |
| String | slot name | PTP string. fujicli declares 25 characters; unmeasured on this body |

Ranges: highlight and shadow tone −2…+4 with half steps allowed (`+1.5` is raw
`15`), colour and sharpness −4…+4, clarity −5…+5, WB shift −9…+9, monochromatic
axes −18…+18. community

### 6.2 Enumerations

| Setting | Values |
|---|---|
| Film simulation | 1 PROVIA · 2 Velvia · 3 ASTIA · 4 Monochrome · 5 Sepia · 6 PRO Neg Hi · 7 PRO Neg Std · 8/9/10 Monochrome +Ye/R/G · 11 Classic Chrome · 12 ACROS · 13/14/15 ACROS +Ye/R/G · 16 ETERNA · 17 Classic Neg · 18 Bleach Bypass · 19 Nostalgic Neg · 20 REALA ACE |
| Grain effect | 1 off · 2 weak small · 3 strong small · 4 weak large · 5 strong large |
| Colour chrome, FX blue, smooth skin | 1 off · 2 weak · 3 strong |
| D-Range priority | 0 off · 1 weak · 2 strong · 32768 auto |
| White balance | 2 auto · 4 daylight · 6 incandescent · 32769/32770/32771 fluorescent 1/2/3 · 32774 shade · 32775 colour temperature · 32800 auto white priority · 32801 auto ambience priority |
| JPEG / HEIF | 1 JPEG · 2 HEIF |

### 6.3 Signedness

Sign is not derivable from the code. Reading unsigned where the camera meant
signed turns −20 into 65516 — a write that worked, reported as a failure.

INT16: `0x5015` · `0xD008` · `0xD00B` · `0xD00C` · `0xD031` · `0xD032` ·
`0xD043` · `0xD104` · `0xD320` · `0xD321` · and the slot twins `0xD193` `0xD194`
`0xD19A` `0xD19B` `0xD19D` `0xD19E` `0xD19F` `0xD1A0` `0xD1A2`

`0xD043` (flash compensation) is INT16 while every other property in the flash
block is UINT16. That is why this has to be a lookup table and cannot be a rule.

### 6.4 High ISO noise reduction

`0xD1A1` looks like its neighbours and is not. Neither linear nor monotonic:

| Menu | Raw | Menu | Raw |
|---|---|---|---|
| +4 | `0x5000` | −1 | `0x3000` |
| +3 | `0x6000` | −2 | `0x4000` |
| +2 | `0x0000` | −3 | `0x7000` |
| +1 | `0x1000` | −4 | `0x8000` |
| 0 | `0x2000` | | |

`+2` is zero and `0` is `0x2000`, so storing a plain menu number here writes the
wrong strength and reports success. community

---

## 7. Write constraints

### Ordering

| Rule | Why |
|---|---|
| Film simulation first | Changing it resets tone and colour on the body, silently undoing anything written before it |
| D-Range priority off before tone curves | While on, highlight and shadow are refused outright |
| WB mode before kelvin | `0xD017` / `0xD19C` mean nothing until the mode is colour temperature |

### Conditional availability

| Setting | Available when |
|---|---|
| Clarity | Image format is JPEG. In HEIF every value is refused |
| Monochromatic axes | Film simulation is Monochrome or ACROS (4, 8, 9, 10, 12, 13, 14, 15). A zero is refused even then |
| Highlight / shadow tone | D-Range priority is off |
| Colour temperature | White balance mode is colour temperature |

---

## 8. Failure reference

| Code | Name | Means here | Do this |
|---|---|---|---|
| `0x200A` | DevicePropNotSupported | The property is not in the camera's advertised list, and the host will not forward it | Change `CONNECTION MODE` — §3 |
| `0x201C` | InvalidDevicePropValue | Value out of range, or the setting is disabled by another setting | Check §7. For clarity, switch the format to JPEG |
| `0x2002` | InvalidObjectHandle | Seen from `GetDevicePropDesc` in RAW CONV mode; `GetDevicePropValue` still works | Read the value, skip the description |
| `0xA001` | Vendor-specific | Seen when probing name length | Shorten the string |

A value that reads back as a large positive number where a negative was written
is not a failure — it is §6.3.

---

## 9. Limits

Not achievable over the cable:

- Changing which slot the camera is using. `0xD18C` moves the editing
  cursor, not the body's selection.
- ISO and exposure compensation. Physical dials.
- Reading which slot the body is on, once anything has been written to
  `0xD18C`.

---

## 10. Provenance

| Project | Body | Contributes |
|---|---|---|
| [eggricesoy/filmkit](https://github.com/eggricesoy/filmkit) | X100VI | `0xD18E` `0xD18F` `0xD193` `0xD194` `0xD197` `0xD198` |
| [karaolidis/fujicli](https://git.karaolidis.com/karaolidis/fujicli) | X-Trans III–V | Names 25 of the 26 registers, `0xD16E` USB mode, `0xD1A3` as lens modulation optimiser, slot name limit of 25 characters |
| [petabyt/fp](https://github.com/petabyt/fp) | X-H1, X-T5 | The profile blob X RAW Studio sends over `0xD185`, with a field list wider than this block |
| [gosku/Filmcase](https://github.com/gosku/Filmcase) | X-S10 | `0xD191` values, `0xD1A1` table, ranges |
| [KyleOndy/dotfiles](https://github.com/KyleOndy/dotfiles) | X-T5 | Long exposure NR inversion |
| p5k369/grawji | — | Monochromatic gating |

Disagreement, and the weight has moved. `0xD1A4` is recorded here as JPEG/HEIF
select, but filmkit and fujicli both call it colour space, and fp's parse of the
profile X RAW Studio sends carries `FileType` and `ColorSpace` as separate
fields — so the two are different registers and only one of them can be this.
The reading here came from a slot whose clarity greyed out, which is what HEIF
does, but that is an inference and the note did not record whether the camera
itself was ever seen saying HEIF. Treat the two community sources as the better
evidence. Not written by this project either way.

`0xD1A3` was carried as long exposure NR on filmkit's single weak sighting.
fujicli names it lens modulation optimiser and fp's profile struct has a
`LensModulationOpt` field, so that is what it is.

Do not use `aradotso/trending-skills` — its table for this block is shifted by
several addresses (film simulation at `0xD18E`, white balance at `0xD191`).

Not covered anywhere: libgphoto2 names nothing between `0xD188` and `0xD1FF`,
and Fujifilm's Camera Control SDK carries no PTP property codes at all.

Unidentified: `0xD1A5`, which reads 7 everywhere.
