# Active Fan-Speed Control During Heating/Cooling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make RoomMind's active heat/cool control loop modulate `climate.set_fan_mode` proportionally to demand, reusing the existing `power_fraction` signal, so Josh can retire a standalone Home Assistant automation that currently fills this gap.

**Architecture:** `power_fraction` (0.0-1.0, already computed by `MPCController.async_evaluate` and already threaded into `async_apply`) is quantized into one of 4 fan-speed bands (`low`/`medlow`/`medhigh`/`high`) by a new pure function with hysteresis, then sent via `climate.set_fan_mode` from inside the existing AC branches of `async_apply`, gated by a new per-device `active_fan_control` opt-in flag. Band-crossing memory lives in a module-level dict (mirroring the existing `_last_commands` cache pattern), since `MPCController` instances are recreated every poll cycle.

**Tech Stack:** Python 3.12, Home Assistant custom integration (`custom_components/roommind`), pytest + pytest-asyncio, voluptuous (config schema validation).

**Full design spec:** `docs/superpowers/specs/2026-06-24-active-fan-speed-control-design.md`

**Scope note:** This plan only touches the simple AC branches of `async_apply` (`mode == MODE_HEATING` / `MODE_COOLING`, no `heat_source_plan`). The multi-device heat-source-orchestration path (`heat_source_plan is not None`) and the no-external-sensor "managed mode" path are intentionally NOT touched — Josh's setup uses neither, and adding fan-speed logic there would be speculative generalization for code paths with no current consumer.

---

### Task 1: Per-device `active_fan_control` opt-in flag

**Files:**
- Modify: `custom_components/roommind/utils/device_utils.py:24-29` (constants), `:215-223` (new helper near `get_idle_action`)
- Modify: `custom_components/roommind/websocket_api.py:313-315` (schema)
- Test: `tests/utils/test_device_utils.py`

- [ ] **Step 1: Write the failing tests**

Add to `tests/utils/test_device_utils.py`, right after `test_get_idle_action_configured` (around line 407):

```python
def test_get_active_fan_control_defaults_false():
    """Empty devices list returns False."""
    assert get_active_fan_control([], "climate.nonexistent") is False


def test_get_active_fan_control_unconfigured_device_defaults_false():
    """Device present but without active_fan_control key defaults to False."""
    devices = [{"entity_id": "climate.ac1", "type": "ac", "role": "auto"}]
    assert get_active_fan_control(devices, "climate.ac1") is False


def test_get_active_fan_control_enabled():
    """Device with active_fan_control=True returns True."""
    devices = [{"entity_id": "climate.ac1", "type": "ac", "role": "auto", "active_fan_control": True}]
    assert get_active_fan_control(devices, "climate.ac1") is True
```

Add `get_active_fan_control` to the import block at the top of the file (alphabetical, after `get_ac_eids`):

```python
from custom_components.roommind.utils.device_utils import (
    SETPOINT_MODE_PROPORTIONAL,
    VALID_DEVICE_TYPES,
    VALID_HEATING_SYSTEM_TYPES,
    build_rooms_devices_map,
    devices_to_legacy,
    ensure_room_has_devices,
    get_ac_eids,
    get_active_fan_control,
    get_all_entity_ids,
    get_device_by_eid,
    get_direct_setpoint_eids,
    get_entity_ids_by_type,
    get_idle_action,
    get_room_heating_system_type,
    get_trv_eids,
    has_reliable_hvac_modes,
    is_ac_type,
    is_trv_type,
    legacy_to_devices,
    migrate_heat_pump_devices,
    room_contributes_to_group,
)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/utils/test_device_utils.py -k active_fan_control -v`
Expected: FAIL with `ImportError: cannot import name 'get_active_fan_control'`

- [ ] **Step 3: Implement `get_active_fan_control`**

In `custom_components/roommind/utils/device_utils.py`, add directly after `get_idle_action` (after line 223):

