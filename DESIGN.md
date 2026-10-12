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

Every other `rgb-*`, `mush-rgb-*` and named `*-color` maps to the neutral `text-secondary`
grey on purpose: icons are monochrome, an active toggle is Signal Orange (`icon_color:
deep-orange` on mushroom entity cards for `switch`/`input_boolean`, mushroom greys it when off).

Set colors through theme variables, never through `card-mod-card`: card-mod applies after the
first render, so icons flashed in their default colors for about a second. Per-card `card_mod`
is fine for threshold colors (CO2, overdue Robbi parts), it only adds color.

## Structure

One sections view, `Zuhause` (`/lovelace/zuhause`), shows only what needs attention or a single
value per area; every area tile opens a subview with the details (back arrow returns home).

| Block on Zuhause | Shows | Detail |
|---|---|---|
| Badges | people, outside temperature, clothing; lights, open door/windows, CO2, heating, laundry only when relevant | none |
| Zu erledigen | one row per due item (Robbi parts, filters, batteries from `sensor.batterien_niedrig`, waste, leak); tap opens `todo.wartung` | To-do panel |
| Empfehlungen | one row per problem listing the affected rooms, only while something is off (UBA/WHO bands, `TBAND` below); red once a room leaves its amber band | Klima |
| Läuft gerade | playing media players with inline controls, hidden when nothing plays | none |
| Bereiche | Licht, Klima, Türen & Fenster, Energie, Wetter, Rollos, Haushalt, Prusa (only when on) | one subview each |

Comfort bands (green, amber one step around it, red outside): living room 20–22 °C, bathroom 22–24,
bedroom and Alex's office 20–21 by day; from 22:00 during Schlafenszeit (`binary_sensor.nachtklima_schlafzimmer`,
the sliding door makes both one air volume) bedroom 17–18 and office 16–19; children's room 19–20 by day,
17–18 during `input_boolean.schlafenszeit`. Humidity 40–60 %, CO2 < 1000 / < 2000 ppm, PM2.5 ≤ 15 / ≤ 35, PM10 ≤ 45 / ≤ 100,
VOC index ≤ 150 / ≤ 250.

Plugins: Mushroom (template cards, chips) and card-mod (threshold colors only). No layout-card, no
Bubble Card: the native sections view and subviews cover layout and drill-down. A dashboard
template must never iterate `states` or `states.sensor`; precompute such lists in a template sensor
(see `sensor.batterien_niedrig`), otherwise every state change re-renders the card and the page
takes 20 s to fill in.

## Rules that apply here

- Signal Orange is the only accent and stays under 10% of a view.
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
