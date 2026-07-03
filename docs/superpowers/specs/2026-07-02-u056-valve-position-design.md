# U056 Accelerometer → True Valve Position — Design Spec

**Date:** 2026-07-02
**Status:** Approved 2026-07-02 (design session, section-by-section)
**Grounding:** `docs/2026-07-02-u056-valve-position.md` (all external dependencies verified with citations)

## Problem

`switch.main_water_valve_robot` is optimistic: it flips ~200 ms after a command, while the
EVC200 physically takes ~10–18 s to swing the ball-valve handle 90°. The valve reports nothing
over Z-Wave. Today the panel covers this with a blind 20 s lockout and displays the commanded
outcome, not reality. There is no way to know the valve actually reached its stop — including
the failure that matters most: **commanded closed but not fully closed**.

## Goal

Mount an M5Stack U056 ACCEL (ADXL345) on the EVC200's rotating handle. The gravity vector gives
ground-truth handle angle. The panel and HA report **provable fully-open / fully-closed**
position, transit progress, and faults.

## Decisions (from the design session)

1. **Surface position on both the panel and HA entities.**
2. **Tilt is the source of truth** for the panel's status display, with an explicit motion
   (transit) state. The optimistic switch state is used to infer intent/direction, and as the
   display source only in sensor-offline fallback.
3. **Faults surface on the panel AND as an HA `problem` binary_sensor** (phone notification via
   an HA-side automation, out of scope here).
4. **Mounting method decided at install; firmware ships tunable calibration substitutions.**
5. **Architecture A — all classification logic on-device.** HA is told the position; it never
   decides it. Rationale: the sensor is wired to the panel; position must survive HA outages
   and restarts. (HA-side derivation was explicitly considered and rejected.)
6. **Fully-seated guarantee:** OPEN/CLOSED are only declared inside tight windows around the
   two calibrated mechanical stops. "Roughly closed" (e.g., stuck at 62°) is a fault, never
   CLOSED.

## Hardware & bus

- New I2C bus `bus_port_a`: **GPIO2 = SDA, GPIO1 = SCL, 50 kHz** (CoreS3 SE Port A Grove).
  Physically separate from `bus_internal` (GPIO12/11 — PMU/touch/expander), so a fault on the
  external cable cannot take down the display stack.
- U056 (ADXL345) at address **0x53**, connected via the on-hand **2 m Grove cable**. Long for
  I2C but acceptable: single device, dedicated bus, 50 kHz. Bring-up watches logs for I2C
  errors (validation step 3).
- Driver: external component `adxl345` from `github.com/jdillenburg/esphome`, **pinned to a
  commit SHA** (single-maintainer repo; driver code gets read during implementation before
  trust). Two sensors configured: `off_vertical` (degrees from vertical — position truth) and
  `jitter` (vibration — advisory motor-activity signal; the valve observably vibrates while
  turning). `accel_x/y/z` are omitted.

## Sensing & calibration

`off_vertical` polled every **250 ms**, `median` filter (window 5) against motor vibration.

Physical geometry (verified on site): handle points **up ≈ 0° when OPEN**, sweeps ~90°
through a vertical plane to **parallel-to-floor ≈ 90° when CLOSED**.

Position classification — tight windows around calibrated stops (NOT wide bands):

| Substitution | Default | Meaning |
|---|---|---|
| `open_angle` | `0.0` | measured angle at the fully-open mechanical stop |
| `closed_angle` | `90.0` | measured angle at the fully-closed mechanical stop |
| `angle_tolerance` | `8.0` | ± window (deg) around each stop |
| `position_settle_time` | `1s` | window must hold this long before position is declared |
| `confirm_timeout` | `30s` | tilt/switch disagreement older than this → fault (backstop watchdog) |
| `jitter_moving_threshold` | `0` (disabled) | jitter above this = motor running; **0 disables all jitter logic** — calibrated at bring-up, units confirmed from driver source |
| `motor_start_timeout` | `5s` | commanded but no jitter above threshold within this → early FAULT ("motor didn't start"); only active when jitter threshold > 0 |

- **OPEN** = |angle − open_angle| ≤ tolerance, held for settle time
- **CLOSED** = |angle − closed_angle| ≤ tolerance, held for settle time
- **MID** = everything else (including "motor quit at 70°")
- **UNKNOWN** = sensor unreadable (NAN) or not yet settled after boot

