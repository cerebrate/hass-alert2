# Alert2 Architecture

> **Version:** 1.20.1  
> **Repository:** https://github.com/redstone99/hass-alert2

## What Is Alert2?

Alert2 is a [Home Assistant](https://www.home-assistant.io/) custom integration that provides
advanced alerting and notification management. It is a spiritual successor to the built-in
`alert` integration, adding native event-based alerts, Jinja2 template support, dynamic
alert generators, supersede/priority logic, snooze/throttle controls, and a companion web UI card.

---

## High-Level Architecture

```
configuration.yaml          UI card (browser)
      │                           │
      │  alert2: ...              │  REST + WebSocket
      ▼                           ▼
┌──────────────────────────────────────────┐
│            Alert2Data                    │  ← hass.data["alert2"]
│  (central coordinator, __init__.py)      │
│                                          │
│  ┌──────────┐  ┌──────────────────────┐  │
│  │  UiMgr   │  │  SupersedeMgr        │  │
│  │ (ui.py)  │  │  (graph of alerts)   │  │
│  └──────────┘  └──────────────────────┘  │
│  ┌──────────────────┐  ┌──────────────┐  │
│  │ DelayedNotifier  │  │ SupersedeNot │  │
│  │     Mgr          │  │    ifyMgr    │  │
│  └──────────────────┘  └──────────────┘  │
└──────────────────────────────────────────┘
         │                    │
         │ creates             │ creates
         ▼                     ▼
  EventAlert entities    ConditionAlert entities
  (alert2.* domain)      (alert2.* domain)
         │
         │ may create
         ▼
  AlertGenerator entities
  (sensor.* domain)
         │
         │ dynamically creates/deletes
         ▼
  More EventAlert / ConditionAlert entities
```

---

## Source Files

| File | Purpose |
|------|---------|
| `__init__.py` | Integration entry point; `Alert2Data` central manager; service registration; reload logic |
| `entities.py` | All entity classes (`EventAlert`, `ConditionAlert`, `AlertGenerator`, `AlertBase`, `AlertCommon`) plus `TriggerCond`, `Tracker`, `MovingSum`, `SupersedeMgr`, `SupersedeNotifyMgr` |
| `config.py` | Voluptuous validation schemas; custom validators (`boolTemplate`, `jtemplate`, `JTemplate`) |
| `config_flow.py` | Minimal `ConfigFlow` — single instance, no user-facing options screen |
| `binary_sensor.py` | Creates `binary_sensor.alert2_ha_startup_done` for startup detection |
| `ui.py` | `UiMgr` — persistent `Store`, REST endpoint, WebSocket handlers for the UI card |
| `util.py` | `report()`, `create_task()`, `DOMAIN`, HA bus event name constants |
| `services.yaml` | Human-readable service descriptions shown in the HA developer tools |

---

## Alert Types

### 1. Event Alert (`EventAlert`)

Fires in response to a discrete event. Think of it as a one-shot notification trigger.

- **Driven by**: a HA trigger spec (`trigger:`) plus an optional `condition:` template.
- **State**: the ISO timestamp of the last firing, or `"has never fired"`.
- **Special variant — Tracked Alert**: an `EventAlert` with no trigger, fired only by
  the `report()` Python API. Used internally for `alert2.error`, `alert2.warning`, etc.

### 2. Condition Alert (`ConditionAlert`)

Tracks a boolean condition over time. Think of it as a continuous sensor that turns on and off.

- **Driven by**: one of:
  - `condition:` — a single template that controls both on and off
  - `threshold:` — numeric hysteresis (value, hysteresis, optional min/max)
  - `condition_on:` / `condition_off:` — explicit split templates
  - `trigger_on:` / `trigger_off:` — event-triggered transitions
  - `manual_on` / `manual_off` — service-call driven
- **State**: `"on"` or `"off"`.
- **Notifications**: on turn-on (`Fire`), periodic reminders while on (`ReminderOn`),
  on turn-off (`StopFiring`).
- **Extras**: `supersedes`, `delay_on_secs`, `done_message`, `ack_required`.

### 3. Alert Generator (`AlertGenerator`)

Dynamically creates and destroys child alerts based on a list-valued template.

- **Driven by**: a `generator:` Jinja2 template that produces a list of elements.
- **Entity type**: `sensor.*` (not `alert2.*`).
- **State**: count of currently generated child alerts.
- **Use case**: generate one alert per device, per zone, per entity matching a regex, etc.
- **Special filter**: adds the `entity_regex` Jinja2 filter for matching entity IDs.

---

## Notification Pipeline

When an alert fires (or turns on/off), notifications flow through several layers:

```
1. Alert fires
       │
       ▼
2. AlertBase._notify_pre_debounce()
       │  • Assembles message string
       │  • Updates last_fired_time / fires_since_last_notify
       │
       ▼
3. SupersedeNotifyMgr.processNotify()
       │  • If this alert is superseded by another alert that just fired,
       │    mark notification as superseded (no send).
       │  • Otherwise optionally wait up to supersede_debounce_secs for
       │    a superseding alert to fire before sending.
       │
       ▼
4. AlertBase._notify_post_debounce()
       │  • Calls can_notify_now() — checks snooze, throttle, reminder timing
       │  • Resolves notifier list (template or literal)
       │  • Builds notification args (message, title, target, data)
       │
       ▼
5. hass.services.async_call('notify', ...)
       │  • Supports both new entity-platform notifiers (notify.*)
       │  • and legacy service-based notifiers
       │  • Handles persistent_notification grouping modes
```

---

## Supersede Logic

Alerts can declare that they *supersede* lower-priority alerts. If a higher-priority alert
is already on when a lower-priority alert fires, the lower-priority notification is suppressed.

- **Configuration**: `supersedes: [{domain: ..., name: ...}]` on a `ConditionAlert`.
- **Graph**: `SupersedeMgr` maintains a directed graph (`supersedesMap`, `supersededByMap`)
  with transitive traversal and cycle detection.
- **Race handling**: `SupersedeNotifyMgr` uses an `asyncio.Event` + timeout to handle
  near-simultaneous firings where ordering is nondeterministic.

---

## Throttling

The `throttle_fires_per_mins` setting limits how many notifications are sent in a given window.

- Implemented via `MovingSum` — a sliding-window bucket counter (10 buckets).
- When the threshold is exceeded, Alert2 sends one "Throttling started" notification, then
  suppresses further notifications until the window clears.
- When throttling ends, a "Throttling ending" notification is sent.

---

## Reminder Scheduling

Alerts can send periodic reminder notifications while a condition remains active.

- Configured via `reminder_frequency_mins` — a list of intervals (can escalate over time).
- The first reminder uses `reminder_frequency_mins[0]`, subsequent ones use `[1]`, `[2]`, etc.
- `AlertBase.reminder_check()` is the central scheduler — it cancels/reschedules on every
  state change, ack, snooze, or throttle transition.

---

## Notification Control

Each alert entity supports three notification states (exposed as `notification_control`
attribute and controllable via the `notification_control` service):

| State | Meaning |
|-------|---------|
| `"enabled"` | Normal operation |
| `"disabled"` | All notifications suppressed indefinitely |
| ISO datetime | Snoozed until that time |

---

## Generator System

Alert generators allow one YAML stanza to produce multiple alerts dynamically:

```yaml
alert2:
  alerts:
    - domain: myapp
      name: "{{ genElem }} fault"
      generator: "{{ states.sensor | selectattr('attributes.group', 'eq', 'mygroup') | map(attribute='entity_id') | list }}"
      condition: "{{ is_state(genEntityId, 'fault') }}"
      message: "Sensor {{ genEntityId }} is faulted"
```

- `genElem` / `genEntityId` / `genIdx` / `genGroups` / `genRaw` are injected template variables.
- The generator list is re-evaluated whenever the tracked entities change state.
- Child alerts are created/destroyed incrementally (not recreated on every list change).

---

## Configuration Layering

Settings are merged in this priority order (highest wins):

```
Built-in defaults
    ↓ overridden by
YAML (configuration.yaml: alert2:)
    ↓ overridden by
UI-created settings (stored in .storage/alert2.ui)
```

The merge happens in `Alert2Data.init2()` / `Alert2Data.noteUiUpdate()`.

Alert-level values then override the effective defaults for each alert instance. This includes voice fields like `voice_proxies_enabled`, `voice_snooze_minutes`, and `voice_event_latch_secs`.

---

## UI Integration

Alert2 provides a companion UI card (`hass-alert2-ui`). The backend serves:

- **REST**: `POST /api/alert2/manageAlert` — create, read, update, delete, validate, search alerts
- **WebSocket**: commands for listing all alerts, reading top-level config, entity navigation

Alert configurations created or edited via the UI are persisted using HA's `homeassistant.helpers.storage.Store`
under the key `alert2.ui`. On reload, these are merged with the YAML config.

---

## Startup Lifecycle

```
1. async_setup() called with YAML config
2. Alert2Data created and stored in hass.data["alert2"]
3. binary_sensor platform loaded asynchronously
4. init2() called:
   a. Built-in defaults applied
   b. YAML defaults applied
   c. UI settings loaded from storage and merged
   d. Internal alerts (alert2.error, alert2.warning, alert2.global_exception) declared
   e. DelayedNotifierMgr started (30s grace period by default)
   f. asyncio exception handler installed
   g. HA event listeners registered
   h. Services registered
5. processConfig() called:
   a. YAML alerts/tracked loaded
   b. UI-created alerts loaded
   c. Entity registry GC scheduled (10s delay)
6. HA fires EVENT_HOMEASSISTANT_STARTED
7. Alerts with early_start=false begin watching their conditions/triggers
```

---

## Entity ID Convention

All alert entities have IDs in the form:

```
alert2.<slugified(domain + "_" + name)>
```

For example, domain `myapp` + name `sensor fault` → `alert2.myapp_sensor_fault`.

Generator sensor entities live in the `sensor.*` namespace.

---

## Testing

Tests use `pytest` with the `pytest-homeassistant-custom-component` plugin.

```bash
pip install -r requirements_test.txt
pytest tests/
```

Key test patterns:
- `do_reload()` helper triggers a config reload mid-test
- `service_calls` fixture captures all `hass.services.async_call` invocations
- `auto_check_empty_calls` fixture ensures no unchecked notifications remain after each test
- `dummy_server.py` provides a `NotifyEntity` mock

---

## External Developer API

Other integrations can programmatically fire Alert2 event alerts:

```python
from custom_components.alert2 import declareEventMulti

# Register event alerts for your integration
await declareEventMulti([
    {'domain': 'my_integration', 'name': 'connection_lost'},
    {'domain': 'my_integration', 'name': 'parse_error'},
])

# Fire an alert
from custom_components.alert2.util import report
report('my_integration', 'connection_lost', 'Lost connection to device XYZ')
```

`report()` is synchronous and thread-safe — it fires the `alert2_report` event via
`loop.call_soon_threadsafe()` so it can be called from worker threads.
