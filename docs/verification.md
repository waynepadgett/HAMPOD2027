# HAMPOD2027 — Verification & Test Strategy

## Automated (CI / sim, no hardware needed)

- **Profile gate**: `hampod-profile validate` (JSON schema + semantic lints) on every
  profile in the tree; that is the biggest correctness lever for adding a new
  radio model (a profile is "the complete spec").
- **Golden behaviour tests**: YAML fixtures of key traces (scripted taps/holds/timings) →
  expected event logs (spoken tokens, hamlib calls, mode transitions), compared byte for
  byte. Run in `--sim` against virtual HAL + fake radios that model timeouts —
  discovers lockup classes without hardware.
- **Latency probes**: instrumented evdev→core→speechd→ALSA path; p50/p99 assertions.
- **Soak + fault injection**: random key-mash + radio power-cycle + `kill -9` of speechd /
  rigd + device unplug mid-utterance; assert core survives, children respawn, zero
  watchdog restarts, flat RSS (valgrind/ASan).
- **Property tests** for the debounce/tap-hold/numeric-projection logic.
- **Config/hampod.conf fuzzing** for robust loading.

## Manual (human gates)

- Bench matrix per brand — each mode × each key (tap/hold) versus the real device
  (IC-7300, FT-891, TS-2000, and the equivalent Elecraft/other supported types).
- Accessibility walkthrough with blind users: intelligibility, timing feel, error-recovery
  UX.
- Radio off/on-mid-operation scenario and audio hot-plug scenario run by hand once
  per release.

All checklists should be plain text, screen-reader-friendly (≤ 72 columns per line)
like the existing `control_flow.txt` transcription.
