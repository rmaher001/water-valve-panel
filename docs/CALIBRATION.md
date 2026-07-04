# Installing and Calibrating the Valve-Position Sensor (ADXL345)

The panel derives the true valve position from an ADXL345 accelerometer mounted
on the valve handle. The handle angle is computed from the sensor's raw X/Y
axes (`atan2(-y, x)`), so the absolute mounting orientation does not matter —
only that the sensor rotates with the handle. Calibration is a two-point
procedure: read the angle at each end stop and bake the two values into the
firmware substitutions.

## 1. Hardware

- **Sensor:** ADXL345 breakout, I²C address `0x53`, wired to the CoreS3 Grove
  Port A (SDA = GPIO2, SCL = GPIO1, 5V, GND).
- **Mounting:** fix the sensor flat on the valve handle so that the sensor's
  X/Y plane is the handle's plane of rotation (i.e., the board face is parallel
  to the handle's swing). Any rotation *within* that plane is fine — the
  calibration absorbs it. Use a rigid mount; a wobbly sensor adds noise to the
  angle and can defeat the settle window.
- **Power note:** the Grove 5V rail on the CoreS3 is switched by the AW9523B
  expander (BUS_EN + BOOST_EN). The firmware enables both at boot and includes
  a one-shot powered retry (90 s after boot) because a cold boot can power the
  rail after the sensor probe has already failed. If the sensor is dead at
  first boot, wait about 3 minutes before concluding anything (see section 2).

## 2. First boot — verify the sensor is alive

Flash the firmware with the default substitutions, then check Home Assistant
(device page → Diagnostic):

- **Valve Accel X / Valve Accel Y** update and sum to roughly gravity
  (√(x²+y²+z²) ≈ 9.8 m/s²).
- **Valve Handle Angle** shows a steady value in degrees.

If these are `unavailable`, the sensor was not found: check wiring/address,
power-cycle, and wait out the retry cycle — the firmware does one powered
reboot retry, each leg waiting 90 s, so a definitive "still dead" takes about
3 minutes from boot. Until the sensor is alive, the panel
runs in fallback mode (mirrors the HA valve switch, shows
"sensor offline" caption).

## 3. Two-point calibration

1. **Drive the valve fully open** (from the panel, HA, or manually with the
   actuator declutched). Wait for it to hit the stop and settle (~5 s still).
2. Read **Valve Handle Angle** in HA → record as `open_angle`.
3. **Drive the valve fully closed.** Wait for the stop.
4. Read the angle again → record as `closed_angle`.

The two values typically differ by ~90° for a quarter-turn ball valve. Sign
and magnitude don't matter as long as the two bands (± `angle_tolerance`)
don't overlap.

## 4. Apply the calibration

Set the measured values as substitutions in your deploy wrapper YAML (they
override the package defaults in `src/main.yaml`), then re-flash:

```yaml
substitutions:
  open_angle: "4.0"      # your measured fully-open angle (deg)
  closed_angle: "86.4"   # your measured fully-closed angle (deg)
packages:
  water_valve_panel: github://rmaher001/water-valve-panel/src/main.yaml@<tag>
```

## 5. Verify

- With the valve at a stop, **Valve Position** reads `open` (or `closed`) and
  holds through at least 30 s of stillness.
- **Valve Position Problem** stays `OK`.
- Cycle the valve once end-to-end: position should pass through `moving` and
  land on the other stop; mid-travel must never read `open`/`closed`.

## Tunables (all substitutions, defaults in `src/main.yaml`)

| Substitution | Default | Meaning |
|---|---|---|
| `open_angle` | `4.0` | Handle angle at the fully-open stop (deg). **Site-specific — always calibrate.** |
| `closed_angle` | `86.4` | Handle angle at the fully-closed stop (deg). **Site-specific — always calibrate.** |
| `angle_tolerance` | `8.0` | ± window around each stop that counts as "at the stop". Widen if the stop angle drifts slightly between cycles; the two bands must never overlap. |
| `position_settle_ms` | `1000` | Angle must hold inside a band this long before the position is declared. |
| `confirm_timeout_ms` | `30000` | Tilt/HA disagreement older than this raises a fault. |
| `jitter_moving_threshold` | `0.0` | Inter-sample jitter delta (m/s²) that counts as "motor running". `0.0` disables jitter logic (default — calibrate on site before enabling). |
| `motor_start_timeout_ms` | `5000` | With jitter enabled: commanded but no vibration within this → fault. |
| `sensor_stale_ms` | `5000` | No angle sample for this long → sensor treated as offline (fallback). |
| `fallback_lockout_ms` | `25000` | Tap lockout while in sensor-offline fallback (covers valve travel time). |

## Optional: calibrating the jitter threshold

Jitter-based motor detection ships **disabled** (`jitter_moving_threshold: "0.0"`).
To enable it: watch the **Valve Jitter** diagnostic while the valve is idle
(note the sample-to-sample variation) and while it is running (variation is
markedly larger). Pick a threshold between the two and set it as a
substitution. Leave it at `0.0` if the margin is not clear-cut.
