# Razer Synapse 3 and its drivers: what they do to these devices, and removing them

Part of the [Razer peripherals protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **PUBLISHED**,
**HOST**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. Times are local time of the
capturing PC (UTC-7). "The capture" is the USBPcap session of 2026-10-30, 13:39 to 14:03, 14 segments, in which Razer
Synapse 3 owned the DeathAdder V2 (`1532:0084`), the Huntsman V2 (`1532:026C`) and the Kraken V3 Pro's HyperSpeed
dongle (`1532:052C`). Bytes are hexadecimal; `cc:ii` is the command class and id of the 90-byte feature-report
protocol of [razer-hid-protocol.md](razer-hid-protocol.md), and arguments are written from the first argument byte.
The headset's serial frames are named as in [razer-kraken-v3-pro.md](razer-kraken-v3-pro.md).

This document describes what Razer's Windows software installed for these three devices on the observed PC, which
device handles each part held, how Synapse 3 re-applies its state, how a third party can take the devices over while
Synapse is installed, how the software was removed on 2026-11-03, and what the devices do on Windows' own drivers
afterwards. Wire formats are in the device documents; this one only names the commands.

Short version (each point is tagged in the sections below): Synapse 3 is a session-0 service that loads one native
"device access" DLL per product and talks to the mouse and the keyboard through their HID control collections, and to
the headset dongle through its virtual COM port, which it holds exclusively. It renders **every** lighting effect on
the PC and streams it as custom frames; it writes a device effect (spectrum cycling) and normal device mode only when
it lets a device go. At every attach it puts the mouse and keyboard in driver mode and rewrites their settings and
onboard profile. Razer's kernel drivers are filters on Windows' HID stack and never own an interface. Stopping the
services in the wrong order brought two of them back within half a minute. Razer's uninstaller left 14 of 27 driver
packages, and Windows bound the keyboard to superseded Razer packages from the driver store until they were deleted.
After a full cleanup every interface runs on an inbox Windows driver and all three devices work with third-party code.

## 1. What was installed

### 1.1 Components and versions

All **HOST** (installed-programs list, version resources and Synapse's module manifest, read on 2026-10-26 and
2026-11-02) unless tagged otherwise.

| Component | Version as recorded | Notes |
|---|---|---|
| Razer Synapse 3 | 3.10.0730.071519 (manifest 3.10.730.71519; UI executable 3.1.627.5) | one entry in the installed-programs list; Razer's unified installer removes its modules, Razer Central, the Game Manager and the Chroma SDK with it (INFERRED from the uninstall, section 7: none of them had its own entry) |
| Synapse device modules | Lily Refresh 3.9.331.30402 (DeathAdder V2), Jamie T2 V2 3.9.630.70502 (Huntsman V2), Krystal T1 3.9.630.62502 (Kraken V3 Pro dongle) | Razer's code names for the three products |
| Device access DLLs ("Razer Device Access", one per product) | 1.0.0.33 | `RzCtl_0084.dll`, `RzCtl_026C.dll`, `RzCtl_052C.dll`; also `RzCtl_0302.dll` 1.0.0.32 for the virtual "Chroma Connect" device |
| THX helper | 1.0.0.33 | `RzTHX052c.exe` ("Razer THX052c Helper"), in the same folder as the device access DLLs |
| Razer Central (account and update hub) | 7.19.1.904 | service `RzActionSvc` ("Razer Central Service") |
| Razer Game Manager Service | 2.4.160.155 | game library scanner, no device role |
| Razer Chroma SDK | runtime 3.38 (REST server 1.10.1.226, service 2.12.0.222; native client `RzChromaSDK64.dll` 3.8.0.233) | 51 per-family device modules, per-PID grid maps |
| THX Spatial Audio USB 1532-052C | 3.0.36.0 and 3.0.30.0 | two visible entries; plus THX V3 APO Presets 3.0.36.1 and 3.0.18, hidden; each of the four existed as a Windows Installer product and a bundle, eight entries in all |
| Autostart | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, value `Synapse3` (`Razer Synapse 3.exe /StartMinimized`) | no scheduled task |
| Disk | Razer folders under both Program Files folders (1.5 GB), ProgramData (970 MB) and the user's local application data (152 MB); `Razer Chroma SDK` under both Program Files folders; THX folders under ProgramData and Program Files; a `Razer` folder in the Windows Installer cache | |
| Registry | `HKLM\SOFTWARE\Razer`, `HKLM\SOFTWARE\WOW6432Node\Razer`, `HKLM\SOFTWARE\WOW6432Node\Razer Chroma SDK`, `HKLM\SOFTWARE\THX`, `HKCU\SOFTWARE\THX`, `HKCU\SOFTWARE\Razer` (account ids only) | |

Razer states that Synapse 3 reached end of support on 2026-02-03 and recommends Synapse 4 (**PUBLISHED**, Razer
support answers 20786 and 14628). Synapse 4 was never installed on the observed PC; web reports name its processes
`RazerAppEngine.exe` and `RzEngineMon.exe` and a service "Razer Elevation Service" (**PUBLISHED**, user reports; not
checked). Nothing in this document is known to apply to Synapse 4.

### 1.2 Services and processes, and what each held

| Service or process | What it does to the devices |
|---|---|
| `Razer Synapse Service` (LocalSystem, session 0) | **The device owner.** Loads one feature module per Synapse feature and the device access DLL per product; every device line in Synapse's logs carries this process's id (**HOST**). Holds the HID control collections of the mouse (interface 0) and the keyboard (interface 3) while a device is attached; the Win32 path it uses for the 90-byte reports (`HidD_SetFeature` on the collection, or `DeviceIoControl` on the Razer control device of section 1.4) is **UNKNOWN**: the DLLs import both (**METADATA**). Holds the dongle's **COM port (interface 4) exclusively** (INFERRED, strong, from its own logs, section 3.5) |
| `Razer Synapse Service Process` (per user) | input hooks, Bluetooth pairing, game detection (**METADATA**, the DLL names in its folder). No device handle (INFERRED: no device line in any log comes from it) |
| `Razer Synapse 3` (the window) | user interface only (INFERRED, same reason) |
| `Razer Chroma SDK Service` (`RzSDKService.exe`) | parent process of `RzDeviceManager`, `RzWDLDeviceManager`, `RzChromaConnectManager`, `RzChromaConnectServer`, `RzAppManager`, `RzBTLEManager`, `RzIoTDeviceManager`, `RzSmartlightingDeviceManager` and `RzDiagnostic` (**HOST**, parent process ids). Its device modules for the mouse and the keyboard import `HidD_SetFeature` / `HidD_GetFeature` (**METADATA**) and log "Lighting enabled" for those two devices (**HOST**), so it can write them itself (INFERRED). Its module for the dongle hands frames to Synapse through shared memory (`RzDeviceAccess_052C_0000`, **HOST**, Synapse's log) and imports no serial-port API (**METADATA**). `RzDeviceManager` enumerates the dongle's COM port at every boot ("Razer COM Port Device found", **HOST**) |
| `Razer Chroma SDK Server` (`RzSDKServer.exe`), `Razer Chroma Stream Server` | the Chroma REST API (section 6) and Chroma streaming; no device handle seen |
| `RzActionSvc` / `Razer Central`, `Razer Game Manager Service` | account, update registration and game scanning; no device role (**HOST**, logs). They stayed running during every session of third-party code in section 5 without touching the devices (OBSERVED 2026-10-31 and 2026-11-01) |
| `RzTHX052c.exe` | configures the THX audio processing objects on the headset's Windows audio endpoints through Windows Core Audio, reached from the device access DLL over IPC (**METADATA**: `CTHXV3Device::Get/Set*` strings, ZeroMQ 4.3.4 and `MMDeviceEnumerator` embedded, no HID or serial API imported). Sidetone is not a THX function (`Get/SetSidetone ... Not supported`, **METADATA**) |
| Windows audio engine | runs the THX APOs installed by the dongle's audio packages (section 1.3) |

The services were all set to start automatically (the Stream Server delayed) and were running on 2026-10-26 (**HOST**).

Synapse and the Chroma SDK serialise their transactions with named mutexes whose format strings differ:
`RzProtocolMutex_%04X_%04X` in the device access DLLs, `RzProtocolMutex_%04X_%X` in the SDK modules (**METADATA**).
Filled with VID and PID they would give two different names for the same device (INFERRED; the runtime names are
**UNKNOWN**). OpenRGB uses `Global\RazerLinkReadWriteGuardMutex` (**PUBLISHED**, OpenRGB
`Controllers/RazerController/RazerDeviceGuard.cpp`), a name no Razer binary contains (**METADATA**), so OpenRGB and
Synapse do not coordinate; an OpenRGB issue reports Synapse overriding OpenRGB's colours while both run
(**PUBLISHED**, OpenRGB issue #536).

### 1.3 Kernel drivers and driver packages

Four kernel services, all demand-start and loaded while the devices were present (**HOST**): `RzDev_0084`,
`RzDev_026c`, `RzDev_052c` (one filter per product; `.sys` versions 1.0.0.68, 1.0.0.73, 1.0.0.71) and `RzCommon`
("Razer Control Service", 1.0.0.77).

Every Razer package is a filter or class package layered on Windows' own HID stack (`HidUsb` with `mouhid`, `kbdhid`
or `hidserv`); none replaces the function driver of a HID interface (**METADATA**, the INFs). The packages, by
original INF name (**METADATA**: INF text; **HOST**: which node used which):

| Device | INF | Binds to | What it adds | DriverVer |
|---|---|---|---|---|
| DeathAdder V2 | `rzdevu_0084_mou.inf` | `HID\VID_1532&PID_0084&MI_00` (mouse collection, the control collection) | upper filter `RzDev_0084`, `ControlDevice` = 1 | 08/18/2020 |
| | `rzdevu_0084_kbd.inf` | `HID\..._0084&MI_01&Col01` (keyboard collection) | upper filter `RzDev_0084` | 08/18/2020 |
| | `rzdevu_0084_dkm.inf` | `USB\VID_1532&PID_0084&MI_01` | lower filter `RzDev_0084` under `HidUsb`; `DkmKeyDevice`, `DkmMouseDevice`, `Virtual_Con` | 08/18/2020 |
| | `rzdevu_0084_vcon.inf` | `RZVIRTUAL\VID_1532&PID_0084&MI_00&Col03` | function driver `RzDev_0084` under Microsoft's `mshidkmdf` (a virtual consumer-control device) | 08/18/2020 |
| | `rz0084dev.inf` | `USB\VID_1532&PID_0084&MI_00` | no filter; registers the co-installer `RazerS3CoinstallerEx.dll` and copies Synapse's setup wizard `RzS3WizardPkgS3.exe` (section 7.6) | 01/02/2017 |
| Huntsman V2 | `rzdevu_026c_hid.inf` | `HID\VID_1532&PID_026C&MI_03` (consumer collection, the control collection) | upper filter `RzDev_026c`, `ControlDevice` = 1 | 08/03/2022 |
| | `rzdevu_026c_mou.inf`, `rzdevu_026c_kbd.inf` | `HID\..._026C&MI_02`, `HID\..._026C&MI_01&Col01` | upper filter `RzDev_026c` (`ControlDevice` = 0 on the mouse collection) | 08/03/2022 |
| | `rzdevu_026c_dkm.inf` | `USB\VID_1532&PID_026C&MI_01` | lower filter `RzDev_026c`; `Virtual_Con_026c` | 08/03/2022 |
| | `rzdevu_026c_vcon.inf` | `RZVIRTUAL\VID_1532&PID_026C&MI_00&Col03` | as for the mouse (consumer and system-control children) | 08/03/2022 |
| | `rz026cdev.inf` | `USB\VID_1532&PID_026C&MI_02` | the co-installer, as for the mouse | 01/02/2017 |
| | superseded copies of the five `rzdevu_026c_*` packages | none while the current ones were present | (section 7.4) | 04/30/2021 |
| Kraken V3 Pro dongle | `rzdevu_052c_dkm.inf` | `USB\VID_1532&PID_052C&MI_03` (media-key interface) | lower filter `RzDev_052c`, `ControlDevice` = 1 | 03/23/2021 |
| | `razer052cdriver.inf` | `USB\VID_1532&PID_052C&MI_00` (USB Audio) | the audio function package, class MEDIA, on top of Windows' `usbaudio`; THX V2 render and capture APOs | 01/12/2021 (19.45.27.818) |
| | `thxusbapo.inf` (provider THX Ltd.) | the same audio interface | extension; adds the THX APO component `thxrtapo.inf` and three software components `thxrtscu.inf`, `thxrtsvc.inf`, `thxrthsa.inf` (three software-device nodes); sets the endpoint effects and the name "Razer Kraken V3 Pro" | 11/23/2023 (3.0.36.0) |
| | none | `USB\VID_1532&PID_052C&MI_04` (the COM port) | Windows' `usbser.inf`, no Razer filter | |
| All | `rzcommonu.inf` (four copies in the store) | `RAZER\DeviceContol` (sic) | function driver `RzCommon` for the `RZCONTROL` devices, new device class "Razer Device" `{1750F915-5639-497C-966C-3A65ACECFCB6}`; **`PnpLockdown=1`** (section 7.5) | 01/12/2024 |

In all, 27 Razer and THX packages were in the driver store on 2026-11-02 (**HOST**). The filter INFs set a device
security descriptor that grants read and write to Administrators and Everyone (**METADATA**,
`D:P(A;;GA;;;SY)(A;;GRGWGX;;;BA)(A;;GRGWGX;;;WD)(A;;GRGWGX;;;RC)`), so the filters never made the control
collections inaccessible to other programs; third-party code opened them through the filters (OBSERVED 2026-10-31).
The dongle's USB-C cable PID `1532:052D` (the headset itself on its charging cable, a CDC-only device; **PUBLISHED**,
openrazer issue #1950) had no Razer package of any kind (**HOST**).

### 1.4 The extra devices the filters create

- **`RZCONTROL\VID_1532&PID_<pid>&MI_00`** "Razer Control Device", class "Razer Device", function driver
  `RzCommon`, one per product (**HOST**). Its parent is the node that carries the filter with `ControlDevice` = 1:
  the mouse's HID interface 0, the keyboard's HID interface 3 and the dongle's USB interface 3 (the `MI_00` in its
  name is fixed, not the real interface number). INFERRED: a private, non-HID side door from Razer's user-mode code
  to the filter (strings `CRzClient::Notify` in `RzCommon.sys`, `eDriverNotify_InputRedirect / InputHook /
  InputNotify / DeviceEvent` in the device access DLLs, **METADATA**).
- **`RZVIRTUAL\VID_1532&PID_<pid>&MI_00&Col03`**, a virtual HID consumer-control device (mouse and keyboard only)
  under the USB interface 1 that carries the lower filter (**HOST**). INFERRED: the path through which the filter
  injects remapped keys (media keys, power functions) while the device is in driver mode.
- The lower ("DKM") filters sit under `HidUsb` on interface 1 of the mouse and the keyboard and on interface 3 of the
  dongle. Synapse's logs name the DeathAdder's DPI buttons `DKM 0x20` / `0x21` and its profile button `DKM 0x50`, the
  Huntsman's Fn key `0x01` and media keys `0x52` to `0x55` (**HOST**); INFERRED: in driver mode those inputs arrive on
  interface 1 as codes for Synapse to act on, which is why they need Synapse while driver mode is on.

Neither LampArray collection (mouse interface 3, keyboard interface 4 collection 1) nor the dongle's COM port carried
a Razer filter (**HOST**).

## 2. The device stacks while Synapse was installed

| Device | Interface | Driver stack (**HOST**, 2026-10-26) | Razer role |
|---|---|---|---|
| DeathAdder V2 | HID 0, mouse collection | `mouhid`, upper filter `RzDev_0084` | control collection; parent of `RZCONTROL` |
| | USB 1 > HID Col01-Col05 | `HidUsb` with lower filter `RzDev_0084`; Col01 also upper filter | DKM input; parent of `RZVIRTUAL` |
| | HID 2, keyboard | `kbdhid` | none |
| | HID 3, LampArray | Microsoft only | Windows Dynamic Lighting |
| Huntsman V2 | HID 0, boot keyboard | `kbdhid` | none |
| | USB 1 > HID Col01-Col05 | `HidUsb` with lower filter `RzDev_026c`; Col01 also upper filter | DKM input; parent of `RZVIRTUAL` |
| | HID 2, mouse collection | `mouhid`, upper filter `RzDev_026c` | input filter only |
| | HID 3, consumer collection | raw HID, upper filter `RzDev_026c` | control collection; parent of `RZCONTROL` |
| | HID 4 Col01, LampArray | Microsoft only | Windows Dynamic Lighting |
| Kraken V3 Pro dongle | USB 0, audio | `usbaudio` with `razer052cdriver.inf` and the THX extension | speakers and microphone endpoints with THX processing |
| | USB 3 > HID | `HidUsb` with lower filter `RzDev_052c`, `hidserv` | media keys and Synapse's input hook only: the collection has no output and no feature report (**PUBLISHED**, openrazer issue #1950, report descriptor of this PID) |
| | USB 4, CDC | `usbser` (a COM port) | the headset's settings channel ([razer-kraken-v3-pro.md](razer-kraken-v3-pro.md)) |

## 3. How Synapse drives the devices and re-applies its state

### 3.1 Lighting is always rendered on the PC

**CAPTURED (segments 1-14).** While Synapse ran, it streamed `0F:03` custom-frame rows without pause: to the mouse one
row of 2 LEDs (size 11) about 25 times a second, to the keyboard 6 rows of 22 cells (size 71 each) about 24 frames a
second. Changing the effect in Synapse (Spectrum Cycling, Wheel, Static) changed only the frame contents; Synapse
never sent a device effect (`0F:02`) for any effect it showed. A "Static" colour was the same frame repeated about 5
times a second. To the headset dongle it streamed colour writes (`0x40` / `0x8C`, mode 0x08): one every 5 s for a
static colour, about 21 a second for breathing, a hue sweep for spectrum, 286 writes in one two-minute segment
(segment 11), and never a headset effect command. So every lighting effect Synapse 3 offers, including the ones the
devices could run themselves, is a host animation, and a device shows Synapse's effect only while Synapse runs.

A device effect is written only when Synapse lets a device go (section 3.3) and, once, a custom-frame apply
(`0F:02` effect 0x08) when a device arrives (section 3.2). Details per device: [razer-hid-protocol.md](razer-hid-protocol.md),
[razer-deathadder-v2.md](razer-deathadder-v2.md), [razer-huntsman-v2.md](razer-huntsman-v2.md),
[razer-kraken-v3-pro.md](razer-kraken-v3-pro.md).

### 3.2 Attach and start

**CAPTURED, mouse replug (segment 5, 13:45:42 to 13:45:45).** In order:

1. Windows enumerated the mouse and its HID class driver read the LampArray attributes on interface 3 and wrote
   `LampArrayControl` (report 0x06) with autonomous mode 1, then 0 about 25 ms later; no lamp update followed. At
   the same moment Windows sent the keyboard's LampArray collection a 9-byte report 0x05 (`01 00 00 6d 00 00 00 00`;
   INFERRED a range update of lamps 0-109 to black). So Windows Dynamic Lighting claims both lamp arrays at every
   arrival, and Synapse's frames are still what the LEDs show.
