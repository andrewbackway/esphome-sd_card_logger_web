
# ESPHome SD Logger & Web Suite

A high-performance, non-blocking logging solution for **ESP32-S3** (ESP-IDF). This suite is designed to handle high-frequency sensor data without impacting the stability of your ESPHome main loop.

## 🚀 The Core Engine: Asynchronous Batching

The `sd_logger` component uses a **producer-consumer architecture** to protect your device's timing:

* **Zero-Wait Sampling:** Sensors are sampled at your defined `log_interval`, batched into a CSV row, and instantly pushed to a **FreeRTOS queue**. 
* **Background I/O:** A dedicated task handles the slow process of writing to the SD card. This ensures that even during a slow "sync" operation on the card, your sensors, pulse counters, and network tasks remain responsive.
* **Wear Leveling:** By batching physical disk commits via `fsync_interval`, the component reduces the overhead and wear associated with frequent small writes.

---

## 🛠 1. `sd_card` (The Hardware Driver)

The driver uses the native ESP-IDF SDMMC peripheral. It supports **FAT32** and **exFAT** (essential for 32GB+ cards).

```yaml
sd_card:
  id: sd_card_1
  clk_pin: 12
  cmd_pin: 11
  data0_pin: 13
  mode_1bit: true         # Set true for LOLIN S3 Pro or 3-wire setups
  # power_ctrl_pin: 10    # Optional: GPIO to power-cycle the SD slot
```

| Key | Description |
| :--- | :--- |
| `mode_1bit` | `true` uses only DATA0 (3-pin). `false` (default) uses 4-bit mode for higher speed. |
| `clk/cmd/data0` | Standard SDMMC pin assignments. |

---

## 📊 2. `sd_logger` (Logs & Batching Config)

This is where you define your data structure. You can create multiple "Log Groups" with different sensors and intervals.

```yaml
time: # requried for accurate timestamps, logging will wait until time is acquired.
  - platform: sntp
    id: sntp_time
    servers:
      - 0.au.pool.ntp.org
      - 1.au.pool.ntp.org
      - 2.au.pool.ntp.org

sd_logger:
  sd_card_id: sd_card_1
  time_id: sntp_time
  fsync_interval: 30s        # How often to physically commit data to the card
  
  logs:
    - name: "High Speed Telemetry"
      folder: "fast_data"
      file_prefix: "stats"
      log_interval: 1s     
      rotation: daily
      sensors:
        - sensor_id: voltage_input
          format: "%.2f"
        - sensor_id: current_draw

    - name: "Environment"
      folder: "env"
      log_interval: 60s 
      rotation: size
      max_file_size: 5242880 # 5MB rotation
      sensors:
        - sensor_id: temp_internal
      text_sensors:
        - sensor_id: system_status
```

### Log Integrity (The Catalog)
The logger maintains a `catalog.bin` file. This tracks if files were closed cleanly. If the device loses power, the logger identifies the "dirty" file on reboot and marks it as `CORRUPT` to prevent data loss in the new log session - Experimental feature.

---

## 🌐 3. `webserver_sd` (The Web Interface)

Access and manage your log files directly from your browser at `http://<ip-address>/file/`.

```yaml
webserver_sd:
  sd_card_id: sd_card_1
  url_prefix: "file"       # Access via /file/
  enable_download: true    # Click files to download CSVs
  enable_upload: false     # Disable for security on loggers
  enable_deletion: true    # Allow remote cleanup of old logs
```

---

## 🏗 Full Implementation Example

Refer to example.yaml

Based on previous work from [n-serrette/esphome_sd_card](https://github.com/n-serrette/esphome_sd_card)

## License
MIT - [Andrew Backway](https://github.com/andrewbackway)