```python
def get_active_fan_control(devices: list[dict], entity_id: str) -> bool:
    """Return whether *entity_id* has active (non-idle) fan-speed control enabled."""
    dev = get_device_by_eid(devices, entity_id)
    if dev is None:
        return False
    return bool(dev.get("active_fan_control", False))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pytest tests/utils/test_device_utils.py -k active_fan_control -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Add the config schema field**

In `custom_components/roommind/websocket_api.py`, modify the device schema block (lines 306-318):

```python
        vol.Optional("devices"): [
            vol.All(
                {
                    vol.Required("entity_id"): str,
                    vol.Required("type"): vol.In(["trv", "ac"]),
                    vol.Optional("role", default="auto"): vol.In(["primary", "secondary", "auto"]),
                    vol.Optional("heating_system_type", default=""): vol.In(["", "radiator", "underfloor"]),
                    vol.Optional("idle_action", default="off"): vol.In(["off", "fan_only", "setback", "low"]),
                    vol.Optional("idle_fan_mode", default="low"): str,
                    vol.Optional("setpoint_mode", default="proportional"): vol.In(["proportional", "direct"]),
                    vol.Optional("active_fan_control", default=False): bool,
                },
                _validate_device_idle_action,
            )
        ],
```

(Only the new `vol.Optional("active_fan_control", default=False): bool,` line is added, directly after the `setpoint_mode` line.)

- [ ] **Step 6: Run the full device_utils + websocket_api test files**

Run: `pytest tests/utils/test_device_utils.py tests/test_websocket_api.py -v`
Expected: PASS, no existing tests broken (the new schema field is optional with a default, so messages without it are unaffected — matches the existing `idle_fan_mode` precedent where schema defaults are not asserted in the `devices` round-trip tests, since those tests call the handler function directly and bypass HA's websocket-layer schema validation).

- [ ] **Step 7: Commit**

```bash
git add custom_components/roommind/utils/device_utils.py custom_components/roommind/websocket_api.py tests/utils/test_device_utils.py
git commit -m "feat: add active_fan_control per-device opt-in flag"
```

---

### Task 2: Fan-speed band constants

**Files:**
- Modify: `custom_components/roommind/const.py:64-65` (add constants after `BANGBANG_COOL_HYSTERESIS`)

- [ ] **Step 1: Add the constants**

In `custom_components/roommind/const.py`, directly after line 65 (`BANGBANG_COOL_HYSTERESIS = 0.2  # ...`):

```python
# Fan-speed bands during active heating/cooling, quantized from power_fraction (0.0-1.0).
# Ordered low -> high; FAN_SPEED_EDGES[i] is the boundary between
# FAN_SPEED_LABELS[i] and FAN_SPEED_LABELS[i + 1].
FAN_SPEED_LABELS = ("low", "medlow", "medhigh", "high")
FAN_SPEED_EDGES = (0.25, 0.50, 0.75)
FAN_SPEED_HYSTERESIS = 0.07  # required overshoot past a boundary before switching bands
```

No test for this step alone — constants are exercised by Task 3's tests.

- [ ] **Step 2: Commit**

```bash
git add custom_components/roommind/const.py
git commit -m "feat: add fan-speed band constants"
```

---

### Task 3: Pure hysteresis band-selection function

**Files:**
- Modify: `custom_components/roommind/control/mpc_controller.py:13-35` (const imports), `:63-64` (module state), `:98-101` (`clear_command_cache`), add new function after `_effective_ac_modes` (around line 627, before `class MPCController` at line 700)
- Test: `tests/control/test_fan_mode.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/control/test_fan_mode.py` (new section at the end of the file):

```python
# ---------------------------------------------------------------------------
# _fan_speed_for_power_fraction — pure hysteresis unit tests
# ---------------------------------------------------------------------------


def test_fan_speed_low_band_default():
    """No previous speed, low power_fraction -> low."""
    assert _fan_speed_for_power_fraction(0.1, None) == "low"


def test_fan_speed_jumps_straight_to_high_from_unknown():
    """No previous speed, high power_fraction -> high (no warmup kick needed)."""
    assert _fan_speed_for_power_fraction(0.9, None) == "high"


def test_fan_speed_stays_put_inside_hysteresis_buffer_going_up():
    """Currently low, power_fraction just below the buffered threshold stays low."""
    assert _fan_speed_for_power_fraction(0.30, "low") == "low"


def test_fan_speed_steps_up_past_buffer():
    """Currently low, power_fraction past the buffered threshold steps up to medlow."""
    assert _fan_speed_for_power_fraction(0.33, "low") == "medlow"


def test_fan_speed_stays_put_inside_hysteresis_buffer_going_down():
    """Currently medlow, power_fraction just above the buffered drop threshold stays medlow."""
    assert _fan_speed_for_power_fraction(0.18, "medlow") == "medlow"


def test_fan_speed_steps_down_past_buffer():
    """Currently medlow, power_fraction below the buffered drop threshold drops to low."""
    assert _fan_speed_for_power_fraction(0.17, "medlow") == "low"


def test_fan_speed_unknown_previous_speed_treated_as_low():
    """An unrecognized previous_speed value (e.g. 'auto') is treated as starting from low."""
    assert _fan_speed_for_power_fraction(0.9, "auto") == "high"
```