2. Requests with transaction ids 0x0C and 0x08, before Synapse's first request: `00:86` (answered busy, status 0x01,
   for 0.4 s), `00:85`, `00:82`. INFERRED: issued by the Razer filter driver itself, since Synapse uses 0x1F for every
   request.
3. Synapse, 1.2 s later, transaction id 0x1F: profile and storage queries (`00:87`, `05:8A`, `05:80`, `05:81`,
   `06:80`, `06:8E`), `00:84` (device mode, read 0x00), **`00:04` 0x03 (driver mode)**, **`0F:10` 0x01** (INFERRED:
   Synapse's "Force Chroma Control", section 4.2), `0F:04` brightness with storage 0x00 then 0x01, **one `0F:02`
   custom-frame apply** (effect 0x08), lift-off settings (`0B:03`, `0B:0B`), polling (`00:05`), DPI (`04:05`) and the
   DPI stage list (`04:06`) with storage 0x01, 20 button mappings (`02:0C`, 10 buttons on 2 layers), then the frame
   stream.

**CAPTURED, Synapse start (segment 14, 14:01:34 to 14:01:38).** The same for both devices from `00:84` on, without
the enumeration and filter-driver reads, and **without** the `0F:02` apply: both answered `00:84` with 0x00, then got
`00:04` 0x03, `0F:10` 0x01, `0F:04` `00 00 ff` and `01 00 ff`, and their settings; frames resumed with no apply.
The keyboard also got `03:00` LED 0x08 off (game-mode indicator), `00:40` `01 08` (polling 1000 Hz, storage 0x01) and
`02:14` `01 00`, which it answered with status 0x05 (not supported). The dongle got two status reads (battery), a
power-saving read and a power-saving re-write `01 84 03` (on, 900 s), and the colour stream resumed.

**Every change made in Synapse's window was sent twice** (two SET/GET pairs 20-30 ms apart) and read back; the
writes of the attach and start sequences went out once (**CAPTURED**). Synapse used
storage 0x01 for brightness, the keyboard's polling, the DPI stage list, the final DPI and the button mappings at
every attach (**CAPTURED**); its logs also show it rewriting onboard profile 1 (24 `Storage_ProfileData` items per
round, two rounds at a service start, one per resume) (**HOST**, logs of 2026-10-23 to 2026-10-26). Whether storage
0x01 writes flash is not shown by the wire; see [razer-hid-protocol.md](razer-hid-protocol.md). Any setting another
program stores on the mouse or the keyboard is overwritten at Synapse's next attach (INFERRED from the above).

### 3.3 Exit

**CAPTURED (segment 13, Synapse exited at 14:00:26).** Within 170 ms:

| Device | What Synapse sent |
|---|---|
| Huntsman V2 | `03:00` LED 0x08 off and LED 0x18 off; `0F:80` (LED regions, reply `05 19 03 06 16`); `0F:82` storage 0x01 LED 0x05 (stored effect, reply spectrum); **`0F:02` `01 00 03 00 00 00`** (spectrum, storage 0x01); `00:84` (reply 0x03); **`00:04` 0x00** (normal mode) |
| DeathAdder V2 | `0F:80` (reply `01 19 03 01 01 04 19 03 01 01`); `0F:82` (reply spectrum, a[4] = 0x28); **`0F:02` `01 00 03 00 28 00`** (spectrum, speed 0x28 = 40, storage 0x01); `00:84` (reply 0x03); **`00:04` 0x00** |
| Kraken V3 Pro dongle | **nothing**: the last frame Synapse sent was a colour write at 14:00:23.867; the colour stream simply stopped |

So after a clean exit the mouse and the keyboard run their own stored spectrum cycle in normal mode. Synapse's logs
name the same release `Lighting_Effect ... LEDEFFECT_SPECTRUMCYCLING`, speed 40, followed by `DeviceConfig_DeviceMode
0x0 (Default)` and "Switch OK to Default Mode" (**HOST**). For the dongle the logs show a release that sets spectrum
cycling and the default device mode, each failing because the headset was off (**HOST**, 2026-10-23 to 2026-10-25);
with the headset on, the capture shows no frame for either (CAPTURED), so that part of the release does not reach the
serial channel (INFERRED). What the headset shows after Synapse's stream stops, before its next power cycle, is
**UNKNOWN**; after a power cycle it runs its own colour cycle (OBSERVED 2026-11-02).

Which process ended at 14:00:26 (the window, the per-user process or the service) was not recorded; the devices were
released either way (CAPTURED). Synapse was started again at 14:01:34.

### 3.4 Sleep, log-off, resume and display power

From Synapse's own logs, 2026-10-23 to 2026-10-26 (**HOST**; no sleep or resume was captured):

- The releases on record (2026-10-23 20:23, 2026-10-24 20:15, 2026-10-25 19:27) each fall within a second of the
  service logging the end of the user session ("User has log off", "Stopping RzDetectMgr"), before sleep or
  shutdown. The trigger is the session ending, not a service stop (INFERRED from the timing).
- After resume the same service process attached the devices again the next morning (2026-10-24 03:44, 2026-10-26
  04:22, after "PowerResume" and a session logon) with the full attach: driver mode, Force Chroma Control, polling,
  DPI, brightness, the custom-frame effect and the onboard-profile rewrite. After a cold boot a new service process
  attached them (2026-10-25 03:23). On two of the four mornings the attach ran twice, 18 s and 61 s apart.
- While attached, the lighting power-state module wrote brightness 0 and 100 to the mouse 45 times on 2026-10-25,
  the last one (brightness 0) one second before the release. INFERRED: Synapse dims the LEDs on its own power and
  display events. One consequence was OBSERVED on 2026-10-31: after a Synapse release the mouse's two LEDs read
  brightness 0, so a program that takes over sees dark LEDs until it writes a brightness. On 2026-11-01, after another
  release, they read 255.

### 3.5 The dongle's COM port and Synapse's start-up race

Synapse's log on two cold boots (2026-10-23 03:42:09 and 2026-10-25 03:23:16) shows two of its feature modules
opening the dongle's COM port on two threads of the same service process at once: the `DeviceStatus` thread got the
port and the `PowerSwitch` thread's open failed with `GetLastError: 0x5` (access denied), followed by "Failed to get
device control (Vid=0x1532, Pid=0x052c)" (**HOST**). That module then stayed without the device for the whole session,
so the headset's connect and battery events had no listener (**HOST**). On re-adds after resume, done from one
thread, no failure was logged (**HOST**). A Windows COM port can only be opened exclusively (**PUBLISHED**, Microsoft,
`CreateFileW` documentation, communications resources). INFERRED (strong; no handle listing was taken):
`Razer Synapse Service` holds the port from attach to release, and **whoever opens it first wins; Synapse does not
retry until the device is re-added**. A program that holds the port while Synapse starts leaves Synapse's headset
features broken for that session.

## 4. Synapse's own files as evidence

### 4.1 Device definitions

Synapse keeps a binary capability file per product (`DD_<code name>.rzdd`), whose tags are the djb2 hashes of ASCII
names such as `RZDD`, `MODUSB` and `MODMSDPI` (**METADATA**, decoded from the files; every tag matched a name found in
Synapse's binaries). They declare what the user interface offers, not what is sent. Where the wire confirms them:

| Product | Declared (**METADATA**) | On the wire |
|---|---|---|
| DeathAdder V2 | DPI 100 to 20000, step 50, 5 default stages (400, 800, 1600, 2400, 3200); polling 125, 500, 1000 Hz (no 250); two LEDs (scroll wheel, logo) | polling through legacy `00:05` (0x02 = 500 Hz, 0x01 = 1000 Hz); the stored stage list read `400 / 800 / 1600 / 2400 / 20000` (CAPTURED `04:06`, OBSERVED 2026-11-01 with `04:86`); `0F:80` lists LED regions 0x01 and 0x04 (CAPTURED) |
| Huntsman V2 | polling 125 to 8000 Hz (user-interface file); LED groups backlight, game mode, macro, volume; 6 x 22 LED layout | polling through `00:40` with a[0] = 0x01 (CAPTURED); game-mode LED 0x08 and LED 0x18 switched off at exit (CAPTURED); `0F:80` reply `05 19 03 06 16` = region 0x05, 6 rows, 22 columns (CAPTURED) |
| Kraken V3 Pro dongle | sleep timeout 900 to 3600 s; haptics levels 1 to 3; sidetone 0 to 100; one LED (logo); 10-band EQ -12 to +12 dB (applied on the PC by THX, section 1.2) | power saving `01 84 03` = on, 0x0384 = 900 s (CAPTURED); haptics read back as on, level 2 of 3 ("Medium" in Synapse) (CAPTURED, segment 10) |

The default files set `ChargingForceChromaControl` `Enabled` = 1 for the mouse and the keyboard (**METADATA**), which
matches the `0F:10` 0x01 sent at every start (CAPTURED; the link is INFERRED). No device folder contains a firmware
updater; firmware updates go through Razer Central (**METADATA**).

### 4.2 Log vocabulary and the wire

Synapse's device access DLLs log every parameter they get or set (**HOST**). Matched against the capture:

| Log line | Wire |
|---|---|
| `DeviceConfig_DeviceMode DeviceMode:0x1 (Synapse)`, "Switch OK to Driver Mode" | `00:04` 0x03 (CAPTURED at every attach and start) |
| `DeviceConfig_DeviceMode DeviceMode:0x0 (Default)`, "Switch OK to Default Mode" | `00:04` 0x00 (CAPTURED at exit) |
| `DeviceConfig_ForceChromaControl Enabled : 1`, "Sending Force Chroma Control - Enabled = 1 to firmware" after "Initial 'WDLState' not found" | `0F:10` 0x01 (CAPTURED right after the mode write; INFERRED to be this parameter: neither openrazer nor OpenRGB sends `0F:10`) |
| `Lighting_Effect LEDEFFECT_CUSTOMEFFECT` | `0F:02` effect 0x08 (CAPTURED at the replug) |
| `Lighting_Effect LEDEFFECT_SPECTRUMCYCLING`, speed 40 | `0F:02` effect 0x03, a[4] = 0x28 (CAPTURED at exit) |
| `CProtocol25::SetFeature : Failed: 80004001 (Cmd: 214, Status: 05)` on the keyboard, each time the onboard-profile module writes `SCROLLWHEELMODE` | `02:14` `01 00` answered with status 0x05 (CAPTURED 14:01:37.664); 0x02/0x14 is openrazer's set-scroll-mode command (**PUBLISHED**, openrazer `driver/razerchromacommon.c`), a mouse command the keyboard rejects |
| `PollingRate 0x3E8` | `00:05` 0x01 (mouse), `00:40` `01 08` (keyboard) |
| dongle `InputHook_DeviceEvent 0x00323100`, `0x00283100`, `0x001E3100` | battery events of 50, 40 and 30 %: the status reply's battery byte took exactly these values 0x32, 0x28, 0x1E in the capture (INFERRED match); `0x03030900` / `0x03020900` = headset connected / disconnected (INFERRED) |
| `FW: 1.0.3.0` (mouse), `FW: 1.0.5.0` (keyboard) | the firmware-version read returns `01 00 00 00` on both (OBSERVED 2026-10-31); how Synapse's four-part versions map onto these bytes is **UNKNOWN** |

Synapse-internal parameter ids in the feature logs (`MwParam 0x04020001` and similar) are callback ids, not wire
values (INFERRED).

## 5. Taking the devices over while Synapse is installed

### 5.1 The services come back if the order is wrong

On 2026-10-31 (OBSERVED, third-party code polling the device state every 2 s) the four device-related services
(`Razer Chroma SDK Service`, `Razer Chroma SDK Server`, `Razer Chroma Stream Server`, `Razer Synapse Service`) were
stopped from an elevated prompt and the Synapse window was closed; Razer Central and the Game Manager were left
running. The services were gone at 20:57:06 and both HID devices answered at 20:57:18. Then **`RzSDKService` and
`Razer Synapse Service` ran again from 20:57:29 to about 20:57:55** (the REST server's URL registration reappeared at
20:57:29, the Synapse service at 20:57:43) and stopped again by themselves. The cause is **UNKNOWN**; INFERRED
candidates are the still-open window or the per-user `Razer Synapse Service Process` restarting what it found
stopped.

On 2026-11-01 (OBSERVED, three sessions) the window was closed **first** and the services stopped after; nothing came
back during a 60 s watch or the whole session. In the third session the order was: start type of the four services
set to Manual, the Synapse window and the per-user `Razer Synapse Service Process` closed, then the four services
stopped in the order above; a lingering `Razer Synapse Service` process exited within 5 s and nothing came back.

### 5.2 A procedure that worked

Read-only checks between the steps; everything below was done on the observed PC on 2026-11-01 and 2026-11-02
(OBSERVED):

1. Set the start type of `Razer Chroma SDK Service`, `Razer Chroma SDK Server`, `Razer Chroma Stream Server` and
   `Razer Synapse Service` to Manual (elevated), so that a restart does not bring them back. Record the old start
   types (here Automatic, the Stream Server Automatic delayed).
2. Exit Synapse from its tray icon and end the per-user `Razer Synapse Service Process`.
3. Stop the four services through the Service Control Manager, in that order. Do not kill the service process: the
   exit writes of section 3.3 (normal device mode, stored effect) are part of Synapse's graceful release, so a killed
   service is expected to leave the mouse and the keyboard in driver mode (INFERRED; a kill was never tested).
4. Watch the process list for a minute. Razer Central and the Game Manager may keep running.
5. Read the device mode (`00:84`) of the mouse and the keyboard. After every clean stop it was 0x00 (OBSERVED);
   if it reads 0x03, write `00:04` 0x00, or the DPI buttons, the Fn layer and the media keys wait for software that
   is gone (INFERRED, section 1.4).
6. Open the dongle's COM port only after `Razer Synapse Service` has stopped. On 2026-11-02 the first open after a
   start of the third-party service failed with access denied and an open 0.4 s later worked, while the Chroma SDK
   services were still running (OBSERVED); INFERRED: the Chroma SDK's device manager had the port open for a moment
   while enumerating it. Retry an access-denied open a few times.

**Hand-back** (OBSERVED 2026-10-31 and 2026-11-01): restore the recorded start types, start the services in reverse
order, then start Synapse 3. Synapse re-applies its own state at attach (section 3.2), so the hand-back needs no
device writes. A third party should also stop any custom-frame stream first; the devices then keep the last frame
until Synapse's attach (INFERRED).

### 5.3 Coexistence without a takeover

While Synapse runs, any colour another program writes over raw HID is replaced within one frame (40 ms) by Synapse's
stream, and settings are rewritten at Synapse's next attach (INFERRED from sections 3.1 and 3.2). Interleaving with
Synapse's request/response pairs is possible because the mutexes are private (section 1.2). Another vendor's advice
is to set every Razer device's lighting to static black or off in Synapse and close Synapse rather than uninstall it
(**PUBLISHED**, SignalRGB troubleshooting page for Razer). The documented ways to light Razer devices while Synapse
runs are in section 6.

## 6. Documented alternatives that go through Razer's or Windows' software

- **Chroma SDK, REST and native** (**PUBLISHED**, Razer's REST documentation and its `HTML5ChromaSDK` sample on
  GitHub, `RazerOfficial/HTML5ChromaSDK`): an application registers with `POST http://localhost:54235/razer/chromasdk`
  (title, author, supported device types), keeps the session alive with `PUT {uri}/heartbeat` (the server drops a
  session after 15 s without commands) and sets effects per device type (`keyboard` 6 x 22 grid, `mouse` 9 x 7
  `CHROMA_CUSTOM2` grid, `chromalink` 5 cells, and others). It controls lighting only: no DPI, polling, device mode or
  firmware reads. On the observed PC the mouse mapped to the SDK's `mouse` type, the keyboard to `keyboard` and the
  dongle to `chromalink` (**METADATA**, the SDK's device list). Razer says the Chroma SDK is no longer actively
  maintained and points game developers to its WYVRN SDK (**PUBLISHED**, Razer WYVRN documentation). It needs the
  Chroma SDK services installed and running.
- **Chroma Broadcast** publishes Synapse's current colours (five colours) to registered partner programs; a client
  must present a Razer-issued application id that is checked against a local allowlist (**HOST**: the broadcast log
  shows a partner verifying "AppId 7" with `broadcast.dat v4.0.1`; **PUBLISHED**, OpenChromaConnect,
  `OpenChromaConnect/ChromaBroadcastAPI`, for the callback layout). On the observed PC the listener was MSI's Mystic
  Light service, so Razer colours could follow onto the motherboard's lighting (**HOST**).
- **Windows Dynamic Lighting.** Both the mouse and the keyboard expose a HID LampArray collection (usage page 0x59)
  that no Razer driver filters (**HOST**). Windows gives a foreground app control first and background apps in the
  user's order; an app that controls lamps in the background needs package identity (**PUBLISHED**, Microsoft Learn,
  "Dynamic lighting"). While Synapse ran, its frames won over Windows (CAPTURED, section 3.2, step 1); Synapse's Chroma
  SDK "WDL Device Manager" logged start-ups but never a device (**HOST**). 2023 user reports on Synapse 3 describe a
  Huntsman V2 turning solid white until Dynamic Lighting was switched off (**PUBLISHED**, Razer Insider forum).
  LampArray details: [razer-hid-protocol.md](razer-hid-protocol.md).

## 7. Removing Synapse, Razer Central, the Chroma SDK and THX (2026-11-03)

What was done on the observed PC, with times (**HOST** unless tagged: read-only checks after each step, run by a
script that lists installed programs, services, processes, kernel drivers, every `VID_1532` device node with its INF
and filters, the driver store and the audio endpoints).

### 7.1 Before (04:43 to 04:45)

- Export the registry trees of section 1.1 and copy Synapse's data folders (88 MB, the profiles and logs); record or
  export any remaps, macros and profiles from Synapse first, since they live only there (section 8.3).
- Export every Razer and THX driver package with `pnputil /export-driver <oemNN.inf> <folder>`. The command failed for
  every package (exit 2) when the target folder did not exist; with an existing folder all 27 exported, and the export
  needed no elevation (OBSERVED 2026-11-03).
- Close Synapse (tray, Exit).

### 7.2 Uninstall (04:52)

1. Settings > Apps > Installed apps > **Razer Synapse** > Uninstall; in Razer's installer, remove everything it
   lists. Razer Central, the Game Manager and the Chroma SDK went with it.
2. THX Spatial Audio has its own entries (two visible, six hidden). On the observed PC all eight were still installed
   at the next check and were removed in the cleanup (section 7.4).
3. **Restart**, not Shut down: with Windows' Fast Startup on, Shut down hibernates the kernel and loaded drivers stay
   loaded.

### 7.3 What the uninstallers left (check at 04:53: 18 pass, 26 warnings, 29 failures)

| Leftover | Effect |
|---|---|
| No Razer program, service, process or autostart entry; the Chroma SDK folders gone | as intended |
| All eight THX entries (four Windows Installer products, four bundles) | THX still installed |
| Folders: Razer under both Program Files folders and ProgramData (104 files), THX under ProgramData (1071 files), Razer in the Windows Installer cache (295 files) | leftovers |
| **Keyboard bound to the superseded 2021 Razer packages** (`rzdevu_026c_hid`, `_mou`, `_kbd`, `_dkm`, `_vcon`) and an older copy of `rzcommonu.inf` | Razer's uninstaller had deleted the current packages, and Windows picked the next-best match from the driver store, so `RzDev_026c` and `RzCommon` were **still loaded** and the keyboard still had a `RZCONTROL` and a `RZVIRTUAL` node |
| Mouse | on Windows' `msmouse.inf`, no filter |
| The co-installer packages `rz0084dev.inf` and `rz026cdev.inf`, still bound to the mouse's USB interface 0 and the keyboard's USB interface 2, and Synapse's setup wizard `RzS3WizardPkg` on disk | the mechanism that installs Synapse when a device is plugged into a new port (section 7.6) |
| The dongle's audio still on `razer052cdriver.inf`, the THX APO component and the three THX software-device nodes | THX still processing the headset's audio |
| `RzDev_0084`, `RzDev_052c` registered but not loaded; all four `.sys` files on disk | |

Razer's uninstaller had deleted 13 of the 27 packages: the current filter packages of all three products and three
of the four copies of `rzcommonu.inf`. Both HID devices were in normal mode (`00:84` = 0x00) right after the uninstall
(OBSERVED 04:53).

### 7.4 Cleanup, phase 1 (04:54, elevated)

1. Save the registry key of each Razer kernel service (for a rollback).
2. Uninstall the THX bundles left, quietly (all four exit 0).
3. Delete the Razer and THX packages with `pnputil /delete-driver <oemNN.inf> /uninstall`, in this order: the THX
   components and extension, the dongle's audio package (**the headset's sound drops for a few seconds and returns on
   Windows' USB audio driver**), the virtual-device packages, the upper filters, the lower filters, the co-installers,
   the control-device class package. 14 packages; two answered exit 3010 (restart needed). The mouse and the keyboard
   stop for a second or two each while they move to Windows' own drivers.
4. `pnputil /scan-devices`, so that every node re-matches an inbox INF.
5. Remove the non-present Razer device nodes left from earlier USB ports with `pnputil /remove-device` (59 on the
   observed PC, including old `RZCONTROL`, `RZVIRTUAL` and `1532:052D` nodes). Their registry entries still name the
   Razer filters; a device plugged into such a port after the filter services are gone would fail to start (INFERRED
   from how filter references work).
6. Delete the kernel services with `sc delete RzDev_0084` (and `RzDev_026c`, `RzDev_052c`, `RzCommon`) once no node
   names them. `RzDev_026c` was still loaded and went at the restart.
7. Remove the folders. Restart (04:58).

Check at 04:59: 33 pass, 0 failures; every device node on an inbox INF.

### 7.5 Cleanup, phase 2 (05:05 and 05:15, elevated)

- `RzDev_0084.sys`, `RzDev_026c.sys` and `RzDev_052c.sys` were deleted.
- **`RzCommon.sys` could not be deleted: access denied.** Its package declares `PnpLockdown=1` (**METADATA**), and
  Windows keeps such driver files owned by TrustedInstaller. With no service and nothing loading it the file was
  inert, but it went only after taking ownership first (`takeown /f <file> /a`, then
  `icacls <file> /grant *S-1-5-32-544:F` for the Administrators group), then deleting it (05:15).
- Registry trees removed: `HKLM\SOFTWARE\THX`, `HKCU\SOFTWARE\Razer`, `HKCU\SOFTWARE\THX` (the `HKLM` Razer trees
  were already gone).

### 7.6 Keep Synapse from coming back

- The co-installer packages are the path back: bound to the mouse's USB interface 0 and the keyboard's USB interface
  2, they run `RazerS3CoinstallerEx.dll`, which copies Synapse's setup wizard (**METADATA**, the INFs). They match
  those interfaces better than Windows' `input.inf`, so a device plugged into a new port picks them again while they
  are in the driver store (INFERRED from INF ranking). Delete them (section 7.4).
- Control Panel > System > Advanced system settings > Hardware > Device Installation Settings > **No**
  (`PreventDeviceMetadataFromNetwork` = 1) stops Windows fetching manufacturers' apps for new devices; set between 05:05 and the cold boot at 05:11.
  Whether it also stops Windows Update from offering the co-installer packages again is **UNKNOWN**; check the driver
  store after the next Windows Update and after plugging a device into another port.

### 7.7 Spatial audio and audio enhancements

- **Changing the dongle's audio driver reset the speakers' spatial audio to Off** (check at 04:59: the Windows API
  reported an all-zero format GUID). The spatial format (here Dolby Atmos for Headphones) had to be chosen again in
  Sound settings. The speaker and microphone endpoints themselves were present after every step.
- After THX was gone, Windows' own "Microsoft Audio Home Theater Effects" APO on the speakers listed VirtualSurround
  and VirtualHeadphones effects next to the spatial format (05:12). Turning **Audio enhancements** Off for the
  speakers removed them, and the spatial format stayed active (05:17).
- Windows' Voice clarity on the microphone stands in for THX's noise reduction; it stayed on through all steps.

### 7.8 Final state (cold boot 05:11; checks 05:12 and 05:17)

A full shutdown (Shift + Shut down), 30 s off, power on. Then: no Razer or THX program, service, process, autostart
entry, scheduled task, kernel service, `.sys` file, driver package, folder or registry tree; no `RZCONTROL`,
`RZVIRTUAL` or THX node; every present `VID_1532` node on an inbox INF with no filter (36 pass, 1 warning: the check
ran unelevated). Both HID devices read normal mode; the headset answered over its COM port (90 % battery, firmware
1.6.0) (OBSERVED 05:12).

## 8. The devices without Synapse

### 8.1 Drivers per interface (**HOST**, 2026-11-03 05:17)

| Device | Interface | Inbox driver |
|---|---|---|
| DeathAdder V2 | HID 0 (mouse, control collection) | `msmouse.inf` |
| Huntsman V2 | HID 3 (consumer, control collection) | `hidserv.inf` |
| | HID 0 (boot keyboard) | `keyboard.inf` |
| Kraken V3 Pro dongle | USB 0 (audio) | `wdma_usb.inf` |
| | USB 4 (COM port) | `usbser.inf` |
| all | the other 31 present nodes | Windows' own INFs, no filter |

### 8.2 What still works (all three devices fully usable by third-party code, OBSERVED 2026-11-03)

- **Lighting:** the mouse and the keyboard run the stored spectrum cycle Synapse wrote at its last exit (CAPTURED,
  section 3.3; OBSERVED 2026-11-01: after a replug the mouse logo showed the colour cycle). Effects and custom frames
  from another program work in normal mode, and custom frames display without `0F:10` (OBSERVED 2026-11-01). Effects
  and DPI written without the storage flag did not survive a replug (OBSERVED 2026-11-01). The headset shows a host
  colour until its next power-off and then its own colour cycle (OBSERVED 2026-11-02).
- **Mouse:** 1000 Hz and the stored DPI stages (400 / 800 / 1600 / 2400 / 20000, stage 4 active) stay on the device
  (OBSERVED 2026-10-31 and 2026-11-01). In normal mode the firmware handles the DPI buttons (INFERRED, section 1.4).
- **Keyboard:** Fn+F10 game mode blocks the Windows key in firmware without Synapse (OBSERVED 2026-11-01). The
  keyswitch optimisation written with `02:02` alone persisted through a replug (OBSERVED 2026-11-01). The keyboard
  powers up at 1000 Hz; 8000 Hz is what Synapse set, and a host has to write it after each power-up (OBSERVED
  2026-11-01). The game-mode LED write `03:00` (LED 0x08) was accepted but no LED lit on this unit (OBSERVED
  2026-11-01).
- **Headset:** sidetone, power saving and the haptics level are stored on the headset and stay as last set; the
  battery and charging state are readable over the COM port (OBSERVED 2026-11-02). Audio, volume wheel and mic mute
  are plain USB Audio and HID (INFERRED from the inbox stack).
- **Windows Dynamic Lighting:** with no Razer software left, Windows can drive the two LampArray collections. Whether
  it competes with another program's raw-HID lighting, and whether the `0F:10` flag Synapse set still matters, were
  not checked (**UNKNOWN**, section 10).

### 8.3 What is lost

- Synapse's software features, which run on the PC in driver mode: button remaps, Hypershift, macros (including
  on-the-fly recording), game-linked profiles, cloud sync.
- Host-rendered effects (all of Synapse's effects are host-rendered, section 3.1), Chroma SDK lighting in games,
  Chroma Studio, Chroma Connect and Chroma Broadcast followers, and the dimming of the LEDs on power and display events
  (section 3.4), unless another program does these.
- Every setting Synapse re-applied that the devices do not keep: 8000 Hz on the keyboard at power-up, Synapse's
  lift-off and calibration settings (written at every attach; whether the mouse keeps them is **UNKNOWN**).
- THX Spatial Audio and the headset's PC-side EQ and microphone processing (Windows spatial formats and Voice
  clarity replace parts of it).
- The headset's battery display and notifications in Synapse (the value stays readable from the device).
- Update notifications through Razer Central. This reference does not cover firmware updates.

## 9. Hazards

| Hazard | Guard |
|---|---|
| Killing `Razer Synapse Service` instead of stopping it skips the release, INFERRED to leave the mouse and keyboard in driver mode, where the DPI buttons, Fn layer and media keys do nothing without Synapse | stop through the Service Control Manager; read `00:84` afterwards and write `00:04` 0x00 if it reads 0x03 |
| The services restart by themselves if the window or the per-user process is still open (section 5.1) | close those first, set the start type to Manual, watch for a minute |
| The dongle's COM port is exclusive; holding it when Synapse starts breaks Synapse's headset features for the session (section 3.5) | open it only while `Razer Synapse Service` is stopped; release it before handing back |
| Raw-HID writes while Synapse or the Chroma SDK runs are overwritten and can interleave with their transactions | do not write while either runs |
| Synapse rewrites settings and the onboard profile with storage 0x01 at every attach (section 3.2) | expect any stored change to be overwritten while Synapse is installed |
| Shut down with Fast Startup keeps Razer kernel drivers loaded | use Restart, or a full shutdown, after uninstalling |
| Superseded Razer packages in the driver store are picked up when the current ones are deleted (section 7.3) | list the store (`pnputil /enum-drivers`) and delete every package whose original name starts with `rz`, `razer` or `thx` |
| The co-installer packages reinstall Synapse when a device meets a new port (section 7.6) | delete them; set Device Installation Settings to No |
| Phantom device nodes still name deleted filter services | remove them with `pnputil /remove-device` |
| `RzCommon.sys` is PnpLockdown-protected | take ownership before deleting; it is inert once its service is deleted |
| Changing the dongle's audio driver resets spatial audio | choose the spatial format again after the change |
| Synapse's code knows a Razer bootloader PID, `1532:110D` (**METADATA**, the device access DLLs and the update module's "Ignoring razer bootloader pid") | never open a device with that PID; see [razer-hid-protocol.md](razer-hid-protocol.md) for commands never to send |

