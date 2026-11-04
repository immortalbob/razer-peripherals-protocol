# Razer Huntsman V2: keyboard protocol (USB `1532:026C`)

Part of the [Razer peripherals protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **PUBLISHED**,
**HOST**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. Times are local time of the
capturing PC (UTC-7). "The capture" is the USBPcap session of 2026-10-30, 13:39 to 14:03, in which Razer Synapse 3
(3.10.0730.071519) drove the keyboard at USB device address 13. The keyboard's interrupt IN reports were filtered out
while capturing, so the capture holds no key presses, and none are reproduced here.

Offsets: `r[n]` counts the 90-byte Razer report as it travels on the wire (on Windows the buffer carries report ID
`0x00` in front of it), and `a[n]` = `r[8 + n]` are the arguments. Commands are written `class:id`, for example
`0F:03`. The shared framing (status, transaction id, data size, class, id, arguments, CRC at `r[88]`), the status codes,
the storage byte and the generic command catalogue are in [razer-hid-protocol.md](razer-hid-protocol.md); this document
restates only what is specific to this keyboard.

Short version (each point is tagged in the sections below): the keyboard is a composite USB device with five HID
interfaces. The Razer control protocol runs on interface 3, a consumer-control collection, as 90-byte feature reports
with report ID 0. The per-key lighting is a 6 × 22 matrix written one row per `0F:03` report; Synapse renders every
effect on the host and streams about 24 frames per second, and never uses a hardware effect while it runs. The
keyboard answers polling-rate and brightness writes "busy" for a few milliseconds before "ok". The polling rate
(1000 or 8000 Hz on the wire so far) is lost at power-off and comes back as 1000 Hz; the keyswitch timing preset is
kept. Game Mode (Fn+F10) works in the firmware without any software, but the game-mode LED command has no visible
light on the observed unit. Interface 4 is a standard HID LampArray that Windows' Dynamic Lighting writes to.

## 1. Device

### 1.1 What the keyboard has

| Item | Finding | Source |
|---|---|---|
| Product | Razer Huntsman V2, full size, optical switches, model code RZ03-0393 | **PUBLISHED**, Razer product pages |
| Lighting | per-key RGB; no underglow, logo or wrist-rest LEDs in any source | **PUBLISHED**, Razer product pages; **PUBLISHED**, openrazer and OpenRGB (one matrix zone) |
| Matrix | 6 rows × 22 columns, one LED region `0x05` | **PUBLISHED**, openrazer (`MATRIX_DIMS = [6, 22]`) and OpenRGB; CAPTURED and OBSERVED: the region list answers `05 19 03 06 16` (section 4.1) |
| Media controls | media keys and a "multi-function digital dial" | **PUBLISHED**, Razer product pages and reviews (section 4.9) |
| Polling | up to 8000 Hz | **PUBLISHED**, Razer ("HyperPolling"); openrazer lists 125, 250, 500, 1000, 2000, 4000 and 8000 Hz |
| Onboard memory | up to five keybinding profiles | **PUBLISHED**, Razer product pages; CAPTURED: `05:8A` answers `05` (section 4.10) |
| Layout | the observed unit is US ANSI | CAPTURED: `00:86` answers `01 00` (layout 1, variant 0) |
| Firmware | `00:81` answers `01 00 00 00` | OBSERVED 2026-10-31. Synapse's log printed "FW: 1.0.5.0" for the same unit (HOST); neither the `00:81` reply nor the USB `bcdDevice` `0x0101` reads as 1.0.5.0, so where Synapse's number comes from is UNKNOWN |
| Synapse's internal name | "Jamie T2 V2" | METADATA: Synapse's device-definition folder for this PID |

### 1.2 Descriptors

**CAPTURED (2026-10-30, 13:39:15)**, from the descriptors USBPcap injected at the start of segment 1 (the keyboard was
not re-enumerated during the capture, so its report descriptors were not captured):

| Item | Value |
|---|---|
| Device descriptor | `12 01 00 02 00 00 00 40 32 15 6C 02 01 01 01 02 00 01`: bcdUSB 2.00, class per interface, 64-byte EP0, VID `1532`, PID `026C`, bcdDevice `0101`, manufacturer string 1, product string 2, **no serial-number string**, one configuration |
| Configuration | `09 02 86 00 05 01 00 A0 FA`: five interfaces, bus-powered with remote wake-up (`A0`), bMaxPower 250 = 500 mA |

Full configuration descriptor as captured, one line per interface (interface, HID, endpoint):

```
09 02 86 00 05 01 00 A0 FA
09 04 00 00 01 03 01 01 00   09 21 11 01 00 01 22 3D 00   07 05 81 03 08 00 01
09 04 01 00 01 03 00 01 00   09 21 00 01 00 01 22 B1 00   07 05 82 03 16 00 01
09 04 02 00 01 03 01 02 00   09 21 00 01 00 01 22 4F 00   07 05 83 03 08 00 01
09 04 03 00 01 03 00 00 00   09 21 00 01 00 01 22 16 00   07 05 84 03 08 00 01
09 04 04 00 01 03 00 01 00   09 21 00 01 00 01 22 5C 01   07 05 85 03 02 00 01
```

- **Every endpoint is interrupt IN with bInterval 1.** At high speed that is one microframe, 125 µs, the interval
  8000 Hz needs; at full speed it would be 1 ms (INFERRED from the USB 2.0 timing rules). The bus speed the keyboard
  ran at is UNKNOWN: the capture does not record it, and the interrupt traffic that would show the real report rate
  was filtered out.
- **No interrupt OUT endpoint.** Feature and output reports travel as control transfers on endpoint 0 (CAPTURED: every
  host-to-keyboard report in the capture was a SET_REPORT on the default pipe).
- **No USB serial number.** Windows therefore derives the instance id from the port (HOST). The Razer serial read with
  `00:82` is a separate value (section 4.12).

### 1.3 Interfaces and collections

| Interface | Class / subclass / protocol | Endpoint | Report descriptor | HID collections (HOST) | Purpose |
|---|---|---|---|---|---|
| 0 | HID 1.11, boot keyboard (`03 01 01`) | `0x81`, 8 bytes | 61 bytes | keyboard `0001/0006` | boot-protocol keys; lock-indicator output report without report ID |
| 1 | HID 1.00, keyboard protocol (`03 00 01`) | `0x82`, 22 bytes | 177 bytes | Col01 keyboard `0001/0006`, Col02 consumer `000C/0001`, Col03 system control `0001/0080`, Col04 and Col05 usage `0001/0000` | full key reports, consumer and system keys; lock-indicator output report ID 1; driver-mode Razer key reports (INFERRED, section 4.9) |
| 2 | HID 1.00, boot mouse (`03 01 02`) | `0x83`, 8 bytes | 79 bytes | mouse `0001/0002` | the dial's rotation, reported as a wheel (section 4.9) |
| 3 | HID 1.00, no boot protocol (`03 00 00`) | `0x84`, 8 bytes | 22 bytes | consumer `000C/0001`, one collection | **the Razer control protocol** (section 2) |
| 4 | HID 1.00, keyboard protocol (`03 00 01`) | `0x85`, 2 bytes | 348 bytes | Col01 LampArray `0059/0001`, Col02 consumer `000C/0001` | Windows Dynamic Lighting (section 4.11) |

Class codes, endpoints and descriptor lengths are CAPTURED; the collections are what Windows enumerated (HOST,
2026-10-26). The purpose of interface 3's interrupt IN endpoint is UNKNOWN: Synapse never needed it for the control
protocol (CAPTURED: every reply was fetched with GET_REPORT).

**Lock indicators (CAPTURED, 13:45:42.245 and .250).** When the mouse was replugged, Windows re-sent the keyboard
indicator state to the keyboard twice: `21 09 01 02 01 00 02 00` (SET_REPORT, output report ID 1, interface 1,
2 bytes) and `21 09 00 02 00 00 01 00` (output report without an ID, interface 0, 1 byte), each with bit 0 set. These
are ordinary HID keyboard LED reports, not part of the Razer protocol.

**Windows drivers.** With Synapse 3 installed (HOST, 2026-10-26) Razer's filter `RzDev_026c` sat below USB interface 1
and above the HID collections of interfaces 1 (Col01), 2 and 3; interface 3 was installed as a raw collection; Razer's
control node `RZCONTROL` was a child of interface 3, and a virtual consumer device `RZVIRTUAL` was a child of USB
interface 1. That virtual device's consumer collection has the same usage, `000C/0001`, under a hardware id containing
`MI_00`, so match the control collection by interface number 3, not by usage alone (INFERRED). After Synapse was
removed (2026-11-03) interface 3 runs on Windows' `hidserv.inf` and interface 0 on `keyboard.inf`, with no Razer
filter (HOST). Details: [razer-software-and-drivers.md](razer-software-and-drivers.md).

### 1.4 Prior work

The protocol for this keyboard was first **PUBLISHED** by openrazer (`driver/razerkbd_driver.c` per-PID cases,
`driver/razerchromacommon.c` command builders, `daemon/openrazer_daemon/hardware/keyboards.py` class
`RazerHuntsmanV2`) and by OpenRGB (`Controllers/RazerController/RazerControllerDetect.cpp` detector on interface 3,
usage page `0x0C`; `Controllers/RazerController/RazerDevices.cpp` 6 × 22 zone with the media keys;
`Controllers/RazerController/RazerController/RazerController.cpp` frames and identity reads). OpenRGB's 2022 merge
request !1373 corrected the zone from 9 rows to 6 for this keyboard (**PUBLISHED**, OpenRGB issue #2673). This
document adds what Synapse actually sends, the bench results and the corrections in section 4.

## 2. Transport

### 2.1 The control channel

| Item | Value | Source |
|---|---|---|
| Interface | 3, consumer collection `000C/0001` | **PUBLISHED**, openrazer (`wIndex` 3 for this PID) and OpenRGB (detector on interface 3, usage page `0x0C`, usage `0x01`); CAPTURED: every Synapse request went to interface 3 |
| Report | feature, report ID `0x00`, 90 bytes on the wire | CAPTURED |
| Set | `21 09 00 03 03 00 5A 00`: SET_REPORT, `wValue 0x0300` (feature, ID 0), `wIndex 3`, 90 bytes | CAPTURED |
| Get | `A1 01 00 03 03 00 5A 00`: GET_REPORT, same `wValue`, `wIndex` and length | CAPTURED |
| Transaction id | `0x1F` on every Synapse request | CAPTURED |
| Open on Windows | a feature-only open of the interface-3 collection, transaction id `0x1F` | OBSERVED 2026-10-31 to 2026-11-03, both under Razer's driver package (Synapse stopped) and on `hidserv.inf` |

### 2.2 Reply timing and busy

- **First read about 3 ms after the write** (CAPTURED: GET_REPORT followed each SET_REPORT by 2 to 3 ms).
- **Reads, mode, LED-state and region commands** answered `0x02` (ok) at the first read (CAPTURED).
- **Polling-rate, brightness, effect and profile writes** first answered `0x01` (busy). Synapse re-read every 1 to
  2 ms until `0x02`, which came 9 ms after the write for `00:40`, 12 ms for a brightness change and 38 ms for `05:02`
  (CAPTURED, 13:47:58, 13:49:24, 14:01:37). The busy replies echo the request. A repeated write of the same value was
  sometimes answered ok at once (CAPTURED, the second brightness write at 13:49:24.711).
- **A busy reply can carry a wrong CRC.** At Synapse's exit (14:00:26.352) the busy reply to `0F:02` `01 00 03 ...`
  already showed `a[1] = 05` instead of the `00` sent, but kept the request's CRC `09`; the ok reply 5 ms later had the
  same arguments and the correct CRC `0C` (CAPTURED). Do not treat a CRC mismatch on a busy reply as a fault; poll on.
- **Reply sizes differ from request sizes.** `00:84` is sent with size 2 and answered with size 1; `0F:80` with size
  0x32 and answered with size 5; `0F:82` with size 0x0C and answered with size 9 (CAPTURED). Match replies on class,
  id and transaction id, not on size.
- **Frames are not read back.** Synapse sent its `0F:03` rows without a GET_REPORT after any of them (CAPTURED: in
  segment 8 the only two reads in its 40 s belonged to the two game-LED writes).
- **Writes twice.** Synapse sent polling-rate and brightness changes twice, 20 to 30 ms apart, each with its own read;
  game-LED and mode writes once (CAPTURED). The keyboard accepted both copies. Sending once and reading until ok is
  enough (INFERRED).

### 2.3 Transaction ids

Synapse used `0x1F` for every command (CAPTURED), and third-party code used `0x1F` for every command it sent from
2026-10-31 on, without a refusal (OBSERVED). openrazer uses `0xFF` for the device-mode, identity, layout and macro-LED
commands of this PID and OpenRGB `0x3F` for everything (**PUBLISHED**); both are reported to work for their users.
Whether this firmware accepts all three was not tested here (UNKNOWN). Use `0x1F`.

### 2.4 Other writers

While Synapse runs, its service streams frames to interface 3 continuously (section 4.2), so any second writer's
frames interleave with Synapse's. Windows Dynamic Lighting writes the LampArray on interface 4 independently (section
4.11). Stop the vendor software before driving the keyboard; the shared coexistence rules are in
[razer-hid-protocol.md](razer-hid-protocol.md) and [razer-software-and-drivers.md](razer-software-and-drivers.md).

## 3. Command set

Size is `r[5]`. "Synapse" = sent by Synapse in the capture; "bench" = sent by third-party code (OBSERVED, dates in
section 4).

| Cmd | Purpose | Request `a[]` | Reply `a[]` | Seen | Use |
|---|---|---|---|---|---|
| `00:81` | firmware version | size 2, none | `01 00 00 00` on this unit | bench; PUBLISHED | read |
| `00:82` | serial number | size 0x16, none | ASCII serial (not reproduced) | bench; PUBLISHED | read |
| `00:84` | get device mode | size 2, `00 00` | `00` normal / `03` driver (size 1) | Synapse; bench | read |
| `00:04` | set device mode | size 2, `03 00` or `00 00` | echo | Synapse | `00 00` only (section 5) |
| `00:86` | layout | size 2, `00 00` | `01 00` = US, black | Synapse | read |
| `00:C0` | get polling rate | size 1, none | code in `a[1]` | bench; PUBLISHED | read |
| `00:40` | set polling rate | size 2, `01 cc` | echo; busy then ok | Synapse; bench | section 4.6 |
| `0F:04` | set brightness | size 3, storage, LED, value | echo; busy then ok | Synapse (LED `00`); PUBLISHED (LED `05`) | section 4.5 |
| `0F:84` | get brightness | size 3, `01 05` | `a[2]` | bench; PUBLISHED | read |
| `0F:02` | hardware effect | storage, LED `05`, effect, parameters | echo | Synapse (exit only); bench | section 4.4 |
| `0F:02` | apply custom frame | size 0x0C, `00 00 08` | echo | bench; PUBLISHED | section 4.2 |
| `0F:03` | custom frame row | size 0x47, `00 00 rr 00 15` + 66 colour bytes | not read | Synapse; bench | section 4.2 |
| `0F:10` | enable streamed frames (INFERRED) | size 1, `01` | echo | Synapse | section 4.2 |
| `0F:80` | LED region list | size 0x32, zeros | `05 19 03 06 16` | Synapse; bench | read |
| `0F:81` | effects of one LED | `05` | | PUBLISHED (OpenRGB) only | not read here |
| `0F:82` | stored effect | size 0x0C, `01 05` | `01 05 03 00 00 01 6E DA FD` | Synapse | read |
| `03:00` | indicator LED on/off | size 3, storage, LED, state | echo | Synapse (LED `08`, `18`); bench (LED `08`) | section 4.8 |
| `03:80` | indicator LED state | size 3, `00 08` | `a[2]` state | bench | read |
| `03:02` | macro LED effect | size 3, `01 07 00` static / `01 07 01` blink | | PUBLISHED (openrazer) only | not exercised |
| `02:82` | get keyswitch timing | size 4, none | four bytes, section 4.7 | bench | read |
| `02:02` | set keyswitch timing | size 4, four bytes | echo | bench; PUBLISHED | presets only |
| `02:15` | store keyswitch timing | size 5, `01` + four bytes | | PUBLISHED (openrazer) only | not needed (section 4.7) |
| `02:14` | mouse scroll-wheel mode | size 2, `01 00` | status `0x05` not supported | Synapse | **do not send** |
| `05:8A` | profile query | size 1, `00` | `05` | Synapse | read; section 4.10 |
| `05:80` | profile query | size 1, `00` | `01` | Synapse | read |
| `05:81` | profile query | size 0x41, zeros | `01 01` | Synapse | read |
| `05:02` | profile write (INFERRED: select profile 1) | size 1, `01` | busy then ok | Synapse | **do not send** |
| `06:80` | storage query | size 2, `00 00` | `00 00` | Synapse | read |
| `06:8E` | storage query | size 0x0E, zeros | `FF FF 00 03 FF C8 00 03 DE F8 00 03 DE F8` | Synapse | read; section 4.10 |

Anything else: do not send (section 5).

## 4. Commands

### 4.1 LED ids on this keyboard

| LED id | What it is | Source |
|---|---|---|
| `0x05` | the key matrix ("backlight"), the only region the firmware lists | CAPTURED and OBSERVED: `0F:80` answers `05 19 03 06 16`; **PUBLISHED**, openrazer and OpenRGB address effects and brightness here |
| `0x00` | "whole device": Synapse's brightness writes, its exit effect write and the custom-frame apply carry it | CAPTURED (sections 4.4, 4.5); **PUBLISHED** for the apply |
| `0x08` | game-mode indicator | CAPTURED (Synapse toggles it with Game Mode); OBSERVED: no visible light on this unit (section 4.8) |
| `0x07` | macro indicator | **PUBLISHED**, openrazer; not exercised here |
| `0x18` | INFERRED volume/mute indicator by the dial | CAPTURED: Synapse writes it off at exit; METADATA: position 24 of Synapse's `LEDID_*` name table is `LEDID_VOLUME`, and Synapse's device definition lists a `VOLUMELED` group for this PID |

The region record `05 19 03 06 16` reads as LED `0x05`, two bytes `19 03` of UNKNOWN meaning (the mouse's regions carry
the same two bytes, see [razer-deathadder-v2.md](razer-deathadder-v2.md)), 6 rows, 22 columns (INFERRED from the
matrix size).

### 4.2 Custom frames

One frame is six row reports, rows 0 to 5:

| `a[]` | Size | Field | Value |
|---|---|---|---|
| 0-1 | 2 | reserved | `00 00` (no storage byte) |
| 2 | 1 | row | 0-5 |
| 3 | 1 | first column | `00` |
| 4 | 1 | last column, inclusive | `15` (21) |
| 5-70 | 66 | colours | 22 × R, G, B, column 0 first |

Data size `0x47` = 71 = 5 + 66 (**PUBLISHED**, openrazer and OpenRGB; CAPTURED).

- **Synapse streams continuously.** In segments 1, 6 to 9, 13 and 14 Synapse sent 80 301 row reports, all full rows
  with size `0x47`, transaction id `0x1F`, a correct CRC and `r[89] = 00`, at 23.4 to 25.1 frames per second, about
  145 row writes per second, whether the effect chosen was Wheel, Static or another (CAPTURED). The six rows of a frame
  went out 2.3 to 2.6 ms apart, and a row's control transfer completed about 0.25 ms after submission (CAPTURED,
  13:50:06). A Synapse effect change only changes the frame contents: no `0F:02` was sent (CAPTURED, segment 9,
  13:50:23 to 13:51:08).