Add `_fan_speed_for_power_fraction` to the existing import from `custom_components.roommind.control.mpc_controller` at the top of the file:

```python
from custom_components.roommind.control.mpc_controller import (
    MPCController,
    _fan_speed_for_power_fraction,
    _last_commands,
    async_idle_device,
    clear_command_cache,
)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/control/test_fan_mode.py -k fan_speed_for_power_fraction -v`
Expected: FAIL with `ImportError: cannot import name '_fan_speed_for_power_fraction'`

- [ ] **Step 3: Add module-level state and the function**

In `custom_components/roommind/control/mpc_controller.py`, modify the const import block (lines 13-35) to add the three new constants (alphabetical among the existing names):

```python
from ..const import (
    AC_BOOST_DELTA_MAX,
    AC_BOOST_DELTA_MIN,
    AC_COOLING_BOOST_TARGET,
    AC_HEATING_BOOST_TARGET,
    APPROACH_RATE_MIN,
    BANGBANG_COOL_HYSTERESIS,
    BANGBANG_HEAT_HYSTERESIS,
    CLIMATE_MODE_COOL_ONLY,
    CLIMATE_MODE_HEAT_ONLY,
    DEFAULT_COMFORT_WEIGHT,
    DEFAULT_OUTDOOR_COOLING_MIN,
    DEFAULT_OUTDOOR_HEATING_MAX,
    FAN_SPEED_EDGES,
    FAN_SPEED_HYSTERESIS,
    FAN_SPEED_LABELS,
    HEATING_BOOST_TARGET,
    MODE_COOLING,
    MODE_HEATING,
    MODE_IDLE,
    PROPORTIONAL_DEADBAND_C,
    PROPORTIONAL_DEADBAND_NEAR_TARGET_C,
    TargetTemps,
    is_override_active,
    make_roommind_context,
)
```

Also add `get_active_fan_control` to the `..utils.device_utils` import block (lines 36-46):

```python
from ..utils.device_utils import (
    DEFAULT_IDLE_SETBACK_OFFSET,
    IDLE_ACTION_FAN_ONLY,
    IDLE_ACTION_LOW,
    IDLE_ACTION_SETBACK,
    get_ac_eids,
    get_active_fan_control,
    get_direct_setpoint_eids,
    get_idle_action,
    get_trv_eids,
    has_reliable_hvac_modes,
)
```

Add the module-level state dict right after `_setpoint_override_warned` (line 64):

```python
_setpoint_override_warned: set[str] = set()
# Last fan-speed band sent per climate entity, for active-control hysteresis.
# Same lifecycle as _last_commands: persists across MPCController instances
# (recreated each cycle), resets on integration reload.
_previous_fan_speeds: dict[str, str] = {}
```

Extend `clear_command_cache` (lines 98-101) to also clear the new dict:

```python
def clear_command_cache() -> None:
    """Clear cached state. Call on integration reload/unload."""
    _last_commands.clear()
    _setpoint_override_warned.clear()
    _previous_fan_speeds.clear()
```

Add the pure function directly after `_effective_ac_modes` (after line 627, before the blank lines leading into `class MPCController`):

```python
def _fan_speed_for_power_fraction(power_fraction: float, previous_speed: str | None) -> str:
    """Quantize power_fraction (0.0-1.0) into a fan-speed band with hysteresis.

    Mirrors the existing mode-stickiness pattern used for heat/cool decisions
    (previous_mode + crossing back past the target, not the original trigger
    threshold): moving to a different band requires crossing its boundary by
    FAN_SPEED_HYSTERESIS, not just touching it, so the fan doesn't flap near
    a threshold.
    """
    try:
        index = FAN_SPEED_LABELS.index(previous_speed)
    except ValueError:
        index = 0  # unknown/no previous speed: start unbiased from the bottom

    # Move up: each edge above the current band must be cleared by the buffer.
    while index < len(FAN_SPEED_EDGES) and power_fraction >= FAN_SPEED_EDGES[index] + FAN_SPEED_HYSTERESIS:
        index += 1
    # Move down: the edge below the current band must be cleared by the buffer.
    while index > 0 and power_fraction < FAN_SPEED_EDGES[index - 1] - FAN_SPEED_HYSTERESIS:
        index -= 1

    return FAN_SPEED_LABELS[index]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pytest tests/control/test_fan_mode.py -k fan_speed_for_power_fraction -v`
