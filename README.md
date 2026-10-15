# HAMPOD2027

A Raspberry-Pi reimplementation of the HAMPOD product — a tactile USB-keypad + speech
interface that makes amateur radio station equipment accessible to visually impaired users.

This repository contains **only the implementation side**: portable engine code, audio/HAL
pipelines, Hamlib integration, device/service profiles, and deployment scripts for
Debian Trixie on Raspberry Pi 3/4/5 (and Zero 2 W).

- Reverse-engineering notes, firmware analyses, Ghidra scripts, and brand-specific
  control-flow specifications live in the separate *private* workspace and are NOT
  checked in here.

Engineering documents: `docs/engineering.md`, `docs/profile_schema.md`, `docs/verification.md`.

Step-by-step roadmap: `ROADMAP.md`.

## Scope boundaries

- No firmware binaries (.hex/.gpr).
- No copyrighted vendor manuals.
- Per-brand machine behavior: YAML *profiles* keyed off the runtime-detected device class.
- Config file: `Software2/config/hampod.conf`.

## Build

Debian Trixie / ARM target with `gcc`, `make`, `libasound2-dev`, `libhamlib-dev`,
`piper`, `ffmpeg`. See `Documentation/scripts/install_hampod.sh` for the
canonical install path once it is ported here (and tested on a fresh image).
