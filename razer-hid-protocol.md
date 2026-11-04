# Razer 90-byte HID feature-report protocol (DeathAdder V2 `1532:0084`, Huntsman V2 `1532:026C`)

Part of the [Razer peripherals protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **PUBLISHED**,
**HOST**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. Times are local time of the
capturing PC (UTC-7). "The capture" is the USBPcap session of 2026-10-30, 13:39 to 14:03, in 14 segments, in which
Razer Synapse 3 owned both devices; segments 1 to 9, 13 and 14 carry their traffic. The mouse was at USB address 6,
and at 22 after it was replugged at 13:45:42; the keyboard was at 13.

Byte numbering: `r[0]` to `r[89]` are the 90 bytes of the report as they travel on the bus. On Windows every feature
buffer starts with the report ID, so the Windows buffer is 91 bytes, `b[0] = 0x00` and `b[n + 1] = r[n]`. The
arguments are called `a[0]` to `a[79]` (`a[n] = r[8 + n]`). A command is written `CC:II`, class then id, for example
`0F:03`.

Short version (each point is tagged in the sections below): both devices take one 90-byte feature report with a
status byte, a transaction id, a size, a command class and id, 80 argument bytes and an XOR checksum. The host sends
it with SET_REPORT and fetches the answer with GET_REPORT on the same collection; the answer echoes the request with
the status filled in and the result in the arguments. Ids with bit 7 set are reads. A storage byte in most write
commands decides whether a setting is stored on the device or lost at the next power cycle. Synapse renders every
lighting effect on the PC and streams per-LED frames (`0F:03`) about 25 times a second; it selects a device-side
effect (`0F:02`) only when it exits, so that the devices run a stored spectrum cycle on their own. The devices also expose a standard HID LampArray collection for Windows
Dynamic Lighting, which is a separate path to the same LEDs.

Device-specific details are in the sibling documents: DPI, DPI stages, polling rate, button mappings and sensor
settings of the mouse in [razer-deathadder-v2.md](razer-deathadder-v2.md); the 6 x 22 key matrix, HyperPolling,
keyswitch optimisation and indicator LEDs of the keyboard in [razer-huntsman-v2.md](razer-huntsman-v2.md). The
Kraken V3 Pro headset does **not** use this protocol ([razer-kraken-v3-pro.md](razer-kraken-v3-pro.md)). Razer's
software and drivers, and how they were removed, are in [razer-software-and-drivers.md](razer-software-and-drivers.md).

## 1. Devices and sources

### 1.1 Where the reports go

| Device | USB interface | HID collection that carries the reports | Windows access to that collection |
|---|---|---|---|
| DeathAdder V2 `1532:0084` | 0 | the mouse collection (usage page `0x0001`, usage `0x0002`) | exclusive to Windows' mouse input; opens only with zero access rights |
| Huntsman V2 `1532:026C` | 3 | a consumer-control collection (usage page `0x000C`, usage `0x0001`) | shared |

- **Interface numbers (CAPTURED).** Every SET_REPORT and GET_REPORT of the protocol went to `wIndex 0` for the mouse
  and `wIndex 3` for the keyboard, from Synapse and from the other requester seen at the mouse's attach (section 4.3).
- **The mouse's descriptor (CAPTURED, 13:45:42.185).** The 94-byte report descriptor of interface 0, read by Windows
  at the replug, declares the mouse input report and, in the same collection, one vendor feature report of 90 bytes
  (usage page `0xFF00`, usage `0x02`) without a report ID. That is this protocol's report.
- **The keyboard's collection.** Interface 3 and usage `0x000C`/`0x0001` are **PUBLISHED** by openrazer (the keyboard
  driver sends this PID's reports to interface 3) and OpenRGB (its detector matches interface 3, usage page `0x0C`,
  usage `0x01`). The keyboard was not replugged during the capture, so its descriptors were not captured. With Razer's
  drivers installed, Razer's filter INF marked exactly these two collections as its control devices and installed the
  keyboard's as a raw device (HOST, before the removal of 2026-11-03).
- **Same protocol elsewhere.** The layout is shared by most Razer mice and keyboards and some accessories, and a
  Razer PWM fan controller uses it too (**PUBLISHED**, openrazer, OpenRGB, LibreHardwareMonitor). Only these two
  devices were observed.

### 1.2 What was observed

| Source | When | What it covered |
|---|---|---|
| The capture | 2026-10-30 13:39 to 14:03 | Synapse 3 driving both devices: polling, brightness, effects, DPI stages, a mouse replug, the game-mode key, Synapse exit (14:00:26) and Synapse start (14:01:34). Input reports were dropped at capture time, so no keystrokes are in it |
| Read-only probe (third-party code) | 2026-10-31 20:57 | Synapse stopped: firmware, device mode, polling rate, DPI, brightness, region list |
| Write tests (third-party code) | 2026-11-01 05:08 to 05:19, 06:07 to 06:13, 06:50 to 06:57 | effects with and without storage, replug persistence, wave direction, custom frames without `0F:10`, DPI stages read, keyswitch, polling |
| Daily use (third-party code) | 2026-11-03 onwards | both devices on Windows' own drivers after Razer's software was removed |

## 2. Transport

### 2.1 USB requests

| Operation | Setup packet (CAPTURED) | Data |
|---|---|---|
| Send a report | `21 09 00 03 ii 00 5A 00`: class, interface; SET_REPORT; `wValue 0x0300` (feature, report ID 0); `wIndex` = interface; 90 bytes | the 90-byte report, no report-ID byte |
| Fetch the answer | `A1 01 00 03 ii 00 5A 00`: GET_REPORT, feature, report ID 0, 90 bytes | the 90-byte answer |

Both go over the default control pipe (CAPTURED). openrazer sends the same two requests directly on Linux
(**PUBLISHED**).

### 2.2 Windows access

- **Buffers.** `HidD_SetFeature` and `HidD_GetFeature` take 91 bytes: `b[0] = 0x00` (the report ID) and the 90
  report bytes (OBSERVED: third-party code since 2026-10-31; **PUBLISHED**, OpenRGB and LibreHardwareMonitor use the
  same 91-byte buffer). The report ID never appears on the bus (CAPTURED: `wLength 0x5A` = 90). Take the length from
  `HidP_GetCaps` (`FeatureReportByteLength`) and refuse a collection that does not report 91 (INFERRED, defensive).