Expected: PASS (7 tests)

- [ ] **Step 5: Run the full fan_mode test file to check for regressions**

Run: `pytest tests/control/test_fan_mode.py -v`
Expected: PASS (all existing idle-path tests plus the 7 new ones)

- [ ] **Step 6: Commit**

```bash
git add custom_components/roommind/const.py custom_components/roommind/control/mpc_controller.py tests/control/test_fan_mode.py
git commit -m "feat: add power_fraction-to-fan-speed hysteresis function"
```

---

### Task 4: `_async_apply_active_fan_speed` method + wire into heating branch

**Files:**
- Modify: `custom_components/roommind/control/mpc_controller.py` — add method after `_proportional_deadband` (around line 1618, before `_call`), wire into the heating AC loop (around line 1520-1544)
- Test: `tests/control/test_fan_mode.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/control/test_fan_mode.py`:

```python
# ---------------------------------------------------------------------------
# async_apply — active fan-speed control (heating)
# ---------------------------------------------------------------------------


@pytest.mark.asyncio
async def test_mpc_apply_heating_sets_fan_speed_when_enabled():
    """AC with active_fan_control=True gets set_fan_mode during heating, banded from power_fraction."""
    _last_commands.clear()
    _previous_fan_speeds.clear()
    hass = build_hass()
    state = MagicMock()
    state.state = "heat"
    state.attributes = {
        "hvac_modes": ["heat", "off"],
        "fan_modes": ["auto", "low", "medlow", "medhigh", "high"],
        "fan_mode": "low",
        "temperature": 20.0,
    }
    hass.states.get = MagicMock(return_value=state)

    room = make_room(thermostats=[], acs=["climate.ac1"])
    room["devices"] = [
        {
            "entity_id": "climate.ac1",
            "type": "ac",
            "role": "auto",
            "heating_system_type": "",
            "active_fan_control": True,
        }
    ]
    model_mgr = RoomModelManager()
    ctrl = MPCController(
        hass,
        room,
        model_manager=model_mgr,
        outdoor_temp=5.0,
        settings={},
        has_external_sensor=True,
    )
    await ctrl.async_apply("heating", 21.0, power_fraction=0.9)

    calls = hass.services.async_call.call_args_list
    fan_calls = [c for c in calls if c[0][1] == "set_fan_mode"]
    assert len(fan_calls) == 1
    assert fan_calls[0][0][2] == {"entity_id": "climate.ac1", "fan_mode": "high"}


@pytest.mark.asyncio
async def test_mpc_apply_heating_no_fan_speed_when_disabled():
    """AC without active_fan_control set gets no set_fan_mode call during heating."""
    _last_commands.clear()
    _previous_fan_speeds.clear()
    hass = build_hass()
    state = MagicMock()
    state.state = "heat"
    state.attributes = {
        "hvac_modes": ["heat", "off"],
        "fan_modes": ["auto", "low", "medlow", "medhigh", "high"],
        "fan_mode": "low",
        "temperature": 20.0,
    }
    hass.states.get = MagicMock(return_value=state)

    room = make_room(thermostats=[], acs=["climate.ac1"])
    model_mgr = RoomModelManager()
    ctrl = MPCController(
        hass,
        room,
        model_manager=model_mgr,
        outdoor_temp=5.0,
        settings={},
        has_external_sensor=True,
    )
    await ctrl.async_apply("heating", 21.0, power_fraction=0.9)

    calls = hass.services.async_call.call_args_list
    fan_calls = [c for c in calls if c[0][1] == "set_fan_mode"]
    assert len(fan_calls) == 0


@pytest.mark.asyncio
async def test_mpc_apply_heating_fan_speed_skips_unsupported_label():
    """Desired band not in device's fan_modes: no set_fan_mode call, hvac/temp calls still happen."""
    _last_commands.clear()
    _previous_fan_speeds.clear()
    hass = build_hass()
    state = MagicMock()
    # state.state="off" (not "heat") so the controller's own set_hvac_mode
    # redundancy check doesn't skip the call we're asserting on below.
    state.state = "off"
    state.attributes = {
        "hvac_modes": ["heat", "off"],
        "fan_modes": ["auto", "low"],  # no "high"
        "fan_mode": "low",
        "temperature": 20.0,
    }
    hass.states.get = MagicMock(return_value=state)

    room = make_room(thermostats=[], acs=["climate.ac1"])
    room["devices"] = [
        {
            "entity_id": "climate.ac1",
            "type": "ac",
            "role": "auto",
            "heating_system_type": "",
            "active_fan_control": True,
        }
    ]
    model_mgr = RoomModelManager()
    ctrl = MPCController(
        hass,
        room,
        model_manager=model_mgr,
        outdoor_temp=5.0,
        settings={},
        has_external_sensor=True,
    )
    await ctrl.async_apply("heating", 21.0, power_fraction=0.9)

    calls = hass.services.async_call.call_args_list
    fan_calls = [c for c in calls if c[0][1] == "set_fan_mode"]
    assert len(fan_calls) == 0
    hvac_calls = [c for c in calls if c[0][1] == "set_hvac_mode"]
    assert len(hvac_calls) >= 1


@pytest.mark.asyncio
async def test_mpc_apply_heating_fan_speed_redundant_skip():
    """Device already reporting the desired fan_mode: no redundant set_fan_mode call."""
    _last_commands.clear()
    _previous_fan_speeds.clear()
    hass = build_hass()
    state = MagicMock()
    state.state = "heat"
    state.attributes = {
        "hvac_modes": ["heat", "off"],
        "fan_modes": ["auto", "low", "medlow", "medhigh", "high"],
        "fan_mode": "high",  # already at the band power_fraction=0.9 would select
        "temperature": 20.0,
    }
    hass.states.get = MagicMock(return_value=state)

    room = make_room(thermostats=[], acs=["climate.ac1"])
    room["devices"] = [
        {
            "entity_id": "climate.ac1",
            "type": "ac",
            "role": "auto",
            "heating_system_type": "",
            "active_fan_control": True,
        }
    ]
    model_mgr = RoomModelManager()
    ctrl = MPCController(
        hass,
        room,
        model_manager=model_mgr,
        outdoor_temp=5.0,
        settings={},
        has_external_sensor=True,
    )
    await ctrl.async_apply("heating", 21.0, power_fraction=0.9)

    calls = hass.services.async_call.call_args_list
    fan_calls = [c for c in calls if c[0][1] == "set_fan_mode"]
    assert len(fan_calls) == 0
```

