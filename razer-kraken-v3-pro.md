# Razer Kraken V3 Pro: headset control over the HyperSpeed dongle (USB `1532:052C`)

Part of the [Razer peripherals protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **PUBLISHED**,
**HOST**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. Times are local time of the
capturing PC (UTC-7). For this device "the capture" means segments 10 to 14 of the 2026-10-30 session, 13:51:12 to
14:03, in which Razer Synapse 3 drove the dongle at USB device address 18. Offsets: a host-to-dongle frame is
numbered `f[0]` to `f[27]`, a dongle-to-host reply `ev[0]` to `ev[30]`, and the reply's values are `v[n] = ev[10 + n]`.
Bytes are written in the order they travel on the wire.

Short version (each point is tagged in the sections below): the dongle is a composite USB device with a USB Audio
function, a one-collection HID consumer-control interface for the headset's media buttons, and a CDC ACM function
(a virtual COM port). Every headset setting goes through the COM port, not through HID: the 90-byte Razer feature
report of the mouse and keyboard does not exist on this device. The host writes 28-byte Bluetooth-HCI-style vendor
command frames (`01 F3 FF 18 02 ...`) and the dongle answers with 31-byte Command Complete events (`04 0E 1C 01 F3 FF
...`), after a radio round trip to the headset of about 0.1 to 0.25 s. The command set is small: battery and charging
(`0x24`), sidetone (`0x25`), power saving (`0x26`), haptics, earcup colour and brightness (group `0x40`), plus
firmware, serial and one unknown read. The dongle also reports link up and down, the battery and the HyperSense
button by itself. Synapse renders every lighting effect on the PC and streams one colour at a time. Sidetone, power
saving and the haptics level are stored on the headset; an earcup colour lasts until the headset is switched off. A
read right after a write often returns the old value. One process at a time can own the COM port.

## 1. Device

### 1.1 What the dongle and the headset are

The Kraken V3 Pro is a wireless headset with 2.4 GHz wireless through a USB-A dongle and a 3.5 mm analogue input,
haptic drivers in both earcups ("HyperSense": a button cycles Low, Medium, High and off), a logo light on each earcup
("Razer Chroma"), a power button, a HyperSense button, a volume wheel and a microphone mute button (**PUBLISHED**,
Razer's Kraken V3 Pro master guide). The guide also states that earcup lighting and the HyperSense button are disabled
while the headset charges.

The USB device `1532:052C` is the dongle. The headset itself has its own USB identity, `1532:052D`, which appears only
while it is connected by its charging cable (section 1.3).

Synapse's code name for the device is "KrystalT1" (METADATA: the device-definition file `DD_KrystalT1.rzdd` and the
module folder of Synapse 3).

The dongle reports NXP Semiconductors as its manufacturer (**PUBLISHED**, `lsusb` output in openrazer issue #1950,
https://github.com/openrazer/openrazer/issues/1950). In NXP's reference design for a USB audio dongle (a K32L2B host
MCU with an NxH3670 radio), the host MCU talks to the radio with Bluetooth HCI vendor-specific commands, and the dongle
offers a virtual COM port through which firmware is transferred to the headset (**PUBLISHED**, NXP application note
AN12647, "K32L2B USB Dongle with NxH3670"). The frames of section 3 are HCI vendor commands, so the dongle most likely
derives from that design (INFERRED), and its COM port is also the likely firmware-update path (section 9).

The observed unit: headset firmware 1.6.0 (OBSERVED, section 5.7), dongle `bcdDevice 09.00` (CAPTURED).

### 1.2 Descriptors

**CAPTURED (2026-10-30, segment 10, 13:51:12)**, from the descriptors USBPcap injected at the start of the segment:

| Item | Value |
|---|---|
| Device descriptor | `12 01 00 02 EF 02 01 40 32 15 2C 05 00 09 01 02 00 01`: bcdUSB 2.00, class `EF/02/01` (functions grouped by interface association descriptors), 64-byte EP0, VID `1532`, PID `052C`, bcdDevice `0900`, manufacturer string 1, product string 2, **no serial-number string** (`iSerialNumber 0`), one configuration |
| Configuration | `09 02 2E 01 06 01 00 80 32`: 302 bytes, six interfaces, configuration value 1, bus-powered without remote wake-up (`80`), bMaxPower 50 = 100 mA |
| Association 1 | `08 0B 00 03 01 01 00 02`: interfaces 0 to 2, Audio, string 2 |
| Interface 0 | `09 04 00 00 01 01 01 00 00`: Audio Control, one endpoint |
| Endpoint `0x81` | `09 05 81 03 08 00 08 00 00`: interrupt IN, 8 bytes, bInterval 8 (audio status) |
| Interface 1, alt 0 / 1 | Audio Streaming, microphone to host: PCM, **1 channel, 16 bit, 48 kHz** (`0B 24 02 01 01 02 10 01 80 BB 00`); endpoint `0x83` isochronous IN, synchronous, 98 bytes, bInterval 1 |
| Interface 2, alt 0 / 1 | Audio Streaming, host to headset: PCM, **2 channels, 16 bit, 48 kHz** (`0B 24 02 01 02 02 10 01 80 BB 00`); endpoint `0x02` isochronous OUT, synchronous, 192 bytes, bInterval 1 |
| Interface 3 | `09 04 03 00 01 03 01 01 00`: HID, boot subclass, **protocol 1 (keyboard)**, one endpoint |
| HID descriptor | `09 21 00 01 00 01 22 25 00`: HID 1.00, one report descriptor of 37 bytes |
| Endpoint `0x84` | `07 05 84 03 08 00 10`: interrupt IN, 8 bytes, bInterval 16 |
| Association 2 | `08 0B 04 02 02 02 00 03`: interfaces 4 and 5, CDC, ACM, string 3 |
| Interface 4 | `09 04 04 00 01 02 02 00 00`: CDC Communications, ACM, no protocol, one endpoint; functional descriptors header CDC 1.10 (`05 24 00 10 01`), call management (`05 24 01 01 01`), ACM capabilities `06` (`04 24 02 06`), union master 4, slave 5 (`05 24 06 04 05`) |
| Endpoint `0x85` | `07 05 85 03 10 00 08`: interrupt IN, 16 bytes, bInterval 8 (CDC notifications) |
| Interface 5 | `09 04 05 00 02 0A 00 00 00`: CDC Data, two endpoints |
| Endpoints `0x86`, `0x06` | `07 05 86 02 40 00 00` bulk IN, 64 bytes; `07 05 06 02 40 00 00` bulk OUT, 64 bytes |

The audio function's topology: microphone input terminal (ID 1) → feature unit 2 (master mute and volume) → USB
streaming output terminal 3; USB streaming input terminal 4 → feature unit 5 (master mute and volume) → speaker output
terminal 6 (CAPTURED). The microphone input terminal declares two channels while its streaming format carries one
(CAPTURED; INFERRED harmless, Windows uses the streaming format).

- **Full speed.** An isochronous endpoint with bInterval 1 that carries 48 000 × 2 × 2 bytes per second in 192-byte
  packets is a 1 ms full-speed frame (INFERRED from the descriptors).
- **The call-management descriptor names data interface 1**, not 5 (`bDataInterface = 01`). Windows' `usbser` and
  Linux's `cdc_acm` work with it (OBSERVED on Windows; **PUBLISHED** for Linux by OmaKraken, which uses the tty),
  presumably because they take the data interface from the union descriptor (INFERRED).
- **ACM capabilities `0x06`**: line coding and control-line state, and Send_Break (CAPTURED; meaning per the USB CDC
  PSTN specification).
- **No serial-number string.** Windows therefore keys the device by its port position, and so should any host
  (INFERRED). The serial that command `0x21` returns (section 5.7) needs I/O and is not reproduced here.
- **Strings.** Only the device and configuration descriptors were captured. The manufacturer string is
  "NXP SEMICONDUCTORS" and the product string "Razer Kraken V3 Pro" (**PUBLISHED**, openrazer issue #1950). Windows
  shows the CDC function's bus-reported description as "USB VCOM" (HOST).
- **The HID report descriptor** (37 bytes, CAPTURED length) is one Consumer Control collection with six one-bit
  inputs, Play/Pause, Volume Up, Volume Down, Next Track, Previous Track and Stop, and two padding bits; no report ID,
  **no output report and no feature report** (**PUBLISHED**, `usbhid-dump` in openrazer issue #1950). Windows exposes
  it as an "HID-compliant consumer control device", usage page `0x000C`, usage `0x0001` (HOST).

### 1.3 The headset on its cable (`1532:052D`)

While the headset is connected by its USB charging cable it also enumerates as `1532:052D`: one CDC ACM function, which
Windows binds to `usbser` and shows as "USB Serial Device (COM3)" with the bus-reported interface string "HCI VCOM"
(HOST; OBSERVED 2026-11-02, while charging). It has no audio and no HID interface (**PUBLISHED**, a descriptor dump in
openrazer issue #1950).

Nothing in Synapse's device files mentions `052D` (METADATA, by absence). Its command set is UNKNOWN; the name "HCI
VCOM" and NXP's reference design suggest a service and firmware-update port (INFERRED). **Never open it** (section 9).

### 1.4 The vendor's view (METADATA)

From a static reading of Synapse 3's files (nothing executed):

- The 052C middleware (`RzCtl_052C.dll`) contains classes named `CComPortDevice` and `CProtocol_NXP` that the
  middleware of the mouse and keyboard does not have. It holds byte templates for every frame of section 4, all
  beginning `01 F3 FF 18`, and no other HCI opcode except the keepalive (section 3.4).
- Its COM-port code opens the port exclusively and overlapped, sets 9600 baud 8N1 with DTR enabled and a read timeout
  that returns at once (section 2.3).
- The device definition limits the power-saving timeout to 900 to 3600 s and the haptics level to 1 to 3; the
  sidetone slider is 0 to 100 %. Synapse's defaults: power saving on after 900 s, haptics on at level 2, brightness
  100 %, sidetone 50 %, lighting effect Spectrum.
- Synapse's LED layout for the device has one LED (one row, one column); its Chroma SDK module works on a 1 × 1 frame.
  There is no per-earcup lighting in any Razer file.
- Synapse's reply handlers expect for group `0x40`: `v[0]` = the sub-id echo, `v[1]` = `01` for success, values from
  `v[2]` on. The capture and the bench confirm this for `0x8A`, `0x8B`, `0x8C` and `0x8E` (section 3.3).

Metadata says what Synapse declares, not what it sends: some templates (`0x20`, `0x21`, `0x8D`, `0x8F`, the `0x22`
write) were never sent in the capture (section 4).

### 1.5 Earlier public work

| Project | What it established |
|---|---|
| OmaKraken, https://github.com/Dankestrick/OmaKraken (`helpers/omakrakenctl`, MIT) | the CDC channel and the frame format, from captures of Synapse in a virtual machine; battery, charging, sidetone, power saving, haptics, brightness and colour commands; streaming at 25 frames per second with a 5 s repeat of a static colour |
| openrazer pull request #2884, https://github.com/openrazer/openrazer/pull/2884 (`driver/razerkraken_driver.c`, unmerged) | brightness and colour over the CDC bulk endpoints, the closed-form checksums, raising DTR and RTS before writing |
| openrazer issue #1950, https://github.com/openrazer/openrazer/issues/1950 | the descriptors and HID report descriptor; that every HID control request to interface 3 fails; a raw Synapse colour frame from an owner's capture |
| ShadowXCC, https://github.com/ShadowXCC/kraken-v3-pro-hypersense-charging (`FINDINGS.md`) | notes on haptics and lighting gates while charging, and two further command bytes (section 4, not verified here) |

No merged open-source project (openrazer, OpenRGB, HeadsetControl) supports `1532:052C` (**PUBLISHED**, by absence as of
2026-10-26). OpenRGB's Kraken V3 controllers drive the wired Kraken V3 family through a HID output report on
interface 3; the 052C interface 3 has no output report, so that path does not apply (**PUBLISHED**, OpenRGB
`Controllers/RazerController/RazerKrakenV3Controller`; INFERRED consequence).

## 2. Transport

### 2.1 Interfaces and Windows drivers

| Interface | Class | Windows driver with Synapse installed | Windows driver after Synapse was removed (2026-11-03) | Purpose |
|---|---|---|---|---|
| 0 to 2 (`MI_00`) | USB Audio 1.0 | `usbaudio` with Razer's THX extension package (`razer052cdriver.inf`) and THX APOs | inbox `wdma_usb.inf` | "Speakers" and "Microphone (Razer Kraken V3 Pro)" |
| 3 (`MI_03`) | HID consumer control | `HidUsb` with the Razer lower filter `RzDev_052c`, plus a sibling "Razer Control Device" node under `RzCommon` | inbox HID drivers, no Razer filter | media buttons and volume wheel |
| 4 and 5 (`MI_04`) | CDC ACM | inbox `usbser.inf`, "USB Serial Device (COMn)" | inbox `usbser.inf`, unchanged | **the settings channel** |

All bindings are HOST. No Razer driver package ever targeted the CDC function (HOST: no Razer INF lists `MI_04`). On
the observed PC the port was COM4; a second, non-present dongle instance had been COM6 (HOST). After the removal all
three functions kept working, and third-party code drove the COM port as before (OBSERVED, 2026-11-03).

### 2.2 Not the Razer HID protocol

The 90-byte feature-report protocol of the mouse and keyboard (see razer-hid-protocol.md) does not apply:

- Interface 3 declares no feature and no output report (**PUBLISHED**, openrazer issue #1950; CAPTURED: HID descriptor
  of interface 3 as above).
- Every HID control request an owner sent to interface 3, including the serial-number read of the 90-byte protocol,
  failed with a stall (**PUBLISHED**, openrazer issue #1950).
- In the capture, Synapse sent nothing to the dongle over HID or endpoint 0 except `SET_CONFIGURATION`; all its traffic
  was CDC bulk on `0x06` and `0x86` (CAPTURED, segments 10 to 14).

So there is no transaction id, no status byte in the Razer sense and no storage flag on this device. An early comment
in openrazer issue #1950 that names a transaction id `0x1F` for this headset most likely read another Razer device in
the same capture (INFERRED: the post lists two other Razer devices on that bus).

### 2.3 Opening the COM port

**Finding the port.** Enumerate the device interfaces of `GUID_DEVINTERFACE_COMPORT`
`{86E0D1E0-8089-11D0-9CE4-08003E301F73}` and pick the one whose hardware ID starts `USB\VID_1532&PID_052C&MI_04`.
Synapse does exactly this and opens the interface path `\\?\usb#vid_1532&pid_052c&mi_04#<instance>#{86e0d1e0-...}`
rather than a `COMn` name (HOST: Synapse's middleware log; METADATA: its SetupAPI imports). Never match by COM number,
and never match `052D`.

**Opening.** A communications resource can only be opened exclusively: `CreateFile` with share mode 0 and
`OPEN_EXISTING` (**PUBLISHED**, Microsoft's `CreateFileW` documentation). A second open while another process holds
the port fails with `ERROR_ACCESS_DENIED` (5) (HOST: Synapse's own log shows its open failing that way; OBSERVED,
section 2.5).

**Line settings.**

| Setting | Synapse (METADATA, static reading of its COM-port code) | Third-party code (OBSERVED 2026-11-02) |
|---|---|---|
| Open flags | `GENERIC_READ \| GENERIC_WRITE`, share 0, `OPEN_EXISTING`, `FILE_FLAG_OVERLAPPED` | the same |
| Baud, format | 9600, 8 data bits, no parity, one stop bit | 9600 8N1 |
| DTR | `DTR_CONTROL_ENABLE` | enabled |
| RTS | left as `GetCommState` returned it | enabled |
| Flow control | `fOutxDsrFlow` and `fDsrSensitivity` cleared, the rest as returned | all hardware and XON/XOFF flow control, error-character and null stripping off (the frames are binary; XON/XOFF would swallow `11` and `13` bytes) |
| Timeouts | `ReadIntervalTimeout` = `MAXDWORD`, all others 0: a read returns at once with what has arrived | a bounded wait per read and per write; the receive buffer purged once after opening |
| Event mask | `SetCommMask(EV_RXCHAR \| EV_TXEMPTY)` | not used |

- On the observed PC, `GetCommState` on a freshly opened port already reported `fBinary`, DTR enable and RTS enable
  (DCB flags `0x1011`), so setting them changed nothing (OBSERVED 2026-11-02).
- The baud rate is meaningless on a USB CDC function (INFERRED); the dongle answered at 9600.
- The Linux projects put the tty in raw mode and raise DTR and RTS; the openrazer PR unbinds `cdc_acm` and sends
  `SET_CONTROL_LINE_STATE` with value `0x0003` (DTR and RTS) to interface 4 before its bulk writes (**PUBLISHED**).
- **Not captured**: Synapse had the port open before the capture began, so its `SET_LINE_CODING` and
  `SET_CONTROL_LINE_STATE` requests are not in it. Whether the dongle answers with DTR low is UNKNOWN: every host seen
  raises DTR, and with DTR and RTS raised and no other preparation the dongle answered the first frame (OBSERVED
  2026-11-02, 18:57:09).
- **Opening and closing repeatedly is harmless.** Five open, read, close cycles six seconds apart (DTR raised and
  dropped each time) all answered on the first frame, and the audio did not drop out (OBSERVED 2026-11-02, 18:59:05 to
  18:59:31). Holding the port for a whole session also works (OBSERVED from 2026-11-02 on).

### 2.4 The byte stream

**On the wire (CAPTURED, segments 10 to 14).** Every host-to-dongle bulk transfer on `0x06` was exactly 28 bytes,
including the keepalive (section 3.4). Every dongle-to-host bulk transfer on `0x86` was one whole packet: 31 bytes for a
Command Complete of opcode `0xFFF3`, 8 bytes for the keepalive's Command Complete, 4 bytes for a vendor event. Nothing
arrived on the CDC notification endpoint `0x85` or the audio status endpoint `0x81`.

**Through Windows (OBSERVED 2026-11-02).** Reads on the port returned the leading `04` of each reply on its own and the
other 30 bytes in the next read. Treat the port as a byte stream:

1. Append everything read to a buffer (cap it; OmaKraken keeps 4096 bytes, **PUBLISHED**).
2. Discard bytes up to the next `04`.
3. When at least 3 bytes are buffered, the packet is `3 + ev[2]` bytes long; wait until it is complete.
4. Classify it (section 3.3 to 3.5) and either match it to the outstanding request or handle it as an event.

### 2.5 Exclusive access and coexistence

- **One owner.** Whoever opens the port first owns it; there is no way to share it (**PUBLISHED**, Microsoft; HOST).
- **Synapse's service is the owner while it runs.** `Razer Synapse Service` opens the port when it attaches the dongle
  and keeps it open until it releases the device at service stop or log-off (HOST: Synapse's middleware and feature
  logs, 2026-10-23 to 2026-10-26). Its log shows two of its own threads racing for the port at service start: the
  loser logged "Failed to open" with error 5, and that Synapse module stayed without the device for the whole session
  (HOST). So Synapse does not retry a failed open until the device is re-added: a third-party program that holds the
  port while Synapse starts leaves Synapse's headset features broken until the next re-plug or service restart
  (INFERRED from the log).
- **Do not open the port while the Synapse service runs.** Third-party code checked for the running service and opened
  nothing in that state (OBSERVED 2026-11-02 18:56 and 2026-11-03); with the service stopped it opened the port at
  once (OBSERVED).
- **A brief holder at start-up.** With Synapse stopped but Razer's Chroma SDK services still installed, the first open
  after a service start was refused with access denied and an open 0.4 s later succeeded (OBSERVED 2026-11-02 19:02:17).
  The Chroma SDK's device manager enumerates the port at every boot ("Razer COM Port Device found ... pid_052c&mi_04",
  HOST), so it most likely opened it for a moment (INFERRED). After Razer's software was removed this did not recur
  (OBSERVED 2026-11-03). Retry an access-denied open after a short delay before reporting the port as busy.
- **No interference with audio.** The audio function is independent of the COM port; holding or toggling the port did
  not interrupt playback (OBSERVED 2026-11-02).
- **Lighting from other programs while Synapse runs** reaches the headset only through Synapse (the Chroma SDK maps
  the dongle to a one-LED ChromaLink device and Synapse streams the colour, METADATA and HOST), so a second program
  cannot drive the earcups directly while Synapse owns the port.

## 3. Frames

### 3.1 Host to dongle: 28 bytes

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 1 | HCI packet indicator | `0x01` = command packet (**PUBLISHED**, Bluetooth Core Specification, UART transport) |
| 1-2 | 2 | opcode | `F3 FF` = `0xFFF3` little-endian: OGF `0x3F` (vendor-specific), OCF `0x3F3` |
| 3 | 1 | parameter length | `0x18` = 24 (bytes 4 to 27) |
| 4 | 1 | target | `0x02` in every frame seen; meaning UNKNOWN (INFERRED: a selector, perhaps "headset") |
| 5 | 1 | direction | `0x20` read, `0x21` write |
| 6 | 1 | command | section 4 |
| 7-26 | 20 | arguments | zero-filled; for group `0x40`, `f[7]` is the sub-id |
| 27 | 1 | checksum | section 3.2 |

CAPTURED: every one of the 741 command frames in segments 10 to 14 (48 reads, 693 writes) began `01 F3 FF 18 02`;
the only other host transfers were 63 keepalives (section 3.4). The layout was first **PUBLISHED** by OmaKraken and the openrazer PR, and Synapse's templates agree (METADATA).

### 3.2 Checksum

`f[27]` = the two's complement of the byte sum of the 20 argument bytes `f[7]` to `f[26]`:
`f[27] = (0x100 − (sum mod 256)) mod 256`. The header and the command byte `f[6]` are **not** included (**PUBLISHED**,
OmaKraken; it equals the openrazer PR's closed forms `0x72 − level` for brightness and `0x6C − R − G − B − f[11]` for
colour).

**Which frames carry it (CAPTURED).** Synapse computes the checksum for every group-`0x40` frame and sends `0x00` in
`f[27]` for its `0x25` and `0x26` writes; the headset accepted those writes with status `0x00` and applied them. For a
read with all-zero arguments both rules give `0x00`. OmaKraken computes the checksum for every frame (**PUBLISHED**),
so the dongle probably ignores `f[27]` outside group `0x40` (INFERRED; only `0x00` was used here). Copy Synapse:
computed for `0x40`, zero otherwise.

Worked examples, all CAPTURED frames:

| Frame `f[6..]` (zeros after the last value omitted) | Sum of `f[7..26]` | `f[27]` |
|---|---|---|
| `40 8C 00 00 08 00 C0 00 00` (colour `C0 00 00`, 13:55:43.262) | `8C + 08 + C0` = `0x154` → `0x54` | `0x100 − 0x54` = **`AC`** |
| `40 8A 00 01 02` (haptics on, Medium, 13:52:05.315) | `8A + 01 + 02` = `0x8D` | **`73`** |
| `40 8E 00 00 FF` (brightness 255, 13:52:03.938) | `8E + FF` = `0x18D` → `0x8D` | **`73`** |
| `40 8B` (haptics read, 13:52:05.063) | `0x8B` | **`75`** |
| `40 8C 00 00 03` (mode 3, no colour, 13:59:19.737) | `8C + 03` = `0x8F` | **`71`** |
| `24` (status read) | 0 | **`00`** |
| `26 01 84 03` (power saving on, 900 s, 13:52:03.373) | would be `0x88` → `78` | **`00`** as sent |

### 3.3 Dongle to host: Command Complete, 31 bytes

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 1 | HCI packet indicator | `0x04` = event packet |
| 1 | 1 | event code | `0x0E` = Command Complete |
| 2 | 1 | parameter length | `0x1C` = 28 |
| 3 | 1 | Num_HCI_Command_Packets | `0x01` |
| 4-5 | 2 | opcode | `F3 FF` |
| 6 | 1 | status | `0x00` = success; no other value was ever seen |
| 7 | 1 | target echo | `0x02` |
| 8 | 1 | direction echo | `0x20` / `0x21` as in `f[5]` (unsolicited packets use either, section 6) |
| 9 | 1 | command echo | as in `f[6]` |
| 10-30 | 21 | values `v[0]` to `v[20]` | per command (section 5) |

**Group `0x40` replies (CAPTURED and OBSERVED):** `v[0]` echoes the sub-id, `v[1]` = `01` means accepted, values start
at `v[2]`, and **`ev[30]` is a checksum** by the same rule as the frames, over `v[0]` to `v[19]`. Example: the haptics
read reply `8B 01 01 01 02 ... 70` (13:52:05.284): `8B + 01 + 01 + 01 + 02` = `0x90`, `0x100 − 0x90` = `0x70`. Every
colour-write reply was `8C 01 00 ... 73` (`0x100 − 0x8D`).

**Other replies** carry no checksum: `ev[30]` was `00` in every `0x22`, `0x24`, `0x25`, `0x26` and `0x27` packet
(CAPTURED, OBSERVED).

**Write replies echo the written values.** A `0x25` or `0x26` write reply carries in `v[0..]` exactly the arguments of
the write (CAPTURED: write `00 02`, reply `00 02`, 13:52:04.438/.564; OBSERVED 2026-11-02 for `0x26`). Synapse treats a
`0x26` write as done only when the echo matches (METADATA). The echo confirms receipt, not that the headset applied
the value (section 5.2).

**Matching.** Accept a packet as the reply to a request only if `ev[1..5]` = `0E 1C 01 F3 FF`, `ev[7]` = `02`,
`ev[8..9]` equal `f[5..6]`, and for group `0x40` also `v[0]` = `f[7]`. The sub-id check matters: the HyperSense button
report (section 6.4) is also a group-`0x40` packet with direction `0x20`.

**Latency (CAPTURED).** Of 89 request/reply pairs matched in the capture, 78 were answered 85 to 240 ms after the
request (median 139 ms); the other matches are ambiguous, because Synapse sent many requests in identical pairs. Third-party code saw 0.1 to 0.2 s
(OBSERVED 2026-11-02). The keepalive (section 3.4) was answered within 0.2 ms. INFERRED: settings commands travel over
the radio to the headset and back, while the keepalive is answered by the dongle itself. A reply timeout of 0.7 s was
enough for every read (OBSERVED).

### 3.4 Keepalive (HCI opcode `0xFC9A`)

| Direction | Bytes (CAPTURED) |
|---|---|
| host → dongle | `01 9A FC 00` + 24 × `00` (28-byte transfer; an HCI command with no parameters, padded) |
| dongle → host | `04 0E 05 01 9A FC 00 01` (Command Complete, status 0, one parameter byte `01`) |

- **When Synapse sends it (CAPTURED).** Not continuously: in stretches at 0.50 s intervals (13:51:44.8 to 46.7 and
  13:59:37.3 to 41.5, each right after a headset link-up; 13:59:46.3 to 14:00:03.1 before Synapse re-read the
  settings), with short runs 6 to 15 ms apart inside them. 63 keepalives in about 12 minutes; none in the whole of
  segments 11 and 14.
- **It is not needed.** Third-party code never sent it; the dongle answered every frame and reported link changes
  for days (OBSERVED 2026-11-02 to 2026-11-04). Its purpose is UNKNOWN (INFERRED: Synapse's own check that the dongle
  is alive). The parser must still recognise its reply if another host sends it.

### 3.5 Vendor events (4 bytes)

| Bytes | Meaning | Source |
|---|---|---|
| `04 FF 01 E8` | headset link up (switched on or back in range) | CAPTURED 13:51:43.804 and 13:59:36.264; OBSERVED |
| `04 FF 01 E9` | headset link down (switched off or out of range) | CAPTURED 13:59:19.482; OBSERVED |

HCI event code `0xFF` is the vendor-specific event (**PUBLISHED**, Bluetooth Core Specification); one parameter byte.
Synapse's middleware logs the same transitions as device events `0x03030900` (up) and `0x03020900` (down) (HOST,
INFERRED mapping by time).

### 3.6 A complete exchange

Status read, CAPTURED 13:52:05.855 / 13:52:06.005:

```
OUT  01 F3 FF 18 02 20 24 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
     |  opcode |  |  |  |  20 argument bytes, all 0                                  checksum 00
     HCI cmd   |  |  |  command 0x24
               |  |  direction 0x20 = read
               |  target 0x02
               parameter length 24

IN   04 0E 1C 01 F3 FF 00 02 20 24 01 0E B6 1E 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
     |  |  |  |  opcode|  |  |  |  v[0] v[1] v[2] v[3]
     |  |  |  |        |  |  |  command echo 0x24
     |  |  |  |        |  |  direction echo
     |  |  |  |        |  target echo
     |  |  |  |        status 00
     |  |  |  Num_HCI_Command_Packets
     |  |  length 28
     |  Command Complete
     HCI event
```

`v[0]` = `01` on battery, `v[1]` = `0E`, `v[2]` = 182 (raw), `v[3]` = `0x1E` = 30 % (section 5.1).

## 4. Command set

| Cmd | Dir | Name | Arguments `f[7..]` | Reply `v[..]` | Tag |
|---|---|---|---|---|---|
| `0x20` | read | firmware version | none | `v[0..2]` = major, minor, patch | OBSERVED (`01 06 00`); METADATA template; not sent by Synapse in the capture |
| `0x21` | read | serial number | none | ASCII in `v[0..14]` | OBSERVED (15 characters, not reproduced); PUBLISHED (OmaKraken); not sent by Synapse in the capture |
| `0x22` | read | unknown | none | `v[0]` | CAPTURED (all zeros, 13:52:02.444); meaning UNKNOWN |
| `0x22` | write | unknown | one byte | | METADATA template only; **do not send** |
| `0x24` | read | status and battery | none | `v[0]` flags, `v[1]`, `v[2]` raw value, `v[3]` battery % | CAPTURED; OBSERVED; PUBLISHED (OmaKraken) |
| `0x25` | read | sidetone | none | `v[0]` on/off, `v[1]` level 0-4 | CAPTURED; OBSERVED; PUBLISHED |
| `0x25` | write | sidetone | `f[7]` on/off, `f[8]` level 0-4 | echo | CAPTURED; OBSERVED; PUBLISHED |
| `0x26` | read | power saving | none | `v[0]` on/off, `v[1..2]` seconds LE | CAPTURED; OBSERVED; PUBLISHED |
| `0x26` | write | power saving | `f[7]` on/off, `f[8..9]` seconds LE | echo | CAPTURED; OBSERVED; PUBLISHED |
| `0x27` | event | unknown, after link-up | | `v[0]` = `01` | CAPTURED (unsolicited, section 6.3); meaning UNKNOWN |
| `0x40/0x8A` | write | haptics | `f[8]` `00`, `f[9]` on/off, `f[10]` level 1-3 | `8A 01` | CAPTURED; OBSERVED; PUBLISHED |
| `0x40/0x8B` | read | haptics | none | `8B 01`, `v[3]` on/off, `v[4]` level | CAPTURED; OBSERVED; METADATA |
| `0x40/0x8C` | write | earcup colour | `f[8..9]` `00 00`, `f[10]` mode `08`, `f[11]` `00`, `f[12..14]` R G B | `8C 01` | CAPTURED; OBSERVED; PUBLISHED |
| `0x40/0x8D` | read | lighting effect | none | layout expected by Synapse in section 5.5 | METADATA only; never sent by anyone observed |
| `0x40/0x8E` | write | brightness | `f[8..9]` `00 00`, `f[10]` 0-255 | `8E 01` | CAPTURED; OBSERVED; PUBLISHED |
| `0x40/0x8F` | read | brightness | none | `8F 01`, `v[4]` expected | METADATA only; never sent by anyone observed |
| `0x40/0x90` | event | HyperSense button | | `90 01 00 01 03` | CAPTURED (unsolicited, section 6.4) |
| `0x93`, `0x94` | ? | lighting / haptics on-off gates | | | PUBLISHED by ShadowXCC only; not captured, not sent here: UNKNOWN |
| opcode `0xFC9A` | | keepalive | none | `01` | CAPTURED (section 3.4) |
| anything else | | | | | **do not send** (section 9) |

Write/read pairs in group `0x40` differ in bit 0 of the sub-id: `8A`/`8B`, `8C`/`8D`, `8E`/`8F` (METADATA). Synapse's
templates contain no other command bytes (METADATA), and the capture showed none (CAPTURED).

## 5. Commands

### 5.1 `0x24`: status and battery

| Value | Meaning | Evidence |
|---|---|---|
| `v[0]` | flags; **bit `0x04` = charging**; `01` on battery, `05` on the charger | CAPTURED `01` throughout (on battery); OBSERVED 2026-11-02: `05` on the charger, `01` after unplugging. OmaKraken reads bit `0x04` as charging (**PUBLISHED**), confirmed |
| `v[1]` | `0E` on battery, `0F` on the charger | CAPTURED, OBSERVED; meaning UNKNOWN (INFERRED: bit 0 also tracks the charger) |
| `v[2]` | a raw value that falls on battery and jumps when charging starts | CAPTURED: 202 → 183 → 169 over 13 minutes on battery; OBSERVED 2026-11-02: 117 at 10 % on battery, 40 right after plugging in, 68 to 75 within a minute |
| `v[3]` | battery per cent, **in 10 % bands** | CAPTURED: only `32`, `28`, `1E` (50, 40, 30); OBSERVED: 10, 20, 30, 40, 50, 60, 80, 90 |

- **Correction to earlier public descriptions.** OmaKraken names `v[2]` "signal" (**PUBLISHED**). It is not a radio
  signal: it fell steadily on battery and dropped from 117 to 40 the moment the charger was connected (OBSERVED
  2026-11-02). INFERRED: a raw battery or charge-current reading. Its scale is UNKNOWN; expose it raw, if at all.
- **10 % bands.** In the capture every value was a multiple of 10, and Synapse's own logged battery events
  (`0x00LL3100` with `LL` = `32`, `28`, `1E`, HOST, 2026-10-25) carry the same bytes; the firmware reports bands, not
  single per cent (INFERRED).
- **The first value is an estimate.** Right after link-up the headset reported 50 % (13:51:45.245) and 40 %
  (13:51:57.804) unsolicited, then 30 % in every polled reply for the rest of the session, while Synapse's battery tile
  showed one bar of three (CAPTURED). On the charger the value went 10 → 20 → 40 → 50 % within the first minute
  (OBSERVED 2026-11-02, 18:57 to 18:59:31). Smooth or delay alerts after a link-up or a charger change.
- **Synapse's handling (METADATA).** It rescales `v[3]` to 0-255 (`v[3] × 255 / 100`) and maps `v[0]` through an
  eight-entry table (0, 4, 7 → one state; 5 → another; 1, 2, 3, 6 → a third). What its states mean is UNKNOWN.
- **Headset off.** With the headset switched off the dongle sends **no reply at all**: no Command Complete with an
  error status, only silence (OBSERVED 2026-11-02, 18:58:15, two reads 0.77 s apart; every start with the headset off
  since). A timeout on `0x24` therefore means "headset not connected", and it is the cheapest first command after
  opening the port.
- **Unsolicited copies** arrive about 1.4 s after each link-up and sometimes later (section 6.3).

### 5.2 `0x25`: sidetone

| Value | Meaning |
|---|---|
| `v[0]` / `f[7]` | `00` off, `01` on |
| `v[1]` / `f[8]` | level 0 to 4 |

- **CAPTURED.** At tab open the headset read `00 04` (off, level 4). The owner's actions produced: enable → `01 04`;
  slider at 73 % → `01 03`; 50 % → `01 02` (13:55:05 to 13:55:25). Synapse's code maps a slider percentage p to a
  level by ≤ 20 → 0, ≤ 40 → 1, ≤ 60 → 2, ≤ 80 → 3, above → 4 and shows levels as 20, 40, 60, 80, 100 % (METADATA);
  73 % → 3 and 50 % → 2 agree.
- **OBSERVED 2026-11-02, 19:03 to 19:07** (headset on the charger): on and off switched the sidetone audibly both ways;
  level 0 was quieter but not silent (0 is the lowest audible step); a change from 2 to 4 was not heard. Every write
  was answered with status `00`. Writes work while charging.
- **Stored on the headset**: the value read back after a restart of the host software and after the headset had been
  off overnight was the last one written (OBSERVED 2026-11-02 19:14 and 2026-11-03 05:08).
- **Stale reads.** See section 7.1: of seven writes, four read back the value from before them.
- Every write carries both fields. Take the other field from the last value you wrote or from a settled read, never
  from a read taken right after another write (OBSERVED: an "off" write that took its level from a stale read sent
  level 1; the headset kept 2).

### 5.3 `0x26`: power saving

| Value | Meaning |
|---|---|
| `v[0]` / `f[7]` | `00` off, `01` on (switch the headset off after the idle time) |
| `v[1..2]` / `f[8..9]` | idle time in **seconds**, u16 little-endian |

**Minutes encoding.** Synapse's slider is in minutes and the wire value is minutes × 60 (CAPTURED, 13:53:27 to
13:53:38):

| Synapse | Seconds | Bytes `f[7..9]` |
|---|---|---|
| on, 15 min | 900 = `0x0384` | `01 84 03` |
| off (timeout kept) | 900 | `00 84 03` |
| on, 60 min | 3600 = `0x0E10` | `01 10 0E` |
| on, 16 min | 960 = `0x03C0` | `01 C0 03` |

- Turning power saving off keeps the timeout in the frame (CAPTURED).
- Synapse's device definition limits the time to 900 to 3600 s (METADATA); OmaKraken offers 15 to 60 min
  (**PUBLISHED**). Values outside that range were never sent; the firmware's own limits are UNKNOWN.
- The write reply echoes the new value and an immediate read returns it (OBSERVED 2026-11-02 19:18: 16 min and back to
  15 min, each confirmed by the first read 0.27 to 0.37 s after the write).
- Stored on the headset (OBSERVED, as in section 5.2). Synapse rewrites the current value at every tab open and at
  start (CAPTURED, section 6.1); the headset accepts the unchanged value.

### 5.4 `0x40/0x8A`, `0x8B`: haptics (HyperSense)

| Field | Write `0x8A` | Read reply `0x8B` |
|---|---|---|
| sub-id | `f[7]` = `8A` | `v[0]` = `8B` |
| accepted | | `v[1]` = `01` |
| reserved | `f[8]` = `00` | `v[2]` = `00` |
| on/off | `f[9]` | `v[3]` |
| level | `f[10]`: `01` Low, `02` Medium, `03` High | `v[4]` |

- **CAPTURED.** At tab open Synapse read `8B 01 01 01 02` (on, Medium). The owner's actions: off → `8A 00 00 02`
  (level kept), on → `01 02`, High → `01 03`, Low → `01 01`, Medium → `01 02` (13:54:01 to 13:54:07). Synapse sent each
  of these writes twice back to back and got one reply `8A 01 00 ... 75` per pair; for on and off the first of the
  pair carried the previous state (INFERRED: Synapse's separate setters for the enable and the level both fire).
- **Correction to earlier public descriptions.** OmaKraken states that haptics cannot be read back (**PUBLISHED**).
  `0x8B` returns the live state: Synapse reads it (CAPTURED), and third-party code saw writes reflected in it within
  0.5 s on battery (OBSERVED 2026-11-02, 19:17).
- **Haptics change only on battery.** On the charger a `0x8A` write is acknowledged (`v[1]` = `01`) but `0x8B` keeps
  returning the old value and nothing changes (OBSERVED 2026-11-02, 19:16:31, eight reads over 1.6 s). This matches
  Razer's guide, which disables the HyperSense button while charging (**PUBLISHED**). On battery, off, on and Low were
  confirmed by the first read and felt by the owner (OBSERVED 19:17).
- **Stored on the headset.** The level written (Low) read back after a host restart and after the headset had been off
  overnight (OBSERVED 2026-11-02 19:26 and 2026-11-03 05:08).
- **The HyperSense button** changes the same state without the host; the dongle reports it (section 6.4).
- **No haptic pulse from a generated tone** (OBSERVED 2026-11-04, third-party code): HyperSense is described as deriving
  its vibration from the audio signal, so a host might trigger a pulse by playing a short low tone on the headset's
  Windows playback endpoint. Tones of 300 and 500 ms at 60 Hz and 800 ms at 40 Hz, at 60 % amplitude, were audible as a
  thump and produced no vibration with HyperSense at Low, at High and switched off, all three the same. A "vibrate now"
  command remains unknown; an audio-driven pulse from a pure tone this short does not work on firmware 1.6.0.
- ShadowXCC describes the level as an index 0 to 3 for 0/35/60/100 % (**PUBLISHED**). On the wire Synapse sends a
  separate on/off byte and levels 1 to 3 (CAPTURED); follow the wire.

### 5.5 `0x40/0x8C`: earcup colour

| Offset | Value |
|---|---|
| `f[7]` | `8C` |
| `f[8..9]` | `00 00` |
| `f[10]` | mode: `08` = direct colour |
| `f[11]` | `00` |
| `f[12..14]` | red, green, blue |
| `f[15..26]` | `00` |
| `f[27]` | checksum |

- **Synapse renders every effect on the PC (CAPTURED, segments 10, 11, 13 and 14).** 651 of the 652 colour frames had
  mode `08`, `f[11]` = `00` and one colour triplet. Static was one frame every 5.0 s with the same colour; Breathing was
  a ramp at about 21 frames per second; Spectrum a hue sweep at the same rate; elsewhere the stream ran at up to 25
  frames per second. The brightness slider sent no brightness command: a static red went out as `C0 00 00`, so Synapse
  scales the colour itself (INFERRED). The frame format was first **PUBLISHED** by OmaKraken and the openrazer PR.
- **Replies are not reliable while streaming.** At one frame per 5 s every frame was answered about 0.2 s later
  (`8C 01 00 ... 73`). During the fast stream of segment 11, 331 colour frames drew 19 replies, none during the 12 s
  ramp from 13:57:00.979 (CAPTURED). Synapse does not wait for colour replies (METADATA: its direct-colour builder
  writes without reading). Do the same: send colour frames without waiting, and drain the replies.
- **`f[11]` and the earcups.** The openrazer PR sends each colour twice, with `f[11]` = `00` and then `01`, "to match
  Synapse dual-zone updates"; OmaKraken sends `01` only (**PUBLISHED**). Synapse sent `00` only, one frame per colour
  (CAPTURED). Third-party code sent `00` only and **both** earcups showed the colour (OBSERVED 2026-11-02 19:19:30).
  Synapse's effect builder writes the number of colours into `f[11]` (METADATA), so `f[11]` is most likely a colour
  count, not an earcup selector (INFERRED). **Correction:** one frame with `00` drives both earcups; whether the earcups
  can be addressed separately is UNKNOWN, and every Razer file models them as one LED (METADATA).
- **Mode `03`.** Once, at 13:59:19.737, 0.26 s after the link-down event of a headset switch-off, Synapse sent
  `40 8C 00 00 03 00 00 00 00 ... 71`: mode `03`, no colour. It got no reply (the headset was off) (CAPTURED). Synapse's
  effect builder maps several of its effects to mode `03`, and its release path logs a "spectrum cycling" effect
  (METADATA, HOST). INFERRED: mode `03` selects the headset's built-in colour cycle. Not tested: do not send it
  (section 9).
- **The colour holds until the headset is switched off (OBSERVED 2026-11-02, 19:19 to 19:27).** A direct colour stayed
  on with nothing sent for minutes, through a restart of the host software. Switching the headset off and on brought
  back the power-on default, a slow colour cycle, with nothing sent. Nothing is stored. **Correction to earlier public
  descriptions:** OmaKraken says the headset falls back to its own cycle when the stream stops (**PUBLISHED**); on
  firmware 1.6.0 it does not. Synapse's exit sent nothing to the earcups (CAPTURED 14:00:26), so under Synapse too the
  last colour stays until power-off (INFERRED).
- **Repeating a static colour** every 5 s, as Synapse does, is not needed to keep it on (OBSERVED, as above). A host
  that does it must stop when it no longer drives the lights, or the colour never ends (OBSERVED 2026-11-02: a
  periodic re-send kept a preview colour on after the preview ended).
- **Charging.** Razer's guide says earcup lighting is disabled while the headset charges (**PUBLISHED**); not checked
  here. ShadowXCC reports that the headset itself turns lighting and haptics off when it goes on the charger
  (**PUBLISHED**), which would also explain section 5.4 (INFERRED).
- **Effect read `0x8D`.** Synapse's handler expects `v[1]` = `01`, `v[4]` mode, `v[5]` colour count, `v[6..11]` two
  colours (METADATA). Never sent in the capture or on the bench; device behaviour UNKNOWN.

### 5.6 `0x40/0x8E`, `0x8F`: brightness

- Write `40 8E 00 00 LL`, `LL` = 0 to 255; reply `8E 01` (CAPTURED: `LL` = `FF` once at tab open, 13:52:03.938).
  Checksum `0x72 − LL` (mod 256) (**PUBLISHED**, openrazer PR; CAPTURED for 255).
- Synapse did not use it for its brightness slider (section 5.5). The openrazer PR sends brightness before the first
  colour after power-on (**PUBLISHED**). Third-party code sent `FF` once after each open or link-up and carried the
  brightness in the colour values; colours showed as expected (OBSERVED 2026-11-02).
- Whether brightness survives a power cycle, and what `00` does to a following colour, is UNKNOWN.
- Read `0x8F`: Synapse's handler expects `v[1]` = `01` and the level in `v[4]` (METADATA). Never sent; UNKNOWN.

### 5.7 `0x20`, `0x21`, `0x22`: identity and the unknown read

- **`0x20` firmware.** Reply `01 06 00` = 1.6.0 (OBSERVED 2026-11-02 18:57:09). Synapse copies `v[0..2]` and adds a
  fourth component 0 (METADATA). Whether this is the dongle's or the headset's firmware is UNKNOWN. An owner reported
  1.10.0 as the latest version (**PUBLISHED**, openrazer issue #1950).
- **`0x21` serial.** Reply: 15 ASCII characters in `v[0..14]` (OBSERVED; the value is not reproduced here). Synapse
  copies exactly 15 bytes; OmaKraken cuts at the first `00` (METADATA, **PUBLISHED**). Whether it is the dongle's or the
  headset's serial is UNKNOWN. Do not log it.
- **`0x22` read.** Synapse read it once at tab open; the reply was all zeros (CAPTURED 13:52:02.294/.444). Meaning
  UNKNOWN. Synapse also has a one-byte `0x22` **write** template (METADATA) that it never sent; it was never sent here
  either. Do not send it.

## 6. Unsolicited packets

### 6.1 What Synapse sends (CAPTURED)

| When | Traffic on the CDC channel |
|---|---|
| Headset switched on (13:51:43.804) | the dongle: link up, then `0x27` and `0x24` reports; Synapse: keepalives for about 2 s; nothing else written until the tab opened |
| Kraken tab opened (13:52:02) | `0x22` read; `0x26` read, write of the same value, read, write; `0x8E` `FF`; `0x25` read `00 04`, write `00 02`, read `00 04`, write `00 04`; `0x8B` read; `0x8A` `00 01 02` twice; `0x24` reads; colour stream |
| Each setting changed | read, write, read, write of the new value, about 0.1 s apart |
| Lighting changed (13:55 to 13:57) | colour frames only (section 5.5) |
| Headset switched off (13:59:19.482) | the dongle: link down; Synapse: one mode-`03` colour frame 0.26 s later |
| Headset switched on (13:59:36.264) | link up, keepalives, unsolicited `0x24` |
| Synapse exit (14:00:26) | nothing; the static-colour repeat had stopped at 14:00:18.862 |
| Synapse start (14:01:36) | `0x24` reads (in pairs), `0x26` read and write twice, colour stream from 14:01:37 |

- Synapse sent many reads in identical pairs at the same instant and got two replies (CAPTURED); INFERRED: two of its
  modules poll independently.
- At tab open Synapse wrote sidetone `00 02` and, after reading back the old `00 04`, wrote `00 04` again (CAPTURED):
  Synapse itself fell for the stale read of section 7.1.
- Synapse never sent `0x20`, `0x21`, `0x8D`, `0x8F` or a `0x22` write in the capture, and nothing resembling a device-
  mode or release command at exit or start (CAPTURED).

### 6.2 Kinds of unsolicited packets

| Packet | Bytes | Tag |
|---|---|---|
| link up / down | `04 FF 01 E8` / `04 FF 01 E9` | CAPTURED, OBSERVED (section 3.5) |
| status `0x24` | `04 0E 1C 01 F3 FF 00 02 20 24 01 0E CA 32 ...` | CAPTURED |
| `0x27` | `04 0E 1C 01 F3 FF 00 02 21 27 01 00 ...` | CAPTURED |
| HyperSense button | `04 0E 1C 01 F3 FF 00 02 20 40 90 01 00 01 03 ... 6B` | CAPTURED |
| keepalive reply | `04 0E 05 01 9A FC 00 01` | CAPTURED, only in answer to a keepalive |

They arrive in the same stream as replies, between a request and its reply. Handle every packet that does not match
the outstanding request as an event; never take an unsolicited `0x24` or `0x90` for the reply of something else.

### 6.3 After a link-up

CAPTURED, headset switched on at 13:51:43.804:

| Time after link-up | Packet |
|---|---|
| 0 | `04 FF 01 E8` |
| +1.12 s | `0x27` event, direction echo `21`, `v[0]` = `01` |
| +1.44 s | `0x24` status `01 0E CA 32` (50 %) |
| +14.0 s | `0x24` status `01 0E B7 28` (40 %) |

The second link-up (13:59:36.264) brought the `0x24` report after +1.44 s again (`01 0E BB 32`). The meaning of
`0x27` is UNKNOWN (INFERRED: "headset ready"; it carries the write direction `21` although nobody wrote). A host can
read the settings once the `0x24` report has arrived; third-party code read them right after the link-up event and
had the status 0.7 s and all settings 1.5 s after it (OBSERVED 2026-11-03 05:08:47).

### 6.4 HyperSense button (`0x40/0x90`)

Pressing the HyperSense button on the right earcup produced, with no request outstanding:

```
04 0E 1C 01 F3 FF 00 02 20 40 90 01 00 01 03 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 6B
```

(CAPTURED 14:00:17.744). `v[0]` = `90`, `v[1]` = `01`, `v[3]` = `01` haptics on, `v[4]` = `03` High: the same layout as
the `0x8B` reply. Haptics had been on at Medium, so the press stepped Medium → High, as the guide's Low, Medium, High,
off cycle predicts (**PUBLISHED**). Checksum `0x100 − (90 + 01 + 01 + 03)` = `6B`. Update the host's haptics state from
it.

### 6.5 What produces nothing

- **Microphone mute button**: pressing and releasing it produced nothing on any endpoint of the dongle, CDC, HID or
  audio status (CAPTURED, segment 12). The mute happens inside the headset and the host cannot see it on USB
  (INFERRED).
- **The power button's media presses** go to the HID consumer interface as standard key reports (**PUBLISHED**,
  report descriptor), not to the CDC channel. **The volume wheel does not reach the host at all** (OBSERVED 2026-11-04):
  it changes the headset's own output level, has end stops, and Windows' endpoint volume does not move when it turns, so
  a host program's volume control (the Windows endpoint) and the wheel act as two independent levels. The mute button
  likewise leaves Windows' recording-endpoint mute untouched (OBSERVED 2026-11-04), consistent with the capture above.
- One HID report `01` then `00` arrived on `0x84` 0.15 s after a sidetone write (CAPTURED 13:52:04.899); bit 0 is the
  first usage of the report descriptor. Cause UNKNOWN; it did not recur.

## 7. Using the channel

### 7.1 Stale reads after a write

A read sent right after a write often returns the value from **before** the write:

- **OBSERVED 2026-11-02, 19:03 to 19:07** (sidetone, read 0.1 to 0.5 s after each write): of seven writes, four read
  back the previous value; the read after writing level 1 returned the level 3 written 26 s earlier.
- **CAPTURED 13:52:04.438 to .724**: Synapse wrote `00 02`, the write reply echoed `00 02`, and the read 30 ms later
  returned `00 04`.
- Power-saving writes read back at once (OBSERVED); haptics writes on battery within 0.5 s (OBSERVED).

INFERRED: the dongle answers reads from its own copy of the headset's settings, which follows the headset only after
the headset has applied the write. Consequences:

1. Neither the write reply nor one read confirms a write. Read back repeatedly until the new value shows: third-party
   code read up to 8 times, 200 ms apart, and every later write on battery was confirmed within 0.27 to 0.55 s (OBSERVED
   2026-11-02, 19:14 to 19:18).
2. Build a write's second field from the value you last wrote or a settled read (section 5.2).
3. If the value never shows, report the write as unconfirmed; on the charger this is the normal outcome for haptics
   (section 5.4).

### 7.2 Pacing

- **One request at a time** for reads and settings writes; wait for the reply or a 0.7 s timeout. Retry a read once
  after a short delay; do not resend writes, read back instead (OBSERVED practice that worked from 2026-11-02 on).
- Synapse sent pairs of identical frames at the same instant and sometimes got one reply (CAPTURED); whether the dongle
  drops a frame that arrives while it waits for the headset is UNKNOWN. Avoid overlapping requests.
- **Colour frames**: at most 25 per second, without waiting for replies (CAPTURED rate; section 5.5). Coalesce: send
  the newest colour, not a backlog.
- Polling the status every 60 s while idle is enough to follow the battery; the link events arrive by themselves
  (OBSERVED).

### 7.3 Recommended start

| Step | Action | Note |
|---|---|---|
| 1 | Check that no other program owns the port (Synapse's service) | section 2.5 |
| 2 | Open the port exclusively, 9600 8N1, DTR and RTS on; on access denied retry once after about 0.5 s | section 2.3 |
| 3 | `0x24` | no reply in 0.7 s, twice → headset off; keep the port and wait for `04 FF 01 E8` |
| 4 | `0x21`, `0x20`, `0x25`, `0x26`, `0x40/0x8B` | each answered in about 0.1 to 0.2 s |
| 5 | `0x40/0x8E` `FF` before the first colour | once per open or link-up |
| 6 | colours as needed | section 7.2 |

This is the sequence third-party code used on every start from 2026-11-02 (OBSERVED: steps 2 to 4 took 0.85 s with the
headset on, 1.5 s with it off).

### 7.4 Leaving

Nothing has to be undone: the dongle has no driver mode, and no release command was seen from Synapse (CAPTURED,
section 6.1). Stop the colour stream and close the port. The last colour stays until the headset is switched off
(section 5.5); the settings stay on the headset. To hand the device back to Synapse, close the port before Synapse's
service starts (section 2.5).

## 8. Standalone behaviour

| Situation | Behaviour | Tag |
|---|---|---|
| Headset off or out of range | the dongle stays enumerated, audio endpoints remain, every command goes unanswered; link down is reported | OBSERVED 2026-11-02 onwards; CAPTURED for the event |
| Headset power-on, no host software | the earcups run a slow colour cycle | OBSERVED 2026-11-02 |
| Host stops sending colour | the last colour stays until the headset is switched off | OBSERVED 2026-11-02 |
| Headset switched off and on | sidetone, power saving and haptics level as last written; colour back to the cycle | OBSERVED 2026-11-02 and 2026-11-03 |
| Charger plugged in or out | the settings channel keeps answering; the headset also enumerates as `052D` | OBSERVED 2026-11-02; **correction:** OmaKraken reports that a charger change can stop the channel until the dongle is re-plugged (**PUBLISHED**); not seen on firmware 1.6.0 |
| On the charger | haptics writes ignored; sidetone and power-saving writes work | OBSERVED 2026-11-02 |
| Windows sleep and resume | after an eight-hour sleep the first write to the port timed out (Win32 error 121, semaphore timeout); closing and reopening the port 2 s later worked | OBSERVED 2026-11-03 04:07 |
| No Razer software installed | all functions work on Windows' inbox drivers | OBSERVED 2026-11-03 onwards |

Recovery: treat a write or read that fails at the port level (not a reply timeout) as "reopen the port", bound every
port call with a timeout, and read the state again after reopening (OBSERVED, as above). A reply timeout alone means
the headset is not connected, not that the port failed.

## 9. Hazards

| Hazard | Guard |
|---|---|
| Firmware update or pairing commands on the same port: NXP's reference dongle transfers headset firmware over its virtual COM port (**PUBLISHED**, AN12647); Razer ships a separate firmware updater and pairing utility for this headset (**PUBLISHED**, Razer support) | Build frames only from an allow-list: opcode `0xFFF3` with `f[4]` = `02`; reads `0x20`, `0x21`, `0x24`, `0x25`, `0x26`, `0x40/0x8B`; writes `0x25`, `0x26`, `0x40/0x8A`, `0x40/0x8C` with mode `08`, `0x40/0x8E`. **Do not send**: any other HCI opcode (the keepalive `0xFC9A` is harmless but unnecessary), any other `f[4]`, any other command byte, the `0x22` write, any other `0x40` sub-id, any `0x8C` mode other than `08` (including `03`), `0x93`, `0x94`. The update and pairing procedures are deliberately not described |
| The headset's cable port `1532:052D` ("HCI VCOM") and Razer boot-loader devices | Never open them. Match `1532:052C` `MI_04` only. Synapse's middleware also references `1532:110D`, which a public driver database lists as a Razer boot loader (METADATA; INFERRED); never open it |
| Factory reset and unpairing exist in Synapse with unknown frames (**PUBLISHED**, Razer's guide; METADATA: generic `DeviceConfig_PairingMode`, `DeviceConfig_UnPair` names) | Not reachable through the allow-list. Unpairing could cut the headset off its dongle (INFERRED) |
| Starving Synapse: it does not retry a failed open (section 2.5) | Never open the port while Synapse's service runs or is starting |
| Unsolicited packets taken for replies | Match direction, command and the `0x40` sub-id (section 3.3) |
| Stale read-back undoing a change (section 7.1) | Read back until the value shows; never build a write from a read taken right after another write |
| Haptics writes silently ignored on the charger | Read `0x8B` after writing; report "not applied" instead of retrying |
| Persistent settings (sidetone, power saving, haptics level) and possible flash wear | Write only committed values, not every slider step; at most about one write per setting per second (INFERRED, wear is UNKNOWN) |
| Power saving off drains the battery | Treat "off" as a deliberate choice; keep the timeout within 900 to 3600 s |
| Radio load: lighting shares the link with audio | At most 25 colour frames per second, as Synapse; coalesce (INFERRED caution; no audio drop was observed at that rate) |
| A colour stream that never ends | Stop sending when nothing drives the lights; the colour persists anyway (section 5.5) |
| A wedged port after resume | Bound every port call; reopen on a port-level failure (section 8) |
| Photosensitivity with host-rendered flashing | Limit full flashes to 3 per second unless the user opts in (INFERRED, general practice) |

## 10. Test vectors

28-byte frames are written with the zero padding shown as `...` before the checksum; replies are 31 bytes with
trailing zeros shown as `...`.

**CAPTURED vectors** (Synapse 3, 2026-10-30, dongle at address 18):

| Id | Meaning | Bytes |
|---|---|---|
| T1 | status read | `01 F3 FF 18 02 20 24` + 20 × `00` + `00` |
| T2 | status reply: on battery, raw 182, 30 % | `04 0E 1C 01 F3 FF 00 02 20 24 01 0E B6 1E` + 17 × `00` |
| T3 | sidetone write: on, level 3 (checksum `00` as sent) | `01 F3 FF 18 02 21 25 01 03 ... 00`; reply `04 0E 1C 01 F3 FF 00 02 21 25 01 03 ...` |
| T4 | power saving write: on, 3600 s | `01 F3 FF 18 02 21 26 01 10 0E ... 00`; reply echoes `01 10 0E` |
| T5 | haptics write: on, High | `01 F3 FF 18 02 21 40 8A 00 01 03 ... 72`; reply `04 0E 1C 01 F3 FF 00 02 21 40 8A 01 00 ... 75` |
| T6 | haptics read and reply: on, Medium | `01 F3 FF 18 02 20 40 8B ... 75`; reply `04 0E 1C 01 F3 FF 00 02 20 40 8B 01 01 01 02 ... 70` |
| T7 | brightness 255 | `01 F3 FF 18 02 21 40 8E 00 00 FF ... 73` |
| T8 | colour `C0 00 00` | `01 F3 FF 18 02 21 40 8C 00 00 08 00 C0 00 00 ... AC`; reply `04 0E 1C 01 F3 FF 00 02 21 40 8C 01 00 ... 73` |
| T9 | colour `03 F5 03` (Spectrum sweep) | `01 F3 FF 18 02 21 40 8C 00 00 08 00 03 F5 03 ... 71` |
| T10 | mode `03` after link-down (do not send) | `01 F3 FF 18 02 21 40 8C 00 00 03 ... 71` |
| T11 | keepalive | `01 9A FC 00` + 24 × `00`; reply `04 0E 05 01 9A FC 00 01` |
| T12 | link up / down | `04 FF 01 E8` / `04 FF 01 E9` |
| T13 | unsolicited `0x27` | `04 0E 1C 01 F3 FF 00 02 21 27 01` + 20 × `00` |
| T14 | HyperSense button: on, High | `04 0E 1C 01 F3 FF 00 02 20 40 90 01 00 01 03 ... 6B` |
| T15 | `0x22` read and reply | `01 F3 FF 18 02 20 22 ... 00`; reply `04 0E 1C 01 F3 FF 00 02 20 22` + 21 × `00` |

T5's checksum: `8A + 01 + 03` = `0x8E`, `0x100 − 0x8E` = `0x72`. T9: `8C + 08 + 03 + F5 + 03` = `0x18F` → `0x8F`,
`0x100 − 0x8F` = `0x71`.

**OBSERVED vectors** (third-party code, 2026-11-02):

| Id | Meaning | Bytes |
|---|---|---|
| T16 | firmware reply: 1.6.0 | `04 0E 1C 01 F3 FF 00 02 20 20 01 06 00 ...` |
| T17 | status on battery, 10 %, raw 117 | `04 0E 1C 01 F3 FF 00 02 20 24 01 0E 75 0A ...` |
| T18 | status on the charger, 20 %, raw 40 | `04 0E 1C 01 F3 FF 00 02 20 24 05 0F 28 14 ...` |
| T19 | sidetone reply: off, level 2 | `04 0E 1C 01 F3 FF 00 02 20 25 00 02 ...` |
| T20 | power-saving reply: on, 900 s | `04 0E 1C 01 F3 FF 00 02 20 26 01 84 03 ...` |
| T21 | haptics reply: on, High | `04 0E 1C 01 F3 FF 00 02 20 40 8B 01 01 01 03 ... 6F` |
| T22 | power saving 16 min | `01 F3 FF 18 02 21 26 01 C0 03 ... 00`; reply echoes `01 C0 03` |
| T23 | colour `FF 00 00` (both earcups red) | `01 F3 FF 18 02 21 40 8C 00 00 08 00 FF 00 00 ... 6D` |
| T24 | headset off | T1 sent twice 0.77 s apart: no bytes at all in reply |

T23 is also byte-identical to the Synapse frame an owner replayed in openrazer issue #1950 (**PUBLISHED**).

**Built from the layouts** (INFERRED arithmetic):

| Id | Meaning | Bytes |
|---|---|---|
| T25 | power saving on, 30 min (1800 s = `0x0708`) | `01 F3 FF 18 02 21 26 01 08 07 ... 00` |
| T26 | brightness 0 | `01 F3 FF 18 02 21 40 8E 00 00 00 ... 72` |
| T27 | colour `00 FF 00` | `01 F3 FF 18 02 21 40 8C 00 00 08 00 00 FF 00 ... 6D` |
| T28 | stream parser: `04` alone, then the other 30 bytes of T2 | one status packet after the second chunk |
| T29 | stream parser: `00 FF` + T12 link up + T2 | 2 bytes discarded, a link-up event, then the status packet |
| T30 | anti-vectors | a parser that requires a checksum in every reply rejects T2; a matcher without the sub-id check takes T14 for a `0x8B` reply; a codec that computes `f[27]` for `0x25` sends `...25 01 03 ... FC` where Synapse sends `00` (both probably accepted, section 3.2) |

## 11. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Whether the dongle answers with DTR low; whether line coding matters | open the port with DTR disabled and send T1; or capture a host opening the port |
| 2 | Meaning of `f[4]` = `02` | none safely: no other value is in use, and probing it is excluded (section 9) |
| 3 | Meaning of `0x22` and of the `0x27` event | a capture of Synapse's device-mode and reset paths; long passive logging of `0x27` |
| 4 | Meaning and scale of `0x24` `v[1]` and `v[2]` | log `v[2]` against battery per cent through a full discharge and a full charge |
| 5 | Whether `0x20` and `0x21` describe the dongle or the headset | compare with the label on the headset and with a second headset paired to the same dongle |
| 6 | Whether mode `03` selects the built-in colour cycle, and what modes `00` to `04` do | a capture of Synapse's effect builder; not by probing |
| 7 | Whether the earcups can be addressed separately (`f[11]` = `01`) | one colour frame with `f[11]` = `01` while both earcups are watched |
| 8 | What the effect and brightness reads `0x8D`, `0x8F` return | one read each |
| 9 | Brightness `00`, and whether brightness persists | a write and a power cycle |
| 10 | Whether earcup lighting is really off while charging, and what `0x93` / `0x94` are | watch the earcups on the charger; a capture of Synapse while charging |
| 11 | Whether frames sent while the dongle waits for the headset are dropped | timed writes with immediate read-back |
| 12 | Whether the first port access after every resume fails | resume tests with a timed first write |
| 13 | The `052D` cable port's command set | not to be probed; only a capture of Razer's own tools, if ever |
| 14 | Battery-band behaviour below 10 % and the low-battery tone | a full discharge on battery |
| 15 | Behaviour of firmware other than 1.6.0 (an owner reports 1.10.0 exists) | a second unit; firmware updates are out of scope |

The firmware-update and pairing procedures are deliberately not on this list.