- **Switching to frames.** OpenRGB sends the apply `0F:02` `00 00 08` (size `0x0C`) after every frame (**PUBLISHED**).
  Synapse sent no apply to the keyboard at all, but `0F:10` `01` right after entering driver mode at its start, and
  its frames displayed (CAPTURED, 14:01:34.814). Third-party code sent one apply, then rows only, without `0F:10`, and
  the frames displayed for 30 s (OBSERVED 2026-11-01 05:16). Either works; one apply per attach is enough (OBSERVED).
  What `0F:10` does by itself is UNKNOWN; it is not in openrazer or OpenRGB.
- **Row addressing.** A frame with one colour per block of 22 cells showed one colour per keyboard row (OBSERVED
  2026-11-01 05:17), so the cell index is row × 22 + column.
- **Smooth streaming.** A host-rendered breathing effect streamed for 20 s ran smoothly (OBSERVED 2026-11-01 05:17).
  The highest safe frame rate is UNKNOWN; Synapse's 24 frames per second is the rate known to be fine.
- **No storage byte.** Rows carry none and the apply carries `00` in that position, so streaming cannot write the
  stored effect (INFERRED from the layouts).

### 4.3 Key positions

The cells for the US ANSI layout (layout code 1). `-` = no key: send black.

| r \ c | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **0** | Esc | - | F1 | F2 | F3 | F4 | F5 | F6 | F7 | F8 | F9 | F10 | F11 | F12 | PrtSc | ScrLk | Pause | M1 | M2 | M3 | M4 | - |
| **1** | Grave | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 0 | Minus | = | Bksp | Ins | Home | PgUp | NumLk | KP/ | KP* | KP- | - |
| **2** | Tab | Q | W | E | R | T | Y | U | I | O | P | [ | ] | Bksl | Del | End | PgDn | KP7 | KP8 | KP9 | KP+ | - |
| **3** | Caps | A | S | D | F | G | H | J | K | L | ; | ' | (ISO #) | Enter | - | - | - | KP4 | KP5 | KP6 | - | - |
| **4** | LShift | (ISO Bksl) | Z | X | C | V | B | N | M | , | . | / | (JIS) | RShift | - | Up | - | KP1 | KP2 | KP3 | KPEnter | - |
| **5** | LCtrl | LWin | LAlt | - | - | - | Space | - | - | - | RAlt | Fn | Menu | RCtrl | Left | Down | Right | - | KP0 | KP. | - | - |

