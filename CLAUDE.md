# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this Home Assistant configuration repository.

## Project Overview

This is a sophisticated German-language Home Assistant configuration for a smart home with extensive automation, energy monitoring, and multi-protocol device integration. The setup emphasizes:

- **Local-first control** (Zigbee, LAN, MQTT over cloud)
- **Energy monitoring** with solar production tracking
- **Circadian rhythm lighting** (Dynamic Lighting via Blackshome blueprint)
- **Room-based automation** with presence detection
- **Multi-user presence** for Rafael and Alex

**Location**: Fürth, Germany
**Language**: German entity/automation naming throughout

---

## Quick Reference

### Environment Constraints

- **No Docker access**: Claude Code has no access to the Docker host. Do not suggest `docker exec` commands for validation, reload, or log viewing.
- **HA REST API** is available via `$HA_URL` (the HA IP, not `homeassistant.local`: the sandbox proxy cannot resolve mDNS) and `$HA_TOKEN` (exported once per session by the SessionStart hook `~/.claude/hooks/ha-token-env.sh` from 1Password). Inside the Bash sandbox curl must go through the proxy: always pass `--noproxy ''`, otherwise the sandbox `NO_PROXY` for 192.168.0.0/16 forces a direct connect that is blocked.
- YAML syntax validation can be done locally with `yamllint`.

### Key Commands

```bash
# Check YAML syntax (local, works without Docker)
yamllint automations_new/**/*.yaml

# Git operations
git -C /Volumes/config status
git -C /Volumes/config diff
git -C /Volumes/config log --oneline -10

# HA REST API (uses $HA_URL and $HA_TOKEN from env)
# Check config
curl -s --noproxy '' -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/config" | jq .version

# Reload automations
curl -s --noproxy '' -X POST -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/services/automation/reload"

# Reload scripts
curl -s --noproxy '' -X POST -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/services/script/reload"

# Call a service (example: script.fade_volume)
curl -s --noproxy '' -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" \
  "$HA_URL/api/services/script/fade_volume" \
  -d '{"target_player":"media_player.homepod_kueche","target_volume":0.20,"duration":30,"curve":"linear"}'

# View recent logs (via WebSocket — /api/error_log returns 404 in HA 2026.3+)
python3 << 'PYEOF'
import asyncio, json, websockets, os
async def get_logs():
    token = os.environ["HA_TOKEN"]
    url = os.environ["HA_URL"].replace("http://","ws://").replace("https://","wss://")
    async with websockets.connect(f"{url}/api/websocket") as ws:
        await ws.recv()
        await ws.send(json.dumps({"type":"auth","access_token":token}))
        msg = json.loads(await ws.recv())
        if msg["type"] != "auth_ok": return print("Auth failed")
        await ws.send(json.dumps({"id":1,"type":"system_log/list"}))
        msg = json.loads(await ws.recv())
        for e in msg.get("result",[])[:30]:
            c = f" (x{e['count']})" if e.get("count",1) > 1 else ""
            print(f"[{e['level'].upper()}] {e.get('name','')}: {e['message'][:150]}{c}\n")
asyncio.run(get_logs())
PYEOF
```

### Important Paths

| Path | Purpose |
|------|---------|
| `configuration.yaml` | Main entry point |
| `automations_new/` | All automations (34 files) |
| `custom_components/` | 24 custom integrations |
| `blueprints/` | 162 blueprints (5 automation + 157 switch_manager) |
| `scripts.yaml` | Reusable scripts |
| `themes/` | 6 UI themes |
| `secrets.yaml` | Sensitive data (never commit) |

---

## Architecture & Structure

### Configuration Architecture

The configuration uses a **modular split-configuration pattern** with `!include` directives:

```
configuration.yaml
├── homeassistant:           # Core settings, auth, MFA
├── http:                    # Proxy & security
├── recorder:                # Database (14-day retention)
├── logger:                  # Log levels
│
├── [Inline Templates]       # 450+ lines of template sensors
├── [Input Helpers]          # input_boolean, input_datetime, etc.
├── [Utility Meters]         # Energy tracking
│
├── automation: !include_dir_merge_list automations_new/
├── script: !include scripts.yaml
├── scene: !include scenes.yaml
├── sensor: !include sensors.yaml
├── binary_sensor: !include binary_sensors.yaml
├── light: !include lights.yaml
├── climate: !include climates.yaml
├── notify: !include notifies.yaml
├── timer: !include timers.yaml
├── group: !include groups.yaml
└── frontend: !include_dir_merge_named themes
```

