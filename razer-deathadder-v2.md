# Razer DeathAdder V2: mouse protocol (USB `1532:0084`)

Part of the [Razer peripherals protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **PUBLISHED**,
**HOST**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. Times are local time of the
capturing PC (UTC-7). "The capture" is the USBPcap session of 2026-10-30, 13:39 to 14:03, in which Synapse 3 drove the
mouse at USB device address 6 and, after a replug at 13:45:42, at address 22. The mouse's interrupt IN reports (pointer
movement and clicks) were dropped at capture time and are not in the files. "Third-party code" is independently
written software that drove the same unit on the bench, with Synapse stopped or, from 2026-11-03, uninstalled.

The shared 90-byte report (framing, CRC, status codes, transaction ids, the storage byte, the generic command catalogue
and how Windows opens the control collection) is described in [razer-hid-protocol.md](razer-hid-protocol.md) and is
not repeated here. Offsets follow that document: `r[0]` is the status byte of the 90-byte report, `r[1]` the
transaction id, `r[5]` the data size, `r[6]` the command class, `r[7]` the command id, `r[88]` the CRC; argument `a[n]`
is `r[8 + n]`. A command is written `class:id`, for example `04:05`; ids with the high bit set are reads. On Windows the
buffer carries the report ID `0x00` in front of `r[0]` (91 bytes); on the wire the SET_REPORT data stage is the 90
bytes alone.

Short version (each point is tagged in the sections below): the mouse is a composite full-speed HID device with four
interfaces. Everything Razer-specific travels as 90-byte feature reports on the boot-mouse collection of interface 0;
interface 3 is a standard HID LampArray with two lamps that Windows Dynamic Lighting uses. The mouse has two RGB LEDs,
the scroll wheel (LED `0x01`) and the logo (LED `0x04`), which Synapse addresses together as LED `0x00`. Synapse never
uses the mouse's own effects while it runs: it streams 25 two-LED frames a second and writes a stored spectrum effect
only when it exits. DPI is set with `04:05`, the five-stage DPI list with `04:06` (which this model accepts, contrary
to earlier descriptions), the polling rate with `00:05`. At every attach Synapse also rewrites the onboard profile:
stage list, DPI, brightness and 20 button mappings (`02:0C`). Writes without the storage flag apply at once and are
lost at the next power cycle.

## 1. Device

### 1.1 What the mouse is

| Item | Value | Source |
|---|---|---|
| Model | Razer DeathAdder V2 (wired), model code RZ01-03210 | **PUBLISHED**, Razer |
| Sensor | Razer Focus+ optical, up to 20 000 DPI, 650 IPS | **PUBLISHED**, Razer's announcement of 2020-01-14 |
| Buttons | eight programmable buttons; Synapse offers ten assignable inputs: left, right, wheel click, wheel up, wheel down, buttons 4 and 5, `SensitivityStageUp`, `SensitivityStageDown` and `OBMButton` (the profile button on the underside) | **PUBLISHED**, Razer; METADATA, Synapse's button-assignment list for PID 132 |
| Lighting | two RGB LEDs, scroll wheel and logo, one LED each; a separate profile LED on the underside driven only by the firmware | **PUBLISHED**, openrazer and OpenRGB; METADATA, Synapse's LED layout for PID `0x0084`; **PUBLISHED**, openrazer issue #1630 for the profile LED |
| Onboard memory | "4+1 profiles"; Synapse's log reports 92 160 bytes of storage and room for 100 macros | **PUBLISHED**, Razer; HOST, Synapse's logs; the same numbers are on the wire (section 4.8) |
| Synapse's internal name | "Lily Refresh" (folder and file names of the device definition) | METADATA |
| Size | 127.0 × 61.7 × 42.7 mm | **PUBLISHED**, Razer |

### 1.2 Descriptors

**CAPTURED (2026-10-30, 13:45:42.173 to .223)**, read by Windows when the mouse was replugged:

| Item | Value |
|---|---|
| Device descriptor | `12 01 00 02 00 00 00 40 32 15 84 00 00 02 01 02 00 01`: bcdUSB 2.00, class defined per interface, 64-byte EP0, VID `1532`, PID `0084`, bcdDevice `0200`, manufacturer string 1, product string 2, **no serial-number string** (`iSerialNumber 0`), one configuration |
| Configuration | `09 02 6D 00 04 01 00 E0 32`: 109 bytes, four interfaces, configuration value 1, attributes `E0` (self-powered and remote wake-up bits set), bMaxPower `32` = 100 mA |
| Interface 0 | `09 04 00 00 01 03 01 02 00`: HID, boot subclass, protocol mouse |
| Interface 1 | `09 04 01 00 01 03 00 01 00`: HID, no boot subclass, protocol keyboard |
| Interface 2 | `09 04 02 00 01 03 01 01 00`: HID, boot subclass, protocol keyboard |
| Interface 3 | `09 04 03 00 01 03 00 02 00`: HID, no boot subclass, protocol mouse |
| Product string | string 2 = `Razer DeathAdder V2` (40-byte descriptor, language `0409`) |

Full configuration descriptor as captured:

```
09 02 6D 00 04 01 00 E0 32
09 04 00 00 01 03 01 02 00   09 21 00 01 00 01 22 5E 00   07 05 81 03 08 00 01
09 04 01 00 01 03 00 01 00   09 21 00 01 00 01 22 9F 00   07 05 82 03 10 00 01
09 04 02 00 01 03 01 01 00   09 21 00 01 00 01 22 3D 00   07 05 83 03 08 00 01
09 04 03 00 01 03 00 02 00   09 21 11 01 00 01 22 45 01   07 05 84 03 02 00 01
```

- **Endpoints.** Each interface has one interrupt IN endpoint, `bInterval 1`: `0x81` 8 bytes, `0x82` 16 bytes, `0x83`
  8 bytes, `0x84` 2 bytes. There is no OUT endpoint, so every host-to-device report is a control transfer on endpoint 0
  (CAPTURED: all host writes in the capture were SET_REPORT requests on the default pipe).