- **The mouse collection is exclusive.** Windows opens mouse and keyboard top-level collections for its own input
  handling, so `CreateFile` with read or write access fails (**PUBLISHED**, Microsoft's "Top-level collections opened
  by Windows for system use"). Open it with desired access 0 and share mode read/write: feature reports still work
  through such a handle (**PUBLISHED**, hidapi's Windows back end does exactly this; OBSERVED, third-party code drove
  the mouse that way from 2026-10-31, with Razer's filter drivers installed and, from 2026-11-03, on `msmouse.inf`).
  The keyboard's consumer collection is shared and opens either way (OBSERVED; on `hidserv.inf` since 2026-11-03,
  HOST).
- **No elevation** should be needed: a zero-access open needs no rights, and the keyboard's consumer collection is not
  held exclusively (INFERRED). While Razer's drivers were installed, their INF granted everyone read and write access
  to both control collections (HOST).
- **Select by interface and usage, never by the first VID/PID match.** Each device exposes several HID collections
  (HOST), and the mouse's interface 3 carries the LampArray collection (section 10) plus a second vendor feature report
  (section 11). While Razer's drivers were installed, virtual child devices with the same VID/PID existed, one of them
  a consumer collection like the keyboard's control collection (HOST, before 2026-11-03).
- **A replug invalidates the handle.** A feature call on the old handle failed with Win32 error 1167 (device not
  connected); after the new device appeared, a fresh open worked about 2.6 s later (OBSERVED, 2026-11-01 05:12 and
  05:14).

### 2.3 One transaction

1. SET_REPORT with status `0x00`.
2. GET_REPORT until the answer's status is no longer `0x01` (busy).

Captured spacing (Synapse's own choice, not a device requirement):

| Device | SET to first GET | GET to GET while busy | Typical busy time |
|---|---|---|---|
| mouse | 10 to 11 ms; the first GET answered `0x02` in every transaction except right after enumeration | about 15 ms (the attach-time requester) | 0.41 s once, right after enumeration (section 4.3) |
| keyboard | 2 to 3 ms | 2 to 3 ms | 8 to 12 ms for brightness, polling and effect writes; 38 ms for `05:02`; reads and most other writes answered `0x02` at the first GET |

(CAPTURED, segments 5 to 8, 13 and 14.) On the keyboard, a write that changed nothing (the second of Synapse's paired
writes, section 4.4) was answered `0x02` at once while the first had been busy, so the busy time is INFERRED to be the
time the firmware spends storing a changed value.

**Custom frames are not read back.** No GET_REPORT followed any of the tens of thousands of `0F:03` writes in the
capture (CAPTURED). The keyboard's six row writes of one frame went out about 2.5 ms apart (CAPTURED, 13:50:30.040 to
.052).

Third-party code polled every 2 ms with a 200 ms budget per transaction and waited 0.5 s before the first command
after opening; no transaction ran out of budget in its logs from 2026-10-31 to 2026-11-04 (OBSERVED). openrazer waits
600 µs before its read and retries up to five times 10 ms apart; OpenRGB waits 2 to 5 ms (**PUBLISHED**).

## 3. Report layout

### 3.1 Fields

| `r[]` | `b[]` | Size | Field | Value |
|---|---|---|---|---|
| | 0 | 1 | report ID (Windows only) | `0x00` |
| 0 | 1 | 1 | status | `0x00` in a request; the device's status in an answer (section 4.1) |
| 1 | 2 | 1 | transaction id | section 4.2; echoed in the answer |
| 2-3 | 3-4 | 2 | remaining packets | `00 00` in every captured report |
| 4 | 5 | 1 | protocol type | `0x00` |
| 5 | 6 | 1 | data size | number of meaningful argument bytes, at most 80 (`0x50`) |
| 6 | 7 | 1 | command class | section 5 |
| 7 | 8 | 1 | command id | bit 7 set = read |
| 8-87 | 9-88 | 80 | arguments `a[0]` to `a[79]` | zero beyond the data size |
| 88 | 89 | 1 | checksum | XOR of `r[2]` to `r[87]` |
| 89 | 90 | 1 | reserved | `0x00` |

- **Source.** The layout is **PUBLISHED** by openrazer (`driver/razercommon.h`, a 90-byte packed structure) and
  OpenRGB (the same with the report-ID byte in front). Every one of the captured reports of both devices fits it, with
  zero padding and a zero reserved byte (CAPTURED: the decoder flags any non-zero byte beyond the data size and found
  none).
- **Remaining packets.** openrazer and LibreHardwareMonitor treat it as big-endian, OpenRGB declares it as a native
  (little-endian) 16-bit field (**PUBLISHED**). It is zero in every captured report, so the order never mattered.
- **Class and id.** A read's id is the write's id with bit 7 set (`0F:04` / `0F:84`) in every pair in the catalogue
  (**PUBLISHED**; CAPTURED for `00:04`/`00:84`, `0F:02`/`0F:82`).

### 3.2 Checksum

XOR of every byte from `r[2]` (remaining packets) to `r[87]` (the last argument) inclusive. The status and the
transaction id are **not** included (**PUBLISHED**, openrazer `razercommon.c`; OpenRGB and LibreHardwareMonitor compute
the same range over `b[3]` to `b[88]`). Every captured request and every captured `0x02` answer carries a correct
checksum (CAPTURED).

**Worked example, CAPTURED 13:42:16.203** (Synapse sets the mouse's brightness to 50 %):

```
r[0..11]   00 1F 00 00 00 03 0F 04 01 00 7F 00     status 00, tid 1F, remaining 0000, proto 00, size 03, 0F:04,
                                                   a[0] 01 (storage), a[1] 00 (LED), a[2] 7F (127)
r[12..87]  all 00
r[88..89]  76 00
```

Checksum: `00 ^ 00 ^ 00 ^ 03 ^ 0F ^ 04 ^ 01 ^ 00 ^ 7F` = `03 ^ 0F` = `0C`; `^ 04` = `08`; `^ 01` = `09`; `^ 7F` =
**`76`**. The status `00` and the transaction id `1F` do not enter it. The answer (13:42:16.213) is the same 90 bytes
with `r[0] = 02` and the same checksum `76`.

**Busy answers carry a stale checksum (CAPTURED, 14:00:26.352).** Synapse sent the keyboard `0F:02` with a[1] = LED
`0x00` and checksum `09`. The two busy answers already showed a[1] = `0x05` but still carried `09`, which does not
match their own bytes (`0C` would); the final `0x02` answer showed a[1] = `0x05` with the correct `0C`. So the
firmware rewrites the arguments first and recomputes the checksum when it finishes (INFERRED). No public project
checks the checksum of an answer (**PUBLISHED**, by absence). Check it on `0x02` answers only.

### 3.3 What an answer contains

| Property | Finding | Tag |
|---|---|---|
| Status | filled in (section 4.1) | CAPTURED |
| Transaction id, class, id | echoed | CAPTURED: every answer of both devices |
| Remaining packets | echoed (always 0) | CAPTURED |
| Data size | **the answer's own size**, which can be smaller than the request's: `0F:80` request size `0x32`, answer `0x05` (keyboard) or `0x0A` (mouse); `0F:82` request `0x0C`, keyboard answer `0x09`; `00:84` request `0x02`, keyboard answer `0x01`; `05:81` request `0x41`, answer `0x02` | CAPTURED |
| Arguments of a write | usually an echo of the request; the keyboard rewrote LED `0x00` to `0x05` in its answer to `0F:02` (section 3.2); the mouse echoed LED `0x00` unchanged | CAPTURED, 14:00:26 |
| Arguments of a read | the result; the request's argument bytes are overwritten | CAPTURED |

**Correction to earlier public descriptions.** openrazer treats the answer as having the request's size (it rejects
an answer size above 80 but otherwise reads the request's layout). The captured answers carry their own, often
smaller, size. Parse a read's result by the answer's data size, and do not reject an answer because its size differs
from the request's.

## 4. Status, transaction ids, storage, device mode

### 4.1 Status codes

| Value | Meaning (**PUBLISHED**, openrazer and OpenRGB agree) | Seen on these devices |
|---|---|---|
| `0x00` | new command (request) | in every request (CAPTURED) |
| `0x01` | busy | keyboard writes (8 to 38 ms), the mouse for 0.41 s after enumeration (CAPTURED) |
| `0x02` | successful | the normal answer (CAPTURED) |
| `0x03` | failure | never seen |
| `0x04` | no response / timeout | never seen |
| `0x05` | not supported | keyboard `02:14` at Synapse start, 14:01:37.664 (CAPTURED) |

- **Busy is not success for a read.** openrazer accepts `0x01` as success ("some commands respond with busy but
  succeed") and parses its arguments (**PUBLISHED**). The captured busy answers carried rewritten arguments with a
  stale checksum (section 3.2). Keep polling while the status is `0x01`; parse arguments only from `0x02`.
- **A write still busy after a long budget** was probably applied (openrazer's reading, **PUBLISHED**). No write in the
  capture or the third-party logs stayed busy longer than 38 ms.
- **`0x05` is final.** The keyboard answered `02:14` with `0x05` at once; Synapse did not retry (CAPTURED). Treat it
  as "this firmware lacks the command" and stop sending it.

### 4.2 Transaction ids

| Requester | Ids | Tag |
|---|---|---|
| Synapse 3 | **`0x1F`** for every request to both devices | CAPTURED: all segments |
| An attach-time requester other than Synapse | `0x0C` and `0x08`, interleaved, right after the mouse's enumeration (section 4.3) | CAPTURED, 13:45:42.265 to .813 |
| openrazer | DeathAdder V2: `0x3F` for every command. Huntsman V2: `0x1F` for lighting, polling, keyswitch and game LED; `0xFF` for firmware, serial, device mode, layout and the macro LED | **PUBLISHED** |
| OpenRGB | DeathAdder V2: `0x1F` (since a 2024 change; `0x3F` before). Huntsman V2: `0x3F` | **PUBLISHED** |
| Third-party code | `0x1F` for everything, on both devices | OBSERVED, 2026-10-31 onwards |

- **The id is echoed** in every answer, including the attach-time requester's `0x0C` and `0x08` (CAPTURED).
- **What the mouse accepts:** `0x1F`, `0x0C` and `0x08` (CAPTURED). **The keyboard:** only `0x1F` was seen. Whether
  either device answers `0x3F` or `0xFF` the same way was not tested here; openrazer and OpenRGB use those ids with
  these PIDs in daily use (**PUBLISHED**), so they are probably accepted (INFERRED). An id of `0x00` is never valid:
  openrazer rewrites it to `0xFF` with a warning (**PUBLISHED**).
- **Use the id to match answers.** It is the only field that tells two requesters apart when both send the same
  command (section 4.3). OpenRGB filters answers by id, class and id for this reason (**PUBLISHED**).

### 4.3 One request at a time: the attach-time reads

At the mouse's replug (CAPTURED, segment 5), 80 ms after Windows configured the device and before Synapse's first
request at 13:45:43.868, two requesters read the mouse:

| Time | Request | Answer |
|---|---|---|
| 13:45:42.265 | tid `0C` `00:86` size 3 | busy, polled about every 15 ms |
| .336, .357, .378, .400, .421, .543, .564, .586, .607, .628 | tid `08` `00:86` size 2, sent ten times | every GET still returned the tid `0C` request, busy |
| .679 | | tid `0C` `00:86`: `0x02`, `00 00 00` (0.41 s after the request) |
| .695 / .726 | tid `0C` `00:85` (answer `01` = 1000 Hz), `00:86` | `0x02` |
| .753 / .775 / .798 | tid `08` `00:86` twice, then `00:82` (serial) | `0x02`; the serial is not reproduced |

- **The device holds one request.** While the `0x0C` request was busy, the `0x08` requester's SETs got no answer of
  their own: every GET returned the pending `0x0C` request (CAPTURED). Whether the firmware discards or queues a SET
  that arrives while it is busy is UNKNOWN; the `0x08` requester behaved as if it was discarded and resent.
- **Who the requesters were.** Synapse used `0x1F` throughout. Razer's filter driver was installed on the mouse's
  control collection at the time (HOST), so these reads are INFERRED to be the filter driver's own; they are gone with
  Razer's drivers. See [razer-software-and-drivers.md](razer-software-and-drivers.md).
- **Settle after enumeration.** The mouse answered busy for 0.41 s after enumeration (CAPTURED). A host should wait
  about half a second after a device appears before its first command, or poll patiently (INFERRED).

### 4.4 Synapse sends user changes twice

Synapse sent each change made in its window as two identical SET/GET pairs 20 to 30 ms apart, each read back
(CAPTURED: polling, brightness, DPI on the mouse; polling and brightness on the keyboard). The game-mode LED `03:00`, the
lighting frames and the writes of its attach sequence (`00:04`, `0F:10`, `04:06`, the stored `04:05`, `0B:0B`,
`02:0C`) went out once (CAPTURED, segments 5 and 8). The reason is UNKNOWN; the protocol does not require it
(OBSERVED: third-party code sends each write once and reads the value back).

### 4.5 The storage byte

Most write commands carry `a[0]` = storage (**PUBLISHED**, openrazer and OpenRGB):

| Value | Name (openrazer / OpenRGB) | Meaning |
|---|---|---|
| `0x00` | `NOSTORE` / `RAZER_STORAGE_NO_SAVE` | applies now; lost at the next power cycle |
| `0x01` | `VARSTORE` / `RAZER_STORAGE_SAVE` | applies now and is stored on the device (profile 1 on devices with onboard profiles, per openrazer) |

- **Unstored writes do not survive a replug (OBSERVED, 2026-11-01).** A static green and then "off" on the mouse's logo
  LED with storage `0x00`, and DPI 800 with storage `0x00`, all took effect; after a replug the logo showed the stored
  spectrum cycle again and the DPI read 2400 again (05:10 to 05:14).
- **Stored values do survive.** The spectrum cycle Synapse writes with storage `0x01` at exit (section 12.2) is what
  the mouse showed after its replug, and the DPI 2400 that Synapse had written with storage `0x01` (CAPTURED,
  13:45:44.933) came back after the replug (OBSERVED, 2026-11-01 05:12 and 05:14).
- **Synapse uses `0x01` routinely** (CAPTURED): brightness on every slider move, the DPI stage list and the final DPI of
  every stage change, the effect at exit; `0x00` for the first brightness and the first two DPI writes of an attach.
  So Synapse writes the stored copy many times a day. Whether `0x01` is a flash write, and the wear budget, are
  UNKNOWN; a few stored writes per minute is what Razer's own software does (INFERRED: keep stored writes to explicit
  user changes, never in animation or curve loops).
- **Not every command has the byte, and not every stored-looking byte stores.** The keyboard's keyswitch write `02:02`
  has no storage byte and still persisted through a replug (OBSERVED, 2026-11-01 06:11). The keyboard's HyperPolling
  write `00:40` carries `a[0] = 0x01` from Synapse, yet the keyboard powered up at 1000 Hz after a replug although
  Synapse had set 8000 Hz (OBSERVED, 2026-11-01 06:11). Details in [razer-huntsman-v2.md](razer-huntsman-v2.md).
- **A read takes the byte too.** `0F:82` and `0F:84` are sent with `a[0] = 0x01` to read the stored value (CAPTURED for
  `0F:82`; **PUBLISHED**, openrazer for `0F:84`). The mouse answered `0F:82` with `a[0] = 0x00` even though the request
  had `0x01`; the keyboard echoed `0x01` (CAPTURED, 14:00:26).

### 4.6 Device mode: `00:84` and `00:04`

| Command | Size | Arguments | Answer |
|---|---|---|---|
| `00:84` get device mode | `0x02` | none | `a[0]` mode, `a[1]` parameter; the keyboard answers with size 1 (mode only) |
| `00:04` set device mode | `0x02` | `a[0]` mode, `a[1] = 0x00` | echo |

| Mode | Meaning (**PUBLISHED**, openrazer) | Seen |
|---|---|---|
| `0x00` | normal ("hardware"): the firmware handles its own special functions | read at every attach and Synapse start; written at Synapse exit (CAPTURED) |
| `0x02` | undocumented factory-test mode; openrazer refuses to send it | never |
| `0x03` | driver ("software"): special functions are reported to host software instead | written by Synapse at every attach and start (CAPTURED) |

- **CAPTURED sequence.** Attach and start: `00:84` (answer `0x00`), then `00:04 03 00`, then `0F:10 01` (section 8.4).
  Exit: the stored effect is written back (section 12.2), then `00:84` (answer `0x03`), then `00:04 00 00`. This settles
  that Synapse's "Driver Mode", logged in its own files as `DeviceMode:0x1`, is wire value `0x03` (CAPTURED; HOST for the
  log names).
- **Driver-mode trap.** In mode `0x03` the special functions depend on the host program that set it. If that program
  is killed instead of exiting, the device stays in `0x03` until a replug or a power cycle (INFERRED from the exit
  sequence and from openrazer's note that a suspend resets the mode, **PUBLISHED**). A host that does not implement
  those functions should read the mode at start and write `0x00` if it finds `0x03`.
- **After a clean exit both devices read `0x00`** (OBSERVED, 2026-10-31 20:57 and at every start of third-party code
  since).

## 5. Command catalogue

Only the commands shared by both devices, or seen on both, are described here. Tag per row; "PUBLISHED" alone means
neither the capture nor the bench exercised it on these devices.

| Command | Name | Request size | Arguments / answer | Mouse | Keyboard | Tag |
|---|---|---|---|---|---|---|
| `00:81` | get firmware version | `0x02` | answer `a[0]` major, `a[1]` minor | yes | yes | **PUBLISHED**; OBSERVED: both answered `01 00 00 00` in `a[0..3]` ("v1.0") on 2026-10-31. Not sent by Synapse in the capture |
| `00:82` | get serial number | `0x16` | answer 22 ASCII bytes | yes | yes | **PUBLISHED**; CAPTURED (mouse, tid `08`, value not reproduced); OBSERVED |
| `00:84` / `00:04` | get / set device mode | `0x02` | section 4.6 | yes | yes | **PUBLISHED**; CAPTURED; OBSERVED (read) |
| `00:85` / `00:05` | get / set polling rate, classic | `0x01` | 1000 / 500 / 125 Hz codes | yes | no | see [razer-deathadder-v2.md](razer-deathadder-v2.md) |
| `00:C0` / `00:40` | get / set polling rate, HyperPolling | `0x01` / `0x02` | up to 8000 Hz | no | yes | see [razer-huntsman-v2.md](razer-huntsman-v2.md) |
| `00:86` | get layout (openrazer: keyboard layout) | `0x02` (also `0x03` from the attach-time requester) | answer `a[0]` layout | answered `00 00` | answered `01 00` | **PUBLISHED** (keyboard); CAPTURED for both |
| `00:87` | unknown read | `0x04` | mouse answer `01 00 03 00` | yes | not seen | CAPTURED (13:45:43.879); meaning UNKNOWN |
| `02:02` / `02:82` | keyswitch optimisation | `0x04` | | no | yes | see [razer-huntsman-v2.md](razer-huntsman-v2.md) |
| `02:0C` | button mapping write | `0x0A` | | yes | no | CAPTURED; do not send (section 11) |
| `02:14` | unknown write | `0x02` | Synapse sends `01 00`; the keyboard answers `0x05` | not seen | not supported | CAPTURED (14:01:37.664) |
| `03:00` / `03:80` | set / get LED state | `0x03` | `a[0]` storage, `a[1]` LED id, `a[2]` 0 or 1 | no | yes (LEDs `0x08`, `0x18`) | **PUBLISHED**; CAPTURED; OBSERVED; see [razer-huntsman-v2.md](razer-huntsman-v2.md) |
| `04:xx` | DPI and DPI stages | | | yes | no | see [razer-deathadder-v2.md](razer-deathadder-v2.md) |
| `05:02` | write, INFERRED "set active profile" | `0x01` | Synapse sends `01`; busy 38 ms, then `0x02` | not seen | yes | CAPTURED (14:01:37.584); do not send |
| `05:80` | read, INFERRED "active profile" | `0x01` | answer `01` | yes | yes | CAPTURED |
| `05:81` | read, INFERRED profile query | `0x41` (all zero) | answer size 2: mouse `05 01`, keyboard `01 01` | yes | yes | CAPTURED |
| `05:8A` | read, INFERRED profile count | `0x01` | answer `05` | yes | yes | CAPTURED |
| `06:80` | read, INFERRED storage query | `0x02` | answer `00 00` | yes | yes | CAPTURED |
| `06:8E` | read, INFERRED storage query | `0x0E` | mouse `00 64 00 01 68 00 00 01 68 00 00 00 00 00`; keyboard `FF FF 00 03 FF C8 00 03 DE F8 00 03 DE F8` | yes | yes | CAPTURED; meaning UNKNOWN |
| `0B:03`, `0B:0B` | sensor settings | | | yes | no | CAPTURED; see [razer-deathadder-v2.md](razer-deathadder-v2.md); do not send |
| `0F:02` | set effect | `0x06` + 3 x colours | section 7 | yes | yes | **PUBLISHED**; CAPTURED (apply and exit effect only); OBSERVED |
| `0F:82` | get stored effect | `0x0C` | section 7.3 | yes | yes | CAPTURED (Synapse exit) |
| `0F:03` | custom frame row | `0x0B` mouse, `0x47` keyboard | section 8 | yes | yes | **PUBLISHED**; CAPTURED; OBSERVED |
| `0F:04` / `0F:84` | set / get brightness | `0x03` | section 9 | yes | yes | **PUBLISHED**; CAPTURED (set); OBSERVED (get) |
| `0F:80` | list LED regions | `0x32` (Synapse) or `0x50` (openrazer, OpenRGB) | section 6.2 | yes | yes | **PUBLISHED**; CAPTURED; OBSERVED |
| `0F:81` | list effects of a region | `0x50` | `a[0]` region; answer `a[1..]` effect ids | UNKNOWN | UNKNOWN | **PUBLISHED** (OpenRGB) only |
| `0F:10` | unknown write, INFERRED "chroma control on" | `0x01` | `01` | yes | yes | CAPTURED; do not send (section 8.4) |

Commands seen in no source for these devices (`00:87`, `02:14`, `05:xx`, `06:xx`, `0F:10`) are named from context only;
their names are INFERRED.

## 6. LED ids and regions

### 6.1 LED ids

`a[1]` of `03:00`, `0F:02`, `0F:04` and their reads (**PUBLISHED**, openrazer and OpenRGB, identical lists):

| Id | Name | On these devices |
|---|---|---|
| `0x00` | whole device ("zero LED") | Synapse's brightness on both devices and its exit effect on both (CAPTURED); the custom-frame apply on both (CAPTURED; **PUBLISHED**) |
| `0x01` | scroll wheel | mouse (region list, CAPTURED; per-LED effects OBSERVED) |
| `0x04` | logo | mouse (region list, CAPTURED; static and off OBSERVED, 2026-11-01) |
| `0x05` | backlight | keyboard key matrix (region list, CAPTURED; openrazer's and OpenRGB's id for this keyboard's brightness and effects, **PUBLISHED**) |
| `0x07` | macro indicator | keyboard (**PUBLISHED**, openrazer); not seen |
| `0x08` | game-mode indicator | keyboard `03:00` (CAPTURED, OBSERVED); no light on this unit (OBSERVED, 2026-11-01) |
| `0x18` | not in openrazer's list | keyboard `03:00` off at Synapse exit, answered `0x02` (CAPTURED, 14:00:26.334); INFERRED a volume indicator, see [razer-huntsman-v2.md](razer-huntsman-v2.md) |

**LED `0x00` against per-LED ids.** openrazer addresses this mouse's two LEDs separately (`0x01`, `0x04`) and the
keyboard as `0x05`; OpenRGB addresses this mouse with `0x00` (**PUBLISHED**). Synapse uses `0x00` for brightness and
the stored effect on both devices (CAPTURED). Third-party code used the per-LED ids for effects with storage `0x00`
and saw no fault (OBSERVED, 2026-11-01). How the firmware combines a `0x00` setting with per-LED settings is UNKNOWN
(open item 6). Public reports of this mouse crashing on effect commands are discussed in
[razer-deathadder-v2.md](razer-deathadder-v2.md).

### 6.2 `0F:80` region list

Request: `0F:80` with no arguments. Synapse sends data size `0x32`; openrazer and OpenRGB `0x50` (**PUBLISHED**); both
were answered (CAPTURED 14:00:26; OBSERVED 2026-10-31 with `0x50`).

| Device | Answer (CAPTURED 14:00:26) | Reading |
|---|---|---|
| mouse | size `0x0A`: `01 19 03 01 01  04 19 03 01 01` | region `0x01` and region `0x04`, each 1 row x 1 column |
| keyboard | size `0x05`: `05 19 03 06 16` | region `0x05`, 6 rows x 22 columns |

Each record is 5 bytes: region (LED) id, two bytes `19 03` of UNKNOWN meaning (the same on both devices), rows,
columns (INFERRED from the keyboard's 6 x 22 matrix, **PUBLISHED** by both projects, and the mouse's two single LEDs).
OpenRGB reads the list as records until an id of `0x00` (**PUBLISHED**); the answer's data size also bounds it
(section 3.3).

## 7. Hardware effects: `0F:02` and `0F:82`

### 7.1 Layout

| Arg | Field | Value |
|---|---|---|
| `a[0]` | storage | section 4.5 |
| `a[1]` | LED id | section 6.1 |
| `a[2]` | effect | table below |
| `a[3]` | flags | sub-mode or direction |
| `a[4]` | rate | speed or duration |
| `a[5]` | colour count | 0 to 2 |
| `a[6]..` | colours | R, G, B per colour |

Data size `0x06 + 3 x colours` (**PUBLISHED**, OpenRGB's documentation of the "extended" effect and every openrazer
constructor for class `0x0F`).

### 7.2 Effect codes

| `a[2]` | Effect | `a[3]` | `a[4]` | Colours | Tag |
|---|---|---|---|---|---|
| `0x00` | none (off) | 0 | 0 | 0 | **PUBLISHED**; OBSERVED (mouse logo, 2026-11-01 05:10; both devices at display-off, 2026-11-03) |
| `0x01` | static | 0 | 0 | 1 | **PUBLISHED**; OBSERVED (mouse logo green, storage `0x00`, 2026-11-01 05:10) |
| `0x02` | breathing | `0x00` random, `0x01` one colour, `0x02` two colours | 0 | 0, 1 or 2 | **PUBLISHED** |
| `0x03` | spectrum cycle | 0 | speed: Synapse writes `0x28` (40) on the mouse, `0x00` on the keyboard | 0 | **PUBLISHED**; CAPTURED (exit); OBSERVED |
| `0x04` | wave | direction (section 7.4) | speed, `0x28` | 0 | **PUBLISHED**; OBSERVED on the keyboard; not offered for this mouse by openrazer |
| `0x05` | reactive | 0 | speed 1 to 4 | 1 | **PUBLISHED** |
| `0x06` | ripple | | | | **PUBLISHED** as an id by OpenRGB only; openrazer renders ripple on the host |
| `0x07` | starlight | 0 | speed 1 to 3 | 0, 1 or 2 | **PUBLISHED** (keyboard) |
| `0x08` | custom frame (apply) | 0 | see section 8.2 | 0 | **PUBLISHED**; CAPTURED; OBSERVED |
| `0x09` | fire | | | | **PUBLISHED** as an id by OpenRGB only |
| `0x0A` | wheel | direction 1 or 2 | `0x28` | 0 | **PUBLISHED** (another keyboard) |

- **Synapse never sends a device effect while it runs.** Its Spectrum Cycling, Wheel and Static effects all went out
  as `0F:03` frames; `0F:02` appeared only as the custom-frame apply at the mouse's attach and as the stored effect at
  exit (CAPTURED, segments 3 and 9).
- **Other effect families.** openrazer also knows class `0x03` effects (`03:0A`, `03:0B`, `03:0C`) for older devices
  and a mouse-specific `03:0D` form with a different argument order (**PUBLISHED**). They share effect names, not
  layouts. These two devices take the class `0x0F` form (**PUBLISHED**; CAPTURED); do not mix the families.
- `0F:81` is OpenRGB's way of asking which effect ids a region supports (**PUBLISHED**). It was not seen in the capture
  or used on the bench.

### 7.3 `0F:82` get stored effect

Request `0F:82`, size `0x0C`, `a[0] = 0x01`, `a[1]` = LED. CAPTURED at Synapse exit (14:00:26):

| Device | Request | Answer |
|---|---|---|
| mouse | `01 01` (LED `0x01`) | size `0x0C`: `00 01 03 00 28 00 00 00 00 00 00 00` = spectrum, rate 40, no colours (storage echoed as `0x00`) |
| keyboard | `01 05` (LED `0x05`) | size `0x09`: `01 05 03 00 00 01 6E DA FD` = spectrum, rate 0, one colour `6E DA FD` |

Synapse then wrote the same effect back with LED `0x00` and storage `0x01` (section 12.2). The stored colour in the
keyboard's spectrum record has no visible use (INFERRED: left over from an earlier effect).

### 7.4 Wave direction

On the keyboard, wave `a[3] = 0x01` runs **left to right** and `a[3] = 0x02` runs **right to left** (OBSERVED,
2026-11-01 05:15 and 05:16, rate `0x28`, storage `0x00`). openrazer clamps the direction to 0..2 and notes that some
devices use 0/1 and others 1/2 (**PUBLISHED**); `0x00` was not tried.

## 8. Custom frames: `0F:03`

### 8.1 Row layout

| Arg | Field | Value |
|---|---|---|
| `a[0]` | reserved | `0x00` |
| `a[1]` | reserved | `0x00` |
| `a[2]` | row | 0 to rows - 1 |
| `a[3]` | first column | |
| `a[4]` | last column, inclusive | |
| `a[5]..` | colours | R, G, B per column |

- **Source.** **PUBLISHED** by openrazer and OpenRGB; CAPTURED in every frame.
- **Data size.** Synapse sends exactly `5 + 3 x columns`: `0x0B` for the mouse's 1 x 2, `0x47` for the keyboard's
  22 columns (CAPTURED). openrazer always sends `0x47` ("most devices are happy with 0x47"), OpenRGB sends the exact
  size (**PUBLISHED**). Both agree for the keyboard; for the mouse only `0x0B` was seen.
- **At most 25 columns** fit in one row write ((80 - 5) / 3, INFERRED arithmetic).
- **No storage byte.** Frames are volatile (INFERRED: nothing in the row can ask for storage, and the devices showed
  their stored effect after every replug).

**CAPTURED examples:**

```
mouse, 13:42:16.224 (one frame; scroll wheel and logo both 16 65 FF):
  r[0..18]  00 1F 00 00 00 0B 0F 03  00 00 00 00 01  16 65 FF  16 65 FF      checksum r[88] = 06

keyboard, 13:50:30.001 (row 3 of a Wheel frame, 22 columns):
  r[0..12]  00 1F 00 00 00 47 0F 03  00 00 03 00 15
  colours   E3 FF 00 x 10, 9A 33 FF, 9A 33 FF, 00 00 00, 9A 33 FF, 00 00 00, 00 00 00, 00 00 00,
            9A 33 FF, 9A 33 FF, 9A 33 FF, 00 00 00, 00 00 00                  checksum r[88] = 5D
```

### 8.2 Switching a device to frames: the apply

`0F:02` with effect `0x08`, storage `0x00`, LED `0x00` makes the device show the frames (**PUBLISHED**, openrazer and
OpenRGB send it after the rows).

| Who | Form | Tag |
|---|---|---|
| Synapse, at the mouse's attach | size `0x06`: `00 00 08 00 01 00`, then one frame, then `00 00 08 00 00 00` (`a[4]` 1 then 0) | CAPTURED, 13:45:44.609 and .641 |
| openrazer | size `0x0C`, `a[0..2] = 00 00 08`, rest 0, after every frame | **PUBLISHED** |
| OpenRGB | after every frame | **PUBLISHED** |
| Third-party code | size `0x0C`, once per attach, then rows only | OBSERVED, 2026-11-01 05:16 (keyboard) |

- **Once is enough.** After the one apply, Synapse streamed rows only; after Synapse's restart at 14:01:34 it streamed
  frames to both devices without any apply at all (CAPTURED). Third-party code that applied once and then sent rows
  only got a correct, smooth display (OBSERVED, 2026-11-01). The meaning of Synapse's `a[4]` 1-then-0 is UNKNOWN.
- **Frames work without `0F:10`** (OBSERVED, 2026-11-01 05:16 to 05:18: the keyboard showed six static colour blocks
  and a 20-second breathing animation from frames, with no `0F:10` sent since Synapse's exit).

### 8.3 Frame rates Synapse streams

| Device | Rows per frame | Frames per second | Tag |
|---|---|---|---|
| mouse | 1 (row 0, 2 LEDs) | 25.0 (23 to 25.8 per segment) | CAPTURED |
| keyboard | 6 (rows 0 to 5, 22 columns each) | 24.4 (24.2 to 25.1 per segment); rows about 2.5 ms apart | CAPTURED |

- The stream runs nonstop while Synapse runs, about 147 keyboard reports and 25 mouse reports a second, whatever the
  effect (CAPTURED, all segments). For a static colour Synapse went on sending identical frames, at a lower rate
  (CAPTURED, 13:43:30 to 13:43:36).
- These are Synapse's rates, not device limits. The fastest safe rate is UNKNOWN (open item 9). Third-party code
  capped its frames at 30 per second without a visible problem (OBSERVED, 2026-11-01).

### 8.4 `0F:10`

`0F:10` with size `0x01` and `a[0] = 0x01`, answered `0x02` on both devices (CAPTURED). Synapse sends it right after
`00:04 03` at every attach and every start (13:45:44.490, 14:01:34.814 and .918) and never sends a `0x00` form, not
even at exit (CAPTURED). Neither openrazer nor OpenRGB has the command (**PUBLISHED**, by absence).

Synapse's logs record `DeviceConfig_ForceChromaControl Enabled : 1` once per attach (HOST, Synapse's logs, before
the removal), so `0F:10 01` is INFERRED to be that setting: a firmware flag that makes the device follow Razer
commands rather than Windows Dynamic Lighting (section 10). Frames work without it (section 8.2). What it changes,
whether it persists, and its read form are UNKNOWN. Do not send it.

## 9. Brightness: `0F:04` and `0F:84`

| Command | Size | Arguments | Answer |
|---|---|---|---|
| `0F:04` set | `0x03` | `a[0]` storage, `a[1]` LED id, `a[2]` brightness 0 to 255 | echo |
| `0F:84` get | `0x03` | `a[0]` storage, `a[1]` LED id | `a[2]` brightness |

- **CAPTURED values.** Synapse's slider is in percent and goes out as `floor(percent x 255 / 100)`: 100 % = `0xFF`,
  51 % = `0x82` (130), 50 % = `0x7F` (127) (13:42:13 to 13:42:22 on the mouse, 13:49:24 to 13:49:28 on the keyboard).
  The 50 % value shows truncation, not rounding (127.5 became 127; INFERRED from the three points).
- **LED `0x00`, storage `0x01`, on both devices** for every slider move; at attach and start, `0x00` storage first,
  then `0x01`, both 255 (CAPTURED). openrazer and OpenRGB address the keyboard's brightness as LED `0x05` and the
  mouse's per LED (**PUBLISHED**).
- **Maximum 255** on both devices (CAPTURED; **PUBLISHED**).
- **Reading per LED (OBSERVED).** `0F:84` with storage `0x01` read 255 for the keyboard's LED `0x05` every time. For the
  mouse's LEDs `0x01` and `0x04` it read 0 on 2026-10-31 20:57 and 255 on 2026-11-01 05:09, while the LEDs were lit
  both times. What the per-LED value means next to Synapse's LED-`0x00` brightness is UNKNOWN (open item 6).
- Synapse's lights-off behaviour (for example at display sleep) is a brightness write of 0 and back (HOST, Synapse's
  logs), not a separate command.

## 10. Windows Dynamic Lighting (HID LampArray)

Both devices also implement the standard HID LampArray interface (usage page `0x59`) that Windows Dynamic Lighting
uses: the mouse on interface 3, the keyboard on interface 4 (HOST; CAPTURED for the mouse's descriptor). It is a
different path to the same LEDs, with standard reports defined by the HID usage tables and Microsoft's Dynamic
Lighting documentation (**PUBLISHED**).

**CAPTURED, at the mouse's replug (13:45:42.239 to .264), Windows' own traffic:**

| Time | Interface | Report | Reading |
|---|---|---|---|
| .239 | mouse if 3 | GET feature 1 (LampArrayAttributes) | 2 lamps, kind 2 (mouse), minimum update interval 33 333 µs (details in [razer-deathadder-v2.md](razer-deathadder-v2.md)) |
| .239 | mouse if 3 | SET 2 / GET 3, twice | attributes of lamp 0 and lamp 1 |
| .239 | mouse if 3 | SET 6 `06 01` | LampArrayControl: autonomous mode on |
| .240 | mouse if 3 | SET 5 `05 01 00 00 01 00 00 00 00` | LampRangeUpdate: lamps 0 to 1, black, update complete |
| .252 | keyboard if 4 | SET 5 `05 01 00 00 6D 00 00 00 00` | LampRangeUpdate: lamps 0 to `0x6D` (109), black (INFERRED with the mouse's layout) |
| .264 | mouse if 3 | SET 6 `06 00` | autonomous mode off: the host drives the lamps |
| .264 | mouse if 3 | SET 4, 43 bytes, lamps 0 and 1 | LampMultiUpdate carrying `00 78 D4` twice |

- `00 78 D4` is Windows' default accent colour, and the Dynamic Lighting setting for both devices was "use accent
  colour" (HOST, Windows' lighting settings), so Windows set both mouse lamps to the accent colour (INFERRED).
- Synapse's `0F:03` frames continued on the mouse's interface 0 and the keyboard's interface 3 throughout (CAPTURED),
  and the effects chosen in Synapse during the session were the ones the devices showed (capture notes). So while
  Synapse ran and `0F:10` was set, the frames won over the LampArray colours (INFERRED).
- **Without Synapse.** Who wins between Dynamic Lighting and a host sending Razer frames, with no `0F:10` sent, was not
  observed. 2023 user reports for this keyboard describe Dynamic Lighting overriding Synapse until it was switched off
  in Windows' settings (**PUBLISHED**, user reports on Razer's forum; not re-checked). A host using the Razer protocol
  should check whether Dynamic Lighting is enabled for the device and warn, and should not write the LampArray
  collection itself (INFERRED).

## 11. What must not be sent

| Never | Why |
|---|---|
| Firmware update, boot-loader or DFU commands of any kind | No public project documents them for these devices, and none is described here. Synapse's files mention a Razer boot-loader PID `1532:110D` (METADATA); a device in that mode is a different USB device. Match exactly `1532:0084` and `1532:026C`, and leave any other Razer PID alone |
| A serial-number or other identity write | none is published; any unknown id in class `0x00` could be one (INFERRED) |
| Device mode `0x02` | undocumented factory-test mode; openrazer refuses it (**PUBLISHED**) |
| Device mode `0x03`, unless the host implements button remapping, macros and the Fn layer itself | the device then waits for host software (section 4.6) |
| `0F:10` | meaning and persistence UNKNOWN (section 8.4) |
| Onboard-profile, storage and mapping writes: `02:0C`, `05:02`, and anything else in classes `0x02`, `0x05`, `0x06` beyond the documented commands | CAPTURED from Synapse, not decoded. Synapse's middleware also names storage reset and factory reset operations whose ids are UNKNOWN (METADATA), so an unknown id in these classes may be one of them |
| Sensor writes `0B:03`, `0B:0B` | partly decoded only ([razer-deathadder-v2.md](razer-deathadder-v2.md)) |
| Pairing and receiver commands (`00:41`, `00:42`, `00:46`, `00:BF`) | for wireless dongles (**PUBLISHED**, openrazer and OpenRGB); OpenRGB sends `00:BF` to every Razer device at detection and wired devices reject it (**PUBLISHED**) |
| Class `0x03` effect commands (`03:0A` to `03:0D`) | other device families, different layouts (section 7.2) |
| Razer reports to any other collection: the mouse's interface 3 (the LampArray collection and a second vendor feature report, ID `0x07`, 63 bytes, usage page `0xFF00`, declared in the same descriptor; CAPTURED, never used in the capture), or virtual devices of Razer's drivers | openrazer's author found in 2020 that this mouse also accepted control reports on interface 3 (**PUBLISHED**), so a stray report there acts on the device (INFERRED) |
| The 90-byte report to the Kraken V3 Pro's dongle `1532:052C` | it does not speak this protocol ([razer-kraken-v3-pro.md](razer-kraken-v3-pro.md)) |
| Any command not in sections 5 to 9 | the protocol has no harmless "unknown command" answer guarantee; the keyboard answered `0x05` to one unknown write, but a different id may act |

## 12. Sequences

### 12.1 Synapse attach and start (CAPTURED)

Mouse replug, 13:45:43.868 to 13:45:45.593, all tid `0x1F` (after the attach-time reads of section 4.3):

1. Reads: `00:87`; then `00:86`, `05:8A`, `05:80`, `05:81`, `06:80`, twice, with one `06:8E` between; then `00:84`
   → `00 00`.
2. `00:04 03 00` (driver mode), `0F:10 01`.
3. `0F:04` `00 00 FF`, then `01 00 FF` (brightness 255, LED 0, unstored then stored).
4. `0F:02` `00 00 08 00 01 00`, one frame, `0F:02` `00 00 08 00 00 00` (the apply).
5. Mouse settings: `0B:03` twice, `0B:0B`, `00:05`, `04:05` twice, `0F:04`, `04:06`, `04:05` stored, `0F:04`, `00:05`
   ([razer-deathadder-v2.md](razer-deathadder-v2.md)).
6. 20 x `02:0C` button mappings, then frames at 25 per second.

Synapse start, 14:01:34.802 (both devices, no replug): `00:84`, `00:04 03 00`, `0F:10 01`, `0F:04` unstored and stored;
2.4 s later the device settings (keyboard: `03:00` LED `0x08` off, `0F:04`, `00:40`, profile and storage reads,
`05:02 01`, `02:14` → not supported; mouse: as steps 5 and 6). No `0F:02` apply; frames resumed on both devices.

### 12.2 Synapse exit (CAPTURED, 14:00:26.329 to .496)

Keyboard first, then mouse:

| Step | Keyboard | Mouse |
|---|---|---|
| 1 | `03:00` `00 08 00`, `03:00` `00 18 00` (indicators off) | |
| 2 | `0F:80` → `05 19 03 06 16` | `0F:80` → `01 19 03 01 01 04 19 03 01 01` |
| 3 | `0F:82` `01 05` → spectrum | `0F:82` `01 01` → spectrum, rate `0x28` |
| 4 | `0F:02` `01 00 03 00 00 00` (busy twice, then `0x02`) | `0F:02` `01 00 03 00 28 00` |
| 5 | `00:84` → `03`; `00:04 00 00` | `00:84` → `03 00`; `00:04 00 00` |

After this both devices run their stored spectrum cycle in normal mode, which is what they showed without Synapse on
the bench (OBSERVED, 2026-10-31 and 2026-11-01).

### 12.3 A host without Razer software

What third-party code did, and found sufficient (OBSERVED, 2026-10-31 to 2026-11-04): open the control collection
(section 2.2), wait 0.5 s, read firmware, serial, device mode (write `00:04 00 00` only if it reads `0x03`), polling
rate, DPI, brightness and the region list; for host-rendered lighting send one apply (section 8.2), then rows only;
for a hardware effect send `0F:02` per LED with storage `0x00`; on exit, hand the LEDs back with a hardware effect
(spectrum) rather than leaving the last frame.

## 13. Hazards

| Hazard | Guard |
|---|---|
| Two requesters on one device (vendor software, the vendor's filter driver at attach, OpenRGB, a second program) read each other's answers or get none (section 4.3) | One owner per device. Match every answer by transaction id, class and id; resend a read only after the budget |
| Busy answers parsed as data | Parse only `0x02` answers; busy answers carry rewritten arguments and a stale checksum (section 3.2) |
| An answer rejected because its data size differs from the request's | Accept the answer's own size (section 3.3) |
| The first commands after enumeration answer busy for about 0.4 s | Wait about 0.5 s after the device appears, then poll (section 4.3) |
| Driver-mode trap after a killed vendor process | Read `00:84` at start; write `00:04 00 00` if it reads `0x03` (section 4.6) |
| Unintended persistence and flash wear from storage `0x01` | Storage `0x00` by default; `0x01` only for explicit user changes, never in loops (section 4.5) |
| Vendor software rewrites the stored state at every attach | Anything stored by another host lasts only until Synapse next starts (CAPTURED: Synapse rewrites brightness, DPI, stages and mappings at every attach). After Synapse was stopped, two of its services started again by themselves about 25 s later (OBSERVED, 2026-10-31); check that no Razer service runs before driving the devices ([razer-software-and-drivers.md](razer-software-and-drivers.md)) |
| Opening the wrong collection, or the exclusive mouse collection with read/write access | Select by interface number and usage; open the mouse with zero access rights (section 2.2) |
| Effect commands addressed to the wrong LED id on the mouse | See [razer-deathadder-v2.md](razer-deathadder-v2.md) for the public crash reports and what was observed |
| Windows Dynamic Lighting driving the same LEDs | Detect and warn; never write the LampArray collection from the Razer driver (section 10) |
| A stale handle after a replug, sleep or re-enumeration | Treat Win32 error 1167 as "disconnected", rediscover by VID, PID, interface and usage, reopen (section 2.2) |
| Mixing effect families | Class `0x0F` only for these devices (section 7.2) |
| Unknown ids | Allowlist the commands of sections 5 to 9 in the one place every report passes (section 11) |
| Photosensitivity from host-streamed frames | Limit full-device flashes to about 3 per second unless the user opts in (INFERRED, general practice) |

## 14. Test vectors

Bytes from `r[0]`; every byte not shown is `00` except the checksum at `r[88]`. On Windows prepend `00`.

**CAPTURED** (Synapse 3, 2026-10-30):

| Id | Meaning | `r[0..]` | `r[88]` |
|---|---|---|---|
| T1 | mouse brightness 50 %, stored, LED 0 (13:42:16.203) | `00 1F 00 00 00 03 0F 04 01 00 7F` | `76` |
| T2 | answer to T1 | `02 1F 00 00 00 03 0F 04 01 00 7F` | `76` |
| T3 | brightness 255 unstored / stored (14:01:34.942 / .977) | `00 1F 00 00 00 03 0F 04 00 00 FF` / `... 01 00 FF` | `F7` / `F6` |
| T4 | get device mode | `00 1F 00 00 00 02 00 84 00 00` | `86` |
| T5 | mouse answer `03 00` / keyboard answer, size 1, `03` | `02 1F 00 00 00 02 00 84 03 00` / `02 1F 00 00 00 01 00 84 03` | `85` / `86` |
| T6 | driver mode / normal mode | `00 1F 00 00 00 02 00 04 03 00` / `... 00 00` | `05` / `06` |
| T7 | `0F:10 01` (do not send; for decoders) | `00 1F 00 00 00 01 0F 10 01` | `1F` |
| T8 | custom-frame apply, `a[4]` 1 / 0 (13:45:44.609 / .641) | `00 1F 00 00 00 06 0F 02 00 00 08 00 01 00` / `... 00 00 08 00 00 00` | `02` / `03` |
| T9 | mouse frame, both LEDs `16 65 FF` (13:42:16.224) | `00 1F 00 00 00 0B 0F 03 00 00 00 00 01 16 65 FF 16 65 FF` | `06` |
| T10 | keyboard frame row 3 (13:50:30.001) | `00 1F 00 00 00 47 0F 03 00 00 03 00 15` + the 66 colour bytes of section 8.1 | `5D` |
| T11 | region list request (Synapse's size) | `00 1F 00 00 00 32 0F 80` | `BD` |
| T12 | region list answers, mouse / keyboard | `02 1F 00 00 00 0A 0F 80 01 19 03 01 01 04 19 03 01 01` / `02 1F 00 00 00 05 0F 80 05 19 03 06 16` | `80` / `85` |
| T13 | get stored effect, keyboard LED 5, and its answer | `00 1F 00 00 00 0C 0F 82 01 05` / `02 1F 00 00 00 09 0F 82 01 05 03 00 00 01 6E DA FD` | `85` / `CB` |
| T14 | stored spectrum, LED 0, mouse / keyboard (exit) | `00 1F 00 00 00 06 0F 02 01 00 03 00 28 00` / `00 1F 00 00 00 06 0F 02 01 00 03 00 00 00` | `21` / `09` |
| T15 | keyboard busy answer to T14 (stale checksum) and final answer | `01 1F 00 00 00 06 0F 02 01 05 03 00 00 00` / `02 1F ... 01 05 03 00 00 00` | `09` (wrong; `0C` would match) / `0C` |
| T16 | game-mode LED on | `00 1F 00 00 00 03 03 00 00 08 01` | `09` |
| T17 | `02:14` and its "not supported" answer | `00 1F 00 00 00 02 02 14 01 00` / `05 1F 00 00 00 02 02 14 01 00` | `15` / `15` |
| T18 | attach-time read, tid `0C`, and a busy answer | `00 0C 00 00 00 03 00 86 00 00 00` / `01 0C 00 00 00 03 00 86 00 00 00` | `85` / `85` |

**Anti-vectors** a decoder must handle: T15's first answer fails the checksum and must not be parsed; T12 and T13
answers have a smaller size than their requests; T18's id is neither `0x1F` nor any id openrazer or OpenRGB uses;
a decoder that includes the transaction id `r[1]` in the checksum gets T1 wrong (`76 ^ 1F` = `69`).

## 15. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Do the devices answer transaction ids `0x3F` and `0xFF` (keyboard: anything but `0x1F`)? | one read of `00:84` with each id |
| 2 | Is a SET that arrives while the device is busy discarded or queued? | two overlapping reads with different ids |
| 3 | Is storage `0x01` a flash write, and with what endurance? | not settleable from the bus; vendor information |
| 4 | Why does the keyboard power up at 1000 Hz although Synapse writes `00:40` with `a[0] = 0x01`? Is `a[0]` there a storage byte at all? | `00:40` with `0x00` and `0x01`, each followed by a replug ([razer-huntsman-v2.md](razer-huntsman-v2.md)) |
| 5 | What `0F:10` does, whether it persists, and its read form | Dynamic Lighting on, Razer frames with and without `0F:10`, then a replug |
| 6 | How LED `0x00` settings combine with per-LED settings (brightness read 0 per LED while lit) | per-LED and LED-0 brightness writes, each read back both ways |
| 7 | The bytes `19 03` of the region records; whether `0F:81` works on these firmwares | one `0F:81` read per region |
| 8 | Meaning of `00:87`, `05:80`, `05:81`, `05:8A`, `06:80`, `06:8E`, `02:14`; `a[4]` of Synapse's apply | a capture of Synapse's profile management |
| 9 | Fastest frame rate each device displays without loss | frame streams at rising rates with a camera |
| 10 | Who wins between Windows Dynamic Lighting and Razer frames with no vendor software running | Dynamic Lighting on, then Razer frames, then the reverse |
| 11 | Mapping of Synapse's four-part firmware versions (`1.0.3.0`, `1.0.5.0` in its logs, HOST) onto the `00:81` answer `01 00 00 00` | a capture of Synapse reading `00:81` |
| 12 | Whether status `0x03` or `0x04` ever occurs on these wired devices | not seen in the capture or on the bench |

## Credits

- **[openrazer](https://github.com/openrazer/openrazer)** published the report structure, the checksum range, the status
  codes and storage names (`driver/razercommon.h`, `driver/razercommon.c`), the command constructors for class `0x0F`
  effects, frames and brightness, device mode, firmware and serial (`driver/razerchromacommon.c`), the per-device
  interface choice, transaction ids and retry timing (`driver/razermouse_driver.c`, `driver/razerkbd_driver.c`), and the
  driver-mode behaviour (`daemon/openrazer_daemon/hardware/device_base.py`).
- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** published the same layout with the report-ID byte, the
  detector's interface and usage choice, its transaction ids, the region and effect-list reads and reply filtering
  (`Controllers/RazerController/RazerController/RazerController.cpp`, `RazerController.h`,
  `Controllers/RazerController/RazerControllerDetect.cpp`, `Controllers/RazerController/RazerDevices.cpp`).
- **[LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor)** independently implemented
  the same report for a Razer fan controller (`LibreHardwareMonitorLib/Hardware/Controller/Razer/RazerFanController.cs`),
  confirming the checksum range, the byte order and the 91-byte Windows buffer.
- **[hidapi](https://github.com/libusb/hidapi)** (`windows/hid.c`) published the zero-access open that makes feature
  reports work on a collection Windows holds exclusively.
- **Microsoft** documents the top-level collections Windows opens for system use and the Dynamic Lighting / HID
  LampArray model.

Nothing from these projects (code, comments or prose) is copied here, only the protocol facts they established.