Add `_previous_fan_speeds` to the existing import block:

```python
from custom_components.roommind.control.mpc_controller import (
    MPCController,
    _fan_speed_for_power_fraction,
    _last_commands,
    _previous_fan_speeds,
    async_idle_device,
    clear_command_cache,
)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/control/test_fan_mode.py -k "test_mpc_apply_heating_sets_fan_speed_when_enabled or test_mpc_apply_heating_no_fan_speed_when_disabled or test_mpc_apply_heating_fan_speed_skips_unsupported_label or test_mpc_apply_heating_fan_speed_redundant_skip" -v`
Expected: FAIL — `test_mpc_apply_heating_sets_fan_speed_when_enabled` fails on the `assert len(fan_calls) == 1` (0 calls made); the other three pass vacuously since no fan logic exists yet (no-op is the current behavior). That's expected — implement the method now so all four assert the right thing.

- [ ] **Step 3: Implement `_async_apply_active_fan_speed`**

In `custom_components/roommind/control/mpc_controller.py`, add this method directly after `_proportional_deadband` (after line 1631, the line before `async def _call`):

```python
    async def _async_apply_active_fan_speed(self, eid: str, power_fraction: float) -> None:
        """Set a proportional fan speed for *eid* while actively heating/cooling.

        Only acts when the device has active_fan_control enabled. Reuses the
        same power_fraction driving the heat/cool intensity decision, so fan
        speed has the same momentum as the rest of the active control.
        """
        if not get_active_fan_control(self._devices, eid):
            return

        previous = _previous_fan_speeds.get(eid)
        desired = _fan_speed_for_power_fraction(power_fraction, previous)
        _previous_fan_speeds[eid] = desired

        state = self.hass.states.get(eid)
        fan_modes: list[str] = (state.attributes.get("fan_modes") or []) if state else []
        if desired not in fan_modes:
            _LOGGER.debug(
                "Area '%s': device '%s' does not support fan_mode '%s' (available: %s)",
                self._area_id,
                eid,
                desired,
                fan_modes,
            )
            return

        if state and state.attributes.get("fan_mode") == desired:
            return

        try:
            await self.hass.services.async_call(
                "climate",
                "set_fan_mode",
                {"entity_id": eid, "fan_mode": desired},
                blocking=True,
                context=make_roommind_context(),
            )
        except Exception:  # noqa: BLE001
            _LOGGER.warning(
                "Area '%s': climate.set_fan_mode('%s') failed on '%s'",
                self._area_id,
                desired,
                eid,
                exc_info=True,
            )
```

