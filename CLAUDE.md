# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a sophisticated Home Assistant configuration with extensive custom components, automations, and UI customizations. The codebase includes both YAML configurations and Python-based custom integrations.

## Key Commands

### Configuration Validation
```bash
# Validate Home Assistant configuration (run from HA CLI or SSH addon)
ha core check

# Check specific configuration file syntax
yamllint automations/*.yaml

# Update blueprints from community
./scripts/blueprints_update.sh
```

### Development Workflow
```bash
# Reload automations without restart
ha automation reload

# Reload scripts
ha script reload  

# Reload template entities
ha template reload

# Full configuration reload (when changing configuration.yaml)
ha core restart
```

## Architecture & Structure

### Configuration Architecture
The configuration follows a **modular split-configuration pattern**:
- `configuration.yaml` - Main entry point using `!include` directives
- Domain-specific files: `lights.yaml`, `sensors.yaml`, `switches.yaml`, etc.
- `automations/` - Room-based automation logic (one file per room/area)
- `custom_components/` - 25+ custom Python integrations
- `themes/` - Custom UI themes (minimalist-desktop, minimalist-mobile)
- `blueprints/` - Reusable automation templates

### Custom Components Structure
Each custom component follows this pattern:
```
custom_components/
  component_name/
    __init__.py       # Component initialization
    manifest.json     # Metadata and dependencies
    config_flow.py    # UI configuration flow (optional)
    sensor.py         # Entity platform implementation
    translations/     # Internationalization files
```

### Automation Pattern
Automations use a consistent room-based structure:
```yaml
- alias: "Room: Action Description"
  trigger:
    - platform: state/time/numeric_state
  condition:
    - condition: state/time/template
  action:
    - choose:
        - conditions: []
          sequence: []
    - default: []
```

## Development Guidelines

### YAML Configuration Best Practices
- **Always use `!include` directives** for modular configuration
- **Entity naming convention**: `domain.location_description` (e.g., `light.kitchen_ceiling`)
- **Template sensors**: Always include `availability` templates to prevent errors
- **Secrets management**: Store sensitive data in `secrets.yaml`, never in main configs

### Custom Component Development
- **Python version**: Components must be compatible with Home Assistant's Python version
- **Async patterns**: Use `async_setup_entry` for modern integrations
- **Config flow**: Implement `config_flow.py` for UI-based configuration
- **Translations**: Include at least English translations in `translations/en.json`
- **Error handling**: Proper exception handling with Home Assistant's logging

### Automation Development
- **Trigger IDs**: Use meaningful trigger IDs for complex automations
- **Conditions**: Combine multiple conditions logically
- **Choose actions**: Use choose/conditions for multi-path logic instead of multiple automations
- **Templates**: Validate Jinja2 templates in Developer Tools before implementing
- **Availability checks**: Always check entity availability in templates

### Database & Performance
- **Recorder configuration**: Optimize `recorder:` settings to exclude chatty entities
- **History retention**: Configure appropriate `purge_keep_days` for performance
- **Entity filters**: Use `include:` and `exclude:` patterns to control what's recorded

## Testing & Validation

### Before Committing Changes
1. **Validate YAML syntax**: Ensure proper indentation and structure
2. **Check configuration**: Run `ha core check` before restarting
3. **Test automations**: Use Developer Tools > Services to test automation logic
4. **Verify templates**: Test Jinja2 templates in Developer Tools > Template
5. **Check logs**: Monitor `home-assistant.log` for errors after changes

### Custom Component Testing
- Test config flow with various input combinations
- Verify entity state updates correctly
- Check integration reload functionality
- Validate translation keys are properly mapped

## Common Patterns & Solutions

### Dynamic Room Lighting
The codebase uses sophisticated room-based lighting with presence detection:
- Motion sensors trigger initial lighting
- Presence detection maintains lights
- Adaptive lighting based on time of day
- Integration with physical switches

### Energy Monitoring
Power calculation templates aggregate consumption:
- Individual device monitoring
- Room-based totals
- Utility meters for tracking periods
- Integration with energy dashboard

### Multi-User Presence
Presence detection combines multiple methods:
- Device trackers (phones, watches)
- Motion/presence sensors
- Calendar integration
- Manual overrides

## Important Files & Locations

- `secrets.yaml` - Sensitive configuration data (API keys, passwords)
- `automations/*.yaml` - Room-specific automation logic
- `custom_components/*/manifest.json` - Component dependencies and metadata
- `themes/*.yaml` - UI theme definitions
- `.storage/` - Home Assistant's internal storage (do not edit directly)
- `www/` - Static web assets for dashboards

## Security Considerations

- **Never commit secrets.yaml** to version control
- **Use Home Assistant secrets**: Reference with `!secret key_name`
- **Trusted networks**: Configure in `configuration.yaml` for local access
- **API security**: Use long-lived access tokens for integrations
- **Component permissions**: Review manifest.json requirements for custom components