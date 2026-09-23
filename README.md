# Home Assistant Weather Dashboard

A sections-view weather dashboard with active alert summary, animated radar, hourly forecast, live lightning tracking, and NWS alerts.


![Alt text](Weather%20Dashboard%20Demo.jpeg)

AI Disclosure: I figured out how to put most of this together, however I had AI sanitize my YAMLs and write this readme. I have since tested this on my setup and it works but YMMV.

## Required HACS custom cards

Install these via HACS → Frontend before importing:

- `weather-radar-card` https://github.com/jpettitt/weather-radar-card
- `blitzortung-lightning-card` https://github.com/timmaurice/lovelace-blitzortung-lightning-card
- `blitzortung.org lightning detector` https://github.com/mrk-its/homeassistant-blitzortung
- `weather-alerts-card` https://github.com/seevee/weather_alerts_card
- `NWS-Alerts` https://github.com/finity69x2/nws_alerts

  NOTE: I do not manage any of the above HACS plugins, if you have issues with those cards please post an issue ticket on the associated github!

## Required entities

Before importing, replace these placeholders in `weather-dashboard.yaml` with your own entity IDs:

| Placeholder | What it should point to |
|---|---|
| `sensor.YOUR_WEATHER_ALERT_SUMMARY` | A sensor with a `full_text` attribute summarizing active alerts (e.g. built from a template or an alert-summary integration) |
| `weather.YOUR_WEATHER_ENTITY` | Your weather integration entity (e.g. Tomorrow.io, Met.no, NWS) |
| `sensor.YOUR_LIGHTNING_DISTANCE` / `_COUNTER` / `_AZIMUTH` | Sensors from the Blitzortung Lightning Detector integration |
| `sensor.YOUR_NWS_ALERTS_ENTITY` | Your NWS alerts sensor |
| `YOUR_CARTO_API_KEY` | A free CARTO API key (used by weather-radar-card for the basemap) |
| `/YOUR-DASHBOARD-PATH` | Optional — only needed if you want the "Home" badge to link elsewhere |


## Import

1. Create a new dashboard in Settings → Dashboards → Add Dashboard → "New dashboard from scratch."
2. Edit it, click the three-dot menu → Edit in YAML, and paste in the contents of `weather-dashboard.yaml` (after filling in your entity IDs above).
3. Save.

## Alert summarization: sensor + automation package

`weather-alert-package.yaml` bundles the template sensor and the automation into a single [HA package](https://www.home-assistant.io/docs/configuration/packages/) — one file instead of two separate imports. It watches your NWS alert sensor and, when the alert count increases, uses a conversation agent to summarize the alert(s) in plain language, stores that summary in `sensor.weather_alert_summary`'s `full_text` attribute (which the dashboard's markdown card reads), and sends a critical push notification for severe alert codes (tornado, flash flood, etc.). It also resets to "No active alerts" when the NWS sensor's alert count drops back to zero.

Enable packages in `configuration.yaml` if you haven't already:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then drop `weather-alert-package.yaml` into a `homeassistant/packages/` folder and restart HA. (Full restart required if you just added the line above, otherwise a YAML reload should suffice)
Placeholders to fill in before restarting:

| Placeholder | What it should point to |
|---|---|
| `sensor.YOUR_NWS_ALERTS_ENTITY` | Your NWS alerts sensor (same one used in the dashboard) |
| `conversation.YOUR_CONVERSATION_AGENT` | A conversation agent entity (can be a local LLM agent or any HA-supported conversation integration) that can process free-text prompts |
| `notify.YOUR_MOBILE_APP_NOTIFY_SERVICE` | Your `notify.mobile_app_*` service for the target device |
| `/YOUR-DASHBOARD-PATH` | Where the push notification should deep-link (e.g. the weather dashboard view) |

Once loaded, this package creates `sensor.weather_alert_summary` — plug that into `sensor.YOUR_WEATHER_ALERT_SUMMARY` in the dashboard YAML.

*(Not using packages? `weather-alert-automation.yaml` and `weather-alert-summary-sensor.yaml` are also included as standalone equivalents — import the automation via copy-paste and add the sensor under `template:` separately.)*

## Setup order

1. Install the required HACS cards (above).
2. Install `weather-alert-package.yaml` as described above, filling in your placeholders.
3. Import `weather-dashboard.yaml` as a new dashboard, filling in your placeholders — including `sensor.weather_alert_summary` from step 2.

## Notes

- Minimum HA version: any release supporting the Sections view (2024.9+).
- The radar card's `carto_api_key` can be left blank, but tile loading may be rate-limited without one.
- The alert summarization automation depends on a conversation agent capable of following the summarization prompt — a local LLM works well since these alerts can contain sensitive-ish info you may not want sent to a cloud service.