### Automations Directory Structure

```
automations_new/                # 34 files
├── areas/                    # 9 room files
│   ├── arbeitszimmer.yaml   # Office
│   ├── badezimmer.yaml      # Bathroom
│   ├── balkon.yaml          # Balcony
│   ├── flur.yaml            # Hallway
│   ├── kinderzimmer.yaml    # Kids' room
│   ├── kueche.yaml          # Kitchen
│   ├── schlafzimmer.yaml    # Bedroom
│   ├── waschzimmer.yaml     # Laundry
│   └── wohnzimmer.yaml      # Living room
│
├── lighting/                 # 4 lighting files
│   ├── adaptive_lighting.yaml    # Manual control detection
│   ├── day_night.yaml            # Time-based transitions, wake-up, sleep mode
│   ├── motion_sensors.yaml       # Motion triggers
│   └── nachtlicht.yaml           # Night light mode
│
├── climate/                  # 2 climate files
│   ├── heating_cooling.yaml # AC, fans, floor heating
│   └── rollos.yaml          # Blind/roller control
│
├── appliances/               # 5 appliance files
│   ├── air_quality.yaml     # Air purifier, humidifier
│   ├── cleaning.yaml        # Robbi vacuum
│   ├── laundry.yaml         # Washer, dryer
│   ├── other.yaml           # Misc devices
│   └── prusa.yaml           # 3D printer
│
├── helpers/                  # 5 helper files
│   ├── automation.yaml      # Automation management, wakeup music
│   ├── energy.yaml          # Energy tracking
│   ├── motion_sync_on_restart.yaml
│   ├── presence.yaml        # Presence detection
│   └── system.yaml          # System checks
│
├── notifications/            # 4 notification files
│   ├── alerts.yaml          # Alert triggers, NAS webhook
│   ├── energy_surplus.yaml  # Solar surplus hints
│   ├── environment.yaml     # Environmental warnings, energy reports
│   └── monitoring.yaml      # Health checks
│
├── scenes/                   # 3 scene files
│   ├── activities.yaml      # Guest, Party modes
│   ├── entertainment.yaml   # Gaming, TV modes
│   └── media_volume.yaml    # Per-scene volume handling
│
└── system/                   # 2 system files
    ├── core.yaml            # System boot/restart, AL startup settings
    └── sabnzbd_nightmode.yaml
```

---

## Naming Conventions

### Entity Naming Pattern

Format: `domain.room_device_type` or `domain.room_description`

**Room Identifiers (German)**:
| German | English | Abbreviation |
|--------|---------|--------------|
| wohnzimmer | Living room | wz |
| schlafzimmer | Bedroom | sz |
| kinderzimmer | Kids' room | kz |
| arbeitszimmer | Office | az |
| kueche | Kitchen | ku |
| badezimmer | Bathroom | bad |
| flur | Hallway | fl |
| waschzimmer | Laundry | wz |
| balkon | Balcony | bal |
| ankleide | Dressing room | ank |

**Examples**:
```yaml
light.wohnzimmer_wand_alle      # Living room wall lights (group)
switch.heizung_schlafzimmer     # Bedroom heating switch
sensor.arbeitszimmer_temperatur # Office temperature
binary_sensor.flur_motion       # Hallway motion sensor
```

### Light Group Naming

All grouping is HA-side (`lights.yaml`); Zigbee2MQTT groups are intentionally not used (caused issues):
1. **Room groups**: `light.{room}_alle` or `alle_{room}_lichter`
2. **Night lights**: `alle_{room}_nachtlichter`

### Automation Naming

Format: `"Category » Description"` (German)

Examples:
- `"Licht » Wohnzimmer Motion"` - Light motion automation
- `"Klima » Heizung Boost"` - Climate heating boost
- `"Scene » Gaming"` - Gaming scene activation
- `"Helper » Motion Sync"` - Helper automation

---

## Device Integration Protocols

### Primary Protocols

| Protocol | Usage | Components |
|----------|-------|------------|
| **Zigbee (ZHA)** | Lights, switches, sensors | Native ZHA integration |
| **MQTT** | Device communication | Meross, Shelly, Qingping, Ecoflow |
| **Local HTTP/UDP** | Smart devices | Meross LAN, Govee LAN, TP-Link |
| **HomeKit** | Apple ecosystem | HomeKit Controller |
| **Matter** | Modern devices | Native Matter support |

