# Status and change log

Part of the [Razer peripherals protocol reference](README.md). One row per command or area; the tags are explained in
the README. "Captured" refers to the 2026-10-30 capture of Synapse 3 (3.10.0730.071519); "observed" to third-party
code on the bench.

## Shared 90-byte protocol (mouse and keyboard)

| Command / area | Name | Layout | Captured | Open |
|---|---|---|---|---|
| framing, checksum | 90-byte report, XOR of `r[2..87]` | complete | yes | |
| status codes | `00` new, `01` busy, `02` ok, `05` not supported | complete | yes | `03` and `04` never seen |
| transaction ids | `1F` (Synapse), `0C` / `08` (at attach, INFERRED the Razer filter driver) | complete | yes | whether other ids are accepted; a SET while busy |
| answer sizes | an answer carries its own size, often smaller than the request's | complete | yes | |
| busy answers | rewritten arguments, stale checksum | complete | yes | |
| storage byte | `a[0]` `00` volatile / `01` stored | complete; replug behaviour observed | yes | flash wear |
| `00:84` / `00:04` | device mode: `00` normal, `03` driver | complete | yes | device state after Synapse is killed |
| `00:81` / `00:82` | firmware, serial (serial not reproduced) | complete | serial only | |
| `03:00` / `03:80` | indicator LED state | complete | yes (LED `08`, `18`) | role of LED `18` |
| `0F:02` / `0F:82` | hardware effect / stored effect | complete | apply at attach, spectrum at exit | breathing, reactive, starlight on these models |
| `0F:03` | custom frame row | complete | yes (about 25 and 24 per second) | highest safe frame rate |
| `0F:04` / `0F:84` | brightness, LED `0x00`, `percent × 255 / 100` truncated | complete | yes | |
| `0F:80` / `0F:81` | regions / effect list | `0F:80` complete | `0F:80` only | bytes `19 03`; `0F:81` |
| `0F:10` | undocumented, sent after every switch to driver mode | one byte | yes | meaning, persistence |
| `05:xx`, `06:xx`, `02:0C`, `02:14` | profile, storage, mapping | bytes, partly decoded | yes | meanings; do not write |
| LampArray coexistence | Windows' reports 1 to 6 at attach | per HUTRR84 | mouse attach | arbitration without Synapse |

## DeathAdder V2 1532:0084

| Command / area | Name | Layout | Captured | Open |
|---|---|---|---|---|
| descriptors | device, configuration, four HID interfaces, report descriptors | complete | yes (replug) | vendor input bytes; interface 1 collections |
| `00:81` / `00:87` | firmware, two-part / four-part | complete | `00:87` yes; `00:81` observed | |
| `00:86` | unknown read | bytes only | yes | meaning |
| `00:85` / `00:05` | polling rate 125 / 500 / 1000 Hz | complete | yes | persistence |
| `04:85` / `04:05` | DPI X/Y with storage flag | complete; volatile and X/Y observed | yes | steps other than 50 |
| `04:86` / `04:06` | DPI stage list | complete | write yes; read observed | volatile form |
| `0F:02`, `0F:03`, `0F:04` | effects, 1 × 2 frames, brightness | complete | yes | column order checked by eye only |
| `02:0C` | button mapping | partial | yes (10 inputs, 2 layers) | key and macro arguments |
| `05:8A` / `05:80` / `05:81` / `06:80` / `06:8E` | profile and storage queries | `06:8E` decoded | yes | the others' meanings |
| `0B:03` / `0B:0B` | sensor settings | bytes only | yes | meaning |
| LampArray (interface 3) | two lamps, attributes, Windows' updates | complete | yes | lamp-to-LED map; report 7 |

## Huntsman V2 1532:026C

| Command / area | Name | Layout | Captured | Open |
|---|---|---|---|---|
| descriptors | five HID interfaces | complete | yes (configuration) | report descriptors; bus speed |
| `00:81` / `00:82` / `00:86` / `00:84` / `00:04` | identity, layout, device mode | complete | `00:86`, `00:84`, `00:04` yes | source of the "1.0.5.0" Synapse shows |
| `00:40` / `00:C0` | polling rate 1000 / 8000 Hz | complete | yes | 125 to 4000 Hz; meaning of `a[0]` |
| key map | 6 × 22 cells → keys, US ANSI | complete | 108 cells Synapse lights | per-cell check; other layouts |
| `0F:02` on LED `05` | hardware effects | complete | spectrum at exit only | static, breathing, reactive, starlight; wave `00` |
| `0F:04` / `0F:84` | brightness | complete | yes (LED `00`) | a write on LED `05` |
| `03:00` / `03:80` | indicator LEDs | complete | LED `08`, `18` | LED `18` and `07` behaviour |
| `02:02` / `02:82` / `02:15` | keyswitch optimisation | complete; `02:02` alone observed | no | units; `02:15` |
| `02:14` | a mouse command, rejected (`05`) | complete | yes | |
| `05:xx` / `06:80` / `06:8E` | profile and storage queries | partly INFERRED | yes | meanings |
| LampArray (interface 4) | Dynamic Lighting | INFERRED | one report | attributes, arbitration |

