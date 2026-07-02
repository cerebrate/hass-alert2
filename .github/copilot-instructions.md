# Copilot Instructions for hass-alert2

## Project Overview

Alert2 is a Home Assistant custom integration (`custom_components/alert2`) that provides
advanced alerting and notification capabilities. It replaces the built-in `alert` integration
with richer templating, event-based firing, generators, supersede logic, and UI management.

Current version: **1.20.1**  
Domain constant: `alert2`  
Generator domain: `alert2generator`  

---

## Repository Layout

```
custom_components/alert2/
  __init__.py       - Entry point, Alert2Data central manager, service registration
  entities.py       - All entity classes (AlertGenerator, EventAlert, ConditionAlert, AlertBase)
  config.py         - Voluptuous schemas + custom template validators
  config_flow.py    - Minimal ConfigFlow (single-instance, no UI options)
  binary_sensor.py  - Startup status sensor (binary_sensor.alert2_ha_startup_done)
  ui.py             - UiMgr: REST API + WebSocket handlers + persistent UI-config storage
  util.py           - Global helpers: report(), create_task(), DOMAIN, event name constants
  services.yaml     - Service descriptions (for HA UI)
tests/
  test_t1.py        - Main integration test suite (pytest + pytest-homeassistant-custom-component)
  test_ui.py        - UI-layer tests
  conftest.py       - Shared pytest fixtures
  dummy_server.py   - Mock notify entity used in tests
```

---

## Key Classes and Their Roles

### `Alert2Data` (`__init__.py`)
Central coordinator stored in `hass.data[DOMAIN]`. Owns:
- `alerts` — dict `{domain: {name: ConditionAlert}}`
- `tracked` — dict `{domain: {name: EventAlert}}`
- `generators` — dict `{name: AlertGenerator}`
- `component` — `EntityComponent` for `alert2` domain entities
- `sensorComponent` — `EntityComponent` for `sensor` domain (generators)
- `supersedeMgr` — `SupersedeMgr` graph for supersede relationships
- `supersedeNotifyMgr` — `SupersedeNotifyMgr` debounce/race logic
- `delayedNotifierMgr` — `DelayedNotifierMgr` for startup grace period
- `uiMgr` — `UiMgr` for UI-created alerts and settings storage

### `AlertBase(AlertCommon, RestoreEntity)` (`entities.py`)
Abstract base for both alert types. Handles:
- State persistence via `RestoreEntity` (last 15 min window)
- Notification control (enabled / disabled / snooze-until datetime)
- Throttling via `MovingSum` (sliding-window bucket counter)
- Reminder scheduling with configurable `reminder_frequency_mins` list
- Ack / unack lifecycle
- `_notify_pre_debounce()` → `supersedeNotifyMgr.processNotify()` → `_notify_post_debounce()`
- Both new entity-platform (`notify.send_message`) and legacy service notifiers are supported

### `EventAlert(AlertBase)` (`entities.py`)
Fires on triggers (HA trigger spec) plus optional condition template.  
State = ISO timestamp of last fire, or `"has never fired"`.  
Also used as a "tracked" alert—one with no trigger, fired only by `report()`.

### `ConditionAlert(AlertBase)` (`entities.py`)
State-tracked alert (`on`/`off`) driven by condition or threshold templates, or explicit
`condition_on`/`condition_off`/`trigger_on`/`trigger_off` split specs.  
State = `"on"` or `"off"`.  
Supports `supersedes`, `delay_on_secs`, `done_notifier`, `done_message`, `reminder_message`.

### `AlertGenerator(AlertCommon, SensorEntity)` (`entities.py`)
Wraps a Jinja2 `generator` template that produces a list. For each element the generator
dynamically creates/deletes child alerts (EventAlert or ConditionAlert).  
State = count of currently generated child alerts.

### `AlertCommon(Entity)` (`entities.py`)
Base for all three entity types. Fires HA bus events on lifecycle changes
(`alert2_create`, `alert2_delete`).

### `TriggerCond` (`entities.py`)
Wraps `async_initialize_triggers` for event alerts and condition alert trigger_on/off.
Evaluates an optional condition template before calling back.

