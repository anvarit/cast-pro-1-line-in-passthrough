# Changelog

Versions are shown in the *Apollo Firmware Version* sensor as `<Apollo version>-passthrough-<this version>`.
Versions 0.1 to 0.6 were built, in order, on 2026-11-16 against Apollo 26.11.3.3; 0.7 followed on 2026-11-17. 0.1 and
0.2 were development builds; the first public file was the 0.6 state.

## 0.7 - Music Assistant Trim, gain up to +24 dB, new defaults

- Solves: a weak line-in source (a computer dock at full volume gives about -31 dBFS) sounded much quieter than Music
  Assistant, and the only tool was *Line In Gain*, which also raises the source's noise.
- New setting *Music Assistant Trim* (dB, 0 to -30, default 0): turns the Music Assistant input down all the time.
  It is applied by the single ducking script, on top of the line-in ducking: no signal, `Mix` and `Music Assistant wins`
  give just the trim; `Duck other` gives trim plus duck amount; `Line In wins` is silent; during an announcement the
  larger of (trim plus announcement ducking) and what line-in wants. Capped at 80 dB; the announcement input and line-in
  are never trimmed. With the trim at 0 the behaviour is exactly that of 0.6.
- *Line In Gain* now goes up to +24 dB (was +12). The range is still safe for the 32-bit math in the audio callback
  (checked in the comments in the file). Gain above +12 dB has not been listened to.
- New first-boot defaults (stored values on existing devices are kept): *Start After* 0.2 s -> 1.0 s, because a short blip
  at 0.2 s caused a false trigger in the 23-hour soak; *Release After* 20 s -> 8 s, because 20 s felt too long before
  Music Assistant came back (announcements are handled separately). *Keep Warm* stays 15 minutes, *Duck Amount* 30 dB.
- README: soak result of 0.6, new troubleshooting entries; `example-device.yaml` pins `ref: v0.7.0`.
- Not changed: Apollo pin (26.11.3.3), `memory-saver`, `diagnostics`.

## 0.6 - Bluetooth proxy and WizMote off at boot (now the optional `memory-saver` package)

- Solves: too little free internal RAM. With passthrough, Music Assistant and an announcement running together the lowest
  free internal heap dropped to about 8 KB.
- `espnow` and `esp32_ble` no longer start at boot (ESP-NOW init is what loads the Wi-Fi driver on a wired unit).
  Runtime paths that would start them again are overridden: `wifi_release_radio`, `wifi_recovery_probe`,
  `wizmote_channel_scan`, and the *Bluetooth Proxy* switch is inert.
- Result on one unit: idle free internal heap about 88 KB -> about 184 KB.
- Trade-off: no Bluetooth proxy, no WizMote. See `packages/memory-saver.yaml`.

## 0.5 - Single owner of ducking, Release After held during announcements

- Solves: during an announcement test the music source came back loud for about a second after the announcement, while
  line-in was supposed to win. Cause: Apollo's player hooks un-duck the music input on every state publish and at the end
  of an announcement, fighting the controller's own ducking; and silence around an announcement counted toward
  *Release After*.
- Apollo's `on_announcement` / `on_state` hooks are replaced by copies that call one script (`lp_duck_sync`), the only
  place that changes ducking. It sends a new ramp only when the target changes.
- The release timer does not run while an announcement is playing.

## 0.4 - Ducking race fix, line-in ducked during announcements

- Solves: the music source briefly un-ducked right after announcement / state edges (first attempt: periodic re-apply of
  the duck; replaced by the cleaner design in 0.5).
- Line-in is ducked by at least 30 dB during an announcement.
- (Internal diagnostic logging for announcement start/stop was used while developing this; it is not part of the
  public file.)

## 0.3 - Keep Warm

- Solves: the start of the audio was cut every time line-in came back after a pause, because the S/PDIF output shuts
  down after silence and needs 1-2 s to restart.
- New setting *Line In Keep Warm* (minutes, default 15, 0 = off): the path stays open after the last signal.
- Ducking now follows the signal, not the open path, so `Always` mode and the warm period no longer mute Music Assistant.

## 0.2 - Memory diagnostics

- Four heap sensors (now the optional `diagnostics` package) to see how much internal RAM is left.

## 0.1 - First working passthrough

- Line-in (PCM1808) as a third mixer input, signal detection, Off / Auto / Always, four priority modes, threshold, start /
  release delays, duck amount, gain, Level / Signal / Active sensors.
- Apollo's self-update removed.
- First flash boots with passthrough `Off`.