- **Report descriptors**: 94 bytes (HID 1.00), 159 bytes (HID 1.00), 61 bytes (HID 1.00) and 325 bytes (HID 1.11)
  (CAPTURED). The same four interface classes and descriptor lengths appear in an `lsusb -v` dump of the same PID from
  2020 (**PUBLISHED**, openrazer issue #1012), so the layout has not changed across firmware since then (INFERRED).
- **Bus speed.** INFERRED full speed: the polling rate tops out at 1000 Hz, which `bInterval 1` gives at full speed,
  and the 2020 dump reports full speed (**PUBLISHED**, openrazer issue #1012). The capture does not record the speed.
- **Power attributes.** The captured `E0` sets the self-powered bit although the mouse is powered from USB. Read the
  attributes as reported, not as a power budget.
- **No USB serial.** Windows keys the device by port (HOST). The firmware's own serial (`00:82`, section 4.1) is the only
  stable identity across ports. The unit's serial is not reproduced here.

### 1.3 Report descriptors

All four were decoded from the captured bytes (CAPTURED; the decoding is INFERRED from the HID item encoding).

**Interface 0, 94 bytes, the control interface.** One application collection, Generic Desktop Mouse, no report IDs:

| Report | Size | Content |
|---|---|---|
| Input | 8 bytes | 5 button bits and 3 padding bits; 2 bytes on vendor page `0xFF00` usage `0x40` (logical −127 to 127; meaning UNKNOWN); wheel, 1 byte relative; X and Y, 16-bit relative, −32768 to 32767 |
| Feature | 90 bytes | vendor page `0xFF00`, usage `0x02`, 90 × 8 bits, item `B1 01` (Constant) |

The 90-byte feature report is the Razer control report. It sits inside the mouse application collection, so Windows
exposes it on the top-level collection `0x0001`/`0x0002` (mouse) of `MI_00`, which Windows opens exclusively for its
mouse driver. Third-party code reaches it by opening that collection with no access rights and using feature calls only
(OBSERVED 2026-10-31 to 2026-11-04; the method is in [razer-hid-protocol.md](razer-hid-protocol.md)).

**Interface 1, 159 bytes.** Five application collections with report IDs: 1 keyboard (8 modifier bits and 14 key
bytes), 2 consumer control (one 16-bit usage, 0 to `0x23C`), 3 system control (usages `0x81` to `0x83`), 4 and 5 Generic
Desktop with usage `0x00` and 15 input bytes each. These carry key and media events for buttons mapped to keys
(INFERRED). Windows lists them as `Col01` to `Col05` of `MI_01` (HOST).

**Interface 2, 61 bytes.** A standard boot keyboard: modifier byte, reserved byte, six key bytes, and a 3-bit LED
output report (Num, Caps and Scroll Lock). Windows wrote the output report `01` (Num Lock on) to it at the replug,
13:45:42.249 (CAPTURED), as it does for every keyboard collection (INFERRED).

**Interface 3, 325 bytes.** One application collection on the Lighting and Illumination page `0x59`, usage LampArray,
with seven feature reports (section 5.1). The usage page, report layouts and meanings are those of the USB-IF HUTRR84
specification (**PUBLISHED**).

### 1.4 Firmware and identity

| Read | Reply on the observed unit | Source |
|---|---|---|
| `00:81` firmware, size `0x02` | `01 00 00 00` in `a[0..3]`, which openrazer and OpenRGB print as "v1.0" (major `a[0]`, minor `a[1]`) | OBSERVED at every start of third-party code, 2026-10-31 to 2026-11-04 |
| `00:87`, size `0x04` | `01 00 03 00` | CAPTURED 13:45:43.879, sent by Synapse at attach |
| Synapse's log | "FW: 1.0.3.0" for this unit | HOST |

`00:87` returns the four-part version that Synapse displays; `00:81` returns only the first two parts (INFERRED from the
matching numbers). Use `00:87` when the full version matters. Neither command is in openrazer or OpenRGB (**PUBLISHED**,
by absence). No other firmware version of this model was examined.

### 1.5 The Windows view (HOST)

| Node | With Synapse installed (until 2026-11-03) | After Synapse was removed (2026-11-03) |
|---|---|---|
| USB `MI_00` | Razer's co-installer package | inbox |
| HID `MI_00`, mouse collection | `mouhid` with the Razer upper filter `RzDev_0084`, and a Razer Control Device child | inbox `msmouse.inf`, no filter |
| USB `MI_01` | Razer lower filter `RzDev_0084`, which also creates a virtual consumer device | inbox |
| HID `MI_02`, boot keyboard | `kbdhid`, no Razer filter | unchanged |
| HID `MI_03`, LampArray | no function driver ("HID-compliant device"), no Razer filter | unchanged |

The mouse was fully usable by third-party code on the inbox drivers, in device mode normal, after a cold boot
(OBSERVED 2026-11-03 05:12). What the Razer drivers do and how they were removed is in
[razer-software-and-drivers.md](razer-software-and-drivers.md).

## 2. Transport

### 2.1 Interfaces

| Interface | Class | Purpose | Use it? |
|---|---|---|---|
| 0 (`MI_00`) | HID boot mouse, collection `0x0001`/`0x0002` | pointer input; **every Razer command** as a 90-byte feature report | yes, feature reports only |
| 1 (`MI_01`) | HID, five collections | key, consumer and system events the firmware emits for remapped buttons | no |
| 2 (`MI_02`) | HID boot keyboard | key events in boot protocol | no |
| 3 (`MI_03`) | HID LampArray, collection `0x0059`/`0x0001` | standard lighting control (section 5) | only through the LampArray reports, never Razer reports |

openrazer binds only the interface with the mouse protocol **and** the boot subclass for this PID, because the 2020
firmware also answered on the other mouse-protocol interface (**PUBLISHED**, openrazer pull request #1093 and
`driver/razermouse_driver.c`); OpenRGB opens interface 0, usage page `0x01`, usage `0x02` (**PUBLISHED**,
`Controllers/RazerController/RazerControllerDetect.cpp`). Interface 3's descriptor declares no 90-byte report (section
5.1); whether the firmware would still accept Razer reports addressed to it is UNKNOWN. Address interface 0 only.

### 2.2 Requests

| Operation | USB request (CAPTURED) |
|---|---|
| Set feature (command) | `21 09 00 03 00 00 5A 00`: class, interface; SET_REPORT; `wValue 0x0300` (feature, report ID 0); `wIndex 0`; 90 bytes |
| Get feature (reply) | `A1 01 00 03 00 00 5A 00`: GET_REPORT; same `wValue` and `wIndex`; 90 bytes |

- **Transaction id.** Synapse sent `0x1F` in every request to the mouse, and the mouse echoed it (CAPTURED, all
  segments). Third-party code used `0x1F` throughout its bench work (OBSERVED 2026-10-31 to 2026-11-04). openrazer sends
  `0x3F` to this PID; OpenRGB switched from `0x3F` to `0x1F` in 2024 (**PUBLISHED**, openrazer
  `driver/razermouse_driver.c`; OpenRGB commit 78c91eb7). `0x3F` was not re-tested here. Razer's filter driver used
  `0x0C` and `0x08` for its own reads after enumeration (CAPTURED, section 6.1).
- **Reply timing.** Synapse read each reply about 10 ms after the request, and the mouse answered `0x02` (success) at
  the first read in every captured exchange except the first 0.4 s after enumeration, when it answered `0x01` (busy)
  (CAPTURED 13:45:42.265 to .679).
- **Frames are not read back.** Synapse sent custom frames as SET_REPORT only, without a GET (CAPTURED: 1 018 SETs and
  5 GETs to the mouse in segment 13).
- Every control transfer to the mouse in segments 5, 13 and 14 completed with USB status 0 (CAPTURED).

### 2.3 Sharing the control channel

Replies are not addressed to a sender, so two programs writing to interface 0 can read each other's replies, and
Synapse rewrites its own state at every attach (section 6). Stop Synapse and the Chroma SDK before driving the mouse.
The details (Synapse's named mutex, OpenRGB's different mutex, the service stop order) are in
[razer-hid-protocol.md](razer-hid-protocol.md) and [razer-software-and-drivers.md](razer-software-and-drivers.md).

## 3. Command set

Commands the mouse is known to accept. "Size" is `r[5]`. Status: CAPTURED = sent by Synapse; OBSERVED = sent by
third-party code; PUBLISHED = sent by an open-source project.

| Command | Purpose | Size | Section | Status |
|---|---|---|---|---|
| `00:81` | firmware version, two parts | `0x02` | 1.4 | OBSERVED; PUBLISHED |
| `00:82` | serial number, 22 ASCII bytes | `0x16` | 4.1 | CAPTURED (filter driver); OBSERVED; PUBLISHED |
| `00:84` / `00:04` | read / set device mode | `0x02` | 4.2 | CAPTURED; OBSERVED (read, and set `0x00` available); PUBLISHED |
| `00:85` / `00:05` | read / set polling rate | `0x01` | 4.3 | CAPTURED; OBSERVED; PUBLISHED |
| `00:86` | UNKNOWN (polled by the filter driver after enumeration) | `0x02`, `0x03` | 4.1 | CAPTURED |
| `00:87` | firmware version, four parts | `0x04` | 1.4 | CAPTURED |
| `04:85` / `04:05` | read / set DPI, X and Y | `0x07` | 4.4 | CAPTURED; OBSERVED; PUBLISHED |
| `04:86` / `04:06` | read / set the DPI stage list | `0x26` | 4.5 | CAPTURED (set); OBSERVED (read) |
| `0F:80` | list LED regions | `0x32` request | 4.6 | CAPTURED; OBSERVED |
| `0F:82` | read the stored effect | `0x0C` | 4.6 | CAPTURED |
| `0F:84` / `0F:04` | read / set brightness | `0x03` | 4.6 | CAPTURED (set, LED `0x00`); OBSERVED (read per LED); PUBLISHED |
| `0F:02` | set effect, including the custom-frame apply | `0x06` to `0x0C` | 4.6, 4.7 | CAPTURED; OBSERVED; PUBLISHED |
| `0F:03` | custom frame row | `0x0B` | 4.7 | CAPTURED; PUBLISHED |
| `0F:10` | UNKNOWN switch, `a[0] = 0x01` | `0x01` | 4.7 | CAPTURED only |
| `02:0C` | button mapping write | `0x0A` | 4.8 | CAPTURED only |
| `05:8A`, `05:80`, `05:81` | profile queries | `0x01`, `0x01`, `0x41` | 4.9 | CAPTURED only |
| `06:80`, `06:8E` | storage queries | `0x02`, `0x0E` | 4.9 | CAPTURED only |
| `0B:03`, `0B:0B` | sensor settings | `0x03`, `0x04` | 4.10 | CAPTURED only |

Everything else is undocumented for this model; section 8 lists what must not be sent.

## 4. Commands

### 4.1 Identity reads

- **`00:82` serial.** 22 ASCII bytes in `a[0..21]` (**PUBLISHED**, openrazer and OpenRGB). Razer's filter driver read it
  right after enumeration (CAPTURED 13:45:42.798); third-party code read it at every start (OBSERVED). The value is a
  per-unit identifier: mask it in logs. It is not reproduced here.
- **`00:86`.** Razer's filter driver sent it with size `0x03` and `0x02` (`a[]` all zero) until the mouse stopped
  answering busy, then twice more, and Synapse sent it with size `0x02` before its profile queries; every successful
  reply was `00 00` (CAPTURED 13:45:42.265 to 13:45:44.295). Its meaning is UNKNOWN. It is harmless as a read in the
  sense that Synapse sends it routinely; nothing needs it.

### 4.2 Device mode (`00:84` / `00:04`)

| `a[0]` | Mode | Who sets it |
|---|---|---|
| `0x00` | normal: the firmware handles the DPI buttons, the profile button and the onboard mappings itself | the power-on state (OBSERVED: read at every start of third-party code with Synapse stopped or removed); Synapse at exit (CAPTURED 14:00:26.480) |
| `0x03` | driver: host software is expected to handle the special buttons | Synapse at every attach and start (CAPTURED 13:45:44.468 and 14:01:34.896) |
| `0x02` | factory test mode | never; openrazer refuses to send it (**PUBLISHED**, `driver/razerchromacommon.c`) |

`a[1]` is `0x00` in every captured write and reply. Synapse reads the mode first and writes only when it changes it
(CAPTURED). Its middleware logs the driver-mode write as "DeviceMode 0x1 (Synapse)" and the release as "0x0 (Default)"
(HOST): those are Synapse's own numbers, and the wire values are `0x03` and `0x00` (CAPTURED).

What exactly stops working in driver mode without host software is UNKNOWN; the DPI and profile buttons are the likely
casualties (INFERRED from the mode's purpose). If a mouse is found in mode `0x03` and Synapse is not running, write
`00:04` `00 00` once.

### 4.3 Polling rate (`00:85` / `00:05`)

| `a[0]` | Rate |
|---|---|
| `0x01` | 1000 Hz |
| `0x02` | 500 Hz |
| `0x08` | 125 Hz |

- **Source.** Codes **PUBLISHED** by openrazer (`driver/razerchromacommon.c`). Synapse offers exactly 125, 500 and 1000
  Hz (METADATA, the device definition). CAPTURED (segment 1, 13:40:42 to 13:40:53): Synapse wrote `02` for 500 Hz and
  `01` for 1000 Hz, each write twice, about 30 ms apart, each answered `0x02`.
- **No storage byte.** The command has no storage argument. Whether the rate survives a power cycle is UNKNOWN: the
  unit always read 1000 Hz on the bench (OBSERVED), which is also the rate Synapse sets at every attach (CAPTURED).
- The newer `00:40` polling command of the Huntsman V2 was not sent to this mouse (CAPTURED); use `00:05`.
- Any other code: openrazer maps it to 500 Hz on the host side (**PUBLISHED**); what the firmware does with it is
  UNKNOWN. Send only the three codes.
- A rate change does not re-enumerate the mouse: no configuration or descriptor request followed the four writes of
  segment 1 (CAPTURED). The descriptors were read only at 1000 Hz, so whether `bInterval` changes with the rate is
  UNKNOWN.

### 4.4 DPI (`04:85` / `04:05`)

| Offset | Size | Field | Value |
|---|---|---|---|
| `a[0]` | 1 | storage | `0x00` apply only, `0x01` apply and store |
| `a[1..2]` | 2 | X | DPI, big-endian |
| `a[3..4]` | 2 | Y | DPI, big-endian |
| `a[5..6]` | 2 | reserved | `00 00` |

Read: `04:85` with `a[0] = 0x00`; the reply carries X in `a[1..2]` and Y in `a[3..4]` (**PUBLISHED**, openrazer;
OBSERVED: 2400 × 2400 at every start of third-party code).

- **Range** 100 to 20 000 in steps of 50 (METADATA: the device definition's DPI entry; **PUBLISHED**: openrazer's
  maximum of 20 000 for this model). Whether the firmware accepts values that are not multiples of 50 is UNKNOWN.
- **What Synapse sends.** For every DPI change it sends `04:05` twice with storage `0x00`, then once with `0x01`, after
  the stage list (CAPTURED 13:44:39.673 to .743, 13:44:44.900 to .960, 13:44:46.859 to .922; section 4.5).
- **Storage `0x00` works.** Third-party code wrote 800 DPI with storage `0x00`: read back 800 × 800 and the pointer
  slowed; after a replug the mouse read 2400 × 2400, the last stored value (OBSERVED 2026-11-01, 05:13 to 05:14). So
  `0x00` applies without storing.
- **X and Y separately.** 800 × 400 with storage `0x00` read back 800 × 400 and the pointer moved more slowly vertically
  (OBSERVED 2026-11-01 06:55).
- **Correction to earlier public descriptions.** openrazer's DPI builder overwrites `a[0]` with `0x01` whatever its
  caller asks for, so every DPI write openrazer sends is a storing write (**PUBLISHED**, `driver/razerchromacommon.c`).
  The storing form is not needed for an immediate change; use `0x00` for frequent changes and `0x01` only when the
  value should survive a power cycle.

### 4.5 DPI stages (`04:86` / `04:06`)

| Offset | Size | Field | Value |
|---|---|---|---|
| `a[0]` | 1 | storage | `0x01` in every captured write; `0x01` in the read request |
| `a[1]` | 1 | active stage | 1-based |
| `a[2]` | 1 | stage count | 1 to 5 |
| `a[3 + 7k]` | 1 | stage k index | 0-based |
| `a[4 + 7k]` | 2 | stage k X | big-endian |
| `a[6 + 7k]` | 2 | stage k Y | big-endian |
| `a[8 + 7k]` | 2 | reserved | `00 00` |

Size `0x26` for both directions.

- **CAPTURED (segment 4).** Switching the active stage in Synapse from 3 to 1 to 4 sent `04:06` with the whole list each
  time, then `04:05` with the new stage's value (section 4.4):

  ```
  13:44:46.838  04:06  01 04 05  00 01 90 01 90 00 00  01 03 20 03 20 00 00  02 06 40 06 40 00 00
                                 03 09 60 09 60 00 00  04 4E 20 4E 20 00 00
  13:44:46.859  04:05  00 09 60 09 60 00 00   (twice)
  13:44:46.922  04:05  01 09 60 09 60 00 00
  ```

  That is stage 4 of 5 active: 400, 800, 1600, 2400 and 20 000 DPI, X = Y. Synapse writes the same list at every attach
  and start (CAPTURED 13:45:44.880, 14:01:37.493).
- **OBSERVED (2026-11-01 06:07).** Third-party code read `04:86` with `a[0] = 0x01`: active stage 4, five stages, 400,
  800, 1600, 2400, 20 000, in the layout above. The list had survived two replugs earlier that morning, so it is stored.
- **Correction to earlier public descriptions.** openrazer enables `04:06` / `04:86` only for the DeathAdder V2 Pro and
  V2 Mini and reports "not supported" for `0x0084` (**PUBLISHED**, `driver/razermouse_driver.c`). This firmware accepts
  both (CAPTURED, OBSERVED). openrazer's builder numbers stages from 0, as Synapse does (**PUBLISHED**).
- The device definition's default list is 400, 800, 1600, 2400, 3200 (METADATA); the observed unit held a user list
  with 20 000 as stage 5.
- A stage list with storage `0x00` was never sent and is UNKNOWN. Every `04:06` seen is a stored write; treat it as a
  write to onboard memory (section 9).
- The DPI stage buttons step through this list on the device in normal mode (INFERRED from the mapping in section 4.8;
  not tested without Synapse).

### 4.6 LEDs, brightness and effects

**LED ids.**

| LED | Id | Evidence |
|---|---|---|
| Scroll wheel | `0x01` | **PUBLISHED** (openrazer `SCROLL_WHEEL_LED`, OpenRGB); METADATA (Synapse's LED layout: id `0x01` at the top of the mouse picture); CAPTURED (region list below) |
| Logo | `0x04` | **PUBLISHED** (openrazer `LOGO_LED`, OpenRGB); METADATA (id `0x04` at the bottom); CAPTURED; OBSERVED: a static green written to `0x04` lit the logo (2026-11-01 05:09) |
| Whole mouse | `0x00` | CAPTURED: Synapse's form for brightness and for the stored effect |

**`0F:80` region list.** Request size `0x32`, `a[]` zero. Reply size `0x0A`: `01 19 03 01 01 04 19 03 01 01`, two
five-byte records, LED `0x01` and LED `0x04`, each followed by `19 03 01 01` (CAPTURED 14:00:26.389; OBSERVED with
request size `0x50`, same records, at every start of third-party code). The trailing bytes are INFERRED to be a region
type and a 1 × 1 size; their exact meaning is UNKNOWN.

**Brightness `0F:04` / `0F:84`.** `a[0]` storage, `a[1]` LED, `a[2]` 0 to 255.

- Synapse writes LED `0x00` with storage `0x01` on every slider move, each value twice: 130 for 51 %, 127 for 50 %, 255
  for 100 % (CAPTURED 13:42:13.184, 13:42:16.203, 13:42:22.904). The mapping is `floor(percent × 255 / 100)`. At attach
  it writes `00 00 FF` and then `01 00 FF` (CAPTURED 13:45:44.514).
- Third-party code read `0F:84` with storage `0x01` per LED (`01 01`, `01 04`): 255 on both on 2026-11-01, but 0 on
  both on 2026-10-31 20:57 (OBSERVED). Synapse writes brightness 0 when the display turns off and 100 % when it comes
  back (HOST: Synapse's logs), so a stored 0 left from a display-off is the likely cause (INFERRED). Read the brightness
  before the first lighting write and set it if it is 0.
- Writing brightness 255 to LED `0x00` raised the per-LED reads to 255 (INFERRED from the sequence: Synapse's attach
  writes, then third-party reads), so LED `0x00` addresses both LEDs.

**Effects `0F:02`.** The layouts are those of [razer-hid-protocol.md](razer-hid-protocol.md). On this mouse:

| Form | Evidence |
|---|---|
| Per LED, storage `0x00`: static `00 04 01 00 00 01 00 FF 00` (size `0x09`), off `00 04 00` (size `0x06`) | OBSERVED 2026-11-01 05:09 to 05:10: logo green, then off, no disconnect; after a replug the logo showed the stored spectrum, so these do not persist |
| Per LED, storage `0x00`, off and spectrum (`00 01 00`, `00 04 00`, `00 01 03`, `00 04 03`) | OBSERVED 2026-11-03 08:30 to 08:36 and again on 2026-11-04: both LEDs went dark on the display-off writes; the spectrum writes followed without a disconnect |
| Per LED, storage `0x01` | **PUBLISHED**, openrazer's only form for this PID; users of openrazer's first driver reported that colours and DPI survived a replug (openrazer pull request #1093). Not sent here |
| LED `0x00`, storage `0x01`, spectrum `01 00 03 00 28 00` (size `0x06`) | CAPTURED at Synapse exit (14:00:26.421) |
| LED `0x00`, storage `0x00`, custom-frame apply `00 00 08 00 01 00` then `00 00 08 00 00 00` (size `0x06`) | CAPTURED at attach (13:45:44.609, .641); section 4.7 |
| LED `0x05` (the "backlight" id of other devices) | a hazard: **PUBLISHED** crash and disconnect reports, section 9 |

- **No hardware effect while Synapse runs.** Changing the effect in Synapse from Spectrum Cycling to Static and back sent
  no `0F:02` at all, only different frame contents (CAPTURED, segment 3, 13:42:36 to 13:43:46).
- **Spectrum speed.** Synapse's stored spectrum carries `a[4] = 0x28` (40), and its middleware logs "speed 40" for the
  effect written at release (HOST). openrazer's spectrum leaves `a[3..5]` zero (**PUBLISHED**). The meaning and range of
  `a[4]` for spectrum are UNKNOWN.
- **Effects the firmware offers.** openrazer drives off, static, breathing (random, one or two colours), spectrum and
  reactive per LED on this PID, and no wave or starlight (**PUBLISHED**, `daemon/openrazer_daemon/hardware/mouse.py`).
  The firmware's own effect list (`0F:81`) was not read (UNKNOWN).

**`0F:82` stored effect.** Request size `0x0C`, `a[0] = 0x01`, `a[1]` LED. Synapse read LED `0x01` at exit; the reply was
`00 01 03 00 28 00`: LED `0x01`, effect `0x03` (spectrum), `a[4] = 0x28` (CAPTURED 14:00:26.410). The reply's `a[0]` is
`0x00` although the request carried `0x01` (CAPTURED); what `a[0]` means in the reply is UNKNOWN. Synapse then wrote the
same effect back to LED `0x00` with storage `0x01` (section 6.5).

### 4.7 Custom frames (`0F:03`) and `0F:10`

A frame is one row with two columns:

| Offset | Size | Field | Value |
|---|---|---|---|
| `a[0..1]` | 2 | reserved | `00 00` |
| `a[2]` | 1 | row | `0x00` |
| `a[3]` | 1 | first column | `0x00` |
| `a[4]` | 1 | last column | `0x01` |
| `a[5..7]` | 3 | column 0 | R, G, B |
| `a[8..10]` | 3 | column 1 | R, G, B |

Size `0x0B`.

- **CAPTURED.** Synapse streamed this frame nonstop in every segment, about 25 per second (23.0 to 25.8 per second per
  block; for example 1 735 frames from 13:39:15.932 to 13:40:25.492). Both columns carried the same colour in every
  frame seen, so the capture does not show which column is which LED.
- **Column order.** Column 0 is the scroll wheel and column 1 the logo in OpenRGB's device table, and an OpenRGB user
  reported a blue wheel and a red logo in direct mode on this model (**PUBLISHED**, OpenRGB `RazerDevices.cpp` and
  issue #3496). Synapse's LED layout lists the wheel first (METADATA). Not checked by eye on the observed unit
  (UNKNOWN).
- **Frame size.** openrazer sends the same row with size `0x47` (**PUBLISHED**); Synapse and OpenRGB use `0x0B`
  (CAPTURED, **PUBLISHED**). Use `0x0B`.
- **Matrix shape.** openrazer's daemon declares this mouse as a 1 × 1 matrix while its kernel driver and OpenRGB use one
  row of two columns (**PUBLISHED**). The captured frames settle it: 1 row × 2 columns (CAPTURED).
- **The apply.** Frames show only after `0F:02` effect `0x08`. Synapse sent it once at the replug, as two writes with
  `a[4] = 0x01` and then `0x00` (size `0x06`, CAPTURED 13:45:44.609 and .641). After a Synapse restart the frames
  resumed without any apply, although Synapse's exit had left the stored spectrum running (CAPTURED, segments 13 and
  14; section 6.2). Send the apply once after each attach and after any hardware effect (INFERRED, the safe reading).
  openrazer and OpenRGB send the apply with size `0x0C` and `a[0..2] = 00 00 08` (**PUBLISHED**). What `a[4]` selects is
  UNKNOWN.
- **Static colours.** When Synapse showed a static colour, the frame stream paused after the fade into it; one repeat of
  the same frame followed 4.3 s later (CAPTURED 13:43:30.531 to 13:43:34.845). Frames hold until replaced (INFERRED).
- **`0F:10`, `a[0] = 0x01`.** Synapse writes it once at every attach and start, right after setting driver mode
  (CAPTURED 13:45:44.490, 14:01:34.918). It is in neither openrazer nor OpenRGB (**PUBLISHED**, by absence). INFERRED:
  the wire form of the setting Synapse's middleware logs as `DeviceConfig_ForceChromaControl Enabled : 1` (HOST), which
  gives Razer lighting priority over the LampArray host (section 5.3). Third-party frames worked on the Huntsman V2
  without it (see [razer-hid-protocol.md](razer-hid-protocol.md)); do not send it until its effect is known.
- **Rate.** The LampArray side declares a minimum update interval of 33 333 µs (section 5.2). Synapse's 25 frames per
  second stay under that rate; no faster limit is known for the Razer path (UNKNOWN).

### 4.8 Button mapping (`02:0C`)

At every attach and start, Synapse wrote 20 mappings, ten inputs on two layers (CAPTURED 13:45:45.050 to .583 and
14:01:37.652 to 14:01:38.236), each once, size `0x0A`:

| Offset | Size | Field | Captured values |
|---|---|---|---|
| `a[0]` | 1 | profile (or storage) | `0x01` in every write |
| `a[1]` | 1 | input id | `01 02 03 0B 0C 0E 09 0A 04 05`, in that order |
| `a[2]` | 1 | layer | `0x00`, then all ten again with `0x01` |
| `a[3]` | 1 | action class | `0x01`, `0x06` or `0x07` |
| `a[4]` | 1 | argument length | `0x01` in every write (INFERRED meaning) |
| `a[5]` | 1 | argument | see below |
| `a[6..9]` | 4 | | `00` |

| Input id | Class, argument | INFERRED meaning |
|---|---|---|
| `0x01`, `0x02`, `0x03` | `01`, same as the id | left, right and wheel click mapped to themselves |
| `0x04`, `0x05` | `01`, same as the id | side buttons 4 and 5 mapped to themselves |
| `0x09`, `0x0A` | `01`, same as the id | wheel up and wheel down mapped to themselves |
| `0x0B`, `0x0C` | `06`, `01` / `02` | DPI stage up / down |
| `0x0E` | `07`, `04` | the profile button's profile function |

Example, input `0x0E` on layer 1: `02:0C` `01 0E 01 07 01 04 00 00 00 00`.

- **Readings.** The seven plain input ids are the button list of Synapse's device definition, `01 02 03 09 0A 04 05`
  (METADATA), and Synapse's ten assignable inputs (section 1.1) match the ten ids (INFERRED). Layer 1 is INFERRED to be
  the Hypershift layer: Synapse's logs record its onboard profile writes "on the normal and Hypershift layers" (HOST).
  `a[0] = 0x01` matches Synapse's logged "profile:1" (HOST) and the storage byte of other commands; which of the two it
  is, is UNKNOWN.
- These were the factory-like mappings of the observed unit. Mappings to keys, macros, other mouse functions and the
  wire form of a longer argument were not captured (UNKNOWN). No key assignment was captured, and none is reproduced.
- **Every mapping write is a stored write** (INFERRED: the onboard profile keeps mappings without Synapse,
  **PUBLISHED**, openrazer issue #2438). Synapse rewrote all 20 at each attach (CAPTURED). Do not send `02:0C` without
  a reason (section 8).

### 4.9 Profile and storage queries

Synapse sent these reads at the replug, before setting driver mode (CAPTURED 13:45:44.164 to 13:45:44.387, each twice):

| Request | Reply | INFERRED meaning |
|---|---|---|
| `05:8A` size `0x01`, `a[0] = 0x00` | `05` | five profile slots ("4+1") |
| `05:80` size `0x01`, `a[0] = 0x00` | `01` | profile 1 active, or one profile stored |
| `05:81` size `0x41`, `a[]` zero | size `0x02`, `05 01` | a profile list; UNKNOWN |
| `06:80` size `0x02` | `00 00` | stored macro count 0; UNKNOWN |
| `06:8E` size `0x0E` | `00 64 00 01 68 00 00 01 68 00 00 00 00 00` | storage information: `00 64` = 100 macros maximum, `00 01 68 00` = 92 160 bytes total, `00 01 68 00` = 92 160 bytes free |

The `06:8E` reading is strong: Synapse logs exactly "max[92160], free[92160] maxmacro[100]" for this unit (HOST).
Synapse also sent `06:80` and `06:8E` between the stage list and the stored DPI at every attach and start (segments 5
and 14), and twice on its own in segment 1 (13:40:25.504 to .644) (CAPTURED).
The profile create, rename, delete and select commands were not captured (UNKNOWN). The reply to a keyboard's
active-profile read `05:84` (**PUBLISHED**, openrazer, for another model) was not tried on this mouse.

### 4.10 Sensor settings (`0B:03`, `0B:0B`)

At every attach and start Synapse wrote `0B:03` `00 04 00` twice and `0B:0B` `00 04 01 01` once (CAPTURED 13:45:44.674
to .726, 14:01:37.258 to .328), with no other sensor command. Synapse's calibration module logs "Hysteresis state OFF"
(asymmetric cut-off off) at every attach (HOST), and its device data declares a lift-off setting with ten levels and
three ten-level register tables (METADATA; the table values are Razer's tuning data and are not reproduced). INFERRED:
`0B:03` and `0B:0B` carry the lift-off and cut-off state; which byte is which is UNKNOWN. The lift-off slider,
asymmetric cut-off, surface calibration and Smart Tracking were not exercised in the capture (UNKNOWN). Do not send
`0B:xx` (section 8).

## 5. LampArray (interface 3)

### 5.1 Reports

From the captured 325-byte descriptor (CAPTURED 13:45:42.223), all feature reports:

| ID | HUTRR84 report | Bytes incl. ID | Fields |
|---|---|---|---|
| 1 | LampArrayAttributesReport | 23 | LampCount (16 bit); BoundingBoxWidth, Height, Depth, LampArrayKind, MinUpdateIntervalInMicroseconds (32 bit each) |
| 2 | LampAttributesRequestReport | 3 | LampId (16 bit) |
| 3 | LampAttributesResponseReport | 28 | LampId (16 bit); PositionX, Y, Z, UpdateLatencyInMicroseconds, LampPurposes (32 bit each); RedLevelCount, GreenLevelCount, BlueLevelCount, IsProgrammable, InputBinding (8 bit each) |
| 4 | LampMultiUpdateReport | 43 | LampCount, LampUpdateFlags (8 bit); 8 LampIds (16 bit); 8 × red, green, blue (8 bit) |
| 5 | LampRangeUpdateReport | 9 | LampUpdateFlags (8 bit); LampIdStart, LampIdEnd (16 bit); red, green, blue (8 bit) |
| 6 | LampArrayControlReport | 2 | AutonomousMode (8 bit, 0 or 1) |
| 7 | vendor page `0xFF00`, usage `0x02` | 64 | 63 bytes; purpose UNKNOWN, never seen in use |

The array declares no IntensityLevelCount and no intensity update channel: lamps take RGB only (CAPTURED descriptor).
Report 7 is not a Razer 90-byte report. Do not send it.

### 5.2 What Windows read

At the replug, within 25 ms of enumeration, Windows read the attributes and both lamps (CAPTURED 13:45:42.239):

```
GET  report 1:  01 02 00 D4 11 01 00 50 EF 01 00 EA A6 00 00 02 00 00 00 35 82 00 00
SET  report 2:  02 00 00
GET  report 3:  03 00 00 10 8B 00 00 74 59 00 00 00 00 00 00 E8 03 00 00 04 00 00 00 FF FF FF 01 00
SET  report 2:  02 01 00
GET  report 3:  03 01 00 04 8D 00 00 F8 6F 01 00 00 00 00 00 E8 03 00 00 04 00 00 00 FF FF FF 01 00
```

| Attribute | Value |
|---|---|
| Lamp count | 2 |
| Bounding box | 70.1 mm wide, 126.8 mm high, 42.73 mm deep |
| LampArrayKind | 2 (mouse) |
| Minimum update interval | 33 333 µs (30 updates per second) |
| Lamp 0 | x 35.6 mm, y 22.9 mm, z 0; update latency 1000 µs; purposes `0x04` (branding); 255 levels per colour; programmable; no input binding |
| Lamp 1 | x 36.1 mm, y 94.2 mm, z 0; otherwise as lamp 0 |

Values are little-endian micrometres (**PUBLISHED**, HUTRR84 field definitions). Lamp 0 sits near the front of the mouse,
where the scroll wheel is, and lamp 1 near the palm, where the logo is (INFERRED from the positions); the firmware's
mapping from lamps to LEDs `0x01` and `0x04` is not documented anywhere (UNKNOWN). The box length matches Razer's
127.0 mm, the depth its 42.7 mm height; the width is 70.1 mm against Razer's 61.7 mm (**PUBLISHED**, Razer).

The trailing `01 00` of report 3 is IsProgrammable = 1 and InputBinding = 0, not an intensity level count: the array
declares no IntensityLevelCount usage (CAPTURED descriptor).

### 5.3 What Windows sent, and who wins

Right after reading the attributes (CAPTURED):

| Time | Report | Bytes | Meaning (HUTRR84) |
|---|---|---|---|
| 13:45:42.239 | 6 | `06 01` | AutonomousMode on |
| 13:45:42.240 | 5 | `05 01 00 00 01 00 00 00 00` | range update, flags `0x01` (update complete), lamps 0 to 1, black |
| 13:45:42.264 | 6 | `06 00` | AutonomousMode off: the host takes the lamps |
| 13:45:42.264 | 4 | `04 02 01` + lamp ids `00 00 01 00` + 12 × `00` + colours `00 78 D4  00 78 D4` + 18 × `00` (43 bytes) | multi update, 2 lamps, flags `0x01`, lamp ids 0 and 1 (u16), then one R, G, B triple per lamp: both `#0078D4` |

No further LampArray report followed in any segment (CAPTURED). Dynamic Lighting was enabled for this mouse, with a
solid effect in the accent colour (HOST, Windows' lighting settings), and `#0078D4` is Windows' default accent
colour, so the update most likely carried that colour (INFERRED). Two seconds later Synapse set driver mode, sent `0F:10` `01` and started its frames on interface 0
(CAPTURED, section 6.1).

How the firmware arbitrates between a LampArray host with AutonomousMode off and Razer commands on interface 0 is
UNKNOWN. The capture shows only that Windows painted the lamps once and that Synapse's frames followed; what the LEDs
showed in between was not recorded. `0F:10` is the most likely switch (INFERRED, section 4.7). Use one lighting path
at a time: either turn Dynamic Lighting off for the mouse, or drive it only through LampArray.

## 6. What Synapse sends

### 6.1 Attach after a replug (CAPTURED, segment 5)

| Time | Sender | Request | Reply |
|---|---|---|---|
| 13:45:42.177 to .223 | Windows | `SET_CONFIGURATION 1`, `SET_IDLE` and report descriptor per interface | |
| 13:45:42.239 to .264 | Windows | LampArray reads and writes (section 5) | |
| 13:45:42.265 to .679 | Razer filter driver, tid `0x0C` and `0x08` | `00:86`, polled | `0x01` busy for 0.4 s, then `00 00` |
| 13:45:42.695 | filter driver | `00:85` | `01` (1000 Hz) |
| 13:45:42.726 to .785 | filter driver | `00:86` three times | `00 00` |
| 13:45:42.798 | filter driver | `00:82` | serial (not reproduced) |
| 13:45:43.868 | Synapse, tid `0x1F` from here on | `00:87` | `01 00 03 00` |
| 13:45:44.143 to .387 | Synapse | `00:86`; `05:8A`, `05:80`, `05:81`; `06:80`, `06:8E`; the same block again | section 4.9 |
| 13:45:44.447 | Synapse | `00:84` | `00 00` (normal) |
| 13:45:44.468 | Synapse | `00:04` `03 00` | driver mode |
| 13:45:44.490 | Synapse | `0F:10` `01` | |
| 13:45:44.514 | Synapse | `0F:04` `00 00 FF`, then `01 00 FF` | |
| 13:45:44.588 | Synapse | `00:86` | `00 00` |
| 13:45:44.609 | Synapse | `0F:02` `00 00 08 00 01 00`, then `00 00 08 00 00 00` | custom-frame apply |
| 13:45:44.674 | Synapse | `0B:03` `00 04 00` twice; `0B:0B` `00 04 01 01` | |
| 13:45:44.758 | Synapse | `00:05` `01` | 1000 Hz |
| 13:45:44.779 | Synapse | `04:05` `00 09 60 09 60 00 00` twice | 2400 DPI, not stored |
| 13:45:44.845 | Synapse | `0F:04` `01 00 FF` | |
| 13:45:44.880 | Synapse | `04:06` stage list, stage 4 active | |
| 13:45:44.901 | Synapse | `06:80` | `00 00` |
| 13:45:44.933 | Synapse | `04:05` `01 09 60 09 60 00 00` | 2400 DPI, stored |
| 13:45:44.954 | Synapse | `06:8E` | storage information |
| 13:45:44.986 | Synapse | `0F:04` `01 00 FF` | |
| 13:45:45.018 | Synapse | `00:05` `01` | |
| 13:45:45.050 to .583 | Synapse | 20 × `02:0C` | section 4.8 |
| 13:45:45.604 on | Synapse | `0F:03` frames, about 25 per second | |

Single frames are interleaved with the writes from 13:45:44.631 on (CAPTURED). Every write was answered `0x02`.

### 6.2 Synapse start (CAPTURED, segment 14, 14:01:34)

The same sequence from `00:84` on, without the enumeration, the filter driver's reads, `00:87`, the profile and storage
queries before `00:84`, and without the `0F:02` apply: `00:84` → `00 00`, `00:04` `03`, `0F:10` `01`, `0F:04` `00 00 FF`
and `01 00 FF`, then (from 14:01:37.258) `0B:03`, `0B:0B`, `00:05`, `04:05` twice, `0F:04`, `04:06`, `06:80`, `04:05`
stored, `06:8E`, `0F:04`, `00:05`, 20 × `02:0C`, and frames from 14:01:38.272 (2 499 frames to the end of the
segment). The mouse had been switched to its stored spectrum effect at Synapse's exit 68 s earlier (section 6.4), yet
no `0F:02` apply preceded the frames. Either frames override a running effect or `0F:10` switches the mouse to them
(INFERRED); what the LEDs showed was not recorded (UNKNOWN).

### 6.3 Settings changes (CAPTURED)

| Change in Synapse | Wire |
|---|---|
| Polling rate (segment 1) | `00:05` with the new code, twice |
| Brightness (segment 2) | `0F:04` storage `0x01`, LED `0x00`, value, twice per step |
| Effect (segment 3) | no command; only the frame colours change |
| Active DPI stage (segment 4) | `04:06` with the full list, then `04:05` storage `0x00` twice and storage `0x01` once |

So Synapse sends some writes twice (the polling rate, brightness, the first DPI write) and attach-time writes mostly
once. The mouse answered `0x02` to the first read every time; the repeat is Synapse's habit, not a requirement of the
mouse (INFERRED).

### 6.4 Synapse exit (CAPTURED, segment 13, 14:00:26)

| Time | Request | Reply |
|---|---|---|
| 14:00:26.305 | last frame | |
| 14:00:26.378 | `0F:80` | `01 19 03 01 01 04 19 03 01 01` |
| 14:00:26.399 | `0F:82` `01 01` | `00 01 03 00 28 00` (stored effect: spectrum) |
| 14:00:26.421 | `0F:02` `01 00 03 00 28 00` | spectrum, stored, LED `0x00` |
| 14:00:26.449 | `00:84` | `03 00` (driver) |
| 14:00:26.480 | `00:04` `00 00` | normal mode |

The mouse then runs its stored spectrum cycle on its own. Synapse's logs show the same release at every user-session
log-off (HOST). What happens if Synapse is killed instead (driver mode left on, frames frozen) was not captured
(UNKNOWN).

## 7. Standalone behaviour and persistence

| Setting | Survives a power cycle? | Evidence |
|---|---|---|
| Effect written with storage `0x00` | no: the stored effect returns | OBSERVED 2026-11-01 05:12 (logo back to the colour cycle after a replug) |
| Effect written with storage `0x01` | yes | **PUBLISHED**, openrazer pull request #1093 (user reports); CAPTURED: Synapse stores the spectrum it leaves running |
| Custom frames | no | INFERRED: frames are live data, and the power-on effect is the stored one (OBSERVED) |
| DPI with storage `0x00` | no: the stored DPI returns | OBSERVED 2026-11-01 05:14 |
| DPI with storage `0x01` | yes | INFERRED: after the replug the mouse read 2400, the value Synapse last stored (OBSERVED, CAPTURED) |
| DPI stage list (`04:06`, storage `0x01`) | yes | OBSERVED: read back after two replugs (2026-11-01 06:07) |
| Brightness with storage `0x01` | UNKNOWN | Synapse rewrote it at every attach, so no bench read isolates it; the per-LED reads changed between sessions (section 4.6) |
| Polling rate | UNKNOWN | the unit always read 1000 Hz (OBSERVED), which Synapse also sets at every attach (CAPTURED) |
| Device mode | the mouse powered up in normal mode on every check | OBSERVED 2026-10-31 to 2026-11-04, including the cold boot after Synapse's removal |
| Button mappings | yes | **PUBLISHED**, openrazer issue #2438 (onboard mappings work on another OS) |

After a replug, the first transaction of third-party code failed with Windows error 1167 (device not connected), and a
fresh handle to the new device instance worked 2.6 s later (OBSERVED 2026-11-01 05:12 and 05:14). Hold no state across
a reconnect; re-read DPI and re-apply any unstored setting.

The gpoulios/deathadderv2 utility, written for this model, writes DPI, the stage list, the polling rate, a static colour
and brightness to the mouse's memory (**PUBLISHED**, as described by the project); its wire forms were not compared
with the capture.

## 8. What must not be sent

| Command | Why |
|---|---|
| `00:04` with `0x02` | factory test mode; openrazer warns that the buttons then emit keystrokes and refuses it (**PUBLISHED**) |
| `00:04` with `0x03` | driver mode without host software that handles the special buttons (section 4.2) |
| `0F:02` or `0F:04` to LED `0x05`, or to any id other than `0x00`, `0x01`, `0x04` | `0x05` crashed or disconnected real units (section 9) |
| `0F:10` | effect unknown (section 4.7) |
| `02:0C` | rewrites the onboard button mapping; only the default-function form was captured (section 4.8) |
| `0B:xx` | sensor lift-off and cut-off; wrong values make tracking fail on the user's surface (INFERRED) |
| `05:xx` and `06:xx` writes | onboard profile and storage management; not captured (section 4.9) |
| Resets | Synapse's middleware names `Storage_Reset`, `Storage_Factory_Reset` and `DeviceConfig_DeviceReset` (METADATA); wire forms unknown |
| Anything firmware-update related | the mouse has a boot loader; a DeathAdder V2 owner reported a unit stuck in a mass-storage boot loader labelled "CRP DISABLD" (**PUBLISHED**, Razer Insider thread 90221), and Synapse's middleware for this PID embeds the id `1532:110D` (METADATA), which usb.ids lists as a Razer boot loader (**PUBLISHED**). Never match, open or write `1532:110D` or a drive with that label |
| Razer reports on interface 3, or LampArray report 7 | section 5.1 |

Reads that Synapse sends routinely (`00:86`, `00:87`, `05:8A`, `05:80`, `05:81`, `06:80`, `06:8E`, `0F:82`) are
harmless in that sense, but nothing in this document depends on them except `00:87`.

## 9. Hazards

| Hazard | Guard |
|---|---|
| **Effects to LED `0x05` crash or disconnect the mouse.** OpenRGB builds before July 2024 sent this PID's effects and brightness to LED `0x05` with storage `0x00`; users reported the mouse going dark with a Windows disconnect sound, a 1-2 s disconnect returning in spectrum, and modes not working (**PUBLISHED**, OpenRGB issues #3496, #2890, #1265). Commit 78c91eb7 ("Adjust LED number for Deathadder V2", 2024-07-04) moved the mouse to LED `0x00` and transaction id `0x1F`, and the issues were closed (**PUBLISHED**). Synapse itself uses LED `0x00` for brightness and its stored effect (CAPTURED), so OpenRGB's current form matches Razer's (INFERRED) | Accept only LED `0x00`, `0x01` and `0x04` in the one place every packet passes. Prefer the per-LED forms, which third-party code used without a fault (OBSERVED 2026-11-01 to 2026-11-04) |
| Flash wear from stored writes | Storage `0x01` writes onboard memory (INFERRED: it survives power loss). Synapse itself writes the stage list, DPI, brightness and 20 mappings at every attach (CAPTURED), so the memory tolerates some, but endurance is UNKNOWN. Use storage `0x00` for anything automated (per-app DPI, animations, display-off dimming); never store from a loop |
| DPI jumps | The stored stage list holds 20 000 DPI on the observed unit (OBSERVED). Clamp to 100 to 20 000 in steps of 50, and restore the DPI read at start when handing the mouse back |
| Driver mode left on | Read `00:84` before the first write; set `0x00` once if it reads `0x03` with no Synapse running (section 4.2) |
| Brightness 0 left by Synapse | Read `0F:84` per LED before the first lighting write; set 255 without storing if it is 0 (section 4.6) |
| Two lighting owners (Synapse, Chroma SDK apps, Windows Dynamic Lighting) | One owner per mouse; stop Synapse and its SDK services; turn Dynamic Lighting off for the mouse when driving it over interface 0 (section 5.3) |
| Stolen replies | Synapse and other tools on interface 0 at the same time; match transaction id, class and id of every reply, and do not run beside Synapse |
| A replug or reset returns the stored state | Treat re-arrival as a new device; re-apply unstored DPI and lighting (section 7) |
| Serial number is a unique identifier | Mask it in logs and reports |
| Photosensitivity from host-rendered flashing | Limit full flashes to 3 per second unless the user opts in (INFERRED, general practice) |

## 10. Test vectors

The 90-byte report from `r[0]`: header `r[0..7]`, arguments from `r[8]`, all other bytes `00`, CRC in `r[88]` as
defined in [razer-hid-protocol.md](razer-hid-protocol.md) (XOR of `r[2]` to `r[87]`). Transaction id `0x1F` throughout.

**CAPTURED** (Synapse 3, 2026-10-30; the CRC values are the ones on the wire):

| Id | Meaning | `r[0..7]` | Arguments | CRC |
|---|---|---|---|---|
| T1 | polling 500 Hz | `00 1F 00 00 00 01 00 05` | `02` | `06` |
| T2 | polling 1000 Hz | `00 1F 00 00 00 01 00 05` | `01` | `05` |
| T3 | brightness 50 %, both LEDs, stored | `00 1F 00 00 00 03 0F 04` | `01 00 7F` | `76` |
| T4 | brightness 100 %, both LEDs, not stored | `00 1F 00 00 00 03 0F 04` | `00 00 FF` | `F7` |
| T5 | stage list, stage 4 of 5 active | `00 1F 00 00 00 26 04 06` | `01 04 05 00 01 90 01 90 00 00 01 03 20 03 20 00 00 02 06 40 06 40 00 00 03 09 60 09 60 00 00 04 4E 20 4E 20 00 00` | `20` |
| T6 | DPI 2400, not stored | `00 1F 00 00 00 07 04 05` | `00 09 60 09 60 00 00` | `06` |
| T7 | DPI 2400, stored | `00 1F 00 00 00 07 04 05` | `01 09 60 09 60 00 00` | `07` |
| T8 | read device mode / reply in driver mode | `00 1F 00 00 00 02 00 84` / `02 1F 00 00 00 02 00 84` | `00 00` / `03 00` | `86` / `85` |
| T9 | normal mode | `00 1F 00 00 00 02 00 04` | `00 00` | `06` |
| T10 | custom-frame apply, first and second write | `00 1F 00 00 00 06 0F 02` | `00 00 08 00 01 00` / `00 00 08 00 00 00` | `02` / `03` |
| T11 | frame, both LEDs green | `00 1F 00 00 00 0B 0F 03` | `00 00 00 00 01 00 FF 00 00 FF 00` | `06` |
| T12 | region list request / reply | `00 1F 00 00 00 32 0F 80` / `02 1F 00 00 00 0A 0F 80` | none / `01 19 03 01 01 04 19 03 01 01` | `BD` / `80` |
| T13 | stored effect, wheel / reply | `00 1F 00 00 00 0C 0F 82` / `02 1F 00 00 00 0C 0F 82` | `01 01` / `00 01 03 00 28 00` | `81` / `AB` |
| T14 | spectrum, stored, both LEDs | `00 1F 00 00 00 06 0F 02` | `01 00 03 00 28 00` | `21` |
| T15 | four-part firmware / reply | `00 1F 00 00 00 04 00 87` / `02 1F 00 00 00 04 00 87` | none / `01 00 03 00` | `83` / `81` |
| T16 | storage information / reply | `00 1F 00 00 00 0E 06 8E` / `02 1F 00 00 00 0E 06 8E` | none / `00 64 00 01 68 00 00 01 68 00 00 00 00 00` | `86` / `E2` |
| T17 | mapping, left button, layer 0 (do not replay) | `00 1F 00 00 00 0A 02 0C` | `01 01 00 01 01 01 00 00 00 00` | `05` |
| T18 | LampArray attributes reply (interface 3, report 1) | | `01 02 00 D4 11 01 00 50 EF 01 00 EA A6 00 00 02 00 00 00 35 82 00 00` | |

**OBSERVED** (third-party code on the same unit; CRCs computed from the layout):

| Id | Meaning | `r[0..7]` | Arguments | CRC |
|---|---|---|---|---|
| T19 | firmware read; reply `a[0..3]` = `01 00 00 00` | `00 1F 00 00 00 02 00 81` | none | `83` |
| T20 | stage list read; reply: active 4, five stages as T5 | `00 1F 00 00 00 26 04 86` | `01` | `A5` |
| T21 | DPI read; reply `00 09 60 09 60` = 2400 × 2400 | `00 1F 00 00 00 07 04 85` | `00` | `86` |
| T22 | brightness read, logo / wheel; reply `a[2]` = `FF` | `00 1F 00 00 00 03 0F 84` | `01 04` / `01 01` | `8D` / `88` |
| T23 | DPI 800 × 400, not stored | `00 1F 00 00 00 07 04 05` | `00 03 20 01 90 00 00` | `B4` |
| T24 | logo static green, not stored | `00 1F 00 00 00 09 0F 02` | `00 04 01 00 00 01 00 FF 00` | `FF` |
| T25 | logo off, not stored / wheel off | `00 1F 00 00 00 06 0F 02` | `00 04 00` / `00 01 00` | `0F` / `0A` |

**Anti-vectors** (must be refused by an encoder):

| Id | Bytes | Why |
|---|---|---|
| T26 | `00 1F 00 00 00 09 0F 02` + `00 05 01 00 00 01 00 FF 00`, CRC `FE` | LED `0x05` (section 9) |
| T27 | `00 1F 00 00 00 02 00 04` + `03 00`, CRC `05` | driver mode without host software (section 8) |

## 11. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Which frame column is which LED, and which LampArray lamp is which LED | one frame with different colours per column; one LampArray update per lamp, with Dynamic Lighting the only owner |
| 2 | The meaning of `0F:10` and how the firmware arbitrates LampArray against Razer commands | with no Razer software running: LampArray control, then `0F:10` `01`, watching the LEDs |
| 3 | `a[4]` of the custom-frame apply and of the spectrum effect | vary one at a time on the bench |
| 4 | Whether the polling rate is stored | set 500 Hz, power-cycle, read `00:85` |
| 5 | `00:86`; the `0F:80` record bytes `19 03 01 01`; the reply's `a[0]` in `0F:82` | comparison with other Razer models; reads only |
| 6 | The profile commands (`05:xx`) and the meaning of `05:81`'s reply; profile create, select and delete | a capture of Synapse's profile pages (Synapse was removed from the observing PC after the capture) |
| 7 | Mapping a button to a key, macro or other function: the `02:0C` argument forms | the same capture |
| 8 | The lift-off and cut-off commands and the bytes of `0B:03` / `0B:0B`; calibration and Smart Tracking | a capture of the calibration page |
| 9 | Whether DPI values that are not multiples of 50 are accepted | one non-storing write of a value such as 1234, read back |
| 10 | The firmware's own effect list (`0F:81`) | one read per LED |
| 11 | What driver mode disables without host software | set `0x03` deliberately with a way back, press each button |
| 12 | The meaning of the input report's two vendor bytes (usage `0xFF00:0x40`) and of interface 1's collections 4 and 5 | an input capture, with consent, of the special buttons |
| 13 | Endurance of stored writes | none practical; keep stored writes rare |

The firmware-update path and the boot loader are deliberately not on this list.