### Key Custom Components (24 total)

**Heavily Used**:
- `powercalc` - Energy calculation for all devices
- `meross_lan` - Meross smart plugs/switches (local)
- `xiaomi_miot` - Xiaomi vacuum, AC, air purifiers

**Active Integrations**:
- `waste_collection_schedule` - Garbage collection (Fürth)
- `dual_smart_thermostat` - Climate control
- `nuki_ng` - Nuki smart lock
- `dwd` - German weather service
- `govee_lan` - Govee LED lights (local)
- `midea_ac` - Midea air conditioning
- `ecoflow_cloud` - EcoFlow power stations
- `hoymiles_wifi` - Solar inverter monitoring

**Lighting**:
- `adaptive_lighting` - Circadian rhythm lighting (10 rooms configured)

---

## Lighting System

### Adaptive Lighting (Circadian)

The system uses the **Adaptive Lighting** custom component for circadian rhythm lighting across all 10 rooms.

**Configuration**: two files, and both are load-bearing for different reasons.

`adaptive_lighting.yaml` **must exist and stay included**. Deleting it on 2026-08-28 destroyed all ten config entries and every AL switch on the next restart, and the fix was to restore both file and `!include` and restart again. The values inside still only apply at first import, so editing them changes nothing, but the file anchors the config entries.

`automations_new/system/core.yaml`, automation `System » Adaptive Lighting Settings nach Neustart`, is where the live values come from. It re-applies them ~30 s after every start. Change settings there.

**Room Configuration**:
| Room | Lights | Reset Time | Notes |
|------|--------|------------|-------|
| Wohnzimmer | 18 lights | 1 hour | Living spaces |
| Küche | 6 lights | 1 hour | Kitchen |
| Schlafzimmer | 3 lights | 1 hour | Bedroom |
| Kinderzimmer | 2 lights | 1 hour | Kids' room |
| Badezimmer | 4 lights | 15 min | Bathroom |
| Waschzimmer | 1 light | 2 hours | Laundry |
| Arbeitszimmer | 5 lights | 1 hour | Office |
| Ankleide | 2 lights | 2 hours | Dressing room |
| Flur | 9 lights | 15 min | Hallway |
| Balkon | 3 lights | 2 hours | Balcony |

**Key Settings** (consistent across rooms):
```yaml
interval: 300              # 5 min update cycle
transition: 10             # Short for IKEA compatibility
min_color_temp: 2200       # Warm (IKEA/Hue compatible)
max_color_temp: 4000       # Cool white maximum
sleep_brightness: 5        # Night mode default; per-room exceptions in system/core.yaml startup automation
sleep_rgb_or_color_temp: color_temp # Uniform 2200K everywhere: the 24 IKEA bulbs cannot do RGB, mixed moods are not acceptable
sleep_color_temp: 2200     # Warmest for sleep
take_over_control: true    # Manual control detection
detect_non_ha_changes: false  # Disabled — prevents random turn-ons with Z2M
skip_redundant_commands: false  # Disabled — prevents state desync with Z2M
include_config_in_attributes: false  # Disabled — reduces HA state overhead
intercept: true            # Intercept light.turn_on calls
brightness_mode: "tanh"    # Smooth brightness curve
```

**Entities per Room**:
- `switch.adaptive_lighting_{room}` - Enable/disable AL for room
- `switch.adaptive_lighting_sleep_mode_{room}` - Force sleep mode

### Yeelight CubeMatrix (3 Stück, LAN)

`light.yeelight_cubematrix_0xdc5475bbc814` (Gaming Licht, Arbeitszimmer), `light.yeelight_cubematrix_0xdc5475bd7828` (Uhrzeit, Wohnzimmer), `light.kinderzimmer_kinderzimmer_nachtlicht`. Sie hängen an den Steckdosen des Willkommenslaufs und gehen mit dem Strom von selbst an. `Licht » Yeelight nach Steckdose aus` (`helpers/presence.yaml`) schaltet Gaming Licht und Kinderzimmer Nachtlicht wieder aus, sobald sie nach `unavailable` als `on` gemeldet werden. Achtung: die Integration meldet nach dem Reconnect zuerst `off` und erst 0,5 bis 15 s später `on`; ein Trigger `unavailable -> on` feuert deshalb nie, und `light.turn_off` ist bei gemeldetem `off` ein No-op (Guard in HA core). Deshalb auf `on` warten, dann ausschalten. Nur Uhrzeit darf mit dem Strom leuchten, mit einer Ausnahme: während `input_boolean.schlafenszeit` bleibt das Kinderzimmer Nachtlicht an, weil `Licht » Kinderzimmer Nachtlichter bei Schlafenszeit` (`areas/kinderzimmer.yaml`) es einschaltet, sobald der Govee Sternenprojektor `light.h60b0` ausgeht (Mitternacht oder App-Timer); bis dahin leuchtet nur der Govee. Eine Automation, die eine der beiden anderen einschalten will, muss das nach dem Verfügbarwerden tun.