Bksl = backslash. M1 to M4 are the media-key area (below).

Evidence:

- **Two independent sources agree on all 104 ANSI keys.** Razer's Chroma SDK ships a grid-to-device map for this PID:
  device cell (r, c) takes the colour of Chroma keyboard-grid cell (r, c + 1), except that the four cells (0,17) to
  (0,20) all take Chroma cell (0,18), cell (0,21) takes (0,21), cell (4,12) takes (0,21) and cell (3,14) takes (1,12)
  (METADATA, the SDK's data file for PID `026C`). Applied to the key positions of the Chroma keyboard grid
  (**PUBLISHED**, openrazer's generic map in `daemon/openrazer_daemon/keyboard.py`), it gives the table above, and so
  does OpenRGB's full-size layout with its Huntsman V2 overlay (**PUBLISHED**,
  `KeyboardLayoutManager/KeyboardLayoutManager.cpp` and `Controllers/RazerController/RazerDevices.cpp`), with no
  mismatch (INFERRED by comparing the two).
- **Synapse lights exactly these cells.** Across all 80 301 captured rows, the cells that were ever non-black are
  exactly the 108 named cells of the table (104 keys and M1 to M4); every `-` and parenthesised cell stayed black in
  every frame (CAPTURED).
- **On the device.** A horizontal rainbow drawn through this table showed as a smooth left-to-right rainbow with no row
  shifted, and frame samples put Grave, Tab, Caps, LShift and LCtrl at the red end, Backspace, Enter and RShift at
  cyan and the numpad at blue to magenta (OBSERVED 2026-11-01 06:27; the owner confirmed it). Each cell was not lit
  one at a time, so a swap of two neighbouring keys would not have shown (UNKNOWN).
- **M1 to M4** are the media-key area. OpenRGB names them Previous, Play/Pause, Next and Mute (**PUBLISHED**).
  Synapse colours all four (CAPTURED), while the Chroma SDK feeds all four from one grid cell (METADATA). Reviews
  describe three media buttons and a dial that mutes when pressed (**PUBLISHED**, reviews, weak), so which physical
  element M4 lights is UNKNOWN.
- **Other layouts.** ISO layouts add "#" at (3,12) and the extra backslash at (4,1); a JIS key sits at (4,12)
  (METADATA, Synapse's LED-layout files and the SDK map; **PUBLISHED**, OpenRGB's ISO overlay). Synapse's own LED-layout
  files use a drawing grid in which some blocks sit one column to the right; they are not the frame index (METADATA,
  INFERRED). The remaining JIS keys are UNKNOWN.

**Correction to earlier public descriptions (INFERRED).** openrazer's daemon places position-based effects (ripple)
with its generic key map, which is in the Chroma-grid convention (Escape at column 1). On this keyboard the device grid
is the Chroma grid shifted one column left, so those effects should land one column to the right of the key
(INFERRED from the two maps; not reported anywhere found and not tested here). A Chroma SDK client must likewise add
one to the column.

### 4.4 Hardware effects

`0F:02` with `a[0]` storage, `a[1]` LED `0x05`, `a[2]` effect id, then parameters (**PUBLISHED**, openrazer's builders;
the generic layout is in [razer-hid-protocol.md](razer-hid-protocol.md)):

| Effect | `a[2]` | Size | `a[3..]` | On this keyboard |
|---|---|---|---|---|
| Off | `00` | 6 | | OBSERVED 2026-11-03: all keys dark |
| Static | `01` | 9 | `00 00 01 R G B` | |
| Breathing | `02` | 6 / 9 / 12 | `00` random; `01 00 01 R G B`; `02 00 02 R1 G1 B1 R2 G2 B2` | |
| Spectrum cycle | `03` | 6 | | OBSERVED 2026-11-01 and 2026-11-03; CAPTURED (stored by Synapse) |
| Wave | `04` | 6 | direction, `28` | OBSERVED 2026-11-01: direction `01` runs left to right, `02` right to left |
| Reactive | `05` | 9 | `00 ss 01 R G B`, speed 1-4 | |
| Starlight | `07` | 6 / 9 / 12 | `00 ss 00`; `00 ss 01 R G B`; `00 ss 02 R1 G1 B1 R2 G2 B2`, speed 1-3 | |

- Third-party code sent off, spectrum and wave with storage `0x00` on LED `0x05`, and each was accepted and shown
  (OBSERVED 2026-11-01 05:15 to 05:18 and 2026-11-03 08:30 to 08:36). Static, breathing, reactive and starlight as
  hardware effects were not exercised on this keyboard.
- openrazer notes that newer firmware may use wave directions `00`/`01` (**PUBLISHED**); on this firmware `01` and
  `02` both work as stated (OBSERVED). `00` was not tried.
- **Stored effect.** At its exit Synapse read the stored effect with `0F:82` `01 05` and got `01 05 03 00 00 01 6E DA
  FD` (spectrum, with a left-over colour), then wrote `0F:02` `01 00 03 00 00 00` (storage `0x01`, LED `0x00`,
  spectrum) before leaving driver mode (CAPTURED, 14:00:26.344 to .357). So the keyboard runs a stored spectrum cycle
  when no host drives it (INFERRED from that sequence).
- The firmware's own effect list (`0F:81`) was not read on this keyboard (UNKNOWN).

### 4.5 Brightness

| Writer | Command | Values |
|---|---|---|
| Synapse | `0F:04` storage `01`, **LED `00`**, value | `7F` for 50 %, `FF` for 100 % (CAPTURED, 13:49:24 and 13:49:28); at its start `00 00 FF`, then `01 00 FF` (CAPTURED, 14:01:34) |
| openrazer, OpenRGB | `0F:04` / `0F:84`, LED `05`, 0-255 | **PUBLISHED** |
| third-party code | `0F:84` `01 05` | read 255 after Synapse (OBSERVED 2026-10-31) |

- 127 for 50 % is consistent with truncating p × 255 / 100 (127.5); two points only (INFERRED).
- The firmware accepts LED `0x00` for brightness (CAPTURED: ok replies). A brightness **write** with LED `0x05` was not
  made here; reading with `0x05` works (OBSERVED).
- Fn+F11 and Fn+F12 step the brightness down and up by 5 in Synapse's mapping (METADATA, Synapse's default key
  mapping). Whether the firmware does the same without Synapse was not checked (UNKNOWN). A host should re-read the
  brightness rather than cache it.

