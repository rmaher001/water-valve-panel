# U056 True Valve Position Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ADXL345 tilt sensing on the valve handle gives the panel and HA provable fully-open / fully-closed position, transit state, and fault detection — replacing the blind 20 s lockout.

**Architecture:** All classification on-device (spec: `docs/superpowers/specs/2026-07-02-u056-valve-position-design.md`). A second I2C bus (Port A) hosts the ADXL345 via a pinned external component. Angle → calibrated stop windows → tilt position; tilt/switch disagreement age drives MOVING/FAULT/MISMATCH; jitter inter-sample delta is an advisory motor-activity signal, shipped disabled. HA receives four entities; the panel UI gains amber transit + fault treatments.

**Tech Stack:** ESPHome (esp-idf, ESP32-S3), LVGL, external component `adxl345` pinned to `jdillenburg/esphome@464fd3402e14a7e4c39f2b72df5a1a001df662ed`.

---

## Driver facts this plan is built on (verified from source at the pinned SHA)

- Config schema: `adxl345:` is a hub list (`MULTI_CONF`); options `range` (default `2G` — correct for gravity sensing), `update_interval` (polling schema, default 1 s), `i2c_id`, `address` (default 0x53), and per-sensor blocks `off_vertical` (unit °), `jitter` (unit m/s²), `accel_x/y/z`.
- `off_vertical` publishes `max(|pitch|, |roll|)` where pitch/roll come from `atan` of axis ratios. It is **0° when the board lies flat, ~90° when on edge**, and is NOT guaranteed monotonic across arbitrary mounts. Consequence: the two valve stops MUST be verified to give well-separated readings (≥ ~40° apart) in the hand-held test **before** mounting; if not, re-orient the module.
- `jitter` publishes the instantaneous magnitude `|x|+|y|+|z|` — **≈ 9.8–17 m/s² at rest** depending on orientation. Vibration appears as sample-to-sample fluctuation, so motor activity = `|jitter[n] − jitter[n−1]| ≥ jitter_moving_threshold`. The threshold is a **delta**, not an absolute level.
- On I2C read failure the driver logs a warning and publishes **nothing** (no NAN). If the device is absent at boot, `setup()` calls `mark_failed()` and the component never polls. Consequences: (a) sensor-offline detection is a **staleness watchdog** (no angle sample for `sensor_stale_ms`); (b) plugging the U056 in after boot requires a panel reboot (validation step 3 already reboots).

## Project conventions (read before Task 1)