Der Govee H60B0 (Star Light Projector, `govee_light_local`) bekommt bewusst `light.turn_on` ohne Parameter: der Sternenmodus ist nur in der Govee-App wählbar, die lokale Schnittstelle kennt nur zwölf Szenen. Die Lampe lädt beim Einschalten die zuletzt in der App gesetzte Szene. Kein `brightness`, `rgb_color` oder `effect` mitschicken, das überschreibt sie.

Bett, TV (`philips_light_15/16`) und Stahlträger gehen nur nachts an (`sun.sun` unter dem Horizont bzw. Nachtlicht-Pfad der Motion-Blueprints); tagsüber sind sie bewusst aus den Motion-Lichtlisten ausgenommen.

### Manual Control Detection

Template binary sensors track manual control state via AL's `manual_control` attribute:
```yaml
binary_sensor.{room}_manual_control
```

Detection logic (from `switch.adaptive_lighting_{room}` attribute):
```yaml
state: >-
  {{ (state_attr('switch.adaptive_lighting_wohnzimmer', 'manual_control') or []) | length > 0 }}
```

When manual control is detected:
- AL stops adapting those specific lights
- After `autoreset_control_seconds` (1-2 hours), control resets
- Scene automations can disable/re-enable AL per room

**Key Files**:
- `adaptive_lighting.yaml` - Anchors the 10 config entries, must not be deleted
- `automations_new/system/core.yaml` - Live AL configuration, applied at every start
- `automations_new/lighting/adaptive_lighting.yaml` - AL-related automations
- `automations_new/lighting/day_night.yaml` - Time-based transitions

---

## Scene System

### Active Scenes

Managed via `input_boolean` helpers:

| Scene | Input Boolean | Purpose |
|-------|---------------|---------|
| Film/Movie | `input_boolean.movie` | TV/movie watching |
| Party | `input_boolean.party` | Party mode lighting |
| Kochen | `input_boolean.kochen` | Cooking mode |
| Essen | `input_boolean.essen` | Dining mode |
| Baden | `input_boolean.baden` | Bath mode |
| Sauna | `input_boolean.sauna` | Sauna mode |
| Arbeit | `input_boolean.arbeit` | Work mode |
| Gaming | `input_boolean.gaming` | Gaming mode |

### Scene Tracking Sensors

```yaml
sensor.active_scenes              # List of active scenes
binary_sensor.any_scene_active    # True if any scene on
binary_sensor.scene_conflict_detected  # Conflict warning
```

### Scene Scripts

```yaml
script.scene_enable_adaptive_lighting   # Enable circadian for room (with scene conflict check)
script.scene_disable_adaptive_lighting  # Disable circadian
script.scene_enable_motion_automations  # Re-enable motion triggers (with scene conflict check)
script.scene_disable_motion_automations # Disable motion triggers
script.al_set_manual_control            # Mark/clear AL manual control per room (mapping lives here)
script.scene_stop_rgb_effect            # Stop a Hue effect, reset to color_temp, turn lights off
script.restore_leselicht                # Restore the reading light if input_boolean.leselicht is on
script.movie_motion_restore             # Re-enable motion automations when the movie scene ends
```

The four `scene_*` scripts take an `exclude_scenes` list of `input_boolean` names and do nothing
while one of them is on. That check only sees input_booleans. A gate on a media_player (e.g. the
AppleTV guard around `automation.motion_schlafzimmer`) still has to sit at the call site.

---

## Climate Control

Defined in `automations_new/climate/heating_cooling.yaml`.