### `Tracker` (`entities.py`)
Wraps `async_track_template_result` for condition/threshold templates.
Handles type coercion (Bool, Str, Float, List) and reports template errors.

### `UiMgr` (`ui.py`)
Provides:
- `Store`-backed persistence of UI-created alerts and global settings
- REST endpoint `POST /api/alert2/manageAlert` (load/create/update/delete/validate/search)
- WebSocket commands for alert listing, top-config, and entity navigation
- On reload, merges YAML base config with UI config

### `SupersedeMgr` (`__init__.py`)
Directed graph (supersedesMap / supersededByMap) for alert hierarchy.  
Provides transitive `supersedesSet()` / `supersededBySet()` traversal.  
Cycle detection on `addNode()`.

### `SupersedeNotifyMgr` (`__init__.py`)
Handles the race condition when two related alerts fire near-simultaneously.  
Uses `asyncio.Event` + `asyncio.wait_for(timeout=debounce_secs)` to delay notification
of lower-priority alerts, letting higher-priority alerts preempt them.

### `DelayedNotifierMgr` (`__init__.py`)
Buffers notifications during the startup grace period (`notifier_startup_grace_secs`, default 30 s)
to wait for notify platforms to finish loading.

---

## Configuration Schemas (`config.py`)

Schemas are built with `voluptuous`. Key schemas:

| Schema | Purpose |
|--------|---------|
| `DEFAULTS_SCHEMA` | Global defaults block in `alert2:` YAML |
| `SINGLE_TRACKED_SCHEMA` | Tracked (event-only) alert |
| `SINGLE_ALERT_SCHEMA_EVENT_NO_GEN` | Event alert without generator |
| `SINGLE_ALERT_SCHEMA_EVENT_GEN` | Event alert with generator |
| `SINGLE_ALERT_SCHEMA_CONDITION_NO_GEN` | Condition alert without generator |
| `SINGLE_ALERT_SCHEMA_CONDITION_GEN` | Condition alert with generator |
| `TOP_LEVEL_SCHEMA` | Top-level `alert2:` config block |

Custom validators: `boolTemplate`, `floatTemplate`, `jtemplate` (JTemplate), `jstringList`,
`jProtectedTrigger`, `jProtectedGeneratorTrigger`, `check_off`.

`JTemplate` extends `template_helper.Template` to move `domains` → `domains_lifecycle`
(preventing expensive state-tracking for generator templates) and adds the
`entity_regex` Jinja2 filter.

---

## Notification Flow

```
report() / trigger fires
    │
    ▼
AlertBase._notify_pre_debounce()
    │  builds message string, updates last_fired_time
    ▼
SupersedeNotifyMgr.processNotify()
    │  waits debounce_secs if superseded-by alerts exist
    ▼
AlertBase._notify_post_debounce()
    │  checks can_notify_now() (snooze / throttle / reminder timing)
    │  resolves notifier list via notifierTemplateToList()
    ▼
hass.services.async_call('notify', ...)
```

`NotificationReason` enum values: `Fire`, `ReminderOn`, `StopFiring`, `Summary`,
`SnoozeEnded`, `ReminderToAck`.

---

## Event Bus Integration

All HA bus events fired by Alert2:

| Constant | Value | When |
|----------|-------|------|
| `EVENT_TYPE` | `alert2_report` | `report()` called—drives alert processing |
| `EVENT_ALERT2_CREATE` | `alert2_create` | Entity added to HA |
| `EVENT_ALERT2_DELETE` | `alert2_delete` | Entity removed from HA |
| `EVENT_ALERT2_FIRE` | `alert2_alert_fire` | EventAlert fires |
| `EVENT_ALERT2_ON` | `alert2_alert_on` | ConditionAlert turns on |
| `EVENT_ALERT2_OFF` | `alert2_alert_off` | ConditionAlert turns off |
| `EVENT_ALERT2_ACK` | `alert2_alert_ack` | Alert acknowledged |
| `EVENT_ALERT2_UNACK` | `alert2_alert_unack` | Alert unacknowledged |

---

## Registered Services

