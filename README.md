# Razer DeathAdder V2, Huntsman V2 and Kraken V3 Pro: USB protocol reference

An independent description of the USB protocols spoken by three Razer peripherals of the Synapse 3 generation, and of
what Razer's own Windows software does with them. It is meant as the reference for running these devices without
Razer's software.

| Device | USB ID | How the host controls it | Document |
|---|---|---|---|
| shared protocol of the mouse and the keyboard | | 90-byte HID feature reports: framing, checksum, status codes, transaction ids, storage flag, device mode, effects, custom frames, brightness, Windows Dynamic Lighting | [razer-hid-protocol.md](razer-hid-protocol.md) |
| DeathAdder V2 (wired mouse) | `1532:0084` | the shared protocol on interface 0; DPI, DPI stages, polling rate, two LEDs, button mappings; a HID LampArray on interface 3 | [razer-deathadder-v2.md](razer-deathadder-v2.md) |
| Huntsman V2 (full-size optical keyboard) | `1532:026C` | the shared protocol; a 6 × 22 LED matrix, polling up to 8000 Hz, indicator LEDs, keyswitch optimisation; a HID LampArray on interface 4 | [razer-huntsman-v2.md](razer-huntsman-v2.md) |
| Kraken V3 Pro (wireless headset) through its HyperSpeed dongle | `1532:052C` | **not** the shared protocol: framed commands over the dongle's virtual COM port (USB CDC); battery, sidetone, haptics, power saving, earcup colour, events | [razer-kraken-v3-pro.md](razer-kraken-v3-pro.md) |
| Razer Synapse 3, Razer Central, the Chroma SDK, THX, and Razer's drivers | | what they install, which handles they hold, how Synapse drives and re-applies, how they were removed and what is lost | [razer-software-and-drivers.md](razer-software-and-drivers.md) |

Coverage per command and change history: [STATUS.md](STATUS.md).

## Why this exists

The 90-byte feature-report protocol is well known from openrazer and OpenRGB, and parts of the Kraken V3 Pro's serial
protocol were published by small projects (see Credits). What was missing: what Synapse 3 actually sends, which of
the published commands these particular models accept, how the devices behave between and without hosts, and how the
headset's control channel works in full. On 2026-10-30 the USB traffic between Synapse 3 and all three devices was
captured with USBPcap while the owner exercised one function at a time; from 2026-10-31 to 2026-11-03 the same units
were driven by independently written code; on 2026-11-03 all Razer software was removed and the devices kept working
on Windows' own drivers. This reference writes down what was seen, which published claims held, and what is still
unknown.

New compared with what was public before, among others:

- **Synapse renders every effect on the PC.** While it runs, it never sends a hardware effect: it streams custom
  frames (`0F:03`) at about 25 per second to the mouse and 24 per second (six rows each) to the keyboard, and colour
  frames to the headset, Static included. It applies a stored hardware effect only once, at exit.
- **Synapse's attach, start and exit sequences**, complete with timing, including an undocumented `0F:10 01` after
  every switch to driver mode, and that frames work without it.
- **Corrections to the shared protocol:** answers carry their own data size, often smaller than the request's; busy
  answers carry rewritten arguments with a stale checksum and must not be parsed; brightness is `percent × 255 / 100`
  truncated, on LED `0x00` with storage `0x01`, on both devices.
- **Persistence measured on the bench:** writes without the storage flag are lost at a replug; the keyswitch setting
  persists without it; the keyboard powers up at 1000 Hz even after Synapse's
  8000 Hz write with the storage flag.
- **Mouse:** the DPI-stage commands `04:06` / `04:86` work on `1532:0084` although published sources mark them
  unsupported; DPI can be applied without storing it; separate X and Y DPI; the four-part firmware read `00:87`; the
  button-mapping layout `02:0C`; the full LampArray descriptor (two lamps, positions, update interval).
- **Keyboard:** the five-interface descriptor set; the exact 108 matrix cells Synapse lights, which fixes the 6 × 22
  key map (one column off in published maps); polling-rate writes; Game Mode handled in firmware; the profile and
  storage queries Synapse makes, decoded.
- **Headset:** the dongle's descriptors, frame and checksum rules per command, every setting's encoding, the
  unsolicited link, battery and button events, the stale read-back after a write, haptics ignored while charging, and
  that the earcup colour lasts until power-off.
- **Windows Dynamic Lighting** writes to the devices' LampArray interfaces when they attach, even with Synapse
  installed (the accent colour, `#0078D4`, to both mouse lamps).
- **Removing Razer's software cleanly:** what the uninstaller leaves, how Windows falls back to superseded Razer
  driver packages, and the inbox driver each interface ends up on.

## Scope

- The control protocols of the three devices on their USB interfaces: reports and frames, commands, answers,
  sequences, timing, persistence and standalone behaviour; how Razer's software and Windows share the devices.