> **Currently stored away (as of 2026-08-27):** the Midea AC (`climate.ac`) and both Smartmi za4
> fans (`fan.schlafzimmer`, `fan.kinderzimmer`) are in the basement and will not go back up until
> summer 2027. Their entities are therefore absent, not broken. The automations already skip
> themselves when the device is gone (commit 2bd73d1), so everything below describes dormant logic
> and stays here for reference.

### Air Conditioning (`climate.ac`, Midea via `midea_ac`)

The AC sits in the Arbeitszimmer (office) and is steered by **"Helper » AC manual"**:

- **Hysteresis**: on at >= 25.5 C, off below 24.5 C (no flapping, no dead zone).
- **Window gates cooling**, not mode: windows closed -> `cool`, windows open -> `off` (cooling with open windows is pointless). There is **no steady-state `fan_only`** anymore (the Midea fan mode is useless for cooling).
- **Outdoor temperature gates cooling** below 20 C (`aussen_zu_kalt`): the room cools down on its own, the compressor is pointless. Source is `sensor.temperatur_aussen` (combined Aqara + Tuya balcony sensors), falling back to `weather.fuerth_bayern`. `sensor.ac_outdoor_temperature` is deliberately unused, it sits on the outdoor unit and picks up its waste heat. Two `numeric_state` triggers on the 20 C threshold wake the automation.
- **No delta block**: when it is cooler outside than in, cooling is *not* blocked, it only sends a push (`lueften_besser`, outdoor <= indoor - 2 C). HA cannot open the windows, so a hard block would just leave the room hot. The ventilation case is already covered by the window gate.
- **Cleanup run**: whenever cooling stops after **>= 10 min** of `cool`, the AC runs `fan_only` (silent) for **30 min** to dry the evaporator (mold protection), then `timer.ac_cleanup` turns it off via "Klima » AC Reinigungslauf beendet". Resuming `cool` cancels the timer. This is the only situation `fan_only` is used.
- **Presence gate** (`zone_anwesend`): Gaming or Schlafenszeit, an active MacBook, or AppleTV playing. **Arbeit deliberately does NOT count**: the flag is shared with Alex, who sits in the other room. No one home -> off.
- **One temperature source**: `sensor.arbeitszimmer_temperatur`, fallback `sensor.ac_indoor_temperature`. There is no mode-dependent switching; after the room swap the bed, the AC and the sensor share one room.
- **Setpoint** 24 C, or **22 C during Schlafenszeit**. Fan follows the distance to target (`medium` from delta 3, back to `silent` below 1.5); Schlafenszeit is always `silent`. Preset **`ieco`** whenever cooling.

> **iECO vs gear**: `preset ieco` and `select.ac_rate_select` (gear_50/75 power limit) are **mutually exclusive** on this Midea (setting one clears the other). Control runs via `ieco`; gear is only useful as a hard wattage cap (e.g. PV coupling).

### Override detection

`climate.ac` and `fan.schlafzimmer` have manual-override detection that pauses the automatic control for 2h (`input_boolean.ac_override`, `input_boolean.ventilator_schlafzimmer_override`). Both guards **ignore reconnects from `unavailable`** and the AC guard uses a **180s echo window** (Midea reports its state ~2 min late; without this the device echo is mistaken for a manual change).

### Fans (Smartmi za4 via `xiaomi_miot`, Cloud mode)

- `fan.schlafzimmer`: controlled by CO2 (`sensor.qingping_air_monitor_lite_co2_carbon_dioxide`) **and** temperature, speed = max of both demands.
- `fan.kinderzimmer`: temperature + radar presence, hysteresis.
- za4 preset is **`Natural Wind`** (not the old miio name `nature`). The za4 need **Cloud** connection mode in xiaomi_miot, the motor power command does not work in local mode.

### Known sensor defect

`sensor.qingping_air_monitor_lite_temperature` reads 4-8 C too low (variable, not a fixed offset). All temperature logic uses `sensor.arbeitszimmer_temperatur` instead; the Qingping is used for CO2 only. `sensor.temperatur_schlafzimmer`, which older notes name here, does not exist as an entity and is referenced nowhere.

### Heating

`switch.heizung_*` floor heating is predictive (morning/afternoon boost from forecast) with a boost timer, a 2h safety watchdog, and restart recovery. Heating uses forecast (`weather.fuerth_bayern` / `sensor.fuerth_daily`), not room temperature.

---

## Energy Monitoring

### Multi-Source Architecture