| Service | Handler | Notes |
|---------|---------|-------|
| `alert2.report` | `Alert2Data.handle_service_report` | Fire event alert externally |
| `alert2.ack_all` | `Alert2Data.ackAll` | Ack all active alerts |
| `alert2.reload` | `Alert2Data.reload_service_handler` | Reload YAML without restart |
| `alert2.notification_control` | `AlertBase.async_notification_control` | Entity service: enable/disable/snooze |
| `alert2.ack` | `AlertBase.async_ack` | Entity service |
| `alert2.unack` | `AlertBase.async_unack` | Entity service |
| `alert2.toggle_ack` | `AlertBase.async_toggle_ack` | Entity service |
| `alert2.manual_on` | `ConditionAlert.async_manual_on` | Entity service |
| `alert2.manual_off` | `ConditionAlert.async_manual_off` | Entity service |
| `alert2.get_display_msg` | `AlertBase.async_get_display_msg` | Entity service, returns response |

---

## Internal Error Reporting

`util.report(domain, name, message)` fires `EVENT_TYPE` on the HA event bus, which is
handled by `Alert2Data.handle_event_report`. Three built-in internal alerts are
auto-declared unless `skip_internal_errors: true`:

- `alert2.error` — internal errors
- `alert2.warning` — internal warnings
- `alert2.global_exception` — unhandled asyncio exceptions (throttled 20 per 60 min by default)

Use `reportIfSafe()` inside `alert2.error` itself to avoid infinite recursion.

---

## State Persistence

`AlertBase` extends `RestoreEntity`. State is dumped every 15 minutes by HA.
Restored fields: `last_notified_time`, `last_tried_notify_time`, `last_fired_time`,
`last_fired_message`, `last_ack_time`, `fires_since_last_notify`, `notified_max_on`,
`reminders_since_fire`, `notification_control`.

---

## Testing

Tests use `pytest` + `pytest-homeassistant-custom-component`.

```bash
# Install deps
pip install -r requirements_test.txt

# Run tests
pytest tests/
```

- `test_t1.py` covers all core alert types, generators, supersede, throttle, snooze, reload.
- `test_ui.py` covers the REST/WebSocket UI layer.
- `conftest.py` provides `hass`, `service_calls`, and `MockConfigEntry` fixtures.
- `alert2.gGcDelaySecs = 0.1` is set in tests to speed up entity registry GC.
- The `auto_check_empty_calls` fixture asserts no unprocessed notify calls remain after each test.

---

## Important Conventions

1. **All entity IDs** are generated by `getPreferredEntityId(domain, name)` →
   `alert2.<slugify(domain_name)>`.
2. **`domain` + `name`** is the logical key for every alert. Never use only `entity_id` as a key.
3. **Generator alerts** use `rawConfig` (not processed config) when constructing child alerts
   to avoid double-interpretation of template fields.
4. **Template self-references** are blocked in `Tracker._result_cb()` to avoid feedback loops.
5. **`JTemplate`** must be used for generator templates (adds `entity_regex` filter and moves
   domains to `domains_lifecycle`). Use `cv.template` for regular alert templates.
6. **Startup ordering**: `async_setup` runs before `async_setup_entry`. Both call `init2()`.
   `init2()` is also called on `reload`. Guards prevent double initialization.
7. **Notifier detection**: `notifierExists()` checks both new entity-platform notifiers
   (`notify.*` entity registry entries) and legacy service-based notifiers.
8. **`create_task()` vs `create_background_task()`**: Both wrap HA task creation with
   exception-reporting and tracking in `global_tasks`. Prefer `create_background_task` for
   loops; `create_task` for one-shot work.

---

## Common Pitfalls

- The `condition` / `threshold` fields are mutually exclusive with `condition_on`/`condition_off`.
  `check_off()` validator enforces this.
- When modifying supersede logic, check both `supersedesMap` and `supersededByMap` for consistency.
- `_notify_pre_debounce` must always be followed by `async_write_ha_state()` and `reminder_check()`
  at the call site—it does not do this itself.
- `loadAlertBlock` in `__init__.py` processes alerts in two stages (supersedes registration first,
  then entity declaration) to handle forward references.
- UI config is layered on top of YAML config. Merging happens in `Alert2Data.noteUiUpdate()`.
  Do not mutate `rawYamlBaseTopConfig` directly.
