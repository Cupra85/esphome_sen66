# ESPHome Component for Sensirion SEN66

Local external component for ESPHome (>= 2025.10.x) to read **Sensirion SEN66** air‑quality sensors via I²C.
Supports **only SEN66** and publishes all signals
available per model (PMx, T, RH, VOC, NOx, CO₂, – see table). Works with **Arduino**.

> This is a *external component* incl. autoclean button.

## Model capability matrix
| Model  | PM | RH/T | VOC | NOx | CO₂ | HCHO |
|:------:|:--:|:----:|:---:|:---:|:---:|:----:|
| SEN60  | ✅ |      |     |     |     |      |
| SEN63C | ✅ | ✅   |     |     | ✅  |      |
| SEN65  | ✅ | ✅   | ✅  | ✅  |     |      |
| SEN66  | ✅ | ✅   | ✅  | ✅  | ✅  |      |
| SEN68  | ✅ | ✅   | ✅  | ✅  |     | ✅   |

## Wiring (ESP32‑C6 example)
| Pin  | Wire | ESP32‑S3 | Note |
|-----:|------|----------|------|
| GND  | Black| GND      | Ground |
| VDD  | Red  | 3V3      | 3.3 V |
| SDA  | Blue | GPIO10    | I²C data |
| SCL  | Yellow | GPIO11  | I²C clock |

