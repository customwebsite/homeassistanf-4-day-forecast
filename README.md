# CFA Fire Danger Forecast for Home Assistant

[![HACS Compatible](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://hacs.xyz/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Home Assistant custom integration that provides real-time **fire danger ratings** and **Total Fire Ban** status for all CFA (Country Fire Authority) fire districts across Victoria, Australia.

Data is sourced from the official [CFA RSS feeds](https://www.cfa.vic.gov.au/rss-feeds), using the AFDRS (Australian Fire Danger Rating System) colour standard.

---

## Features

- **4-day fire danger forecast** — Today, Tomorrow, Day 3, Day 4 (configurable 1–4 days)
- **Total Fire Ban detection** — district-specific with statewide declaration support
- **All 9 CFA fire districts** supported
- **Max Severity sensor** — highest rating across all forecast days (great for automations)
- **Feed Status diagnostic sensor** — monitors CFA data feed health
- **Combined feed with fallback** — single HTTP request for all districts, automatic fallback to individual feeds on failure
- **AFDRS-compliant colours** — Moderate=green, High=yellow, Extreme=orange, Catastrophic=red
- **Rating-specific icons** — visual differentiation per severity level
- **Lovelace card auto-registration** — card works immediately after install, no manual resource URL needed
- **Stale entity cleanup** — orphaned entities removed automatically on config changes
- **UI-based configuration** via Config Flow (no YAML required)
- **Configurable update interval** (default: 30 minutes)

## Sensors Created

For each configured district, the integration creates **10 sensors**:

| Sensor | Example Entity ID | State |
|---|---|---|
| Fire Danger Rating Today | `sensor.cfa_central_fire_district_fire_danger_rating_today` | `MODERATE` |
| Fire Danger Rating Tomorrow | `sensor.cfa_central_fire_district_fire_danger_rating_tomorrow` | `HIGH` |
| Fire Danger Rating Day 3 | `sensor.cfa_central_fire_district_fire_danger_rating_day_3` | `EXTREME` |
| Fire Danger Rating Day 4 | `sensor.cfa_central_fire_district_fire_danger_rating_day_4` | `NO RATING` |
| Total Fire Ban Today | `sensor.cfa_central_fire_district_total_fire_ban_today` | `Yes` / `No` |
| Total Fire Ban Tomorrow | `sensor.cfa_central_fire_district_total_fire_ban_tomorrow` | `Yes` / `No` |
| Total Fire Ban Day 3 | `sensor.cfa_central_fire_district_total_fire_ban_day_3` | `Yes` / `No` |
| Total Fire Ban Day 4 | `sensor.cfa_central_fire_district_total_fire_ban_day_4` | `Yes` / `No` |
| Max Fire Danger Rating | `sensor.cfa_central_fire_district_max_fire_danger_rating` | `EXTREME` |
| Feed Status | `sensor.cfa_central_fire_district_feed_status` | `ok` / `degraded` / `failed` |

The number of rating and TFB sensors depends on the **Forecast days** setting (1–4, default 4).

### Sensor Attributes

**Fire Danger Rating sensors** include:
- `date` — Forecast date label (e.g. "Wednesday, 12 February 2026")
- `severity` — Numeric severity (0–5)
- `colour` — AFDRS hex colour for the rating
- `total_fire_ban` — Boolean TFB status for the same day
- `forecast_issued_at` — BoM forecast issue timestamp
- `feed_published` — RSS feed publication time

**Max Severity sensor** includes:
- `severity` — Numeric severity of the worst day
- `colour` — Hex colour for the worst rating
- `any_total_fire_ban` — True if any day has a TFB
- `worst_day` — Date label of the worst day
- `forecast_days` — Number of forecast days available
- `feed_source` — `combined` or `individual` (which feed strategy was used)

**Feed Status sensor** (diagnostic) includes:
- `feed_source` — `combined` or `individual`
- `combined_failures` — Consecutive combined-feed failure count
- `fallback_active` — True when in sustained fallback mode (3+ consecutive combined failures)
- `failed_districts` — List of district slugs that failed (if any)
- `last_error` — Error message from the most recent failure
- `last_successful_update` — ISO timestamp of the last successful data fetch

## Feed Resilience

The integration uses a two-tier fetching strategy:

1. **Primary**: Combined RSS feed — a single HTTP request returns data for all 9 districts
2. **Fallback**: Individual per-district RSS feeds — one request per district, fetched concurrently

If the combined feed fails, the integration automatically falls back to individual feeds. After 3 consecutive combined-feed failures, it enters sustained fallback mode (skipping the combined feed timeout) and periodically retries to auto-recover. Individual feed failures are isolated — if one district's feed is down, the others continue updating.

The Feed Status diagnostic sensor and the Lovelace card's status indicator make this visible without checking logs.

## Lovelace Card

A custom card is included and auto-registered on install. To use it:

```yaml
type: custom:cfa-fire-forecast-card
title: Fire Danger Forecast
districts:
  - slug: central
    name: Central
```

For multiple districts:

```yaml
type: custom:cfa-fire-forecast-card
title: Fire Danger Forecast
districts:
  - slug: central
    name: Central
  - slug: north-east
    name: North East
  - slug: east-gippsland
    name: East Gippsland
```

Card options:
- `title` — Card header text (default: "Fire Danger Forecast")
- `show_title` — Set to `false` to hide the header
- `districts` — List of districts with `slug` and optional `name`

The card displays a subtle feed health indicator dot in the footer: green (ok), amber (degraded/fallback), or red (failed).

## Supported Districts

| District Slug | Display Name |
|---|---|
| `central` | Central |
| `north-central` | North Central |
| `south-west` | South West |
| `northern-country` | Northern Country |
| `north-east` | North East |
| `mallee` | Mallee |
| `wimmera` | Wimmera |
| `east-gippsland` | East Gippsland |
| `west-and-south-gippsland` | West & South Gippsland |

You can add multiple districts — each creates its own set of sensors. All districts share a single HTTP request to the combined feed.

## Installation

### HACS (Recommended)

1. Open HACS in your Home Assistant instance
2. Click the **three dots** menu → **Custom repositories**
3. Add the repository URL as an **Integration**
4. Search for **CFA Fire Danger Forecast** and install it
5. Restart Home Assistant

### Manual

1. Copy the `custom_components/cfa_fire_forecast` folder into your Home Assistant `config/custom_components/` directory
2. Restart Home Assistant

## Configuration

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **CFA Fire Danger Forecast**
3. Select your fire district from the dropdown
4. Done! Sensors will appear within moments

### Options

After adding the integration, click **Configure** on the integration page to adjust:

- **Update interval** — How often to poll the CFA feed (300–86400 seconds, default 1800)
- **Forecast days** — How many days of forecast sensors to create (1–4, default 4)

## Automation Examples

### Notify on Extreme or Catastrophic rating

```yaml
automation:
  - alias: "CFA Extreme Fire Danger Alert"
    trigger:
      - platform: state
        entity_id: sensor.cfa_central_fire_district_max_fire_danger_rating
        to:
          - "EXTREME"
          - "CATASTROPHIC"
    action:
      - service: notify.mobile_app_your_phone
        data:
          title: "🔥 Fire Danger Warning"
          message: >
            {{ trigger.to_state.state }} fire danger rating forecast for
            {{ state_attr(trigger.entity_id, 'worst_day') }}
            in the Central fire district.
```

### Notify on Total Fire Ban

```yaml
automation:
  - alias: "CFA Total Fire Ban Alert"
    trigger:
      - platform: state
        entity_id: sensor.cfa_central_fire_district_total_fire_ban_today
        to: "Yes"
    action:
      - service: notify.mobile_app_your_phone
        data:
          title: "🚫 Total Fire Ban Today"
          message: "A Total Fire Ban is in effect today for the Central fire district."
```

### Notify on feed problems

```yaml
automation:
  - alias: "CFA Feed Problem Alert"
    trigger:
      - platform: state
        entity_id: sensor.cfa_central_fire_district_feed_status
        to: "failed"
        for: "02:00:00"
    action:
      - service: notify.mobile_app_your_phone
        data:
          title: "⚠️ CFA Feed Issue"
          message: >
            CFA fire data feed has been unavailable for 2 hours.
            Fire danger ratings may be stale.
```

## Fire Danger Ratings (AFDRS)

| Rating | Severity | Colour | Icon |
|---|---|---|---|
| NO RATING | 0 | Grey | `mdi:shield-check` |
| LOW-MODERATE | 1 | Light Green | `mdi:fire` |
| MODERATE | 2 | Green | `mdi:fire` |
| HIGH | 3 | Yellow | `mdi:fire-alert` |
| EXTREME | 4 | Orange | `mdi:alert-octagon` |
| CATASTROPHIC | 5 | Red | `mdi:skull-crossbones` |

Colours follow the official Australian Fire Danger Rating System (AFDRS) introduced in September 2022.

## Data Source

Data is fetched from the official CFA RSS feeds:
- **Combined feed** (primary): `https://www.cfa.vic.gov.au/cfa/rssfeed/tfbfdrforecast_rss.xml`
- **Individual feeds** (fallback): `https://www.cfa.vic.gov.au/cfa/rssfeed/{district}-firedistrict_rss.xml`

## Disclaimer

This information is for general reference only. Always check the [official CFA website](https://www.cfa.vic.gov.au/warnings-restrictions/total-fire-bans-and-ratings) for the most current fire danger ratings and restrictions.

**In case of fire emergency, call 000 immediately.**

## Credits

- **Data Source**: [Country Fire Authority (CFA)](https://www.cfa.vic.gov.au/) Victoria, Australia
- **Original WordPress Plugin**: [cfa-4-day-forecast](https://github.com/customwebsite/cfa-4-day-forecast) by Shaun Haddrill
- **Home Assistant Integration**: Adapted for HACS

## License

MIT License — see [LICENSE](LICENSE) for details.
