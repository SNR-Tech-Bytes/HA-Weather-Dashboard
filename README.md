# Home Assistant Weather Dashboard

A sections-view weather dashboard with active alert summary, animated radar, hourly forecast, live lightning tracking, and NWS alerts.

![Alt text](Weather%20Dashboard%20Demo.jpeg)

AI Disclosure: I had AI sanitize and harden these YAMLs and write this readme. I have since tested this on my setup and it works but YMMV.

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

## Alert summary sensor (package)

`weather-alert-package.yaml` is an [HA package](https://www.home-assistant.io/docs/configuration/packages/) containing just the template sensor. It listens for a `weather_alert_summary_updated` event (fired by the automation below) and stores the summary text in `sensor.weather_alert_summary`'s `full_text` attribute — which the dashboard's markdown card reads. It also resets to "No active alerts" when your NWS sensor's alert count drops back to zero.

Enable packages in `configuration.yaml` if you haven't already:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then drop `weather-alert-package.yaml` into a `homeassistant/packages/` folder and restart HA. (Full restart required if you just added the line above, otherwise a YAML reload should suffice.)

Placeholder to fill in before restarting:

| Placeholder | What it should point to |
|---|---|
| `sensor.YOUR_NWS_ALERTS_ENTITY` | Your NWS alerts sensor (same one used in the dashboard) |

Once loaded, this package creates `sensor.weather_alert_summary` — plug that into `sensor.YOUR_WEATHER_ALERT_SUMMARY` in the dashboard YAML.

## Alert summarization automation

`weather-alert-automation.yaml` is a standalone automation — import it via Settings → Automations & Scenes → Add Automation → Edit in YAML, and paste in its contents. It watches your NWS alert sensor and, when the alert count increases, uses a conversation agent to summarize the alert(s) in plain language, fires the `weather_alert_summary_updated` event that the sensor package above listens for, and sends a critical push notification for severe alert codes (tornado, flash flood, etc.).

Placeholders to fill in before importing:

| Placeholder | What it should point to |
|---|---|
| `sensor.YOUR_NWS_ALERTS_ENTITY` | Same NWS alerts sensor as the package and dashboard |
| `conversation.YOUR_CONVERSATION_AGENT` | Optional. A conversation agent entity (local LLM or any HA-supported conversation integration) that can process free-text prompts |
| `notify.YOUR_MOBILE_APP_NOTIFY_SERVICE` | Your `notify.mobile_app_*` service for the target device |
| `/YOUR-DASHBOARD-PATH` | Where the push notification should deep-link (e.g. the weather dashboard view) |

**No conversation agent? No problem.** If you don't have a local LLM or any conversation integration set up, you don't need to touch the `agent_id` field at all — the automation is hardened to fall back gracefully. If the conversation agent call fails or isn't configured, it skips the AI summary and just uses the raw NWS alert text (event, severity, headline, description, instructions) instead. The sensor still updates and critical notifications still fire — you just get the unsummarized alert text rather than a friendlier 2–3 sentence version.

## Setup order

1. Install the required HACS cards (above).
2. Install `weather-alert-package.yaml` as described above, filling in your placeholder.
3. Import `weather-alert-automation.yaml` as a new automation, filling in your placeholders.
4. Import `weather-dashboard.yaml` as a new dashboard, filling in your placeholders — including `sensor.weather_alert_summary` from step 2.

## Notes

- Minimum HA version: any release supporting the Sections view (2024.9+).
- The radar card's `carto_api_key` can be left blank, but tile loading may be rate-limited without one.
- The alert summarization automation works with or without a conversation agent — with one, you get an AI-generated plain-language summary; without, it falls back to the raw alert text. A local LLM is worth it here specifically because these alerts can contain sensitive-ish info you may not want sent to a cloud service.
