# DESIGN.md

Design intent for the Home Assistant frontend (Lovelace dashboard and the `Linear` theme).
This repo follows the Rafael Design System, **Web port, dark appearance only**. Token values
live in the design system (`rafael-design-system/tokens/src/`); this file names the mapping
and the deviations, it does not copy the token tables.

## Where the design lives

| What | Where |
|---|---|
| Theme (colors, radius, mushroom variables, card-mod rules) | `themes/linear/linear.yaml` |
| Dashboard (live, storage mode) | `.storage/lovelace.lovelace`, edited in the UI or via the raw editor |
| Dashboard snapshot (not loaded by HA) | `dashboard.yaml` |

Reload after a theme edit: `frontend.reload_themes`, no restart.

## Token mapping (dark)

| Design system role | Theme variable |
|---|---|
| `surface-ground` (true black) | `primary-background-color`, `app-header-background-color`, `sidebar-background-color` |
| `surface-stone` | `secondary-background-color` |
| `surface-elevated` | `ha-card-background`, `card-background-color` |
| `border-default` | `ha-card-border-color`, `divider-color` |
| `text-primary` / `text-secondary` | `primary-text-color` / `secondary-text-color` |
| Signal Orange (`brand-indicator`, `brand-progress`) | `primary-color`, `app-header-selection-bar-color`, sliders, switches |
| `danger` / `warning` / `available` (dark) | `rgb-red` / `rgb-amber`, `rgb-orange` / `rgb-green` |

Every other `rgb-*` color maps to the neutral `text-secondary` grey on purpose: icons are
monochrome, state is carried by orange plus a non-color cue.

## Rules that apply here

- Signal Orange is the only accent and stays under 10% of a view. Light cards show their
  slider only when on (`collapsible_controls`), otherwise four sliders break the rule.
- The active navigation item is an ink icon, never an orange one; the orange tab underline
  is additive.
- Hierarchy through size, not weight: mushroom card names are Regular (`mush-card-primary-font-weight: 400`).
- Touch targets at least 44px: chips use `mush-chip-height: 44px`.
- No decorative continuous motion: no `icon_animation` on cards.
- One idea per card; a card is named after its device or room, never after the media title.

## Deviations

| Deviation | Reason |
|---|---|
| Card radius 12px instead of the Web port's 8px | The dashboard is used mostly in the iPhone companion app; 12px matches the iOS port there. |
| Roboto instead of Colaborate / Cal Sans UI | Home Assistant ships Roboto. Loading the design system fonts needs a CSS resource and the font files under `www/`; not done yet. |
| No light appearance | The dashboard is only used dark. |
| Light-card sliders use the lamp's own color (`use_light_color: true`) | Shows the actual color temperature of Adaptive Lighting. Revisit if the slider color should become Signal Orange. |