## Quick start (ESP‑IDF)
```yaml
esphome:
  name: sen6x_idf

esp32:
  board: esp32-c6-devkitc-1
  framework:
    type: arduino

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
api:
ota:

i2c:
  sda: GPIO10
  scl: GPIO11
  id: bus_a
  scan: true

external_components:
  - source:
      type: git
      url: https://github.com/Cupra85/esphome_sen6x_test
      ref: main
    components: sen6x

sensor:
  - platform: sen6x
    id: sen66
    pm_1_0:
      name: "PM 0.3-1 µm"
      id: sen66_pm10
      accuracy_decimals: 1
      icon: mdi:chemical-weapon
    pm_2_5:
      name: "PM 1-2.5 µm"
      id: sen66_pm25
      accuracy_decimals: 1
      icon: mdi:chemical-weapon
    pm_4_0:
      name: "PM 2.5-4 µm"
      id: sen66_pm40
      accuracy_decimals: 1
      icon: mdi:chemical-weapon
    pm_10_0:
      name: "PM 4-10 µm"
      id: sen66_pm100
      accuracy_decimals: 1 
      icon: mdi:chemical-weapon
    co2:
      name: "CO₂"
      id: sen66_co2
      accuracy_decimals: 0
      icon: mdi:air-filter
    temperature:
      name: "Temperatur"
      accuracy_decimals: 1
      id: sen66_temperature_top
      icon: mdi:thermometer
    humidity:
      name: "Luftfeuchtigkeit"
      accuracy_decimals: 0
      id: sen66_humidity
      icon: mdi:water-percent
    voc:
      name: "VOC"
      id: sen66_voc
      algorithm_tuning:
        index_offset: 100
        learning_time_offset_hours: 12
        learning_time_gain_hours: 12
        gating_max_duration_minutes: 180
        std_initial: 50
        gain_factor: 230
      icon: mdi:air-filter
      filters:
        - sliding_window_moving_average:
            window_size: 9
            send_every: 1
    nox:
      name: "NOx"
      id: sen66_nox
      algorithm_tuning:
        index_offset: 100
        learning_time_offset_hours: 12
        learning_time_gain_hours: 12
        gating_max_duration_minutes: 180
        std_initial: 50
        gain_factor: 230
      icon: mdi:air-filter
      filters:
        - sliding_window_moving_average:
            window_size: 9
            send_every: 1
    store_baseline: true
    update_interval: 10s

  - platform: template
    name: "Ambiente Temperature"
    id: sen66_ambient_temperature
    unit_of_measurement: "°C"
    accuracy_decimals: 1
    icon: mdi:home-thermometer
    update_interval: 10s
    lambda: |-
      return id(sen66_temperature_top).state + id(temperature_offset_input).state;

  - platform: template
    name: "Ambiente Luftfeuchtigkeit"
    id: sen66_ambient_humidity
    unit_of_measurement: "%"
    accuracy_decimals: 0
    icon: mdi:home-percent
    update_interval: 10s
    lambda: |-
      return id(sen66_humidity).state + id(humidity_offset_input).state;

  - platform: template
    name: "PM gesammt"
    id: sen66_pm010
    unit_of_measurement: "µg/m³"
    accuracy_decimals: 1
    icon: mdi:chemical-weapon
    update_interval: 10s
    lambda: |-
      return id(sen66_pm10).state + id(sen66_pm25).state + id(sen66_pm40).state + id(sen66_pm100).state;
    
  - platform: template  
    name: "Total VOC (Mølhave)"
    id: sen66_tvoc
    unit_of_measurement: "mg/m³"
    accuracy_decimals: 2
    icon: mdi:air-filter
    update_interval: 10s
    lambda: |-
      if (isnan(id(sen66_voc).state)) {
        return NAN;
      }
      float voc_index = id(sen66_voc).state;
      return ((log(501 - voc_index) - 6.24) * (-996.94)) * 0.001;

  - platform: template
    name: "IAQ Index"
    id: iaq_index
    unit_of_measurement: "IAQ"
    accuracy_decimals: 0
    lambda: |-
      float iaq = 100.0; // Start value for good air quality

      // CO2 contribution (400-1500 ppm is optimal, >2000 ppm is bad)
      float co2 = id(sen66_co2).state;
      if (!isnan(co2)) {
        if (co2 > 1500) {
          iaq += (co2 - 1500) / 5.0; // Higher CO2 increases IAQ (worse air quality)
        }
      }

      // TVOC contribution (0-0.25 mg/m³ is good, >1.0 mg/m³ is bad)
      float tvoc = id(sen66_tvoc).state;
      if (!isnan(tvoc)) {
        if (tvoc > 0.25) {
          iaq += (tvoc - 0.25) * 100.0; // Higher VOC levels increase IAQ (worse air quality)
        }
      }

      // Particulate matter
      float pm10 = id(sen66_pm10).state;
      float pm25 = id(sen66_pm25).state;
      float pm40 = id(sen66_pm40).state;
      float pm100 = id(sen66_pm100).state;
      if (!isnan(pm10)) {
        if (pm10 > 5) {
          iaq += (pm10 - 5) * 2.0; // Higher PM2.5 increases IAQ
        }
      }
      if (!isnan(pm25)) {
        if (pm25 > 10) {
          iaq += (pm25 - 10) * 2.0; // Higher PM2.5 increases IAQ
        }
      }
      if (!isnan(pm40)) {
        if (pm40 > 10) {
          iaq += (pm40 - 10) * 2.0; // Higher PM4.0 increases IAQ
        }
      }
      if (!isnan(pm100)) {
        if (pm100 > 20) {
          iaq += (pm100 - 20) * 2.0; // Higher PM10 increases IAQ
        }
      }

      // Humidity contribution (40-60% is optimal)
      float humidity = id(sen66_humidity).state;
      if (!isnan(humidity)) {
        if (humidity < 30 || humidity > 70) {
          //iaq += 10; // Extreme humidity adds to IAQ (worse air quality)
        }
      }

      // Temperature contribution (20-24°C is optimal)
      float temp = id(sen66_temperature_top).state;
      if (!isnan(temp)) {
        if (temp < 16 || temp > 28) {
          //iaq += 10; // Extreme temperatures add to IAQ (worse air quality)
        }
      }

      // Limit IAQ value to range 100-400
      if (iaq > 400) iaq = 400;
      if (iaq < 100) iaq = 100;

      return iaq;
    filters:
      - sliding_window_moving_average:
          window_size: 5
          send_every: 1

```

## Notes
- Default I²C address: **0x6B** (SEN66).
- Start continuous measurement: 0x0021 (SEN66).
- Read measured values (examples): 0x0471 (SEN63C), 0x0446 (SEN65), 0x0300 (SEN66), 0x0467 (SEN68), 0xEC05 (SEN60).
- Scaling per datasheet:
  - PMx [µg/m³] = value / 10
  - RH [%] = value / 100
  - T [°C] = value / 200
  - VOC index = value / 10
  - NOx index = value / 10
  - CO₂ [ppm] = value

## License
MIT