- Not covered: the devices' ordinary input reports (pointer, keys; deliberately filtered out of the capture), the
  headset's audio streams, macro contents, and firmware updates, which are deliberately **not** described beyond the
  warning not to trigger them.
- One unit of each device was observed, with Razer Synapse 3 (3.10.0730.071519) as the vendor host. Other firmware
  versions, keyboard layouts other than US ANSI, and other Razer models may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **CAPTURED** | seen on the USB bus in the capture of Synapse 3 listed below; the time of day (and segment) is given so the transfer can be found in the raw files |
| **METADATA** | read from Razer's installed files without running them: Synapse's device-definition and LED-layout data, printable strings and static tables of its DLLs, driver INF files. Metadata says what the software *declares*, not what it sends |
| **PUBLISHED** | first published by an open-source project, by Razer or Microsoft documentation, or in a public forum, which is named. Where the capture or the bench re-checked the fact, the statement says so |
| **HOST** | read on the capturing PC without driving the device: Windows device properties, driver bindings and registry, Windows' lighting settings, and Synapse's own log files; used only to explain wire values |
| **OBSERVED** | seen while driving the device with independently written third-party code (not the vendor software), with the date |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

## Capture and bench sessions

**The capture** (2026-10-30, 13:39 to 14:03, local time of the capturing PC, UTC-7). Fourteen USBPcap segments of the
whole root hub with injected descriptors, filtered afterwards to the three Razer devices. Synapse 3 owned all three
throughout; the owner did one thing at a time and noted the time. The mouse's and keyboard's interrupt IN reports
(movement, clicks, key presses) were dropped while filtering and are in no file.

| Segment | Time | What was exercised |
|---|---|---|
| 1 | 13:39-13:41 | mouse polling rate 1000 → 500 → 1000 Hz, twice |
| 2 | 13:41-13:42 | mouse brightness 100 → 50 → 100 % |
| 3 | 13:42-13:43 | mouse effect Spectrum Cycling → Static → back |
| 4 | 13:43-13:45 | mouse DPI stage 3 → 1 → 4 |
| 5 | 13:45-13:46 | mouse unplugged and replugged: enumeration, Windows' LampArray traffic, Synapse's attach sequence |
| 6 | 13:46-13:48 | keyboard polling rate 1000 → 8000 → 1000 Hz |
| 7 | 13:48-13:49 | keyboard brightness 100 → 50 → 100 % |
| 8 | 13:49-13:50 | Game Mode (Fn+F10) on and off |
| 9 | 13:50-13:51 | keyboard effect Wheel → Static → Wheel |
| 10 | 13:51-13:55 | headset on; Synapse's headset tab; power saving, haptics and sidetone changes |
| 11 | 13:55-13:57 | headset lighting: brightness, Static, Breathing, Spectrum |
| 12 | 13:57-13:59 | microphone mute and unmute; headset off and on |
| 13 | 13:59-14:00 | the headset's HyperSense button; Synapse exit at 14:00:26 |
| 14 | 14:00-14:03 | Synapse start at 14:01:34 |

The raw capture files are not published (they were filtered from captures of the whole root hub). Serial numbers are
masked throughout. The decoded bytes that matter are quoted in the documents.

**Bench sessions** (OBSERVED; independent third-party code driving the same units, with Synapse stopped or, from
2026-11-03, uninstalled):

| Date | What was checked |
|---|---|
| 2026-10-31 | read-only probe of the mouse and keyboard: identity, firmware, polling rate, DPI, brightness; Synapse services restarting by themselves |
| 2026-11-01 | first writes: effects and DPI with and without the storage flag across a replug, custom frames without `0F:10`, the keyboard matrix row by row, wave directions, the DPI-stage read, keyswitch optimisation, the keyboard's power-up polling rate, an 8000 Hz write, separate X/Y DPI, the Game Mode LED, Fn+F10 without Synapse, a key-position map checked by the owner |
| 2026-11-02 | the headset over the dongle's COM port: read probe, then sidetone, haptics, power saving and earcup colour writes |
| 2026-11-03 | all Razer software removed; all three devices driven on Windows' inbox drivers |

## Sources

The facts in this reference come from these sources only:

1. A USB capture of the vendor software (Synapse 3) driving the devices, made with USBPcap on the owner's own PC.
2. Razer's installed files, read without running them: device-definition and LED-layout data, printable strings of
   Synapse's DLLs, driver INF files.
3. Windows' device properties, driver bindings and lighting settings, and Synapse's own logs on the same PC, read
   without opening the devices.
4. The open-source projects, documentation and forum reports listed under Credits, for the PUBLISHED statements.
5. For the OBSERVED statements: the devices' own answers and behaviour when driven by independently written
   third-party code on the same PC.

No vendor source code was used or seen, no code was disassembled, and no firmware was read. The only vendor strings
reproduced are names from Razer's files and logs and strings that appear on the wire.