Now wire it into the heating branch's AC loop. Find this block (around line 1520-1544):

```python
            for eid in self.acs:
                if eid in _forced_off:
                    await async_idle_device(self.hass, eid, self._devices, area_id=self._area_id, targets=targets)
                    continue
                ha_t = ha_ac_direct if eid in self._direct_eids else ha_ac_target
                ac_state = self.hass.states.get(eid)
                ac_modes = _effective_ac_modes(ac_state)
                if "heat" in ac_modes:
                    ac_mode = "heat"
                elif "heat_cool" in ac_modes:
                    ac_mode = "heat_cool"
                elif "auto" in ac_modes:
                    ac_mode = "auto"
                else:
                    ac_mode = ""
                if ac_mode:
                    await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": ac_mode})
                    await self._call(
                        "set_temperature",
                        {"entity_id": eid, "temperature": ha_t, "hvac_mode": ac_mode},
                        temp_intent="heat",
                        deadband=self._proportional_deadband(eid, current_temp, effective_target),
                    )
                else:
                    await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": "off"})
```

Replace the `if ac_mode:` branch's body to add the fan-speed call after `set_temperature`:

```python
                if ac_mode:
                    await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": ac_mode})
                    await self._call(
                        "set_temperature",
                        {"entity_id": eid, "temperature": ha_t, "hvac_mode": ac_mode},
                        temp_intent="heat",
                        deadband=self._proportional_deadband(eid, current_temp, effective_target),
                    )
                    await self._async_apply_active_fan_speed(eid, power_fraction)
                else:
                    await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": "off"})
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pytest tests/control/test_fan_mode.py -k "test_mpc_apply_heating_sets_fan_speed_when_enabled or test_mpc_apply_heating_no_fan_speed_when_disabled or test_mpc_apply_heating_fan_speed_skips_unsupported_label or test_mpc_apply_heating_fan_speed_redundant_skip" -v`
Expected: PASS (4 tests)

- [ ] **Step 5: Run the full fan_mode test file to check for regressions**

Run: `pytest tests/control/test_fan_mode.py -v`
Expected: PASS, all tests including pre-existing idle-path tests.

- [ ] **Step 6: Commit**

```bash
git add custom_components/roommind/control/mpc_controller.py tests/control/test_fan_mode.py
git commit -m "feat: apply active fan-speed control during heating"
```

---

### Task 5: Wire into cooling branch

**Files:**
- Modify: `custom_components/roommind/control/mpc_controller.py` — cooling AC loop (around line 1557-1567)
- Test: `tests/control/test_fan_mode.py`

- [ ] **Step 1: Write the failing test**

Append to `tests/control/test_fan_mode.py`:

```python
# ---------------------------------------------------------------------------
# async_apply — active fan-speed control (cooling)
# ---------------------------------------------------------------------------


@pytest.mark.asyncio
async def test_mpc_apply_cooling_sets_fan_speed_when_enabled():
    """AC with active_fan_control=True gets set_fan_mode during cooling, banded from power_fraction."""
    _last_commands.clear()
    _previous_fan_speeds.clear()
    hass = build_hass()
    state = MagicMock()
    state.state = "cool"
    state.attributes = {
        "hvac_modes": ["cool", "off"],
        "fan_modes": ["auto", "low", "medlow", "medhigh", "high"],
        "fan_mode": "low",
        "temperature": 24.0,
    }
    hass.states.get = MagicMock(return_value=state)

    room = make_room(thermostats=[], acs=["climate.ac1"])
    room["devices"] = [
        {
            "entity_id": "climate.ac1",
            "type": "ac",
            "role": "auto",
            "heating_system_type": "",
            "active_fan_control": True,
        }
    ]
    model_mgr = RoomModelManager()
    ctrl = MPCController(
        hass,
        room,
        model_manager=model_mgr,
        outdoor_temp=30.0,
        settings={},
        has_external_sensor=True,
    )
    await ctrl.async_apply("cooling", 23.0, power_fraction=0.6)

    calls = hass.services.async_call.call_args_list
    fan_calls = [c for c in calls if c[0][1] == "set_fan_mode"]
    assert len(fan_calls) == 1
    assert fan_calls[0][0][2] == {"entity_id": "climate.ac1", "fan_mode": "medhigh"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/control/test_fan_mode.py -k test_mpc_apply_cooling_sets_fan_speed_when_enabled -v`
