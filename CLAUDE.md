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

### Key Commands

```bash
# Validate Home Assistant configuration
docker exec homeassistant ha core check

# Check YAML syntax
yamllint automations_new/**/*.yaml

# Reload without restart
docker exec homeassistant ha automation reload
docker exec homeassistant ha script reload
docker exec homeassistant ha template reload

# Full restart (only when changing configuration.yaml)
docker exec homeassistant ha core restart

# View logs
docker logs homeassistant
tail -100 /Volumes/config/home-assistant.log

# Git operations
git -C /Volumes/config status
git -C /Volumes/config diff
git -C /Volumes/config log --oneline -10
```

### Important Paths

| Path | Purpose |
|------|---------|
| `configuration.yaml` | Main entry point (899 lines) |
| `automations_new/` | All automations (34 files, 6,740 lines) |
| `custom_components/` | 24 custom integrations |
| `blueprints/` | 195 automation blueprints |
| `scripts.yaml` | Reusable scripts (14 scripts) |
| `themes/` | 6 UI themes |
| `secrets.yaml` | Sensitive data (never commit) |

---

## Architecture & Structure

### Configuration Architecture

The configuration uses a **modular split-configuration pattern** with `!include` directives:

```
configuration.yaml (899 lines)
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
automations_new/
├── areas/                    # 9 room-specific files
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
├── lighting/                 # 8 lighting files
│   ├── adaptive_lighting.yaml    # Manual control detection
│   ├── circadian.yaml            # Dynamic Lighting triggers
│   ├── day_night.yaml            # Time-based transitions
│   ├── motion_sensors.yaml       # Motion triggers
│   ├── nachtlicht.yaml           # Night light mode
│   └── parabolic_sunrise_*.yaml  # Wake-up lighting (4 rooms)
│
├── climate/                  # 2 climate files
│   ├── heating_cooling.yaml # HVAC automation
│   └── rollos.yaml          # Blind/roller control
│
├── helpers/                  # 5 helper files
│   ├── automation.yaml      # Automation management
│   ├── energy.yaml          # Energy tracking
│   ├── motion_sync_on_restart.yaml
│   ├── presence.yaml        # Presence detection (571 lines)
│   └── system.yaml          # System checks
│
├── notifications/            # 3 notification files
│   ├── alerts.yaml          # Alert triggers
│   ├── environment.yaml     # Environmental warnings
│   └── monitoring.yaml      # Health checks
│
├── scenes/                   # 2 scene files
│   ├── activities.yaml      # Guest, Party modes
│   └── entertainment.yaml   # Gaming, TV modes
│
└── system/
    └── core.yaml            # System boot/restart
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

Two-layer aggregation:
1. **Z2M subgroups**: Device-level grouping in Zigbee2MQTT
2. **Room groups**: `light.{room}_alle` or `alle_{room}_lichter`
3. **Night lights**: `alle_{room}_nachtlichter`

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

**Configuration**: `adaptive_lighting.yaml` (386 lines)

**Room Configuration**:
| Room | Lights | Reset Time | Notes |
|------|--------|------------|-------|
| Wohnzimmer | 5 groups/lights | 1 hour | Living spaces |
| Küche | 3 groups | 1 hour | Kitchen |
| Schlafzimmer | 3 groups/lights | 1 hour | Bedroom |
| Kinderzimmer | 3 lights | 1 hour | Kids' room |
| Badezimmer | 3 groups/lights | 2 hours | Bathroom |
| Waschzimmer | 1 light | 2 hours | Laundry |
| Arbeitszimmer | 3 groups/lights | 1 hour | Office |
| Ankleide | 1 group | 2 hours | Dressing room |
| Flur | 2 groups | 2 hours | Hallway |
| Balkon | 2 groups/lights | 2 hours | Balcony |

**Key Settings** (consistent across rooms):
```yaml
interval: 300              # 5 min update cycle
transition: 10             # Short for IKEA compatibility
min_color_temp: 2200       # Warm (IKEA/Hue compatible)
max_color_temp: 4000       # Cool white maximum
sleep_brightness: 10       # Night mode brightness
sleep_color_temp: 2200     # Warmest for sleep
take_over_control: true    # Manual control detection
detect_non_ha_changes: true  # Critical for Zigbee2MQTT
intercept: true            # Intercept light.turn_on calls
brightness_mode: "tanh"    # Smooth brightness curve
```

**Entities per Room**:
- `switch.adaptive_lighting_{room}` - Enable/disable AL for room
- `switch.adaptive_lighting_sleep_mode_{room}` - Force sleep mode

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
- `adaptive_lighting.yaml` - Main AL configuration (10 rooms)
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
script.scene_enable_adaptive_lighting   # Enable circadian for room
script.scene_disable_adaptive_lighting  # Disable circadian
script.scene_enable_motion_automations  # Re-enable motion triggers
script.scene_disable_motion_automations # Disable motion triggers
script.scene_check_conflicts            # Validate scene combinations
```

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