## 10. Read-only checks after a removal

| Check | Expected (**HOST**, 2026-11-03) |
|---|---|
| Installed apps | no Razer or THX entry (the hidden THX products show only in the uninstall registry keys) |
| Services and processes | none whose name starts with `Razer` or `Rz`; no `RzActionSvc` |
| Kernel drivers | `RzDev_*` and `RzCommon` not registered; no `RzDev_*.sys` or `RzCommon.sys` in the drivers folder |
| Driver store (`pnputil /enum-drivers`) | no package with an original name `rz*`, `razer*` or `thx*` |
| Device nodes | no `RZCONTROL`, `RZVIRTUAL` or THX software-device node; no `UpperFilters` / `LowerFilters` value naming `RzDev_*` on any `VID_1532` node; drivers as in section 8.1 |
| Setup wizard | no `RzS3WizardPkg` folder |
| Device Installation Settings | No |
| Device mode of mouse and keyboard (`00:84`) | 0x00 |
| Audio | the headset's speaker and microphone endpoints present; spatial format set again |

## 11. Open items

| Item | Status | What would close it |
|---|---|---|
| Device mode after a **killed** Synapse service | **UNKNOWN** (INFERRED driver mode stays on) | kill the service once on a test install, then read `00:84` |
| Why `RzSDKService` and `Razer Synapse Service` restarted on 2026-10-31 | **UNKNOWN** | repeat the wrong order with a process-start trace (parent process of the restarted services) |
| Which process holds the dongle's COM port | INFERRED (strong) `Razer Synapse Service` | an elevated, read-only handle listing of the port's device object with Synapse running |
| What briefly held the COM port with only the Chroma SDK running (2026-11-02) | INFERRED `RzDeviceManager` | the same handle listing at a service start |
| Effect and persistence of `0F:10` (Force Chroma Control) | **UNKNOWN** | change Windows Dynamic Lighting for the mouse or keyboard with no Razer software installed, before and after sending `0F:10` 0x00 and 0x01 |
| Whether Windows Dynamic Lighting competes with raw-HID lighting now that Synapse is gone | **UNKNOWN** (not checked) | switch a Dynamic Lighting effect on while another program streams frames, and watch the LEDs |
| Whether Device Installation Settings = No also stops Windows Update offering the co-installer packages | **UNKNOWN** | list the driver store after Windows Updates and after plugging each device into a new port |
| What the headset shows after Synapse's stream stops, before a power cycle | **UNKNOWN** | exit Synapse with the headset on and a non-default colour streaming, and look |
| Which part of Synapse ends the release at an "exit" (window, per-user process or service), and whether closing the window alone releases the devices | **UNKNOWN** | capture a tray Exit with the service left running |
| Sleep and resume on the wire | not captured (only Synapse's logs, section 3.4) | a capture across one sleep cycle with Synapse installed |
| Exact runtime names of `RzProtocolMutex_*` for Synapse and the Chroma SDK | **UNKNOWN** | an elevated handle listing filtered on `RzProtocolMutex` |
| Mapping of Synapse's firmware strings (1.0.3.0, 1.0.5.0) onto the `01 00 00 00` reply | **UNKNOWN** | a second unit with another firmware version |
| Synapse 4's services, processes and wire behaviour | not observed | the same analysis on a Synapse 4 install |
