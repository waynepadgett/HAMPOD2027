# HAMPOD2027 — Engineering

This document is the *public* engineering rationale. It describes the product
shape and implementation guidance without reverse-engineering detail.

## 1. What the device must do

The product is an accessibility interface: a blind amateur-radio operator (or any
institutional user) uses a USB numeric keypad and audio feedback to control radios
and related station equipment (antenna rotators, power/SWR meters, other
serial-compatible gear), via a Raspberry Pi.

The dominant design constraints:

1. **Low-latency key response** — ≤ 200 ms keypress→speech start on Pi 5, ≤ 300 ms
   on Pi 3, within typical load.
2. **Interruptible speech** — a keypress cancels the current announcement fast
   (20–50 ms) and queues the next one. This is a flow / responsiveness
   property, not a safety property.
3. **Responsive state detection** — radio power on/off, VFO/frequency changes,
   USB keypad / audio hotplug must be noticed without any loop "waiting" on them.
4. **No lockups, ever** — the device cannot assume the user can see an error
   screen; a stuck process is effectively a dead product.

Constraints source: see the requirements appendix (Pi 3/4/5, Zero 2 W, Debian
Trixie, ALSA audio, libhamlib, piper, gcc/make).

## 2. Runtime topology

Three units, star topology:

```
                ┌────────────────────┐
   evdev       │  CORE (event loop) │   /etc+hampod.conf, profiles/*.yaml
  ───────────► │  keys · mode FSM   │ ◄──── SIGHUP reload
   eventN       │  timers(timerfd)   │
                └────┬──────────┬────┘
      SOCK_SEQPACKET │          │ SOCK_SEQPACKET
                  ┌──▼──┐   ┌──▼──┐
                  │ SPEECHD │  │ RIGD │    hamlib
                  │ piper │  │ worker│
                  │ ALSA  │  │       │
                  └───────┘  └───────┘
```

- The **core** is the only process with product logic: debounce, tap/hold timing,
  mode finite-state machine (interpreted from the active profile), announcement
  selection, config reload, watchdog heartbeat.
- Two **supervised, restartable children**: speechd (piper TTS + ALSA) and rigd
  (hamlib). Third-party (piper, hamlib drivers) is where blocks/segfaults live;
  isolation means key responsiveness cannot be dragged down. Children are
  respawned with backoff and reset.
- **Transport**: `AF_UNIX SOCK_SEQPACKET socketpair`. Message boundaries preserved,
  bidirectional, no FIFO framing, no named-pipe deadlocks; star topology (no
  process-to-process traffic) makes a deadlock *cycle* structurally impossible.
- **Core blocking rule**: the only blocking call is `epoll_wait`, bounded by the
  earliest armed `timerfd`. Every fd is `O_NONBLOCK`, every send
  `MSG_DONTWAIT`, every request has a deadline. A missed deadline becomes an
  event, not a hang. `CLOCK_MONOTONIC` only. Heartbeat via `sd_notify`.

Lockup-avoidance tools: in-code deadline checks, systemd `WatchdogSec`, RSS
caps (valgrind/ASan builds), a RAM log ring (so diagnostics survive a read-only
rootfs), and fault-injection testing.

Rejected alternatives: multi-process FIFO mesh (deadlock and framing bugs),
single-process threads only (crash isolation lost where the buggy code lives).

## 3. Speech pipeline (the real latency lever)

Do not synthesize everything at runtime.

- **Pre-generate** the static vocabulary (mode names, units, band names, error
  phrases, letters, digits) with piper at install time → stable `.pcm` cache.
  Static responses become pure playback, comfortably under the 100 ms
  TTS-command-to-audio target.
- **Compose** dynamic phrases (frequencies) from cached segments (digits + "MHz" +
  band), concatenated with small gaps. This is the same architecture the original
  product used for speech.
- Runtime piper synthesis only for cache misses, deadline-bounded. ALSA settings
  per hardware constraints: small period, double buffer, underrun refill.
- **Barge-in**: persistent audio handle, `snd_pcm_drop()` for immediate stop
  (never `snd_pcm_drain`); queue flush; seconds-spell response within ~20-50 ms.

## 4. Device model, profiles, and setup

Everything is described by a **profile** (YAML, one file per device model), so
new radio models or station equipment (power/SWR meters, rotators, etc.) can be
supported with data changes, not recompiles:

- A **device type** is the top-level concept (`radio`, `rotator`, `powermeter`,
  …), and every supported type has a profile that defines its mode tables, key
  maps, announcement text, command/service sequences, timed ladders, and
  capability set.
- `hampod.conf` = environment (devices, audio, volumes, which device is active).
  **Profile** = behavior. The system detects the active profile based on the
  connected device.
- **Multi-device / setup mode**: the system records the last-seen USB topology.
  When a device appears in a new port, it enters **setup mode**: if a Hamlib
  query gives a confident guess, that guess is presented for **confirmation**;
  otherwise the operator picks from a narrowed list (device class → brand → model),
  and only profiles that actually exist are offered. A generic "Hamlib-backed"
  profile can serve Hamlib-capable models without a full authored profile.
  Comm parameters start from the device driver default; a fallback is offered
  only on a reported comm error.
- **USB keypad**: read via the Linux input subsystem (`evdev`), with HAL
  enumeration and deterministic selection (preferred board, never blind
  `hw:2,0`); hotplug is expected and handled.
- **123 vs 789** numeric layout is one global profile switch (no per-device
  divergence), applied as a projection mapping.
- **`[A]` key is a stateful modifier**: a tap toggles "SHIFT" announce mode and
  stays until another key is pressed or A is pressed to undo — not a chord.

## 5. Test seams and verification

All OS-facing capabilities (`input`, `audio/synth`, `radio`, `clock`, `log`)
sit behind a small HAL vtable: real impls on the Pi, `virtual` impls for
golden tests + a headless `hampod --sim` mode.

Automated:
- Profile validation (schema + semantic lints)
- Golden behavior tests: scripted key traces → expected event logs, compared
  exactly.
- Latency probes with recorded timestamps, run in CI on real Pi hardware.
- Soak/fault-injection: random key-mash + radio power-cycle + `kill -9` children
  + device unplug mid-utterance; assert: core survives, children respawn,
  watchdog restarts == 0, RSS flat.
- Fuzz tests for the config loader and profile loader.

Manual (human gates): per-mode bench checklists with real radios, accessibility
walkthroughs with actual blind users (intelligibility, timing feel,
error recovery), radio-off/on and audio hot-plug scenarios.

## 6. Key performance decisions

- Language: **C** (gcc/make, per target constraints). C11, HAL vtable, simulator
  build target `make sim`.
- Persistent piper subprocess; ALSA is opened once.
- Vocabulary pre-generated at install, or on first boot.
- Async-IO exclusively, `epoll` + `timerfd`, bounded waits.
- cfg/reload plus runtime profile switch (no restart needed for device changes).