### Input Number (Thresholds)
```yaml
# Climate thresholds
input_number.ac_temp_hot, ac_temp_warm, ac_temp_comfortable

# Blind positions (0-100%)
input_number.rollo_position_closed, partial, half, open

# Fan speeds (0-100%)
input_number.fan_speed_low, medium, high, max
```

### Input Select (Choices)
```yaml
input_select.wakeup_routine_phase  # Wake-up phase tracking
```

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
2. **Check configuration**: `docker exec homeassistant ha core check`
3. **Test templates**: Developer Tools > Template
4. **Test automations**: Developer Tools > Services
5. **Check logs**: `docker logs homeassistant`

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
| `notify.rafael_mail` | Rafael (email) |
| `notify.alex_mail` | Alex (email) |
| `notify.critical_alerts` | Both (mobile + email) |

### TTS Notifications

Uses `chime_tts` custom component for audio announcements with notification chimes.

---

## File Reference

### Root Configuration Files

| File | Lines | Purpose |
|------|-------|---------|
| `configuration.yaml` | 899 | Main configuration |
| `adaptive_lighting.yaml` | 386 | Circadian lighting (10 rooms) |
| `scripts.yaml` | 559 | Reusable scripts |
| `sensors.yaml` | 360 | Template/platform sensors |
| `lights.yaml` | 139 | Light groups |
| `binary_sensors.yaml` | 78 | Binary sensors |
| `notifies.yaml` | 63 | Notification services |
| `climates.yaml` | 137 | Climate config |
| `dashboard.yaml` | 1,097 | Lovelace UI |

### Supporting Directories

| Directory | Purpose |
|-----------|---------|
| `.storage/` | HA internal storage (do not edit) |
| `blueprints/` | 195 automation blueprints |
| `custom_components/` | 24 custom integrations |
| `themes/` | 6 UI themes |
| `www/` | Static web assets |
| `esphome/` | ESPHome device configs |
| `backups/` | Configuration backups |

---

## Security Considerations

- **Never commit `secrets.yaml`** to version control
- **Use `!secret key_name`** for all sensitive data
- **Review `manifest.json`** permissions for custom components
- **IP bans managed** in `ip_bans.yaml`
- **Trusted networks** configured in `configuration.yaml`

---

## Adaptive Lighting Optimizations

### Zigbee2MQTT Compatibility (Current)

The Adaptive Lighting configuration is optimized for Zigbee2MQTT with mixed Philips Hue & IKEA lights:

**IKEA Bulb Handling**:
- Short transitions (10s) prevent IKEA bulb lockups
- `separate_turn_on_commands: false` reduces MQTT traffic
- `send_split_delay: 50` (50ms between commands per docs)

**Color Temperature Range**:
- Min: 2200K (warmest IKEA/Hue compatible)
- Max: 4000K (IKEA maximum cool white)
- Sleep: 2200K at 10% brightness

**Interval & Reset**:
- 5-minute update interval (balances smoothness vs traffic)
- 1-hour autoreset for main rooms
- 2-hour autoreset for utility/transition areas

---

## Troubleshooting

### Common Issues

**Lights not following circadian schedule**:
1. Check `switch.adaptive_lighting_{room}` is on
2. Check `binary_sensor.{room}_manual_control` - manual control pauses AL
3. Check `binary_sensor.any_scene_active` - scenes may disable AL
4. Verify light is in AL config: `adaptive_lighting.yaml`

**Automations not triggering**:
1. Check if automation is enabled: `automation.{name}` state
2. Verify entity availability in Developer Tools > States
3. Check logs for errors: `docker logs homeassistant | grep -i error`

**Scene conflicts**:
1. Check `binary_sensor.scene_conflict_detected`
2. Review `sensor.active_scenes` for incompatible combinations
3. Only one major scene should be active at a time

### Log Locations

```bash
# Main log
/Volumes/config/home-assistant.log

# Fault log (errors only)
/Volumes/config/home-assistant.log.fault

# Docker logs
docker logs homeassistant --tail 100
```
