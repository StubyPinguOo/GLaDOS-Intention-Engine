# GLaDOS Weather Satellite (v3.1.1)
> **Temporal & Atmospheric Awareness Module**

The **Weather Satellite** is a specific `template` sensor configuration required to give GLaDOS "Future Sight."

### 📡 Contextual Theory

Standard Home Assistant weather entities (`weather.home`) typically only report **current** conditions (Temperature, Humidity, Sun) in their state attributes. They do **not** expose upcoming forecast data (tomorrow's rain, next week's temperature) in a way that an LLM can easily read.


---


**The Solution:**

This configuration creates a specialized sensor (`sensor.glados_weather_context`) that:

1.  **Triggers** hourly OR upon system restart.

2.  **Calls** the weather service to fetch the *daily* forecast.

3.  **Stores** that rich data in a custom attribute.

4.  **Feeds** the System Prompt so GLaDOS can answer: *"Will it rain on Thursday?"*


---


### 🛠 Installation Protocol

**1. Locate the Configuration File:**

Open your Home Assistant "File Editor" or "Studio Code Server" and open the `configuration.yaml` file (found in the root `/config/` directory).

**2. Insert the Telemetry Logic:**

Paste the code block below into your `configuration.yaml`.

* **Note:** If you already have a `template:` section, paste only the `- trigger:` block underneath it.

* **Note:** If you do not have a `template:` section, paste the entire block exactly as shown.


---


```yaml
# ----------------------------------------------------------------------
# GLaDOS WEATHER SATELLITE (Temporal Context Provider)
# ----------------------------------------------------------------------
template:
  - trigger:
      - platform: time_pattern
        hours: "/1"             # Refresh data every hour
      - platform: homeassistant
        event: start            # CRITICAL: Ensures data exists immediately after reboot
    action:
      - action: weather.get_forecasts
        target:
          entity_id: weather.forecast_home  # <--- UPDATE THIS if your entity is different
        data:
            type: daily
        response_variable: weather_data
    sensor:
      - name: "GLaDOS Weather Context"
        unique_id: glados_weather_context
        state: "{{ now().strftime('%H:%M') }}"
        icon: mdi:satellite-uplink
        attributes:
          forecast: "{{ weather_data['weather.forecast_home'].forecast }}"