1. **Grid Monitoring**: 3-phase power meter via MQTT
   - `sensor.strommesser_phase_a/b/c_active_power`
   - `sensor.stromzahler_saldiert` (net balance)

2. **Solar Production**: Hoymiles WiFi inverter
   - DTU data via `hoymiles_wifi` component
   - Daily tracking: `sensor.ac_energy_daily`

3. **Device Power**: Powercalc integration
   - Automatic calculation for all lights
   - Smart plug measurements (Meross, Shelly, TP-Link)

4. **Utility Meters**: Daily/weekly/monthly cycles
   - `sensor.taglicher_netzbezug` (daily import)
   - `sensor.tagliche_einspeisung` (daily export)

---

## Input Helpers Reference

### Input Boolean (Scene/Mode Flags)
```yaml
input_boolean.movie, party, kochen, essen, baden, sauna, arbeit, gaming
input_boolean.frost_warning_sent
input_boolean.guest_or_away
```

### Input DateTime (Schedules)
```yaml
input_datetime.aufwachzeit_arbeitstag    # Workday wake time
input_datetime.aufwachzeit_freier_tag    # Weekend wake time
input_datetime.einschlafzeit_arbeitstag  # Workday sleep time
input_datetime.einschlafzeit_freier_tag  # Weekend sleep time
```

### Input Number
```yaml
input_number.saved_volume_*          # Volume before a scene, restored afterwards
input_number.al_tagesmax_helligkeit  # Daytime ceiling for Adaptive Lighting
```

**Removed on 2026-08-28:** twelve sliders that looked like thresholds and were read by nothing.
`ac_temp_hot/warm/comfortable`, `rollo_position_closed/partial/half/open` and
`fan_speed_low/medium/high/max` sat on the dashboard promising control that did not exist: the
climate automation computes `ziel_setpoint` and `ziel_speed` itself, and `rollos.yaml` uses
`position_west` and friends. Also gone: `input_boolean.test_morning_routine`,
`input_datetime.wakeup_routine_started` and `input_select.wakeup_routine_phase`, none of which
any automation ever read. There is no `input_select:` block in YAML any more;
`input_select.trockner` is a UI helper and untouched.

---

## Common Automation Patterns

### Pattern 1: Trigger ID Routing

```yaml
triggers:
  - trigger: state
    entity_id: light.example
    to: 'on'
    id: 'light-on'
  - trigger: state
    entity_id: light.example
    to: 'off'
    id: 'light-off'

actions:
  - choose:
    - conditions:
        - condition: trigger
          id: 'light-on'
      sequence: [...]
    - conditions:
        - condition: trigger
          id: 'light-off'
      sequence: [...]
```

### Pattern 2: Blueprint Usage

```yaml
- id: '1745924154398'
  alias: Motion » Esstisch
  use_blueprint:
    path: Blackshome/sensor-light.yaml
    input:
      motion_trigger: [...]
      light_switch:
        entity_id: light.wohnzimmer_esstisch_alle
      # Dynamic Lighting parameters
      include_dynamic_lighting: true
      dynamic_lighting_mode: sun_elevation
```

### Pattern 3: Manual Control Detection

```yaml
conditions:
  - condition: template
    value_template: >
      {{ trigger.to_state.context.parent_id is none or
         trigger.to_state.context.user_id is not none }}
```

### Pattern 4: Room Area Discovery

```yaml
variables:
  area_name: "{{ area_name(trigger.entity_id) }}"
  lights_in_area: >
    {{ expand(area_entities(area_name))
       | selectattr('domain', 'eq', 'light')
       | map(attribute='entity_id')
       | list }}
```

### Pattern 5: Presence-Based Actions

```yaml
conditions:
  - condition: state
    entity_id: person.rafael
    state: 'home'
  - condition: or
    conditions:
      - condition: state
        entity_id: binary_sensor.arbeitszimmer_motion
        state: 'on'
      - condition: state
        entity_id: binary_sensor.arbeitszimmer_presence
        state: 'on'
```

---

## Development Guidelines

### YAML Best Practices

1. **Use `!include` directives** for modular configuration
2. **Always include `availability` templates** for template sensors
3. **Store secrets in `secrets.yaml`** - never hardcode credentials
4. **Use German room names** consistently for entity IDs
5. **Add trigger IDs** for complex multi-trigger automations

### Template Sensor Requirements

