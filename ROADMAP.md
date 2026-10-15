# HAMPOD2027 Roadmap (public-side, tracking against HAMPOD Master Plan)

Each step is sized to be independently verifiable. Plan order is given by the
private `master_plan.md`; this mirror just tracks the public scope. Source authority: the private
workspace's `master_plan.md` schedules the steps; this file mirrors only the
public-facing scope and checkpoint.

## Phase 0 — Inputs & decisions
### Also pre requisite (from master plan Phase 1, performed privately)
 - 0.5 Brand reverse engineering scoped for Icom (done), Yaesu, Kenwood, Elecraft (private;
       repo holds only the subsequent hardware/engine build), followed by brand_diffs.md
       so that the YAML profile shapes are driven by a verified variability surface.

- 0.1 Requirements snapshot (copied requirements only)
- 0.2 Architecture ADR (process model, profiles, HAL)
- 0.3 Latency budget
- 0.4 Repo layout + makefile targets

## Phase 3 — engine running on Pi (period to earliest demo)

- 3.1 test harness `--sim`
- 3.2 Core event loop + evdev keypad
- 3.3 `speechd` (piper + ALSA + vocab cache, barge-in ≤ 20–50 ms)
- 3.4 `rigd` (hamlib, watchdog timeouts)
- 3.5 Config + startup (`hampod.conf`, runtime radio switch)
- 3.6 Smoke bench on a real IC-7300

## Phase 4-6 — Icom-first full parity, then Yaesu/Kenwood/Elecraft profiles, hardening.

Important: phases 1-2 (firmware reverse-engineering) are *private workspace*
activities and are NOT tracked in this repository.

See [master_plan.md archived privately] for the full detail.
