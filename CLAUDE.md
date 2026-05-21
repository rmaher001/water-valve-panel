# CLAUDE.md — water-valve-panel

This repo is the source of truth for the ESPHome firmware on the garage water valve touch panel (M5Stack CoreS3 SE).

## Deploy flow

Production deploy goes through `homelab-esphome`:
1. Edit + commit YAML here; push to `origin/main`.
2. Tag a release: `git tag vX.Y.Z && git push --tags`.
3. In `homelab-esphome/water-valve-panel.yaml`, bump the github:// URL ref to the new tag.
4. Commit + push `homelab-esphome`.
5. On HA, run `script.deploy_esphome` (force-syncs to `/config/esphome/`).
6. Flash via ESPHome Build Dashboard add-on (manual click).

Do NOT edit the deployed YAML on HA directly — the deploy pipeline force-overrides on the next run.

## What lives where

- `src/main.yaml` — the real ESPHome config. Single source of truth for the device firmware.
- `src/fonts/` — custom font assets (MDI TTF for valve icons).
- `secrets.yaml` — NEVER committed. Local builds resolve secrets via this file; production builds substitute them via ESPHome Build Dashboard.

## Hardware

- M5Stack CoreS3 **SE** variant. No camera, no IMU, no proximity/light sensor (the SE omits the LTR-553ALS). Code that touches those features will fail to compile.

## Related infra

- Z-Wave valve: EcoNet EVC200 (firmware 1.79), retailer brand "Bulldog". HA entity `switch.main_water_valve_robot`.
- Whole-home flow: Droplet, HA entity `sensor.droplet_flow_rate` (gal/min) and `sensor.droplet_daily_water` (gal).