## Kraken V3 Pro 1532:052C (dongle)

| Command / area | Name | Layout | Captured | Open |
|---|---|---|---|---|
| descriptors | dongle device and configuration | complete | yes | strings |
| CDC open and line settings | 9600 8N1, DTR/RTS | complete | no (port already open) | behaviour with DTR low |
| `0xFFF3` frame and reply | 28-byte frame, 31-byte reply, checksum | complete | yes | meaning of `f[4] = 02` |
| `0xFC9A` | keepalive | complete | yes (63, in stretches) | purpose; not needed |
| `04 FF 01 E8` / `E9` | link up / down events | complete | yes | |
| `0x20` / `0x21` | firmware / serial (serial not reproduced) | complete | no (observed) | whether dongle or headset |
| `0x22` | unknown | one byte | read only | meaning; never write |
| `0x24` | status and battery | `v[0]` to `v[3]` | yes; observed | scale of `v[1]`, `v[2]` |
| `0x25` | sidetone on, level 0 to 4 | complete | yes; observed | audibility per level |
| `0x26` | power saving on, seconds | complete | yes; observed | limits outside 900 to 3600 s |
| `0x27` | event after link-up | `v[0] = 01` | yes | meaning |
| `0x40` / `0x8A`, `0x8B` | haptics write / read | complete | yes; observed | |
| `0x40` / `0x8C` | earcup colour | mode `08`, `f[11]` `00` | yes (652 frames); observed | `f[11] = 01`; modes `00` to `04` (`03` seen once) |
| `0x40` / `0x8D` | effect read | expected layout only | no | the device's reply |
| `0x40` / `0x8E`, `0x8F` | brightness write / read | write complete | write yes | read; persistence; value 0 |
| `0x40` / `0x90` | HyperSense button event | complete | yes | |
| `0x93` / `0x94` | gates published elsewhere | none | no | everything |

## Software and drivers

| Area | Layout | Captured | Open |
|---|---|---|---|
| Synapse attach sequence (mouse replug) | complete | yes (segment 5) | |
| Synapse start sequence | complete | yes (segment 14) | |
| Synapse exit and release | complete | yes (segment 13) | headset state after the colour stream stops |
| host-rendered lighting | complete | yes | |
| sleep, log-off, resume | from Synapse's logs | no | a wire capture |
| the dongle's COM port ownership | exclusive; start-up race inside Synapse | n/a | |
| service restarts, takeover order | observed | n/a | cause of the restarts |
| Synapse killed rather than closed | | no | device mode afterwards |
| removal and leftovers | complete (HOST) | n/a | Windows Update re-offering Razer co-installers |
| Dynamic Lighting without Synapse | partly | Windows' writes at attach | competition with raw HID |

Firmware update: **deliberately not described** for any of the three devices.

## Change log

- **2026-11-04, update.** Kraken V3 Pro: the volume wheel changes the headset's own level only (end stops, Windows'
  endpoint volume unmoved) and the mute button leaves Windows' recording-endpoint mute untouched, both OBSERVED by
  turning and pressing them while watching Windows; the earlier reading of the HID descriptor (wheel = volume
  increment/decrement keys) is withdrawn for the wheel. Also OBSERVED: a short low tone (40-60 Hz, 0.3-0.8 s) on the
  headset's playback endpoint produces no HyperSense vibration at any level (section 5.4), so an audio-driven haptic
  pulse is not available this way.
- **2026-11-04, first edition.** Written from one USBPcap capture of Synapse 3 in 14 segments (2026-10-30, 13:39 to
  14:03), Razer's installed files, Windows' device properties and Synapse's logs on the same PC, bench sessions with
  independently written code (2026-10-31 to 2026-11-03), the removal of all Razer software on 2026-11-03, and the
  projects credited in the README.
