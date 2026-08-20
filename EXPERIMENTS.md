# Experiment log

What has been run against the camera and what it did. The first two tables are
the point of this file: what is known to work, and what is known not to, so that
neither gets attempted again from scratch.

Body: GFX100RF. 2026-08.

## Works

Each of these was run and observed. The conditions are part of the result — the
same operation under different conditions is a different row.

| Do this | Transport | Host | Camera mode | Evidence |
|---|---|---|---|---|
| Write live settings | USB | macOS | USB TETHER SHOOTING | run |
| Write live settings | USB | iOS | USB TETHER SHOOTING | run |
| Write the slot block | USB | macOS | USB TETHER SHOOTING | wrote `0xD192` = 11 to C7, moved the cursor away and back, still there |
| Write the slot block | USB | iOS | USB RAW CONV. | run |
| Write the slot cursor `0xD18C` | USB | either | TETHER or RAW CONV. | run |
| Write the slot name `0xD18D` | USB | either | USB TETHER SHOOTING | run — lands on the cursor's slot, which is rarely the one you meant |

## Does not work

Do not attempt these again. The reason is in the last column; none of them is a
bug that more effort fixes.

| This fails | Transport | Host | Camera mode | Why |
|---|---|---|---|---|
| Writing the slot block | USB | iOS | USB TETHER SHOOTING | `0x200A`. Apple's imaging stack drops properties the camera has not advertised, before they leave the phone. Not fixable in the app — use RAW CONV. |
| Anything useful | USB | either | USB CARD READER | 5 properties advertised |
| Live settings | USB | either | USB RAW CONV. | not advertised in that mode |
| Clarity while the camera is on HEIF | any | either | any | `0x201C` for every value. Switch the format to JPEG first |
| Monochromatic colour on a colour simulation | any | either | any | refused. A zero is refused even on a black and white one |
| Writing zero to the monochromatic axes | any | either | any | refused |
| Changing which slot the body is using | any | either | any | `0xD18C` moves the editing cursor only |
| Reading which slot the body is on, after writing the cursor | any | either | any | the register reports what was written to it, across sessions and reconnects |
| Setting ISO or exposure compensation | any | either | any | physical dials |
| Image transfer over the camera's own access point | Wireless AP | macOS | — | attempted; no transfer ever completed. The phone app it mirrors was abandoned for the cable over instability |

## Not worth searching for again

| Question | Status |
|---|---|
| `0xD1A5` | Unidentified by every project that has looked — filmkit, fujicli and fp all omit or flag it. Only measurement will settle it |
| Names for this block in libgphoto2 | Never existed — 669 historical revisions of `ptp.h` checked |
| PTP property codes in Fujifilm's official SDK | There are none. It uses its own API numbering and exposes no per-slot registers |
| `aradotso/trending-skills` as a reference | Its table is shifted by several addresses. Wrong |
| `0xD1B2`'s values in published sources | Documented nowhere. Measure it |
| Fuji constants in go-mtpfs | Seven, all shooting properties. Nothing in the slot block |

---

## The variables

Four things change the answer, and every result below is a point in this space:

| Variable | Levels |
|---|---|
| Transport | USB · Wireless, camera raises its own access point · Wireless, camera joins a network |
| Host | macOS · iOS |
| Camera CONNECTION MODE | USB CARD READER · USB TETHER SHOOTING · USB RAW CONV. · WIRELESS TETHER SHOOTING FIXED |
| Target | Live settings · Slot cursor `0xD18C` · Slot name `0xD18D` · Slot block `0xD18E`–`0xD1A5` |

CONNECTION MODE is one setting with both USB and wireless entries in it, so the
transport and the mode are not independent: picking a wireless mode replaces
whichever USB mode was selected.

## Findings that changed the approach

1 · The refusal is the host, not the camera. The two slot-block rows above are the same
camera, same mode, same registers, different computer. macOS forwards a property
the camera never advertised; iOS answers `0x200A` without it reaching the
camera. The slot name is the control: `0xD18D` is advertised in that mode and goes
through from the same phone in the same session.

2 · The slot cursor is sticky. `0xD18C` keeps whatever was last written to
it across sessions and cable reconnects, and then reports that instead of the
slot the body is on. Writing a name without having just set the cursor puts it
on an unpredictable slot — observed, not theorised. Changing the C slot on the
body resets it.

3 · Some settings are refused rather than clamped. Clarity returns `0x201C`
for every value while the camera is set to HEIF. Monochromatic colour is refused
on a colour simulation, and a zero is refused even on a black and white one.

4 · Sign is not guessable. Sharpness −20 was written correctly and read back
as 65516. The write was never the problem; the decode was. Same for the tone
curves, colour, clarity, and the white balance shifts.

5 · Noise reduction is not a linear scale. `0xD1A1` maps +2 to `0x0000` and
0 to `0x2000`, and is not monotonic. A plain menu number stored there writes the
wrong strength and reports success.

## Wireless tethering, in progress

The camera joined the house network and the mode was set to WIRELESS TETHER
SHOOTING FIXED. A port scan of the subnet found the camera at `192.168.0.18`
with a USI Wi-Fi module and no open ports at all — not 55740, 55741, 55742
or 15740.

That is the expected behaviour, not a fault. In this mode the roles are
reversed: the computer listens and the camera dials in.

The sequence, from libfuji's `discovery.c`:

| Step | Detail |
|---|---|
| 1 | Computer listens on TCP `51560` |
| 2 | Computer broadcasts to the subnet on UDP `51562` — `DISCOVERY * HTTP/1.1`, `SERVICE: PCSS/1.0` |
| 3 | Camera connects to `51560`, announcing `DSC`, `CAMERANAME`, `DSCPORT` |
| 4 | Computer answers `HTTP/1.1 200 OK` and closes |
| 5 | Computer connects back on the announced port and speaks Fuji PTP |

Two things worth carrying forward. libfuji hardcodes the broadcast to
`192.168.1.255`, which reaches nothing on a `192.168.0.x` network; the address
is derived from the interface here instead. And the port for this path is
`51560`, not the `55740` used when the camera raises its own access point.

Implemented as `FujiTetherDiscovery`, run with `fuji-cli wifi-tether en0`.
The invitation goes out; the camera had not dialled in as of writing.

### What this run is for

One question: does `0xD192` answer over this transport. The cursor is not
enough — it is advertised over USB as well, so it would pass either way.

If it answers, the RAW CONV. detour disappears and the phone can write slots and
live settings from one mode. If it does not, wireless offers nothing here.

## Untested

- Whether the camera can shoot while in USB RAW CONV. mode. Asserted earlier
  without being checked.
- Whether `0xD1B2` really takes 1 for JPEG. It is only written when clarity has
  already been refused, and that path has not been exercised.
- The slot name length limit on this body. fujicli declares 25 characters across
  the X-Trans generations it covers, and the GFX100RF is not one of them. The app
  now writes at 25 and reads the name back, so the next slot write settles it.
- Whether `0xD16E` can be written to put the camera into RAW conversion mode.
  fujicli documents `6` as that mode. If it takes, the app can stop asking anyone
  to change CONNECTION MODE by hand.
- `0xD1A5`, unidentified by anyone.
- Whether the wireless tether path exposes the slot block — the run above.
- iOS on the wireless tether path. It needs a UDP broadcast, and that requires
  the multicast entitlement, which Apple grants by application. The
  access-point path needs no entitlement, but is the route already abandoned.