- The only source file is `src/main.yaml` (~570 lines, single-file repo convention — do not split it).
- **Never call `esphome` directly and never call any venv binary.** Config validation = the `/esphome-check` skill against `local.yaml` (it routes through the project's pinned-toolchain wrapper). Expected success output ends with `Configuration is valid!`.
- Run `rm -rf .esphome/packages` before any validate/compile after changing `external_components` (stale-cache lesson).
- No test harness exists for ESPHome YAML. The RED/GREEN loop for each task is: edit → `/esphome-check local.yaml` → expect `Configuration is valid!` → commit. Lambda C++ errors surface at first compile (Task 6, staged deploy) — which is deliberately the zero-behavior-change sensor-less deploy.
- Flashing the production panel ALWAYS requires the owner's explicit go (dashboard Install click). No task in this plan flashes anything.
- Commits go to `main` (repo convention for this project). No AI references in commit messages.
- This is a PUBLIC repo: never commit `local.yaml`, `secrets.yaml`, personal paths, or private-infra names. `git add` only the files each task names.

## File structure

- Modify: `src/main.yaml` — all firmware changes (substitutions, bus, sensors, globals, scripts, LVGL, fonts).
- No new files. No wrapper change expected in the private deploy monorepo except the version-ref bump at ship time.

---

### Task 1: Fonts + substitutions groundwork

No behavior change: adds the glyphs the new UI strings need and the calibration substitutions with safe defaults.

**Files:**
- Modify: `src/main.yaml` (font blocks ~lines 127–157; new `substitutions:` block above `esphome:`)

- [ ] **Step 1: Add the substitutions block**

At the very top of `src/main.yaml` (before the `esphome:` block), insert:

```yaml
# ----- U056 valve-position calibration (override in the deploy wrapper after install) -----
substitutions:
  open_angle: "0.0"              # measured off_vertical at fully-open stop (deg)
  closed_angle: "90.0"           # measured off_vertical at fully-closed stop (deg)
  angle_tolerance: "8.0"         # +/- window around each stop (deg)
  position_settle_ms: "1000"     # band must hold this long before position is declared
  confirm_timeout_ms: "30000"    # disagreement older than this -> fault (backstop)
  jitter_moving_threshold: "0.0" # inter-sample jitter delta (m/s^2); 0 = jitter logic disabled
  motor_start_timeout_ms: "5000" # commanded but no jitter delta within this -> early fault
  sensor_stale_ms: "5000"        # no angle sample for this long -> tilt UNKNOWN
  fallback_lockout_ms: "25000"   # legacy tap lockout in sensor-offline fallback (was 20s; manual cites ~18s travel)
```

(Spec names `position_settle_time: 1s` etc.; the `_ms` integer forms are the same values in lambda-safe units.)

- [ ] **Step 2: Add the missing glyphs**

In the `font:` section:
- `mdi_34` glyphs list: append `"\U000F0026"` (mdi alert) → `glyphs: ["\U000F1068", "\U000F1067", "\U000F0026"]`
- `text_14` glyphs string: append `°?` → `glyphs: " abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789·:.,—●'°?"`
- `text_18` glyphs string: append `…` → `glyphs: " abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ·…"`

- [ ] **Step 3: Validate**

Invoke the `/esphome-check` skill on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 4: Commit**

```bash
git add src/main.yaml
git commit -m "Add valve-position substitutions and UI font glyphs"
```

---

### Task 2: Port A bus, pinned ADXL345 component, sensors, HA diagnostics

Adds hardware support and the two diagnostic HA entities. Still no behavior change: nothing consumes the readings yet.

**Files:**
- Modify: `src/main.yaml` (`external_components` ~line 36; `i2c:` ~line 41; new `adxl345:` block after `aw9523b:`; `sensor:` section ~line 201)

- [ ] **Step 1: Pin the external component**

Extend the existing `external_components:` list:

```yaml
external_components:
  - source: github://m5stack/esphome-yaml/components
    components: [axp2101, aw88298, aw9523b]
  - source:
      type: git
      url: https://github.com/jdillenburg/esphome
      ref: 464fd3402e14a7e4c39f2b72df5a1a001df662ed
    components: [adxl345]
```

- [ ] **Step 2: Add the Port A bus**

Extend the `i2c:` list:

```yaml
i2c:
  - id: bus_internal
    sda: GPIO12
    scl: GPIO11
  - id: bus_port_a
    sda: GPIO2
    scl: GPIO1
    frequency: 50kHz
    scan: false
```

(`scan: false` — the 2 m cable makes boot-time bus scans noisy; the driver probes 0x53 itself and logs found/not-found.)

- [ ] **Step 3: Add the ADXL345 hub with raw internal sensors**

After the `aw9523b:` block:

```yaml
# ----- U056 ADXL345 on Port A: valve-handle tilt + vibration -----
adxl345:
  - id: valve_accel
    i2c_id: bus_port_a
    address: 0x53
    range: 2G
    update_interval: 250ms
    off_vertical:
      id: valve_angle_raw
      internal: true
      filters:
        - median:
            window_size: 5
            send_every: 1
            send_first_at: 1
      on_value:
        then:
          # Task 3 Step 4 replaces this stub with a chained eval_valve_state
          # call. That script does not exist yet — do NOT reference it here.
          - lambda: |-
              id(last_angle_ms) = millis();
    jitter:
      id: valve_jitter_raw
      internal: true
      on_value:
        then:
          - lambda: |-
              if (!isnan(id(jitter_prev)) && ${jitter_moving_threshold} > 0.0f) {
                if (fabsf(x - id(jitter_prev)) >= ${jitter_moving_threshold})
                  id(last_motor_activity) = millis();
              }
              id(jitter_prev) = x;
```

(The globals these lambdas touch are added in Step 4 of this task, so the config validates standalone. The `off_vertical` `on_value` above is deliberately a stub — Task 3 defines `eval_valve_state` and un-stubs it.)

- [ ] **Step 4: Add the globals these sensors need**

Append to the existing `globals:` list:

```yaml
  - id: last_angle_ms
    type: uint32_t
    restore_value: no
    initial_value: '0'
  - id: jitter_prev
    type: float
    restore_value: no
    initial_value: 'NAN'
  - id: last_motor_activity
    type: uint32_t
    restore_value: no
    initial_value: '0'
```

- [ ] **Step 5: Add the HA diagnostic copy sensors**

Append to the existing `sensor:` list (after `droplet_today`):

```yaml
  - platform: copy
    source_id: valve_angle_raw
    name: "Valve Handle Angle"
    entity_category: diagnostic
    filters:
      - delta: 1.0
  - platform: copy
    source_id: valve_jitter_raw
    name: "Valve Jitter"
    entity_category: diagnostic
    filters:
      - delta: 0.5
```

- [ ] **Step 6: Purge package cache, validate**

```bash
rm -rf .esphome/packages
```
Invoke `/esphome-check` on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 7: Commit**

```bash
git add src/main.yaml
git commit -m "Add Port A bus, pinned ADXL345 component, tilt/jitter sensors"
```

---

### Task 3: Position classification + Valve Position entity

Turns raw angle into a settled tilt position (open/closed/mid/unknown) with the staleness watchdog, published to HA.

**Files:**
- Modify: `src/main.yaml` (`globals:`, `script:`, `text_sensor:`, `interval:`; un-stub the Task 2 `off_vertical` `on_value`)

- [ ] **Step 1: Add classification globals**

Append to `globals:`:

```yaml
  - id: tilt_position          # -1 boot sentinel, 0 unknown, 1 open, 2 closed, 3 mid
    type: int
    restore_value: no
    initial_value: '-1'
  - id: band_candidate
    type: int
    restore_value: no
    initial_value: '0'
  - id: band_since
    type: uint32_t
    restore_value: no
    initial_value: '0'
```

- [ ] **Step 2: Add the Valve Position text sensor**

Append to the existing `text_sensor:` list:

```yaml
  - platform: template
    id: valve_position_ts
    name: "Valve Position"
    update_interval: never
```

- [ ] **Step 3: Add the eval script (classification half)**

Add to `script:`:

```yaml
  - id: eval_valve_state
    mode: single
    then:
      - lambda: |-
          const uint32_t now = millis();

          // ---- tilt position from calibrated stop windows + settle ----
          int pos = 0;  // unknown
          bool stale = (id(last_angle_ms) == 0) ||
                       (now - id(last_angle_ms) > (uint32_t) ${sensor_stale_ms});
          if (!stale) {
            float a = id(valve_angle_raw).state;
            int band = 3;  // mid
            if (fabsf(a - ${open_angle}) <= ${angle_tolerance}) band = 1;
            else if (fabsf(a - ${closed_angle}) <= ${angle_tolerance}) band = 2;
            if (band != id(band_candidate)) {
              id(band_candidate) = band;
              id(band_since) = now;
            }
            if (now - id(band_since) >= (uint32_t) ${position_settle_ms})
              pos = id(band_candidate);
            else
              pos = (id(tilt_position) > 0) ? id(tilt_position) : 0;
          }
          if (pos != id(tilt_position)) {
            id(tilt_position) = pos;
            const char *s = pos == 1 ? "open" : pos == 2 ? "closed"
                          : pos == 3 ? "mid" : "unknown";
            id(valve_position_ts).publish_state(s);
          }
      - script.execute: render_state
```

- [ ] **Step 4: Un-stub the angle on_value**

Replace the Task 2 stub (including its two comment lines) with the final form:

```yaml
      on_value:
        then:
          - lambda: |-
              id(last_angle_ms) = millis();
          - script.execute: eval_valve_state
```

- [ ] **Step 5: Re-point the 5 s interval at eval**

In `interval:`, change the first entry from `render_state` to a 1 s cadence driving eval (render is chained from eval; timers below need ~1 s resolution):

```yaml
interval:
  - interval: 1s
    then:
      - script.execute: eval_valve_state
```

(Leave the 30 s time-label interval and the 800 ms + 50 ms lambdas untouched.)

- [ ] **Step 6: Validate**

Invoke `/esphome-check` on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 7: Commit**

```bash
git add src/main.yaml
git commit -m "Classify tilt position with settle + staleness; expose Valve Position"
```

---

### Task 4: Disagreement state machine, faults, problem entity

Adds the MOVING/FAULT/MISMATCH logic and the HA problem sensor. UI still renders the old way (Task 5 consumes these globals).

**Files:**
- Modify: `src/main.yaml` (`globals:`, `binary_sensor:` (new top-level section), extend `eval_valve_state`)

- [ ] **Step 1: Add state-machine globals**

Append to `globals:`:

```yaml
  - id: disagree_since         # 0 = tilt and switch agree (or tracking paused)
    type: uint32_t
    restore_value: no
    initial_value: '0'
  - id: expect_motion_since    # set on tap / switch-origin change; 0 = none
    type: uint32_t
    restore_value: no
    initial_value: '0'
  - id: fault_kind             # 0 none, 1 stuck, 2 mismatch, 3 motor-didn't-start
    type: int
    restore_value: no
    initial_value: '0'
```

- [ ] **Step 2: Add the problem binary sensor**

New top-level section (this file has none yet):

```yaml
binary_sensor:
  - platform: template
    id: valve_problem
    name: "Valve Position Problem"
    device_class: problem
```

- [ ] **Step 3: Extend eval_valve_state with the state machine**

Insert into the `eval_valve_state` lambda, after the position-classification block and before the closing of the lambda (i.e., before the `- script.execute: render_state` line — one lambda, two halves):

```cpp
          // ---- tilt/switch agreement, motion, faults ----
          bool connected = id(ha_api).is_connected();
          std::string vs = connected ? id(valve_state).state : std::string("");
          bool sw_known = connected && (vs == "on" || vs == "off");
          int sw_pos = (vs == "on") ? 1 : 2;

          int fk = id(fault_kind);
          if (!sw_known || pos == 0) {
            // Disconnected or sensor-offline fallback: pause tracking.
            id(disagree_since) = 0;
            if (pos == 0) fk = 0;   // no tilt truth -> no tilt-based faults
          } else if (pos == sw_pos) {
            // Agreement: confirmed. Clear everything, unlock the button.
            id(disagree_since) = 0;
            id(expect_motion_since) = 0;
            fk = 0;
            if (id(actuating)) id(actuating) = false;
          } else {
            if (id(disagree_since) == 0) id(disagree_since) = now;
            uint32_t age = now - id(disagree_since);
            bool jitter_on = ${jitter_moving_threshold} > 0.0f;
            if (fk == 0) {
              if (age >= (uint32_t) ${confirm_timeout_ms}) {
                fk = (pos == 3) ? 1 : 2;          // stuck mid-travel vs seated-at-wrong-stop
              } else if (jitter_on && id(expect_motion_since) != 0 &&
                         now - id(expect_motion_since) >= (uint32_t) ${motor_start_timeout_ms} &&
                         id(last_motor_activity) < id(expect_motion_since)) {
                fk = 3;                            // commanded, motor never started
              } else if (jitter_on && pos == 3 &&
                         id(last_motor_activity) != 0 &&
                         now - id(last_motor_activity) >= 2 * (uint32_t) ${position_settle_ms}) {
                fk = 1;                            // was moving, quit mid-travel
              }
            }
          }
          if (fk != id(fault_kind)) {
            id(fault_kind) = fk;
            if (fk != 0 && id(actuating)) id(actuating) = false;  // re-arm button for retry
          }
          bool problem = (fk != 0);
          if (!id(valve_problem).has_state() || id(valve_problem).state != problem)
            id(valve_problem).publish_state(problem);
```

- [ ] **Step 4: Mark switch-origin commands as expected motion**

**Replace** the existing `text_sensor:` `valve_state` `on_value` `then:` block (the old block is a lone `render_state` call plus a NOTE comment about not clearing `actuating` — both are superseded; eval now owns the `actuating` lifecycle and chains into render):

```yaml
    on_value:
      then:
        - lambda: |-
            // A switch flip means a command was issued somewhere (panel tap
            // already set expect_motion_since; HA/automation flips set it here).
            if (id(tilt_position) > 0 && id(expect_motion_since) == 0) {
              std::string v = x;
              int sw = (v == "on") ? 1 : (v == "off") ? 2 : 0;
              if (sw != 0 && sw != id(tilt_position))
                id(expect_motion_since) = millis();
            }
        - script.execute: eval_valve_state
```

(Replace the old `- script.execute: render_state` line — eval chains into render.)

- [ ] **Step 5: Validate**

Invoke `/esphome-check` on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 6: Commit**

```bash
git add src/main.yaml
git commit -m "Add tilt/switch disagreement state machine, faults, problem entity"
```

---

### Task 5: UI — render the new states, rewire the button

The visible half: amber MOVING, fault treatments, MISMATCH, sensor-offline caption, live-position-while-disconnected, confirmation-driven button lockout.

**Files:**
- Modify: `src/main.yaml` (`render_state` script; `action_btn` `on_short_click`)

- [ ] **Step 1: Replace the render_state lambda**

Replace the entire `render_state` lambda body with:

```cpp
          const char* OPEN_GLYPH   = "\xF3\xB1\x81\xA8";  // U+F1068 valve-open
          const char* CLOSED_GLYPH = "\xF3\xB1\x81\xA7";  // U+F1067 valve-closed
          const char* ALERT_GLYPH  = "\xF3\xB0\x80\xA6";  // U+F0026 alert

          const lv_color_t GREEN = lv_color_hex(0x1d8348);
          const lv_color_t RED   = lv_color_hex(0xb62324);
          const lv_color_t AMBER = lv_color_hex(0xb9770e);
          const lv_color_t GREY  = lv_color_hex(0x9a9a9f);
          const lv_color_t CAPGREY = lv_color_hex(0x7a7a82);

          bool connected = id(ha_api).is_connected();
          std::string vs = connected ? id(valve_state).state : std::string("");
          bool sw_known = connected && (vs == "on" || vs == "off");
          bool sw_open = (vs == "on");
          int pos = id(tilt_position);   // <=0 unknown, 1 open, 2 closed, 3 mid
          int fk = id(fault_kind);

          // ---- helpers ----
          auto btn_active = [&](bool red_close) {
            lv_obj_set_style_bg_color(id(action_btn), lv_color_hex(red_close ? 0xc0392b : 0x2ea14a), 0);
            lv_obj_set_style_bg_grad_color(id(action_btn), lv_color_hex(red_close ? 0x962419 : 0x1c6a31), 0);
            lv_obj_set_style_shadow_opa(id(action_btn), 97, 0);
            lv_label_set_text(id(action_icon), red_close ? CLOSED_GLYPH : OPEN_GLYPH);
            lv_label_set_text(id(action_text), red_close ? "CLOSE" : "OPEN");
          };
          auto btn_grey = [&](const char* icon, const char* txt) {
            lv_obj_set_style_bg_color(id(action_btn), lv_color_hex(0xc1c1c5), 0);
            lv_obj_set_style_bg_grad_color(id(action_btn), lv_color_hex(0xc1c1c5), 0);
            lv_obj_set_style_shadow_opa(id(action_btn), 0, 0);
            lv_label_set_text(id(action_icon), icon);
            lv_label_set_text(id(action_text), txt);
          };
          auto status_set = [&](const char* icon, lv_color_t icol, const char* txt, lv_color_t tcol) {
            lv_label_set_text(id(status_icon), icon);
            lv_obj_set_style_text_color(id(status_icon), icol, 0);
            lv_label_set_text(id(status_text), txt);
            lv_obj_set_style_text_color(id(status_text), tcol, 0);
          };
          auto caption = [&](const char* txt, lv_color_t col) {
            lv_label_set_text(id(last_known_label), txt);
            lv_obj_set_style_text_color(id(last_known_label), col, 0);
            lv_obj_clear_flag(id(last_known_label), LV_OBJ_FLAG_HIDDEN);
          };
          auto caption_off = [&]() {
            lv_obj_add_flag(id(last_known_label), LV_OBJ_FLAG_HIDDEN);
          };
          auto flow_row = [&](bool live) {
            if (!live) {
              lv_label_set_text(id(flow_rate_label), "flow: —");
              lv_label_set_text(id(flow_today_label), "today: —");
              lv_obj_set_width(id(flow_bar_fill), 0);
              return;
            }
            float rate = id(droplet_flow).has_state() ? id(droplet_flow).state : 0.0f;
            if (isnan(rate)) rate = 0.0f;
            float today = id(droplet_today).has_state() ? id(droplet_today).state : 0.0f;
            if (isnan(today)) today = 0.0f;
            char rate_buf[24], today_buf[24];
            snprintf(rate_buf, sizeof(rate_buf), "flow: %.1f gpm", rate);
            snprintf(today_buf, sizeof(today_buf), "today: %.0f gal", today);
            lv_label_set_text(id(flow_rate_label), rate_buf);
            lv_label_set_text(id(flow_today_label), today_buf);
            float scale_lin = rate / 15.0f;
            if (scale_lin > 1.0f) scale_lin = 1.0f;
            int w = (rate <= 0.05f) ? 0 : (int)(sqrtf(scale_lin) * 290);
            lv_obj_set_width(id(flow_bar_fill), w);
          };
          auto status_idle = [&](bool open) {
            status_set(open ? OPEN_GLYPH : CLOSED_GLYPH, open ? GREEN : RED,
                       open ? "Water Valve · OPEN" : "Water Valve · CLOSED",
                       open ? GREEN : RED);
          };

          // ---- top bar ----
          lv_label_set_text(id(conn_label), connected ? "connected" : "disconnected");
          lv_obj_set_style_text_color(id(conn_dot), connected ? GREEN : lv_color_hex(0xb62324), 0);

          // ================= DISCONNECTED =================
          if (!connected) {
            btn_grey(sw_open ? CLOSED_GLYPH : OPEN_GLYPH, sw_open ? "CLOSE" : "OPEN");
            if (pos == 1 || pos == 2) {
              // Upgraded: live position from tilt, full color.
              status_idle(pos == 1);
              caption("can't reach HA — position live from tilt sensor", CAPGREY);
            } else {
              // Legacy grey last-known (tilt unknown or mid).
              lv_obj_set_style_text_color(id(status_icon), GREY, 0);
              lv_obj_set_style_text_color(id(status_text), GREY, 0);
              caption("last known state — can't reach HA", CAPGREY);
            }
            flow_row(false);
            return;
          }

          // ================= SENSOR-OFFLINE FALLBACK =================
          if (pos <= 0) {
            bool unknown = vs.empty() || vs == "unavailable" || vs == "unknown";
            if (unknown) {
              lv_obj_set_style_text_color(id(status_icon), GREY, 0);
              lv_obj_set_style_text_color(id(status_text), GREY, 0);
              btn_grey(CLOSED_GLYPH, "CLOSE");
              caption("valve state unknown", CAPGREY);
              flow_row(true);
              return;
            }
            if (id(actuating)) {
              bool to_closed = id(actuating_to_closed);
              btn_grey(to_closed ? CLOSED_GLYPH : OPEN_GLYPH, to_closed ? "CLOSING…" : "OPENING…");
              status_idle(!to_closed);
            } else {
              btn_active(sw_open);
              status_idle(sw_open);
            }
            caption("position sensor offline — showing HA switch state", CAPGREY);
            flow_row(true);
            return;
          }

          // ================= TILT-TRUTH STATES =================
          flow_row(true);
          bool moving = (id(disagree_since) != 0) && (fk == 0);

          if (fk == 3) {                       // motor didn't start
            status_set(ALERT_GLYPH, AMBER,
                       sw_open ? "NOT FULLY OPEN" : "NOT FULLY CLOSED", RED);
            caption("motor didn't start — check valve", AMBER);
            btn_active(!sw_open);              // retry the switch target: off->CLOSE(red), on->OPEN(green)
          } else if (fk == 1) {                // stuck mid-travel
            char buf[48];
            float a = id(valve_angle_raw).state;
            snprintf(buf, sizeof(buf), "stopped at %.0f° — tap %s to retry",
                     a, sw_open ? "OPEN" : "CLOSE");
            status_set(ALERT_GLYPH, AMBER,
                       sw_open ? "NOT FULLY OPEN" : "NOT FULLY CLOSED", RED);
            caption(buf, AMBER);
            btn_active(!sw_open);
          } else if (fk == 2) {                // mismatch: seated at the wrong stop
            status_idle(pos == 1);
            caption("HA switch disagrees — moved manually?", AMBER);
            btn_active(pos == 1);              // offer the action implied by physical truth
          } else if (moving) {
            bool to_closed = (id(valve_state).state != "on");
            btn_grey(to_closed ? CLOSED_GLYPH : OPEN_GLYPH, to_closed ? "CLOSING…" : "OPENING…");
            status_set(to_closed ? CLOSED_GLYPH : OPEN_GLYPH, AMBER,
                       to_closed ? "Water Valve · CLOSING…" : "Water Valve · OPENING…", AMBER);
            char buf[48];
            snprintf(buf, sizeof(buf), "waiting for valve to confirm — %.0f°",
                     id(valve_angle_raw).state);
            caption(buf, CAPGREY);
          } else {                             // idle, tilt-confirmed
            status_idle(pos == 1);
            btn_active(pos == 1);
            caption_off();
          }
```

- [ ] **Step 2: Rewire on_short_click**

Replace the `on_short_click` `then:` block of `action_btn` with:

```yaml
            on_short_click:
              then:
                - lambda: |-
                    if (!id(ha_api).is_connected()) return;
                    int pos = id(tilt_position);
                    std::string vsv = id(valve_state).state;
                    bool sw_known = (vsv == "on" || vsv == "off");
                    if (pos <= 0) {
                      // FALLBACK: legacy gating, legacy timer clears in the timeout below.
                      if (id(actuating)) return;
                      if (!sw_known) return;
                      id(actuating) = true;
                      id(actuating_to_closed) = (vsv == "on");
                    } else {
                      if (id(actuating) && id(fault_kind) == 0) return;  // MOVING: locked
                      if (!sw_known) return;
                      bool to_closed;
                      if (id(fault_kind) == 2) to_closed = (pos == 1);   // MISMATCH: act on tilt truth
                      else if (id(fault_kind) != 0) to_closed = (vsv == "off"); // retry switch target
                      else to_closed = (vsv == "on");                    // idle: normal toggle
                      id(actuating) = true;
                      id(actuating_to_closed) = to_closed;
                      id(fault_kind) = 0;                                 // leave fault, fresh attempt
                      id(disagree_since) = millis();
                      id(expect_motion_since) = millis();
                      id(valve_problem).publish_state(false);
                    }
                - script.execute: render_state
                - if:
                    condition:
                      lambda: 'return id(actuating) && id(actuating_to_closed);'
                    then:
                      - homeassistant.service:
                          service: switch.turn_off
                          data:
                            entity_id: switch.main_water_valve_robot
                    else:
                      - if:
                          condition:
                            lambda: 'return id(actuating) && !id(actuating_to_closed);'
                          then:
                            - homeassistant.service:
                                service: switch.turn_on
                                data:
                                  entity_id: switch.main_water_valve_robot
                # Legacy fallback lockout: only meaningful when tilt is unknown
                # (with tilt active, agreement/fault clears actuating first in eval).
                - delay: !lambda 'return (uint32_t) ${fallback_lockout_ms};'
                - lambda: 'if (id(tilt_position) <= 0) id(actuating) = false;'
                - script.execute: render_state
```

- [ ] **Step 3: Validate**

Invoke `/esphome-check` on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 4: Commit**

```bash
git add src/main.yaml
git commit -m "Render tilt-truth UI states; confirmation-driven button lockout"
```

---

### Task 6: Ship + staged bring-up (operator in the loop)

**Files:**
- Modify: none in this repo beyond what's committed; version-ref bump happens in the private deploy monorepo.

- [ ] **Step 1: Final validation from a clean cache**

```bash
rm -rf .esphome/packages
```
Invoke `/esphome-check` on `local.yaml`. Expected: `Configuration is valid!`

- [ ] **Step 2: Push and tag**

```bash
git push
git tag v0.2.0
git push --tags
```
(Push to this repo requires the public-push confirmation flow — the owner has approved firmware pushes as the normal deploy path; the guard hook's override prefix is used per command.)

- [ ] **Step 3: Bump the deploy wrapper (private monorepo)**

In the private deploy monorepo's `water-valve-panel.yaml`, change the `github://…/src/main.yaml@v0.1.3` ref to `@v0.2.0`; commit + push there.

- [ ] **Step 4: Staged deploy — sensor-less first (owner clicks)**

Owner runs the HA deploy sync, then clicks Install on the Build Dashboard tile. First compile proves all lambdas. Expected on-device: today's behavior + "position sensor offline — showing HA switch state" caption; HA shows 4 new entities (Angle/Jitter unavailable, Position `unknown`, Problem `off`).

- [ ] **Step 5: Attach U056, reboot, verify I2C**

Plug the 2 m Grove cable into Port A, power-cycle the panel. In the device log expect `Found ADXL345 at 0x53`; no repeated `Failed to read data` warnings (the long-cable watch-item).

- [ ] **Step 6: Hand-held walk-through (before mounting)**

Rotate the unmounted U056 slowly: watch `Valve Handle Angle` in HA and the panel walking open → mid → closed states. **Verify the two intended rest orientations read ≥ ~40° apart** (driver's `max(|pitch|,|roll|)` caveat) — if not, choose a different module orientation for the mount. Cosmetic MISMATCH/MOVING panel states during this are expected.

- [ ] **Step 7: Mount + calibrate**

Mount on the handle. Run the valve to each stop (owner's taps), read `Valve Handle Angle` at both, set `open_angle` / `closed_angle` (and optionally tightened `angle_tolerance`) — either in this repo's substitutions (new tag) or as overrides in the deploy wrapper (preferred: no repo churn). Same session: watch `Valve Jitter` at rest vs during a commanded cycle; set `jitter_moving_threshold` to ~50% of the observed in-motion inter-sample delta, comfortably above rest-noise delta. Redeploy.

- [ ] **Step 8: Failure drills**

- Unplug the Grove cable → within ~5 s panel shows the sensor-offline caption; Position → `unknown`. Replug + reboot restores.
- Clutch-pin manual move → MISMATCH caption + `Valve Position Problem` on; re-sync via the offered button or by matching the switch in HA.
- Temporarily set `confirm_timeout_ms: "8000"` (wrapper override), command a cycle → FAULT-STUCK appears mid-travel, then clears on arrival… verify, then restore `30000`.

- [ ] **Step 9: Record completion**

Update the project knowledge graph entity with: deployed tag, calibrated angles, jitter threshold, drill results. (Owner-side note: the problem-sensor phone notification is a follow-up HA automation, out of firmware scope.)

---

## Self-review checklist (run after writing, before handoff)

- Spec coverage: bus/component pin (T2), calibrated windows + settle + staleness (T3), state machine + faults + jitter advisory (T4), all seven UI treatments + button gating + legacy 25 s fallback (T5), HA entities (T2 angle/jitter, T3 position, T4 problem), validation stages + calibration + drills (T6), font glyphs + amber (T1, T5).
- No placeholders; every code step shows the code.
- Type consistency: `tilt_position` int sentinel −1/0/1/2/3 consistent across T3–T5; `fault_kind` 0–3 consistent across T4–T5; substitution names identical everywhere.