Always include availability to prevent errors:
```yaml
template:
  - sensor:
      - name: "Example Sensor"
        state: "{{ states('sensor.source') }}"
        availability: "{{ states('sensor.source') not in ['unknown', 'unavailable'] }}"
```

### Automation Development

1. **Test templates first** in Developer Tools > Template
2. **Use choose/conditions** for multi-path logic
3. **Add logbook entries** for debugging critical automations
4. **Check for scene conflicts** before enabling lighting changes

### Recorder Optimization

The recorder is configured to:
- Retain 14 days of history
- Include specific energy sensors
- Exclude chatty entities
- Commit every 3 seconds

---

## Testing & Validation

### Before Committing Changes

1. **Validate YAML syntax**: `yamllint file.yaml`
2. **Test templates**: Developer Tools > Template
3. **Test automations**: Developer Tools > Services
4. **Check logs**: Use the WebSocket `system_log/list` command (see Key Commands above)

### Health Check Locations

- **Watchman**: Reports unavailable entities and config issues
- **System automations**: `automations_new/system/core.yaml`
- **Monitoring**: `automations_new/notifications/monitoring.yaml`

---

## Notification System

### Notification Groups

| Service | Recipients |
|---------|------------|
| `notify.family` | Rafael + Alex (mobile) |
| `notify.rafael` | Rafael (MacBook + iPhone) |
| `notify.alex` | Alex (MacBook + iPhone) |
| `notify.critical_alerts` | Both (mobile push) |
| `notify.admin_only` | Rafael (mobile push) |

### Notification Strategy

E-Mail-Notifications (SMTP) wurden vollstaendig entfernt und ersetzt durch:
- **Push** (`notify.rafael` / `notify.alex`) fuer sofortige Aufmerksamkeit
- **Persistent Notification** (`persistent_notification.create`) als Erinnerung in der HA-Oberflaeche

Jede Persistent Notification hat eine eindeutige `notification_id` (z.B. `robbi_hauptbuerste`, `waschmaschine_fertig`), damit sie nicht doppelt erscheint und manuell weggeklickt werden kann.

### TTS Notifications

Uses `chime_tts` custom component for audio announcements with notification chimes.

---

## File Reference

### Root Configuration Files

No line counts here on purpose: they were wrong in 8 of 9 rows before they were
removed (audit 2026-08-19). Run `wc -l` when you need one.

| File | Purpose |
|------|---------|
| `configuration.yaml` | Main configuration |
| `adaptive_lighting.yaml` | Anchors the AL config entries, values only apply at first import |
| `automations_new/system/core.yaml` | Circadian lighting (10 rooms), live values |
| `scripts.yaml` | Reusable scripts |
| `sensors.yaml` | Template/platform sensors |
| `lights.yaml` | Light groups |
| `binary_sensors.yaml` | Binary sensors |
| `notifies.yaml` | Notification services |
| `climates.yaml` | Climate config |
| `dashboard.yaml` | Lovelace UI |

### Supporting Directories

| Directory | Purpose |
|-----------|---------|
| `.storage/` | HA internal storage (do not edit) |
| `blueprints/` | 162 blueprints (5 automation + 157 switch_manager) |
| `custom_components/` | 24 custom integrations |
| `themes/` | 6 UI themes |
| `www/` | Static web assets |
| `esphome/` | ESPHome device configs |
| `backups/` | Configuration backups |

---

## Security Considerations

- **Never commit `secrets.yaml`** to version control
- **Use `!secret key_name`** for all sensitive data
- **The repo `raaaf/home-assistant` is PUBLIC.** Anything committed here is world-readable
  immediately. Treat every id, token, host name and internal URL as a credential.
- **Webhook ids are credentials.** A `webhook_id` is the only authentication for
  `/api/webhook/<id>`, so it belongs in `secrets.yaml`, never inline. Currently required key:
  `nas_webhook_id` (Synology Hyper Backup, `notifications/alerts.yaml`). There is no
  `secrets.yaml.example`, so a fresh checkout fails on the missing key rather than silently.
- **`local_only: true` is not a second factor.** It resolves the client IP through
  `use_x_forwarded_for` + `trusted_proxies` in `configuration.yaml`. If a reverse proxy or tunnel
  host is missing from `trusted_proxies`, every request looks local.
- **Webhook payloads are attacker-controlled.** Never pass `trigger.json.*` as a bare `message` to
  `notify.*`: the mobile_app platform reads certain exact message values as device commands. Embed
  the text in a fixed sentence and cap its length.
