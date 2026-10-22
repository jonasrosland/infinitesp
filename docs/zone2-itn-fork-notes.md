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
3. Zone 2 room temperature is on **`041E`**, not `3B02`. Payload: room temp whole °F at **byte 6** (and often **11**); byte 10 is cool setpoint — wrong offset showed ~36.7°C until fixed.

## Code changes (summary)

- Passive capture for aux addresses `0x21`–`0x28`
- `declare_zone`, active mask fallbacks (declared climate zones, ZC `0319`)
- `zone_room_temp_c_` / aux decode from `041E`
- Comfort row **`400B`** fallback when `3B03` setpoints are empty
- `infinitesp.yaml`: two climates — First floor / Second floor

## Verify

After OTA to `infinitesp.local`, **Second floor Temperature** ~21.1°C (~70°F) with comfort profiles from slow `400B` poll.

## Upstream

Consider opening a PR to nebulous when ready; this doc is for maintaining the fork.