Expected: FAIL — `assert len(fan_calls) == 1` (0 calls made, cooling branch doesn't call the new method yet)

- [ ] **Step 3: Wire into the cooling branch**

Find this block (around line 1557-1567):

```python
            for eid in self.acs:
                if eid in _forced_off:
                    continue
                ha_t = ha_cool_direct if eid in self._direct_eids else ha_target
                await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": "cool"})
                await self._call(
                    "set_temperature",
                    {"entity_id": eid, "temperature": ha_t, "hvac_mode": "cool"},
                    temp_intent="cool",
                    deadband=self._proportional_deadband(eid, current_temp, effective_target),
                )
```

Add the fan-speed call after `set_temperature`:

```python
            for eid in self.acs:
                if eid in _forced_off:
                    continue
                ha_t = ha_cool_direct if eid in self._direct_eids else ha_target
                await self._call("set_hvac_mode", {"entity_id": eid, "hvac_mode": "cool"})
                await self._call(
                    "set_temperature",
                    {"entity_id": eid, "temperature": ha_t, "hvac_mode": "cool"},
                    temp_intent="cool",
                    deadband=self._proportional_deadband(eid, current_temp, effective_target),
                )
                await self._async_apply_active_fan_speed(eid, power_fraction)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/control/test_fan_mode.py -k test_mpc_apply_cooling_sets_fan_speed_when_enabled -v`
Expected: PASS

- [ ] **Step 5: Run the full fan_mode test file to check for regressions**

Run: `pytest tests/control/test_fan_mode.py -v`
Expected: PASS, all tests.

- [ ] **Step 6: Commit**

```bash
git add custom_components/roommind/control/mpc_controller.py tests/control/test_fan_mode.py
git commit -m "feat: apply active fan-speed control during cooling"
```

---

### Task 6: Full regression run

**Files:** none (verification only)

- [ ] **Step 1: Run the entire test suite**

Run: `pytest -v`
Expected: PASS, no failures anywhere in the suite (in particular `tests/control/` and `tests/test_websocket_api.py` and `tests/utils/`).

- [ ] **Step 2: Run lint/type checks**

Run: `ruff check custom_components/roommind/control/mpc_controller.py custom_components/roommind/utils/device_utils.py custom_components/roommind/websocket_api.py custom_components/roommind/const.py`
Run: `mypy custom_components/roommind/control/mpc_controller.py custom_components/roommind/utils/device_utils.py`
Expected: no errors. Fix any issues found and re-run before proceeding.

- [ ] **Step 3: Commit if any lint/type fixes were needed**

```bash
git add -A
git commit -m "fix: lint/type cleanup for active fan-speed control"
```

(Skip this step entirely if Step 2 had no findings.)

---

### Task 7 (manual, not code): Migrate Josh's live Home Assistant config

This task has no code changes — it is the Home Assistant config migration described in the design spec's "Migration off the existing automation" section. Do this only after Tasks 1-6 are merged and Josh has confirmed the new behavior works correctly on his real aircon for a few cycles.

- [ ] **Step 1:** Confirm `climate.aircon`'s fan speed changes correctly during a real heat/cool cycle with `active_fan_control: true` set in RoomMind's room config for that device (via the RoomMind frontend), while `automation.aircon_fan_auto_control` is still enabled (so there's no gap in coverage during verification).
- [ ] **Step 2:** Disable `automation.aircon_fan_auto_control` in Home Assistant (turn off, don't delete yet).
- [ ] **Step 3:** Observe at least one full heat or cool cycle with the automation disabled and RoomMind as sole fan-speed driver.
- [ ] **Step 4:** After a few days of confirmed-good behavior, delete `automation.aircon_fan_auto_control` outright.