### 4.6 Polling rate

`00:40` set, `00:C0` get, code in `a[1]`:

| Code | `01` | `02` | `04` | `08` | `10` | `20` | `40` |
|---|---|---|---|---|---|---|---|
| Rate (Hz) | 8000 | 4000 | 2000 | 1000 | 500 | 250 | 125 |

- **PUBLISHED** by openrazer, which sends `a[0] = 00` and notes that Razer's software sends some devices each request
  with `a[0] = 00` and `01`, "maybe varstore".
- **CAPTURED (segment 6).** Synapse's 1000 → 8000 → 1000 Hz change sent `00:40` `01 01` at 13:47:58.419 and `01 08` at
  13:48:02.149, each twice, always with `a[0] = 01`, never `00`; nothing else accompanied it, so the rates above
  1000 Hz use the same command. At its start Synapse re-sent the saved rate (`01 08` twice, 14:01:37).
- **OBSERVED 2026-11-01.** Third-party code wrote `00:40` `01 01` (06:51:01) and read 8000 Hz back with `00:C0`. The
  reply carries the code in `a[1]`. Rates other than 1000 and 8000 Hz were not exercised.
- **Not kept through power-off.** With Synapse stopped, the keyboard read 8000 Hz (as Synapse had left it), and after an
  unplug and replug it read **1000 Hz** (OBSERVED 2026-10-31 20:57 and 2026-11-01 06:11). Synapse writes with
  `a[0] = 01` every time, so `a[0] = 01` does not make the rate survive a power cycle (INFERRED); whether it stores
  anything is UNKNOWN. Software that wants 8000 Hz must write it at every attach, as Synapse does.