- **Review `manifest.json`** permissions for custom components
- **IP bans managed** in `ip_bans.yaml`
- **Trusted networks** configured in `configuration.yaml`

---

## Adaptive Lighting Optimizations

### Zigbee2MQTT Compatibility (Current)

The Adaptive Lighting configuration is optimized for Zigbee2MQTT with mixed Philips Hue & IKEA lights:

**IKEA Bulb Handling**:
- Short transitions (10s) prevent IKEA bulb lockups
- `separate_turn_on_commands: true` enables IKEA fade-in (ON first, then brightness)
- `send_split_delay: 300` (300ms delay for IKEA to process ON before brightness)
- `current_level_startup: 1` set on all IKEA bulbs via Z2M (starts at min brightness)
- IKEA ignores `transition` on turn_on from off — only works when already on (dimming)
- 23 IKEA lights (GU10, E27, JETSTROM panels), 28 Philips Hue lights (full transition support)

**Color Temperature Range**:
- Min: 2200K (warmest IKEA/Hue compatible)
- Max: 4000K (IKEA maximum cool white)
- Sleep: 2200K everywhere (no RGB, so Philips and IKEA look identical); brightness per room: Balkon 40%, Wohnzimmer/Kueche/Waschzimmer/Ankleide 20%, Arbeitszimmer (Bett) 15%, Badezimmer/Flur 10%, Kinderzimmer 5%, default 5%, Schlafzimmer (Stehlampe) 1%. Ankleide is excluded from the sleep-mode switch list (clothes are picked there at night), so its sleep_brightness of 20% only applies as a fallback if it is ever switched on

**IKEA Fade-In Solution**:
IKEA TRADFRI/JETSTROM bulbs ignore the `transition` parameter when turning on from off.
The workaround requires two settings working together:
1. **Z2M**: `current_level_startup: 1` on all IKEA bulbs (starts at minimum brightness)
2. **AL**: `separate_turn_on_commands: true` + `send_split_delay: 300` (sends ON first, then brightness+transition 300ms later)

This is applied via `System » Adaptive Lighting Settings nach Neustart` automation in `system/core.yaml`.

AL reads its config from **Config Entries**, which the YAML seeds once and then anchors: remove the YAML and HA drops the entries on the next restart. Change settings in the startup automation in `automations_new/system/core.yaml`, never anywhere else, and remember that `change_switch_settings` is runtime-only and does not persist into the config entry.

**Exception, `max_brightness`:** that field belongs to `Licht » Tageshelligkeit nach Aussenlicht` (eight rooms with daylight, driven by the balcony lux sensor via `input_number.al_tagesmax_helligkeit`) and to fixed values for Flur and Ankleide in the startup automation. Do not set it anywhere else. That automation now writes to AL only when the level changes or after a restart, the end of Schlafenszeit, or the end of a scene, because every `change_switch_settings` call restarts AL's own interval listener and forces an adaptation with `initial_transition` instead of `transition`.

**Interval & Reset**:
- 5-minute update interval (smooth sunrise/sunset transitions)
- 1-hour autoreset for main rooms
- 2-hour autoreset for utility/transition areas

---

## Troubleshooting

### Common Issues

**Lights not following circadian schedule**:
1. Check `switch.adaptive_lighting_{room}` is on
2. Check `binary_sensor.{room}_manual_control` - manual control pauses AL
3. Check `binary_sensor.any_scene_active` - scenes may disable AL
4. Verify light is in AL config: `adaptive_lighting.yaml` (the `lights:` lists)

**Automations not triggering**:
1. Check if automation is enabled: `automation.{name}` state
2. Verify entity availability in Developer Tools > States
3. Check logs for errors via WebSocket `system_log/list` (see Key Commands)

**Scene conflicts**:
1. Check `binary_sensor.scene_conflict_detected`
2. Review `sensor.active_scenes` for incompatible combinations
3. Only one major scene should be active at a time

### Log Access

```bash
# Active log file lives inside Docker container — not directly accessible
# Old rotated logs may be available on the mounted volume:
/Volumes/config/home-assistant.log.1    # Previous rotation
/Volumes/config/home-assistant.log.old  # Older rotation

# Live logs: Use WebSocket system_log/list (see Key Commands above)
# This is the only reliable way to access current HA logs from outside Docker
```
