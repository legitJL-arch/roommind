# Active fan-speed control during heating/cooling — design

## Problem

RoomMind's active control loop (`MPCController.async_apply`, `control/mpc_controller.py`)
never calls `climate.set_fan_mode`. Fan speed is only ever touched by the idle/setback
path (`async_idle_device`'s `idle_action="fan_only"` + `idle_fan_mode`). During active
heating/cooling, the device's fan speed sits wherever it was last left.

Confirmed via code review and upstream issue history (snazzybean/roommind#18, closed
2026-03-15: "Full proportional fan speed control during active heating/cooling is still
on the roadmap") that this is a genuine, unaddressed gap — not a duplicate effort.

Josh currently fills this gap with a standalone Home Assistant automation
(`automation.aircon_fan_auto_control`) that steps `input_select.aircon_fan` based on
the raw temperature/target differential, with a 10-minute "kick to High" warmup on
mode start. This project replaces that automation with logic built directly into the
RoomMind fork, so fan speed is decided by the same control loop — and the same
momentum/hysteresis — as the heat/cool decision itself, rather than a second,
independent system writing to the same device.

## Scope

- Per-device opt-in (new `active_fan_control` config flag), not global. Today only one
  device (`climate.aircon`, a Mitsubishi Heavy Industries SRK35ZSA-W via IR/template
  climate) will have it enabled.
- Active control path only (`async_apply`, `mode in {HEATING, COOLING}`). The idle path
  (`async_idle_device`) is unchanged.

## Architecture

`async_apply` already receives `power_fraction: float` (0.0–1.0), computed by
`async_evaluate` — either from the MPC optimizer's plan (`_evaluate_mpc`) or, when model
confidence is too low, a binary 0.0/1.0 from the bang-bang fallback (`_evaluate_bangbang`).
Today `power_fraction` is used only to compute a proportional AC setpoint "boost"
(`mpc_controller.py` ~1481-1568).

We add a new step in the same AC branches (`mode == MODE_HEATING` / `MODE_COOLING`,
`eid in self.acs`): after the existing `set_hvac_mode`/`set_temperature` calls, compute
a fan-speed band from `power_fraction` and call `set_fan_mode` if the device has
`active_fan_control` enabled.

This deliberately reuses the exact value already driving heat/cool intensity, so fan
speed has the same momentum/confidence characteristics as the rest of the active
control decision — not a second, independently-tuned signal.

New per-area state: `_previous_fan_speed: dict[str, str]` on `MPCController`, parallel
to the existing `previous_mode`/`_mode_on_since` mode-stickiness state, giving the
hysteresis memory across polling cycles.

No new HA entity, no new automation. This rides the same `climate.set_fan_mode` service
call path `async_idle_device` already uses.

## Band thresholds and hysteresis

4 usable speeds map to `power_fraction` quartiles (excluding `auto`, which is the
device's own internal algorithm, not ours to drive):

| `power_fraction` | Fan speed |
|---|---|
| ≥ 0.75 | `high` |
| ≥ 0.50 | `medhigh` |
| ≥ 0.25 | `medlow` |
| < 0.25 | `low` |

Hysteresis mirrors the existing mode-stickiness idiom (`previous_mode` + crossing back
past the target, not the original trigger threshold): a fixed buffer of `0.07` is
required to cross a boundary in either direction, relative to the *current* band's
threshold(s). E.g. currently `low`: only steps up to `medlow` once `power_fraction ≥
0.32` (0.25 + 0.07). Currently `medlow`: only drops to `low` once `power_fraction <
0.18` (0.25 − 0.07). Same buffer reused at all three boundaries (0.25/0.50/0.75).

The buffer is a fixed module-level constant (e.g. `FAN_SPEED_HYSTERESIS = 0.07`),
following the precedent of `BANGBANG_HEAT_HYSTERESIS`/`BANGBANG_COOL_HYSTERESIS` — not
per-device configurable. With only one device in use, per-device tuning would be
premature.

No "kick to high on mode start" warmup is needed: Josh's temperature sensor has already
been relocated to the far end of the long room it serves, so a large initial
temperature gap (and therefore a large `power_fraction`) already occurs naturally at
the start of a heating/cooling cycle.

## Config and capability checks

- New per-device boolean `active_fan_control` (default `false`) added to the same
  options schema as `idle_fan_mode` (`websocket_api.py` device config block).
  Unconfigured devices are completely unaffected — zero behavior change unless
  explicitly opted in.
- Band labels are hardcoded as `"low"`, `"medlow"`, `"medhigh"`, `"high"` — they already
  match `climate.aircon`'s actual `fan_modes` attribute exactly. No configurable
  label-mapping is added; it would be unused complexity for a single-consumer feature.
- Before calling `set_fan_mode`, the chosen band label must be present in the device's
  live `fan_modes` attribute (mirrors the existing containment check in
  `async_idle_device` for `idle_fan_mode`). If absent, skip the call silently (debug
  log only) — `hvac_mode`/`temperature` calls proceed unaffected. This keeps the feature
  safe to enable on a future device with a different fan vocabulary without crashing or
  sending an invalid command.

## Migration off the existing automation

Home Assistant config change, not part of the code fork:

1. Ship and verify the RoomMind change first; don't remove the existing safety net
   before the replacement is proven.
2. Disable (don't delete) `automation.aircon_fan_auto_control` once the new logic is
   confirmed live.
3. Set `active_fan_control: true` for `climate.aircon` in RoomMind's device config.
4. After a few days of confirmed-good behavior, delete the automation outright.

No changes needed to `automation.aircon_settings_changed` or
`automation.aircon_command_visibility_roommind` — both react generically to
`input_select.aircon_fan` state changes regardless of writer, so they keep working
unmodified once RoomMind becomes the writer instead of the old automation.

RoomMind becomes the sole owner of fan speed for this device; the standalone automation
is retired, not kept as a parallel fallback, to avoid two systems racing to set the
same `input_select`.

## Testing

Follow the existing pattern in `tests/control/test_fan_mode.py` (`MagicMock` HA state
with `fan_modes`/`fan_mode` attributes, assert on `hass.services.async_call`):

1. **Pure hysteresis unit tests** for the band-selection function in isolation: given
   `(power_fraction, previous_band)`, assert correct output at and around every
   boundary, both directions, including "stays put" cases inside the buffer zone (e.g.
   `previous=low, power_fraction=0.30` → stays `low`; `power_fraction=0.33` → flips to
   `medlow`).
2. **`async_apply` integration tests** (heating and cooling branches): assert
   `climate.set_fan_mode` is called with the correct value when `active_fan_control`
   is enabled and `power_fraction` crosses a boundary.
3. **Capability skip test**: device's `fan_modes` missing one of the 4 labels → no
   `set_fan_mode` call, no exception, `hvac_mode`/`temperature` calls still happen.
4. **Opt-out test**: `active_fan_control` unset/false → no `set_fan_mode` call at all,
   confirming zero behavior change for every other device.
5. **No regression to idle path**: existing `test_async_idle_device_fan_only` and
   related idle-path tests keep passing untouched.
