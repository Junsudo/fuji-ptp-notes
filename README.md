# Fujifilm GFX100RF over USB: custom setting slots

Notes from working out how to write film recipes to a GFX100RF's custom setting
slots (C1–C7) over a USB cable, from a Mac and from an iPhone.

Everything marked **measured** was established on a real GFX100RF by writing a
value, reading it back, and checking the camera's own menu. Everything marked
**community** comes from other people's reverse engineering and is credited.
Where the two disagree, both readings are given.

## The short version

A custom slot is not the live camera state. It has its own mirror of the picture
settings at `0xD18E`–`0xD1A5`, reached by parking a cursor (`0xD18C`) on the slot
first. Writing the live registers while a slot is selected does not edit that
slot.

Which registers a host can reach depends on the camera's connection mode, and on
whether the host is a Mac or a phone. That is the part that cost the most time.

## Connection mode decides what exists

`CONNECTION MODE` in the camera's menu changes the advertised property list
completely:

| Mode | Properties advertised | Slot block advertised |
|---|---|---|
| USB CARD READER | 5 | no |
| USB TETHER SHOOTING | 286 | **no** |
| USB RAW CONV. | 61 | **yes** |

In tethered shooting the slot registers exist and answer, but the camera does
not list them. In RAW conversion mode it does.

## macOS and iOS do not behave the same

Both use ImageCaptureCore and `requestSendPTPCommand`. They differ on
unadvertised properties:

- **macOS** forwards them regardless. Slot writes work in tethered mode.
- **iOS** returns `0x200A` (DevicePropNotSupported) for anything the camera has
  not advertised. Slot writes fail in tethered mode and work in RAW CONV.

Evidence that this is about the advertised list rather than the slot block
specifically: `0xD18D` (slot name) is advertised in tethered mode and writes
successfully from an iPhone, while `0xD192` in the same block and same session
returns `0x200A`. Whether the refusal is generated on the phone or by the camera
was not determined — that would need a USB capture between phone and body.

So, from a phone:

- **Writing a stored slot** → set the camera to `USB RAW CONV.`
- **Writing the live settings** → set the camera to `USB TETHER SHOOTING`

## The slot block

Cursor first: write the slot number (1–7) to `0xD18C`, then read or write the
block. The value written to the cursor persists across sessions and cable
reconnects, and the register then reports that value rather than the slot the
body is actually on — so once written, it is no longer a way to find out where
the camera is. Changing the C slot on the body resets it.

| Code | Meaning | Encoding | Basis |
|---|---|---|---|
| `0xD18C` | Slot cursor (C1–C7) | 1–7 | measured |
| `0xD18D` | Slot name | string, ~10 chars | measured |
| `0xD18E` | Image size | 7 = L 3:2 observed | community |
| `0xD18F` | Image quality | | community |
| `0xD190` | Dynamic range | 100 / 200 / 400 | measured |
| `0xD191` | D-Range priority | must be 0 for tone curves | measured |
| `0xD192` | Film simulation | | measured |
| `0xD193` | Mono warm–cool | int16, ×10, B&W sims only, rejects 0 | community ×4 |
| `0xD194` | Mono magenta–green | as above | community ×4 |
| `0xD195` | Grain effect | 1 off … 5 strong large | measured |
| `0xD196` | Colour chrome | 1 off, 2 weak, 3 strong | measured |
| `0xD197` | Colour chrome FX blue | 1 / 2 / 3 | community ×2 |
| `0xD198` | Smooth skin | 1 off, 2 weak, 3 strong | community ×2 |
| `0xD199` | White balance | | measured |
| `0xD19A` | WB shift red–cyan | **int16** | measured |
| `0xD19B` | WB shift blue–yellow | **int16** | measured |
| `0xD19C` | Colour temperature | kelvin | measured |
| `0xD19D` | Highlight tone | **int16**, ×10 | measured |
| `0xD19E` | Shadow tone | **int16**, ×10 | measured |
| `0xD19F` | Colour | **int16**, ×10 | measured |
| `0xD1A0` | Sharpness | **int16**, ×10 | measured |
| `0xD1A1` | Noise reduction | ×10 | measured (disputed) |
| `0xD1A2` | Clarity | **int16**, ×10; disabled in HEIF | measured |
| `0xD1A3` | Long exposure NR | 1 = on; other values unknown | community, weak |
| `0xD1A4` | JPEG / HEIF select | 1 = JPEG, 2 = HEIF | measured (disputed) |
| `0xD1A5` | Unknown | always 7 | — |

