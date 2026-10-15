# HAMPOD2027 Profile Schema (draft)

A profile is a file per supported device model. Field names are illustrative;
the authoritative schema is the JSON Schema that the `hampod-profile validate`
command enforces. Only profileable capabilities appear; there are no device-
specific branches in the engine (the engine is a generic interpreter).

```yaml
schema_version: 1
name: hampod profile name
device_type: radio           # radio | rotator | powermeter | ...
family: icom                 # informs the install-time default suggestion
# detection rules for setup mode:
detect:
  - kind: hamlib query
    result_contains: "<string>"
    weight: 10
  - kind: usb vidpid
    value: 0x0695/0x...
    weight: 5

# global rendering preferences for this operator (normally from hampod.conf):
# key_layout: 123 | 789  (global; same across devices)

bands: [ { id, low_hz, high_hz, step_hz } ]
modes: [ { id, letter, entry_keys, handlers } ]        # mode tables + tap/held tables
keymap:                                                # indexing like the original firmware
  mode_base:          { key0: action, ... }             # keys with {type: tap/held actions}
  mode_set:           { ... }
  mode_frequency:     { ... }
# action types: adjust, speak, commit, ladder, hamlib-call, switch-mode, modifier,
#   prompt, run-service-key

speech_vocab:
  mode_names: [ { key, text } ]
  digits: { 0: "zero", ... }
  units: { mhz: "megahertz", khz: "kilohertz" }
  frequencies_compose: true

radio:
  hamlib_model_id: 3073
  serial_baud_default: 19200       # overridden by Hamlib's driver default where available
  serial_baud_fallback: 9600
  poll:
    freq_hz: 10
    presence_probe: "get_vfo_a"
  timeouts_ms:
    connect: 2000
    freq_read: 50
    write: 100
  resets: { reset_init, reinit }  # hamlib call sequences with deadlines
  sequences:                        # frequency/band/mode/band-switch maps
    freq_set: "rigctl set_freq {hz}"
  ...

timeouts_ms:                       # ladder keys, key-debounce, hold-detection
  tap_hold_ms: 600
  beep_ms: ...

ladders:                           # configuration-mode parameter ladders
  - index: 0
    name: mode selector
    max: 21
    on_apply: [ ... ]

services:
  gate_byte: 0x25-equivalent        # N/A on the Pi; retained for behavioral parity
  service_keys: [ ... ]
  fast_announcement: on-keypress interrupt

# non-radio types add extra sections (meter.readings, rotator.tell_azimuth,
# hamlib-capability string).
```

### key_layout semantics

- A Pi ` key_layout = 123 | 789 ` lives in `hampod.conf` or the operator's
  profile; the engine maps the physical numeric label (`event.code`) → key letter
  before profile lookup.

### Setup mode

- Trigger: a USB port in `hampod.conf` no longer matches.
- Sequence: enumerate; if Hamlib guess is high-confidence, confirm;
  otherwise a **narrowed selection** (class → brand → model) excludes models
  without published profiles. Persist the mapping to `hampod.conf` on completion.

### Barge-in

- Trigger: any valid key event.
- Action: sensor voice cancel (< 20-50 ms target); queue next announcement;
  drop pending audio, never `snd_pcm_drain`.

### Quality and coverage

- Unmatched Hamlib-capable models are served by a generic profile with
  `hamlib-capable` behavior and limited announcements; behavior degrades
  gracefully (notifications "model not fully supported" etc.) and remains
  measurable (counts of missed behavior in `--sim` tests).

(Authoritative master spec: kept in the private engineering workspace;
this schema is the public, implementation-facing view.)