**Calibration procedure (post-mount):** run the valve to each stop, read the diagnostic angle
entity in HA, set `open_angle`/`closed_angle` to the measured values, re-flash. The mount can
sit at any skew — calibration absorbs it. Tolerance can be tightened after observing
repeatability across a few cycles.

**Jitter calibration (same session):** observe the diagnostic jitter entity at rest and
during a commanded cycle; set `jitter_moving_threshold` between idle noise and motor
vibration. Until then the threshold stays `0` and all jitter logic is inert — the firmware
behaves exactly per the base (angle-only) design. The observed motor vibration is also why
the median filter + settle time exist: readings are noisy during travel and clean at the
stops.

## State machine (on-device)

Inputs: tilt position (OPEN/CLOSED/MID/UNKNOWN), HA switch state (optimistic), API
connectivity, button taps.

Core rule — panel state derives from tilt/switch **agreement and disagreement age**:

| Tilt | Switch | Disagreement age | Panel state |
|---|---|---|---|
| at a stop | same stop | — | **IDLE_OPEN / IDLE_CLOSED** |
| MID or wrong stop | target | < `confirm_timeout` | **MOVING** toward switch target |
| MID | target | ≥ `confirm_timeout` | **FAULT-STUCK** ("NOT FULLY OPEN/CLOSED") |
| at a stop | the other stop | ≥ `confirm_timeout` | **MISMATCH** (manual clutch-pin move) |
| UNKNOWN | any | — | **FALLBACK** (today's optimistic logic) |

- **FAULT-STUCK** names the missed target from the switch's direction: commanded off + tilt
  MID → "NOT FULLY CLOSED"; commanded on + tilt MID → "NOT FULLY OPEN".
- **Jitter (advisory, active only when `jitter_moving_threshold` > 0)** accelerates fault
  detection; it never overrides angle truth:
  - *Early "didn't start" fault:* command issued but no jitter within `motor_start_timeout`
    → FAULT immediately (caption "motor didn't start — check valve") instead of waiting the
    full `confirm_timeout`.
  - *Early "stuck" fault:* during MOVING, jitter ceases while angle sits MID for
    2 × `position_settle_time` → FAULT-STUCK early (motor quit mid-travel).
  - `confirm_timeout` remains the unconditional backstop either way; if the jitter signal
    proves unreliable, setting the threshold back to 0 restores the approved base design.
- **MISMATCH** shows tilt truth (the valve IS fully seated — HA is wrong), caption
  "HA switch disagrees — moved manually?", button offers the action implied by physical
  reality (valve closed → OPEN button).
- **Problem sensor** is on in FAULT-STUCK and MISMATCH; clears when tilt and switch agree.
- Fault states exit on renewed agreement or on a retry tap (→ MOVING with a fresh timer).
- **HA DISCONNECTED is an overlay, not a table row:** the tilt-side state is computed as
  above; disconnection re-skins the result (button greyed, flow dashes, caption) without
  changing position logic. While disconnected the switch input is unavailable, so
  disagreement/fault tracking pauses — position is tilt-only. Tilt UNKNOWN *and*
  disconnected together render today's grey "last known state" screen.

**Button gating:** locked during MOVING (unlocks on tilt confirmation — replaces the fixed
20 s timer); re-armed in FAULT-STUCK and MISMATCH for retry; gated off when disconnected or
state is unknown. The tap handler still sets `actuating` immediately to prevent the rapid-tap
race (fix 027f277); it is cleared by confirmation, by fault entry, or — in FALLBACK mode
only — by the legacy 20 s timer.

## Panel UI (six mockup states, plus MISMATCH as an idle variant — approved 2026-07-02)

Mockups reviewed and approved in the 2026-07-02 design session. Summary:

1. **OPEN (idle)** — as today (red CLOSE button, green status), now ground truth.
2. **MOVING** — button grey "CLOSING…/OPENING…", status row **amber** "Water Valve ·
   CLOSING…", caption shows live angle: "waiting for valve to confirm — 62°".
3. **CLOSED (confirmed)** — as today (green OPEN button, red status); only when fully seated.
4. **FAULT-STUCK** — status red "⚠ NOT FULLY CLOSED" (or OPEN), amber caption
   "stopped at 62° — tap CLOSE to retry", button re-armed in retry color. Direction-agnostic.
   The jitter early-fault variants reuse this layout with the caption "motor didn't start —
   check valve" (no angle to report — the handle never left the stop).
5. **SENSOR OFFLINE (fallback)** — today's optimistic UI + caption "position sensor offline —
   showing HA switch state". Taps work via legacy lockout.
6. **HA DISCONNECTED (upgraded)** — status row stays live and full-color from tilt (today it
   greys out as "last known state"); button greyed (commands need HA); flow figures dashed.
   Caption: "can't reach HA — position live from tilt sensor".
- **MISMATCH** renders as the tilt-truth idle state + disagree caption + problem colors on the
  caption line (amber), button re-armed.

New color: **amber `0xb9770e`** for transit/warning captions. Status-row amber for MOVING.

**Font glyph additions required:** `…` in `text_18`; `°` and `?` in `text_14`; MDI alert
(U+F0026) in `mdi_34` for the ⚠ status icon slot. Button font `text_28` already has `…`.

## HA entities (native API, no HA config changes)

| Entity | Type | Values | Notes |
|---|---|---|---|
| Valve Handle Angle | `sensor` (diagnostic category) | degrees | delta-filter throttled to keep HA history sane |
| Valve Jitter | `sensor` (diagnostic category) | driver units | for threshold calibration + motor-activity history; delta-throttled |
| Valve Position | `text_sensor` | `open` / `closed` / `mid` / `unknown` | physical truth only — deliberately NOT the panel state-machine state; HA derives "moving" etc. from position + switch + problem |
| Valve Position Problem | `binary_sensor`, `device_class: problem` | on/off | on = FAULT-STUCK or MISMATCH persisting |

## Error handling

- **Sensor NAN / I2C failure** → position UNKNOWN → entire state machine disengages; panel
  runs exactly today's optimistic logic plus the offline caption. Ground truth degrades to the
  status quo, never below it.
- **HA disconnected** → position keeps working (it's local); only commanding and flow data
  degrade.
- **Boot** → UNKNOWN until first settled reading (~1–2 s), then live.

## Non-goals / future work (explicitly out of scope)

- HA notification automation for the problem sensor (HA-side, later).
- Droplet flow-rate cross-check ("closed but flow nonzero") — HA-side automation enabled by
  these entities, later.
- 3D-printed mounting bracket (mount method decided at install).

## Validation plan (one production device — no bench unit exists)

1. `/esphome-check` locally at every step (no device needed).
2. **Deploy tilt-enabled firmware with no sensor attached.** By construction this runs
   FALLBACK (= today's behavior + offline caption + four unknown/unavailable entities). Normal pipeline
   per `CLAUDE.md`: tag → bump the pinned ref in the deploy wrapper → deploy sync → manual
   Install click on the Build Dashboard. No wrapper change expected.
3. **Plug U056 into Port A** (2 m cable), reboot panel, check logs for clean I2C reads at 0x53.
4. **Hand-held walk-through:** rotate the unmounted U056 0→90°; watch angle entity + panel walk
   open → mid → closed. Cosmetic MISMATCH/MOVING states expected; valve untouched. Tune
   filters/settle here.
5. **Mount on handle, calibrate** (procedure above), re-flash with measured angles.
   Same session: run a commanded cycle, read the jitter entity at rest vs. in motion, set
   `jitter_moving_threshold` (jitter logic stays inert until this step).
6. **Failure drills:** unplug cable (→ FALLBACK); clutch-pin manual move (→ MISMATCH +
   problem sensor); stuck-travel fault via temporarily shortened `confirm_timeout` (not by
   fighting the motor).

Every flash is OTA to the production panel via an explicit manual Install click. Steps 2–4
cannot affect valve control by construction.

## Implementation notes

- `rm -rf .esphome/packages` before any flash pulling `github://` refs (stale-cache lesson,
  2026-05-20).
- Pin the `jdillenburg/esphome` external component to a commit SHA; read the driver source
  before first flash.
- Deploy flow per `CLAUDE.md` (this repo) — tag `vX.Y.Z`, bump the pinned ref in the deploy
  wrapper.
- `local.yaml` (gitignored, local validation harness) is used only by `/esphome-check`.
- The EVC200 manual cites up to ~18 s of travel; the legacy FALLBACK-mode 20 s lockout keeps
  only ~2 s of margin. Revisit that constant (e.g., 25 s) during implementation.