### 4.7 Keyswitch optimisation

`02:02` set, `02:82` get, four bytes (**PUBLISHED**, openrazer). Synapse's preset file names the four fields and holds
two presets (METADATA, Synapse's `KeyswitchOptimization.xml` for this PID):

| `a[]` | Field (Synapse's name) | Typing | Gaming |
|---|---|---|---|
| 0 | DebounceMake | `00` | `00` |
| 1 | MaskMake | `14` (20) | `00` |
| 2 | DebounceBreak | `00` | `00` |
| 3 | MaskBreak | `28` (40) | `00` |

- The keyboard read `00 14 00 28` (Typing) after Synapse (OBSERVED 2026-11-01 06:07).
- **`02:02` alone persists.** Third-party code wrote Gaming with `02:02` only, read it back, unplugged and replugged the
  keyboard: it still read Gaming, and typing worked; Typing was then written back and read back (OBSERVED 2026-11-01
  06:09 to 06:12). openrazer always follows `02:02` with `02:15` `01` + the same four bytes (**PUBLISHED**); on this
  firmware that second command is not needed for persistence. What `02:15` adds is UNKNOWN.
- Units: UNKNOWN (INFERRED milliseconds). Synapse's change between presets was not captured. Values other than the two
  presets are not published anywhere; do not send them (section 9).

### 4.8 Game Mode and the indicator LEDs

`03:00` set and `03:80` get, `a[0]` storage, `a[1]` LED id, `a[2]` state (**PUBLISHED**, openrazer).

- **Synapse (CAPTURED, segment 8).** Pressing Fn+F10 in driver mode reached the host; Synapse answered with `03:00`
  `00 08 01` (13:50:06.936) and, on the next press, `00 08 00` (13:50:10.937), one write each and nothing else. At its
  start it writes `00 08 00` (14:01:37.246); at its exit `00 08 00` and `00 18 00` (14:00:26.329 and .334). Every reply
  was ok at the first read.
- **No visible light.** Third-party code wrote `03:00` `00 08 01` with Synapse stopped: accepted, read back on with
  `03:80`, but nothing visible changed on the keyboard; `00 08 00` likewise (OBSERVED 2026-11-01 06:53 and 06:54). An
  accepted, read-back LED state does not prove a lit indicator.
- **Game Mode in the firmware.** With Synapse stopped and the keyboard in normal mode, Fn+F10 blocked the Windows key
  (OBSERVED 2026-11-01, the owner). Earlier descriptions put Game Mode in host software: openrazer blocks the keys in
  its kernel event handler and Synapse in its filter driver (**PUBLISHED**; METADATA, Synapse's GameMode settings
  list the Windows keys, Alt+Tab and Alt+F4). In normal mode the firmware does it itself.
- **LED `0x18`.** Synapse switching it off at exit is the only wire evidence for a third indicator. Its colour, its
  behaviour and whether a host can light it are UNKNOWN. The macro LED `0x07` and `03:02` (static or blink) are
  **PUBLISHED** by openrazer and were not exercised.

### 4.9 Device mode, Fn layer, media keys and the dial

The device-mode command `00:04` / `00:84` is in [razer-hid-protocol.md](razer-hid-protocol.md). On this keyboard:

- Synapse read `00` and wrote **`03 00` (driver mode)** at its start (14:01:34.807) and, at its exit, read `03` and wrote
  **`00 00` (normal)** as its last command (14:00:26.364) (CAPTURED). With Synapse stopped normally, third-party code
  read normal mode (OBSERVED 2026-10-31 to 2026-11-03).
- **Driver mode** hands the special keys to the host: the firmware reports Fn and the media keys as Razer codes in a
  report starting with `0x04` (`0x01` Fn; `0x52` Mute, `0x53` Next, `0x54` Previous, `0x55` Play/Pause), and openrazer
  treats the 22-byte form of that report as a key bitmap (**PUBLISHED**, openrazer). Synapse maps the same codes to the
  media scan codes and replays them (HOST, Synapse's mapping log). Interface 1's 22-byte endpoint is the only one of
  that size (CAPTURED), so it most likely carries these reports (INFERRED).
- **The Fn layer** in Synapse's default mapping: Fn+F9 macro recording, Fn+F10 Game Mode, Fn+F11 / Fn+F12 brightness
  down / up, Fn+Pause sleep, Fn+Menu next onboard profile; Synapse logs that this model has no function lock
  (METADATA and HOST, Synapse's defaults and logs). openrazer's translation table for this PID has the same F-row and
  Pause functions (**PUBLISHED**). In normal mode Fn+F10 works without software (OBSERVED, section 4.8); the others were
  not checked (UNKNOWN).
- **The dial** reports rotation as a mouse wheel on interface 2: openrazer maps this PID's wheel to a volume control
  (**PUBLISHED**), and Synapse saw it as `SCROLL UP` / `SCROLL DN` and mapped it to Volume Up / Down (HOST, Synapse's
  log). Whether the firmware sends a wheel or consumer volume usages in normal mode, and which consumer collection
  carries the media keys in normal mode, are UNKNOWN (the capture dropped input reports). Reviews say the dial is
  pressable and mutes (**PUBLISHED**, weak).
- A driver mode left on by crashed software leaves the Fn layer, media keys and dial without a handler (INFERRED from
  the above); the fix is one `00:04` `00 00`.

### 4.10 Profile and storage queries

At its start Synapse sent four class-5 and two class-6 commands that no open-source project documents (CAPTURED,
14:01:37.529 to .642). Their names are INFERRED from the replies and from Synapse's logs:

| Cmd | Reply | INFERRED reading |
|---|---|---|
| `05:8A` | `05` | number of onboard profile slots (Razer advertises five) |
| `05:80` | `01` | number of stored profiles, or the active one (Synapse logged "Found 1 Profiles", HOST) |
| `05:81` (65-byte request) | `01 01` | profile list: count 1, profile 1 |
| `05:02` `01` (write) | busy, ok after 35 ms | select or open profile 1 |
| `06:80` | `00 00` | UNKNOWN |
| `06:8E` | `FF FF 00 03 FF C8 00 03 DE F8 00 03 DE F8` | storage sizes, big-endian: `0x0003FFC8` = 262 088 and twice `0x0003DEF8` = 253 688 |

The `06:8E` reading is supported by Synapse's own log for this keyboard, which reports a storage maximum of 262 088
bytes and warns that it is smaller than a free size of 507 376 = 2 × 253 688 (HOST): Synapse evidently added the two
free fields. The leading `FF FF` is UNKNOWN. Synapse also polled `06:80` and `06:8E` twice while running (CAPTURED,
13:46:46 to 13:46:47). Synapse rewrote onboard profile 1 at every device start, with brightness 100, polling
1000 Hz and the Typing timing, according to its logs (HOST, 2026-10-23 to 2026-10-26); the commands that carry those
writes were not identified in the capture (UNKNOWN). The reads are harmless; `05:02` is a write and is left alone
(section 5).

### 4.11 The LampArray interface

Interface 4, collection 1 is a standard HID LampArray (usage page `0x59`) with an inbox driver and no Razer filter
(HOST). Windows Dynamic Lighting was enabled for it on the observed PC (HOST, 2026-10-26, Dynamic Lighting registry
settings), and Synapse marked the PID as Dynamic-Lighting capable (HOST, Synapse's settings).

- **CAPTURED (13:45:42.252).** When the mouse was replugged, Windows sent the keyboard's interface 4 one 9-byte report:
  `05 01 00 00 6D 00 00 00 00`. The report descriptor was not captured, so its layout is not known. Read with the
  field order of Microsoft's sample LampArray descriptor (report `0x05` = LampRangeUpdate: update flags, first and last
  lamp id as 16-bit values, then colour channels), it is "update complete, lamps 0 to 109, colour zero", which would
  make **110 lamps** (INFERRED). Synapse's frames kept showing afterwards (CAPTURED: the stream continued at 13:45:42.278).
- 110 lamps is two more than the 108 lit matrix cells (section 4.3), and Synapse's log reported a lighting region with
  `numIndex : 106` (HOST). Lamp count, positions and key bindings need a read of the LampArray attributes (UNKNOWN).
- A 2023 Razer Insider thread reports that Windows' Dynamic Lighting, switched on by default, held this keyboard on
  solid white until it was turned off (**PUBLISHED**, community answer). Who wins between Dynamic Lighting and the
  Razer protocol without Synapse is UNKNOWN; the shared rules are in [razer-hid-protocol.md](razer-hid-protocol.md).

### 4.12 Identity reads

- `00:81`: `01 00 00 00` (OBSERVED 2026-10-31 20:57). openrazer and OpenRGB print `a[0].a[1]`, here "1.0"
  (**PUBLISHED**).
- `00:82`: an ASCII serial is present (OBSERVED; not reproduced). The USB descriptor has none (section 1.2).
- `00:86`: `01 00`, layout 1 (US) and variant 0 (CAPTURED). OpenRGB publishes the layout code table (1 US, 6 UK, 12
  Japan, and so on) and the variant byte (`00` black, `80` quartz, `82` mercury) (**PUBLISHED**,
  `Controllers/RazerController/RazerController/RazerController.h`). Synapse logged layout 1 and rejected ISO, JIS and
  F13-F24 keys for it (HOST), which agrees.

## 5. What must not be sent

- **`02:14`** (a mouse command). Synapse sends it from a shared profile item and the keyboard answers status `0x05`
  (CAPTURED, 14:01:37.664; HOST: Synapse logged the same rejection at every start).
- **`05:02` and any other class-5 or class-6 write.** They act on onboard profiles and storage whose layout is unknown
  (section 4.10).
- **`00:04` with a mode other than `00` or `03`.** Mode `02` is a factory test mode that openrazer refuses to send
  (**PUBLISHED**). Mode `03` hands the Fn layer, media keys and dial to host software (section 4.9); send it only if
  that software exists.
- **`02:02` values other than the two presets.** Unpublished, and a bad debounce makes the keyboard hard to use for the
  very commands that would fix it (INFERRED).
- **Indicator LED ids other than `0x07`, `0x08` and `0x18`, and `0x18` with state 1.** The volume/mute indicator's
  behaviour is UNKNOWN.
- **Firmware update, boot loader, storage or factory reset.** Synapse's middleware names storage-reset and factory-reset
  operations (METADATA), but their commands are not known and are not described here. No firmware-update command for
  this model is published. Do not probe for them.
- **Unknown classes and ids.** There is no safe way to learn what an undocumented command does from its reply. Keep
  an allowlist (the "Use" column of section 3).

## 6. Sequences

### 6.1 Synapse start (CAPTURED, segment 14)

| Time | Packet | Reply |
|---|---|---|
| 14:01:34.802 | `00:84` | `00` |
| .807 | `00:04` `03 00` (driver mode) | ok |
| .814 | `0F:10` `01` | ok |
| .837 | `0F:04` `00 00 FF` | ok |
| .843 | `0F:04` `01 00 FF` | ok |
| 14:01:37.209 | first frame (six `0F:03` rows) | |
| .246 | `03:00` `00 08 00` (game LED off) | ok |
| .254 | `0F:04` `01 00 FF` | ok |
| .475 | `00:40` `01 08` (1000 Hz) | busy, ok |
| .519, .524 | `00:86` twice | `01 00` |
| .529 to .642 | `05:8A`, `05:80` (each twice), `05:81` twice, `06:8E`, `06:80`, `05:02` `01`, `06:8E` | section 4.10 |
| .644 | `0F:04` `01 00 FF` | ok |
| .649 | `00:40` `01 08` | busy, ok |
| .659 | `02:14` `01 00` | status `0x05` |
| .716 on | frames, 24 per second | |

No `0F:02` was sent, and nothing went to the keyboard for the 2.4 s between the brightness writes and the first frame
(CAPTURED).

### 6.2 Synapse exit (CAPTURED, segment 13, 14:00:26)

| Time | Packet | Reply |
|---|---|---|
| .326 | last frame | |
| .329 | `03:00` `00 08 00` | ok |
| .334 | `03:00` `00 18 00` | ok |
| .339 | `0F:80` (size `0x32`) | `05 19 03 06 16` |
| .344 | `0F:82` `01 05` | `01 05 03 00 00 01 6E DA FD` |
| .349 | `0F:02` `01 00 03 00 00 00` (stored spectrum) | busy (CRC stale), ok at .357 |
| .359 | `00:84` | `03` |
| .364 | `00:04` `00 00` (normal mode) | ok |

### 6.3 Settings changes in Synapse (CAPTURED)

| Segment | Change in Synapse | Wire |
|---|---|---|
| 6 | polling 1000 → 8000 → 1000 Hz | `00:40` `01 01`, then `01 08`, each twice, busy then ok |
| 7 | brightness 100 → 50 → 100 % | `0F:04` `01 00 7F`, then `01 00 FF`, each twice, busy then ok |
| 8 | Game Mode on, off (Fn+F10) | `03:00` `00 08 01`, then `00 08 00`, once each |
| 9 | effect Wheel → Static → Wheel | frame contents only |

### 6.4 Taking the keyboard over without Synapse

What third-party code did on the bench (OBSERVED 2026-10-31 to 2026-11-04), as one workable order:

1. Reads only: `00:81`, `00:82`, `00:84`, `00:C0`, `0F:84` `01 05`, `0F:80`, `03:80` `00 08`, `02:82`.
2. If `00:84` reads `03`, offer the user one `00:04` `00 00` (it was never needed on this PC: Synapse leaves normal mode
   on a clean exit, CAPTURED).
3. If wanted, `00:40` `01 01` for 8000 Hz at every attach (section 4.6).
4. For lighting: one `0F:02` `00 00 08`, then six `0F:03` rows per frame; or a hardware effect on LED `0x05` with
   storage `0x00`.
5. On exit: a hardware effect with storage `0x00` (spectrum was used), so the keys are not left on a frozen frame.

## 7. What can be read back

| Value | Read with | Note |
|---|---|---|
| firmware, serial, layout, device mode | `00:81`, `00:82`, `00:86`, `00:84` | |
| polling rate | `00:C0` | code in `a[1]` |
| brightness | `0F:84` `01 05` | |
| keyswitch timing | `02:82` | |
| indicator LED state | `03:80` | the state, not whether anything lights (section 4.8) |
| stored effect | `0F:82` `01 05` | CAPTURED for Synapse; not read by third-party code |
| LED regions | `0F:80` | |
| frame contents | none | keep the last frame on the host |

## 8. Standalone behaviour

| Situation | Behaviour | Tag |
|---|---|---|
| Power-up without software | polling 1000 Hz, normal mode | OBSERVED 2026-11-01 (replug with Synapse stopped) |
| Keyswitch preset after power-off | kept | OBSERVED 2026-11-01 |
| Polling rate after power-off | lost, back to 1000 Hz | OBSERVED 2026-11-01 |
| Lighting with no host | the stored effect (spectrum on this unit, as Synapse leaves it) | INFERRED from Synapse's exit writes (section 6.2) |
| Effects written with storage `0x00` | shown until changed; survival of a replug not tested on this keyboard | OBSERVED 2026-11-01 (shown); UNKNOWN (replug). On the mouse they did not survive a replug ([razer-deathadder-v2.md](razer-deathadder-v2.md)) |
| Game Mode (Fn+F10), no software | blocks the Windows key | OBSERVED 2026-11-01 |
| Host stops streaming frames | the last frame stays (INFERRED; Synapse writes a stored effect at exit to avoid it) | INFERRED |
| Synapse exits cleanly | normal mode, stored spectrum | CAPTURED |

## 9. Hazards

| Hazard | Guard |
|---|---|
| 8000 Hz polling: up to eight times the interrupt completions of 1000 Hz on the host; whether the link really runs at high speed is UNKNOWN (section 1.2) | INFERRED. Raise the rate only on the user's request; the keyboard falls back to 1000 Hz at power-up anyway (OBSERVED) |
| Keyswitch timing is stored by `02:02` alone (OBSERVED), and an unpublished value can make typing unreliable | Presets only; read back after writing |
| Indicator LED accepted and read back but not lit (LED `0x08` on this unit) | Do not present an indicator as working because the write succeeded |
| Busy replies with a stale CRC (section 2.2) | Poll on a busy reply whatever its CRC; check the CRC on the final ok |
| Interleaving with another writer (Synapse streams 145 row writes per second) | One writer per keyboard; stop the vendor software first |
| Matching the wrong collection (Razer's virtual consumer device has the same usage) | Match interface 3 and usage `000C/0001` together |
| Wrong key map (Chroma-grid or openrazer generic map: one column off) | Use the device grid of section 4.3 |
| A mouse command (`02:14`) or a profile write (`05:02`) from shared code | Per-model allowlist (section 5) |
| Driver mode left on: Fn layer, media keys and dial dead | Read `00:84` at attach; offer `00:04` `00 00` |
| Windows Dynamic Lighting on interface 4 competing for the LEDs | Do not change Windows settings silently; tell the user when frames are accepted but not shown |
| Fn+F11/F12 and Fn+Menu change state behind the host (Synapse's mapping; firmware behaviour UNKNOWN) | Re-read brightness and settings; never cache them for long |
| Photosensitivity: whole-keyboard flashing from host-streamed frames | Limit full flashes to 3 per second unless the user opts in (INFERRED, general practice) |

## 10. Test vectors

Bytes `r[0..7]` (status, transaction id, two remaining-packet bytes, protocol, size, class, id), then `a[]`; every byte
not shown is `00`; the CRC is `r[88]`, `r[89]` is `00`. On Windows, prefix report ID `00` (91 bytes).

**CAPTURED** (Synapse 3, 2026-10-30):

| Id | Meaning | `r[0..7]` | `a[]` | CRC |
|---|---|---|---|---|
| T1 | set 8000 Hz | `00 1F 00 00 00 02 00 40` | `01 01` | `42` |
| T2 | its busy reply, then ok | `01 1F ...` / `02 1F 00 00 00 02 00 40` | `01 01` | `42` |
| T3 | set 1000 Hz | `00 1F 00 00 00 02 00 40` | `01 08` | `4B` |
| T4 | brightness 50 %, storage 1, LED 0 | `00 1F 00 00 00 03 0F 04` | `01 00 7F` | `76` |
| T5 | brightness 100 %, storage 1, LED 0 | `00 1F 00 00 00 03 0F 04` | `01 00 FF` | `F6` |
| T6 | brightness 100 %, storage 0, LED 0 | `00 1F 00 00 00 03 0F 04` | `00 00 FF` | `F7` |
| T7 | game LED on / off | `00 1F 00 00 00 03 03 00` | `00 08 01` / `00 08 00` | `09` / `08` |
| T8 | LED `0x18` off | `00 1F 00 00 00 03 03 00` | `00 18 00` | `18` |
| T9 | region list, reply | `02 1F 00 00 00 05 0F 80` | `05 19 03 06 16` | `85` |
| T10 | stored effect read, reply | `00 1F 00 00 00 0C 0F 82`, `02 1F 00 00 00 09 0F 82` | `01 05`; `01 05 03 00 00 01 6E DA FD` | `85`; `CB` |
| T11 | stored spectrum at exit; busy reply with stale CRC; ok reply | `00 1F 00 00 00 06 0F 02`; `01 1F ...`; `02 1F ...` | `01 00 03 00 00 00`; `01 05 03 00 00 00`; same | `09`; `09` (should be `0C`); `0C` |
| T12 | device mode read, reply driver; set normal | `02 1F 00 00 00 01 00 84`; `00 1F 00 00 00 02 00 04` | `03`; `00 00` | `86`; `06` |
| T13 | set driver mode; enable streamed frames | `00 1F 00 00 00 02 00 04`; `00 1F 00 00 00 01 0F 10` | `03 00`; `01` | `05`; `1F` |
| T14 | layout read, reply | `00 1F 00 00 00 02 00 86`, `02 1F 00 00 00 02 00 86` | `00 00`; `01 00` | `84`; `85` |
| T15 | storage query reply | `02 1F 00 00 00 0E 06 8E` | `FF FF 00 03 FF C8 00 03 DE F8 00 03 DE F8` | `B2` |
| T16 | `02:14` rejected | `05 1F 00 00 00 02 02 14` | `01 00` | `15` |
| T17 | frame row header (every captured row) | `00 1F 00 00 00 47 0F 03` | `00 00 rr 00 15` + 66 colour bytes | XOR as usual |

**OBSERVED** (sent or read by third-party code on this keyboard, 2026-10-31 to 2026-11-03; CRCs computed from the
layout):

| Id | Meaning | `r[0..7]` | `a[]` | CRC |
|---|---|---|---|---|
| T18 | firmware read; reply on this unit | `00 1F 00 00 00 02 00 81`; `02 1F ...` | none; `01 00 00 00` | `83`; `82` |
| T19 | polling read; reply 8000 Hz | `00 1F 00 00 00 01 00 C0`; `02 1F ...` | none; `00 01` | `C1`; `C0` |
| T20 | brightness read, storage 1, LED 5; reply 255 | `00 1F 00 00 00 03 0F 84`; `02 1F ...` | `01 05`; `01 05 FF` | `8C`; `73` |
| T21 | keyswitch read; reply Typing | `00 1F 00 00 00 04 02 82`; `02 1F ...` | none; `00 14 00 28` | `84`; `B8` |
| T22 | keyswitch Gaming / Typing | `00 1F 00 00 00 04 02 02` | `00 00 00 00` / `00 14 00 28` | `04` / `38` |
| T23 | wave left to right / right to left, storage 0 | `00 1F 00 00 00 06 0F 02` | `00 05 04 01 28` / `00 05 04 02 28` | `23` / `20` |
| T24 | spectrum / off, storage 0 | `00 1F 00 00 00 06 0F 02` | `00 05 03` / `00 05 00` | `0D` / `0E` |
| T25 | apply custom frame | `00 1F 00 00 00 0C 0F 02` | `00 00 08` | `09` |
| T26 | game LED read | `00 1F 00 00 00 03 03 80` | `00 08` | `88` |

**Built from the layouts** (INFERRED arithmetic):

| Id | Meaning | `r[0..7]` | `a[]` | CRC |
|---|---|---|---|---|
| T27 | row 0, all 22 cells red | `00 1F 00 00 00 47 0F 03` | `00 00 00 00 15` + 22 × `FF 00 00` | `5E` |
| T28 | row 5, all black | `00 1F 00 00 00 47 0F 03` | `00 00 05 00 15` | `5B` |
| T29 | static red, storage 0 | `00 1F 00 00 00 09 0F 02` | `00 05 01 00 00 01 FF 00 00` | `FE` |

Anti-vectors: a codec should refuse `00:04` `02 00`, any `02:14`, any `05:02`, and a `0F:03` row above 5 or a last
column above `15`; a parser that requires the reply size to equal the request size rejects T9, T10 and T12. A polling
write with `a[0] = 00` is openrazer's form and untested here.

## 11. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | The five report descriptors (61, 177, 79, 22 and 348 bytes) | a capture that includes the keyboard's enumeration |
| 2 | Bus speed; the real report rate at 8000 Hz | the same capture, with interrupt IN timing (not contents) kept |
| 3 | Each cell of section 4.3 on the device; what M1 to M4 light; the JIS cells | light one cell at a time |
| 4 | Media keys and dial in normal mode: collection, usages, dial press | a HID input viewer with Synapse absent |
| 5 | LED `0x18`: what it lights and whether a host may set it; the macro LED `0x07` | a careful bench test, one write each, with a restore |
| 6 | What `0F:10` `01` does; whether Dynamic Lighting overrides raw frames without Synapse | bench tests with Dynamic Lighting on and off |
| 7 | LampArray attributes: lamp count (110 INFERRED), positions, key bindings, minimum update interval | a read-only LampArray query |
| 8 | Meaning of `05:8A`, `05:80`, `05:81`, `05:02`, `06:80` and the `FF FF` of `06:8E`; Synapse's profile-write commands | a capture of Synapse saving a profile (Synapse is no longer installed on the observing PC) |
| 9 | Brightness write on LED `0x05`; the percent mapping beyond 50 and 100 % | one write and read; a capture of more slider positions |
| 10 | Polling rates 125 to 4000 Hz; openrazer's `a[0] = 00` form | one write and read each |
| 11 | Keyswitch units and what `02:15` adds | capture of a Synapse preset change |
| 12 | Where Synapse's "1.0.5.0" comes from | a capture of Synapse's identity reads at enumeration |
| 13 | Wave direction `00`; effect list `0F:81`; static, breathing, reactive and starlight as hardware effects | one write each |
| 14 | Whether storage-0 effects survive a replug on this keyboard | write one, replug, look |
| 15 | Fn+F9, F11, F12, Pause and Menu without software | press each with Synapse absent |
| 16 | Transaction ids `0x3F` and `0xFF` | one firmware read each |
| 17 | The purpose of interface 3's interrupt IN endpoint | a capture with that endpoint kept |
| 18 | The two bytes `19 03` of the region record | compare with other Razer devices |