### Disputed

Two registers were measured here but are named differently by filmkit, which
tested on an X100VI only:

- `0xD1A1` — measured as noise reduction on a GFX100RF; filmkit reports it as a
  high-ISO NR sentinel always reading `0x8000` and not stored in presets.
- `0xD1A4` — measured as JPEG/HEIF select on a GFX100RF; filmkit reports colour
  space (1 = sRGB).

Different bodies may genuinely differ. Neither is written by this project.

### Still unknown

`0xD191` reads 0 on every preset filmkit scanned, and `0xD1A5` reads 7. Neither
has been identified by anyone.

## Signed properties

Sign is not guessable and getting it wrong turns −20 into 65516. These are
INT16 rather than UINT16:

`0x5015` sharpness, `0xD008` colour, `0xD00B`/`0xD00C` WB shift,
`0xD031`/`0xD104` monochromatic colour, `0xD032` clarity, `0xD043` flash
compensation, `0xD320`/`0xD321` tone curves, and the slot equivalents
`0xD193`, `0xD194`, `0xD19A`, `0xD19B`, `0xD19D`, `0xD19E`, `0xD19F`, `0xD1A0`,
`0xD1A2`.

Note that `0xD043` is INT16 while every other property in the flash block is
UINT16, which is what makes a lookup table necessary rather than a rule.

## Order matters

- **Film simulation first.** Changing it resets tone and colour on the body, so
  anything written before it is silently undone.
- **D-Range priority before the tone curves.** While it is on, the camera
  rejects highlight and shadow outright. A recipe that shapes tone has to turn
  it off first even when it says nothing about it.
- **White balance mode before kelvin.** `0xD017` only means something once the
  mode is colour temperature.

## Settings the camera refuses rather than clamps

- **Clarity in HEIF.** Every value is refused with `0x201C`. Switching the image
  format to JPEG first makes the same write take.
- **Monochromatic colour on a colour simulation.** Refused, and a zero is
  refused even on a black and white simulation.

## What cannot be done over the cable

- **Changing which slot the camera is using.** `0xD18C` moves the editing
  cursor, not the body's active slot.
- **ISO and exposure compensation** sit behind physical dials.
- **Reading which slot the body is on**, once anything has been written to
  `0xD18C`.

## Sources

Community entries above come from:

- [eggricesoy/filmkit](https://github.com/eggricesoy/filmkit) — WebUSB preset
  editor. Its property map came from Wireshark captures of Fujifilm's own X RAW
  Studio on an X100VI, and it agrees with every register measured here.
- [gosku/Filmcase](https://github.com/gosku/Filmcase) — Python recipe pusher,
  tested on an X-S10, named independently after the official SDK's function
  names.
- [KyleOndy/dotfiles](https://github.com/KyleOndy/dotfiles) `nix/pkgs/helios` —
  includes a description of the backup blob format, in which the long exposure
  NR byte is inverted relative to the PTP property.
- p5k369/grawji — Python, same mono-axis gating.

Two places worth knowing do **not** document this block:

- **libgphoto2** has never named anything between `0xD188` and `0xD1FF`; a scan
  of 669 historical revisions of `camlibs/ptp2/ptp.h` found no such define in
  any of them. Its X100VI capture lists the whole block as unknown.
- **Fujifilm's own Camera Control SDK** contains no PTP property codes at all —
  it uses its own API numbering and does not expose per-slot registers.
