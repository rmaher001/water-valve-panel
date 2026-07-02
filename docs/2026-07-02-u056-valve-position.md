# U056 Accelerometer → True Valve Position (resume notes)

**Status as of 2026-07-02:** grounded verification COMPLETE (per `grounded-planning` skill). Design NOT started. Resume with a brainstorming/design session in this repo, then spec → plan → implement.

## Goal

Mount the M5Stack U056 ACCEL unit on the EVC200 Bulldog's rotating valve handle and report **ground-truth open/closed position** on the panel. The valve reports nothing over Z-Wave — `switch.main_water_valve_robot` in HA is optimistic (flips instantly on command, ~10–18 s before the valve physically finishes).

## Dependencies & Grounding (all verified 2026-07-02)

| Dependency | Fact | Citation |
|---|---|---|
| U056 unit | ADXL345 3-axis accel, I2C addr **0x53**, ±16g @ 13-bit, 1.7–5V, HY2.0-4P Grove, 20cm cable included, LEGO-hole mounting | https://docs.m5stack.com/en/unit/accel |
| CoreS3 SE Port A | Grove = GND / 5V / **G2 (SDA)** / **G1 (SCL)** — separate from internal bus G12/G11 (PMU/touch/expander) | https://docs.m5stack.com/en/core/M5CoreS3%20SE |
| This firmware | `src/main.yaml` defines only `bus_internal` (GPIO12/11). **GPIO1/2 unused** → clean second `i2c:` bus for Port A | grep of `src/main.yaml` |
| ESPHome driver | **No ADXL345 in ESPHome core** (feature-requests#511 still open). Working external component: `github.com/jdillenburg/esphome` → `components/adxl345`. Exposes `accel_x/y/z` (m/s²), **`off_vertical` (deg from vertical)**, `jitter`; config: address, range, update_interval; multi-bus OK | https://github.com/jdillenburg/esphome/tree/main/components/adxl345 |
| EVC200 Bulldog | Bolts to valve body; arm swings the ball-valve handle 90° in ~18 s; manual clutch pin on motor underside | https://uploads-ssl.webflow.com/5b30ffa2f7f0bd2e6c05e950/5b35889b501b8e0c35b855d1_evc200-bulldog-valve-robot-user-s-guide-v4.pdf |

External-component caveat: single-maintainer repo — **pin a commit ref** in `external_components:` and read the driver code before trusting it.

## Physical constraints (answered 2026-07-02)

- **Handle geometry (ideal):** handle points UP (perpendicular to floor) when OPEN; rotates ~90° counter-clockwise to parallel-to-floor when CLOSED. Sweep is through a **vertical plane** → gravity vector swings the full ~9.8 m/s² between states.
  - `off_vertical` maps directly: **open ≈ 0°, closed ≈ 90°**; threshold ~45° = huge margin.
  - Bonus: mid-travel angle + `jitter` (motor vibration) can signal a third "actuating" state during the ~18 s sweep.
- **Cable:** a **2m Grove cable** is on hand. Long for I2C but workable: dedicated Port A bus, single device, ESPHome default 50 kHz clock. Watch device logs for I2C errors during bring-up — watch-item, not blocker.

## Sketch (unvalidated — design session decides)

```yaml
i2c:
  - id: bus_internal        # existing, GPIO12/11 — unchanged
  - id: bus_port_a
    sda: GPIO2
    scl: GPIO1
    frequency: 50kHz

external_components:
  - source: {type: git, url: https://github.com/jdillenburg/esphome, ref: <PINNED_COMMIT>}
    components: [adxl345]

adxl345:
  - id: valve_accel
    i2c_id: bus_port_a
    address: 0x53
    off_vertical: {name: "Valve Handle Angle"}
# + template binary_sensor: angle > 45° for N s → CLOSED, else OPEN
```

## Open design decisions (for the brainstorming session)

1. Where the true position surfaces: panel UI, HA entity, or both.
2. How it reconciles with the current optimistic `switch.main_water_valve_robot` display (e.g., replace state source vs. show "moving…" until tilt confirms vs. mismatch alert).
3. Debounce/threshold values and the optional "actuating" third state.
4. Physical mounting of the U056 on the handle (LEGO holes / zip tie / bracket) + cable routing panel → valve.

## Process expectations for the next session

- Deploy flow is in `CLAUDE.md` here (tag → bump the pinned ref in the deploy wrapper → deploy sync → dashboard flash). This feature likely needs **no wrapper change**.
- `rm -rf .esphome/packages` before flashing anything pulled via `github://` (stale-cache lesson, 2026-05-20).
- Validate the config before any flash; flash via the HA Build Dashboard.
