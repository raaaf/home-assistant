# Home Assistant Configuration

German-language Home Assistant setup for a smart home in Fürth, Germany. Local-first control (Zigbee, LAN, MQTT over cloud), solar energy monitoring, circadian lighting, and room-based automation with presence detection for two users (Rafael and Alex).

## Highlights

- **Local-first**: Zigbee (ZHA), MQTT, and LAN integrations preferred over cloud.
- **Adaptive Lighting**: circadian rhythm across 10 rooms (Blackshome blueprint + Adaptive Lighting component).
- **Climate**: Midea AC with hysteresis + window-aware cool/fan logic and `ieco`, CO2/temperature-driven fans, predictive floor heating. See the Climate Control section in `CLAUDE.md`.
- **Energy**: 3-phase grid metering, Hoymiles solar, EcoFlow storage, powercalc per-device.
- **Notifications**: push + persistent notifications (no email/SMTP).

## Structure

```
configuration.yaml        # main entry point (split via !include)
automations_new/          # all automations, grouped by domain
  areas/  lighting/  climate/  helpers/  notifications/  scenes/  system/
custom_components/         # 24 custom integrations
blueprints/  themes/  esphome/  www/
```

## Documentation

Full architecture, naming conventions, entity reference, and automation patterns live in **[CLAUDE.md](CLAUDE.md)**.

## Local checks

```bash
# YAML syntax
yamllint automations_new/**/*.yaml

# HA config check (uses $HA_URL and $HA_TOKEN)
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" "$HA_URL/api/config/core/check_config"
```

## Security

Secrets live in `secrets.yaml` (never committed) and are referenced via `!secret`. Do not commit credentials, tokens, or `.env` files.
