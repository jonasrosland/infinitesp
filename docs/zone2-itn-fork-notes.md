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
4. **Live vs profile:** `Z#HTSP?` / `Z#FAN?` read the SAM **`3B03` mirror**, which ITN often leaves empty. **Profile** heat/cool/fan live in table **40** (`400A` zone 1, `400B` zone 2, …). Fan **3** = **HIGH**, **0** = **AUTO**.

## Code changes (summary)

### Zone 2 / bus capture (initial fork commit)

- Passive capture for aux addresses `0x21`–`0x28`
- `declare_zone`, active mask fallbacks (declared climate zones, ZC `0319`)
- `zone_room_temp_c_` / aux decode from `041E`
- Comfort row **`400A`/`400B`** fallback when `3B03` setpoints are empty
- `infinitesp.yaml`: two climates — First floor / Second floor

### ITN Home Assistant / control follow-ups

- **`041E` decode:** prefer offset **11** before **6** for room temp.
- **Setpoints:** fill missing heat **or** cool from the zone Home (or Manual) comfort row when SAM `3B03` is empty or partial.
- **Climate mode:** apply global **3B02** mode/stage even when a zone’s room temp is not ready yet (fixes zone 1 stuck on **fan only** while zone 2 showed **heat**).
- **HA setpoint snapback:** empty **3B03** poll readback no longer clears commanded setpoints or hold state; HA/ASCII edits on ITN write the **MANUAL** comfort row and arm a **hold**. Preset readback treats **MANUAL** as **Hold indefinitely**, not Per schedule.
- **Comfort profile + season (SAM extensions):** see [sam-ascii.md](sam-ascii.md) and table below.

### SAM extensions quick reference (ITN)

Connect: `nc infinitesp.local 23`

| Goal | Commands |
|------|----------|
| Set season Home setpoints | `Z1HOME!66,76` / `Z2HOME!66,76` |
| Home fan Auto (profile) | `Z1HOMEFAN!AUTO` / `Z2HOMEFAN!AUTO` |
| Home on schedule (no hold) | `Z1ACT!home` |
| **Home + permanent hold (season)** | `Z1ACT!homehold` / `Z2ACT!homehold` |
| Live fan now | `Z1FAN!AUTO` (in addition to `HOMEFAN!` if needed) |
| Cancel hold | `Z1HOLD!off` |
| HA-style live SP (may not stick on ITN) | `Z1HTSP!72` / `Z1CLSP!76` |

**`homehold`** applies Home setpoints/fan, arms a **permanent hold**, and **copies Home → Manual** in table 40 so a leftover Manual row (e.g. **76°** from an HA slider) does not confuse readback.

## Season setup (one Home profile, permanent hold)

Use once per season (or after an HA edit put you on Manual at the wrong temp):

```text
Z1HOME!66,76
Z1HOMEFAN!AUTO
Z1ACT!homehold
Z2HOME!66,76
Z2HOMEFAN!AUTO
Z2ACT!homehold
```

Adjust **`66,76`** for your season. Verify with `REPORT?` (registers **400A** / **400B**) or HA climate entities after refresh.

## Verify

- **Second floor** room temp tracks the Smart Sensor (~70°F), not the heat setpoint.
- Both zones **Home** row: **66 / 76**, fan **0** (Auto) after season script.
- **`MODE?`** and both climate cards agree on system mode (**heat** in a heat-mode test).
- After **`homehold`**, zone 1 **Manual** row matches **Home** (not 76 / 76 from an old HA hold).
- SAM **`3B03`** mirror: `zones_holding` bit 0 set = zone 1 permanent hold ( **`Z1HOLD?`** may still say OFF on ITN).

## Home Assistant notes

- **Idle** = not actively heating/cooling; normal when above setpoint or no stage call.
- **Heat mode** still shows a **cool** bound in HA (two-point entity); that is the profile cool limit, not “cooling now.”
- Changing temperature in HA on ITN uses **MANUAL + hold**; use **`homehold`** on the bus to return to **Home** for the season.
- Prefer **Home** preset or **`Z#ACT!homehold`** over sliding temp when you only use one profile per season.

## Upstream

Consider opening a PR to nebulous when ready; this doc is for maintaining the fork.