## Credits

Projects are linked to their repositories; the facts were taken from their source trees (default branch) and issue
trackers as of October 2026.

- **[openrazer](https://github.com/openrazer/openrazer)** (Linux drivers and daemon for Razer devices) established
  the 90-byte report, its checksum, status codes and transaction ids, the storage flag, and most command constructors
  (`driver/razercommon.h`, `driver/razercommon.c`, `driver/razerchromacommon.c`, `driver/razermouse_driver.c`,
  `driver/razerkbd_driver.c`, `daemon/openrazer_daemon/hardware/`). Its pull request #2884 (`driver/razerkraken_driver.c`)
  and issue #1950 cover the Kraken V3 Pro.
- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** (`Controllers/RazerController/`: `RazerController/`,
  `RazerControllerDetect.cpp`, `RazerDevices.cpp`, `RazerDeviceGuard.cpp`; `KeyboardLayoutManager/`) published device
  tables, matrix sizes, the LED-0 brightness fix for the DeathAdder V2 and the Windows named mutex Razer software uses.
- **[OmaKraken](https://github.com/Dankestrick/OmaKraken)** (`helpers/omakrakenctl`) and
  **[kraken-v3-pro-hypersense-charging](https://github.com/ShadowXCC/kraken-v3-pro-hypersense-charging)**
  (`FINDINGS.md`) published parts of the Kraken V3 Pro's serial protocol.
- **[gpoulios/deathadderv2](https://github.com/gpoulios/deathadderv2)** configures the DeathAdder V2 without Synapse.
- **[LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor)**
  (`LibreHardwareMonitorLib/Hardware/Controller/Razer/RazerFanController.cs`) is an independent implementation of the
  same report for another Razer device.
- **[hidapi](https://github.com/libusb/hidapi)** (`windows/hid.c`), for how feature reports reach the device on
  Windows.
- **Microsoft** documents Dynamic Lighting and the HID LampArray usages (HUTRR84, from the USB-IF); **Razer**
  documents the Chroma SDK and answers support questions; **Razer Insider** forum threads report Dynamic Lighting
  conflicts. These are named where they are used.
- **Razer** is the vendor whose software, drivers and devices were observed. Product, command and setting names are
  Razer's.

Nothing from these projects (code, comments or prose) is copied here, only the protocol facts they established;
byte layouts and constants that they already publish are stated with credit.

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to capture more

The capture was made with [USBPcap](https://desowin.org/usbpcap/) on Windows 11.

1. Install USBPcap. Run `USBPcapCMD.exe` once without arguments: it lists the root hubs (`\\.\USBPcap1`, ...) and
   their devices. Find the one that carries the Razer devices.
2. Capture the root hub in short segments, one action each, from an elevated prompt:

   ```
   USBPcapCMD.exe -d \\.\USBPcap1 -A --inject-descriptors -o razer-seg1.pcap
   ```

   `-A` captures every device on the root hub; without it and without `--devices` USBPcapCMD exits at once. Stop
   with Ctrl+C (a script that sends Ctrl+C to USBPcapCMD should ignore it itself).
3. **Keep your keystrokes out of it.** The root hub carries the keyboard and usually the mouse: filter each segment to
   the Razer devices at once, drop the keyboard's and mouse's interrupt IN transfers, and delete the raw files. After a
   replug, a sleep or a reboot the device addresses change and the injected descriptor list can be stale; check which
   address actually carries the traffic.
4. Expect Synapse's frame streams in every segment: `0F:03` writes to the mouse and keyboard and colour frames to the
   dongle, many per second. Set them aside and the rest is what you did.
5. Mouse and keyboard commands are SET_REPORT / GET_REPORT (feature) on the default pipe; the headset's commands are
   bulk transfers on the dongle's CDC data interface. Each USBPcap record starts with a header
   `u16 headerLength, u64 irpId, u32 status, u16 function, u8 info, u16 bus, u16 device, u8 endpoint,
   u8 transferType, u32 dataLength`.

Never replay captured packets to a device, and never trigger a firmware update to capture it.

Still uncaptured, in case you want to contribute: hardware effects other than spectrum as Synapse would apply them
(it never did while running), macros and profile writes, the mouse's other settings (lift-off, calibration), the
keyboard's keyswitch setting as Synapse writes it, the headset's brightness read and effect read, and what the
devices do when Synapse is killed rather than closed. See [STATUS.md](STATUS.md).

## Safety

The command sets of these devices include serial-number writes, onboard-profile and macro storage, firmware-update and
boot-loader entry; the documents list those with a do-not-send warning. Apart from the OBSERVED statements, nothing in
this reference was tested beyond observing the vendor software; anyone sending commands to their own devices does so at
their own risk. Writing with the storage flag set may write the devices' flash; prefer volatile writes for anything
sent often. A headset over a radio link answers late and sometimes not at all: never assume a write took effect
without reading it back.
