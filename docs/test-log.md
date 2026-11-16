# Test log

One CAST PRO-1, wired Ethernet, Apollo 26.11.3.3, ESPHome 2026.11.0-dev (Docker image `esphome/esphome:2026.11.0-dev20261104`).
Speakers on the optical output; a computer on the line-in; Music Assistant and Home Assistant announcements as the
other sources. Versions refer to [CHANGELOG.md](../CHANGELOG.md). Builds were flashed by OTA.
Where a result below was observed by ear it says so; numbers come from the device's own heap sensors and logs.

## Build verification of this repository's files

`esphome config` and `esphome compile` (no upload), ESPHome `2026.11.0-dev20261104` container, `--network host --memory 4g`.
Both variants: config valid, compile clean (only three warnings from upstream code: `PACKED` redefined in `bt_types.h`,
a `%d` format in Apollo's `wizmote.yaml`, `cos_LSF_QA` in the Opus library).

| Variant | OTA image | Factory image | Flash use | Static RAM |
| --- | --- | --- | --- | --- |
| main file only | 2,242,144 B | 2,307,680 B | 57.0 % of 3,932,160 B | 44.1 % |
| main + memory-saver + diagnostics | 2,240,576 B | 2,306,112 B | 57.0 % | 44.1 % |

The remote-package form (`url`, `ref`, `files:` list, as in `example-device.yaml`) resolves to exactly the same
configuration as local `!include`s, for the main file alone and for all three files. (Checked against a local git
repository; the `github.com` URL itself was not exercised before publication.)
The generated C++ of the full variant is identical to the author's development build 0.6, which runs on the author's
device, apart from: version strings, the removed verbose logging of the announcement player, and three first-boot defaults
(Start After 0.5 s -> 0.2 s, Release After 5 s -> 20 s, Duck Amount 20 dB -> 30 dB; the author's device already ran with
its own stored values).

## Stage 1-2: build, first flash with passthrough Off (0.1)

- Config valid, compile clean (only warnings from Apollo's code), image about 2.24 MB.
- Flashed with passthrough Off: entities as expected, Home Assistant pairing and API key kept, Music Assistant player
  unchanged, 12 minutes without reboot, no warnings or errors in the log.

## Memory with the ADC and S/PDIF output running (0.2, Bluetooth and WizMote still on)

| State | Free internal | Lowest free since boot | Largest internal block |
| --- | --- | --- | --- |
| Passthrough Off, idle (3 min) | 108 KB | 105 KB | 63 KB |
| Passthrough Always (5.5 min) | 35 KB | 33 KB | 27 KB |

Starting the ADC, mixer input and S/PDIF output costs about 73 KB of internal RAM; stable afterwards. Line In Level
ranged from about -77 dBFS (noise floor of the source at rest) to -31 dBFS (signal); Signal toggled accordingly.

## Keep warm (0.3)

Observation that led to it: in Auto, the start of the sound was cut after pauses (not in Always, not while Music
Assistant played; a shorter Start After made no difference). After the change the path stays open for the Keep Warm time.
Idle memory in Auto with the ADC running: about 88 KB free, 86 KB lowest.

## Announcements (0.4, verbose logging of the announcement player enabled)

Home Assistant TTS to the CAST PRO-1's player, passthrough idle, no Music Assistant:

| # | gap since previous | result |
| --- | --- | --- |
| A | first after boot | started |
| B | 23 s | started |
| C | 22 s | started |
| D | 95 s | started |
| E | 22 s | started |
| F | 18 s | started |
| G | 0.7 s after F | did **not** start (no state change, nothing logged) |
| H | 20 s | started |

- Seven of seven announcements spaced 18-95 s apart started, each in about 0.6 s, with no retry or error line. The
  earlier "never starts" cases (a minute or two after another announcement) did **not** reproduce here. Cause still unknown.
- G vanished because of the player's queue design: every new announcement clears the playlist before pushing its own URI,
  so two announcements within the time the first needs to start collapse into one (latest wins). Nothing is logged.
- Memory: lowest free internal heap fell from 86 KB to about 15 KB during the first announcements with passthrough
  idle and no Music Assistant (an announcement takes about 50-70 KB of internal RAM for a second). No reboot.
  In a run with Sendspin and passthrough active it fell to about 8 KB, which prompted 0.6.

## Listening test of ducking around an announcement (0.4, by ear)

Setup: Music Assistant radio via Sendspin, computer on line-in, Line In wins, announcement ducking -40 dB, Release After
5 s, one TTS announcement. Heard: the announcement, the computer dropped during it and came back, **but the radio was
loud for about a second before the computer came back**. Bug.

Cause (from the device's sensor history): the line-in itself was silent for about 6 s (longer than Release After 5 s,
starting during the announcement), so the controller released the signal and un-ducked the radio; it ducked it again 1.1 s
later when the computer came back. Why the source went silent is not known; the ADC kept delivering noise-floor samples.
Fixed in 0.5 (single owner of ducking; Release After not counted during announcements). In that timeline the computer was
back 4.1 s after the announcement ended, so 0.5 would have kept the radio muted, with about 0.9 s margin at Release After 5 s.
The default Release After is now 20 s.

## Bluetooth proxy and WizMote off (0.6)

| State (Auto, ADC on, nothing playing) | 0.4 | 0.6 |
| --- | --- | --- |
| Free internal heap | 88 KB | 184 KB |
| Lowest free internal heap | 86 KB | 182 KB (after boot) |
| Largest internal block | 47 KB | 139 KB |
| Free PSRAM | 4.33 MB | 4.41 MB |

Log after boot: Wi-Fi disabled, ESP-NOW disabled, Bluetooth stack not enabled, no warning or error lines over 3.5 minutes.
Just before the flash (0.4 with Music Assistant radio playing): 27 KB free, lowest since boot 8 KB, largest block 12 KB.
Not tested on hardware: pulling the Ethernet cable (Wi-Fi fallback with memory-saver).

## Listening tests on 0.6 (by ear)

Setup: Music Assistant radio via Sendspin, computer on line-in, Line In wins, Release After 20 s.

| # | State | Gap since previous | Result |
| --- | --- | --- | --- |
| A | radio only | first | announcement heard, radio ducked and came back (as stock) |
| B | radio + line-in | 44 s | announcement did **not** start (the unexplained timing issue again) |
| C | radio + line-in | about 90 s after A | announcement heard over the ducked line-in |

Memory in this worst case (radio + line-in + announcement): lowest free internal heap since boot 105 KB (was 8 KB on
0.4). Free with radio + line-in playing: about 126 KB (was about 29 KB).

## Earlier listening tests (0.2-0.3, by ear)

- Always: clean sound, level fine, delay barely noticeable.
- Auto: release after silence and restart work; start cut of 1-2 s after a pause on 0.2, gone with keep-warm on 0.3.
- Priority *Line In wins*, *Duck other* (30 dB preferred over 20 dB) and *Mix* with Music Assistant radio: all as
  labelled, clean mixing.

## Not tested

- 24 h soak (clicks, drift, reboots): **running**, started 2026-11-16.
- Priority mode *Music Assistant wins*.
- Streaming line-in to Music Assistant (Sendspin Source) with this firmware, and the local passthrough pausing while it does.
- Latency in milliseconds.
- Wi-Fi fallback with `memory-saver` on hardware.
- Whether receivers still enter standby while the keep-warm optical signal is running.
- Any other CAST PRO-1 unit, other ESPHome versions, Wi-Fi-only operation.
