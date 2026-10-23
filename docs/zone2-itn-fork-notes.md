# Zone 2 / Next Gen ITN fork notes (jonasrosland)

Personal fork of [nebulous/infinitesp](https://github.com/nebulous/infinitesp) with changes for **SYSTXCCITN01** (Infinity Next Gen) and a **two-zone** setup.

## Hardware (verified on wire)

- Device: ESPHome InfinitESP on `infinitesp.local`
- Thermostat: **SYSTXCCITN01** at bus `0x20`
- Zone controller: **EXCALIBUR 4 ZONE** at `0x60` (passive; `zone_controller_address` not enabled)
- Zone 2 **Smart Sensor** at bus **`0x22`**, register **`041E`** (thermostat polls `20>22` READ `041E`)
- SAM emulation at `0x92`

## Problem

1. Next Gen control does **not** populate SAM **`3B02` / `3B03`** for polled reads → active-zone mask reads as `0`, zone 2 entities bail on inactive-zone checks.
2. Smart Sensor traffic on **`0x21`–`0x28` / `041E`** was not stored by passive capture (only IDU, ODU, ZC).
3. Zone 2 room temperature is on **`041E`**, not `3B02`. On ITN Smart Sensor payloads, **byte 11** is room temp (whole °F); **bytes 6–7** often mirror heat/cool setpoints — reading byte 6 first made HA show the heat setpoint as room temp.

## Code changes (summary)

### Zone 2 / bus capture (initial fork commit)

- Passive capture for aux addresses `0x21`–`0x28`
- `declare_zone`, active mask fallbacks (declared climate zones, ZC `0319`)
- `zone_room_temp_c_` / aux decode from `041E`
- Comfort row **`400A`/`400B`** fallback when `3B03` setpoints are empty
- `infinitesp.yaml`: two climates — First floor / Second floor

### ITN Home Assistant / control follow-ups

- **`041E` decode:** prefer offset **11** before **6** for room temp.
- **Setpoints:** fill missing heat **or** cool from the zone Home comfort row when SAM `3B03` is empty or partial (e.g. after `Z#HTSP!` only).
- **Climate mode:** apply global **3B02** mode/stage even when a zone’s room temp is not ready yet (fixes zone 1 stuck on **fan only** while zone 2 showed **heat**).
- **Comfort profile writes (SAM extensions):**
  - `Z#HOME!heat,cool` — update Home row in table 40 (`400A`+zone−1)
  - `Z#ACT!home|away|sleep|wake|manual` — apply that activity’s setpoints/fan via `3B03` with hold cleared (`hold=0`)

See [sam-ascii.md](sam-ascii.md) for the full command list.

## Verify

- **Second floor** room temp tracks the Smart Sensor (~70°F), not the heat setpoint.
- Both zones show **66 / 76** Home setpoints in HA when configured via `Z#HOME!66,76`.
- **`MODE?`** and both climate cards agree on system mode (**heat** in a heat-mode test).
- `Z1HOME!66,76` / `Z2HOME!66,76` then `Z1ACT!home` / `Z2ACT!home`: thermostat ACKs **400A**/**400B** writes.

## Upstream

Consider opening a PR to nebulous when ready; this doc is for maintaining the fork.
