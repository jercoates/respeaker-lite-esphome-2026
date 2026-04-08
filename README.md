# ReSpeaker Lite ESPHome — ESPHome 2026.3.x Compatible

This is a working ESPHome YAML configuration for the **Seeed Studio ReSpeaker Lite** voice satellite, updated for compatibility with **ESPHome 2026.3.x**.

It is based on the excellent work by [formatBCE](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration). The original configuration breaks on ESPHome 2026.3.x due to a number of breaking changes in the YAML parser and ESPHome API. This repo contains the fixes needed to get it compiling and running again.

Tested on **ESPHome 2026.3.1** with **Home Assistant** and **Wyoming Satellite** / local Whisper + Piper.

---

## Hardware

- [Seeed Studio ReSpeaker Lite](https://wiki.seeedstudio.com/respeaker_lite/) (ESP32-S3, dual microphone, onboard AIC3204 codec)
- Works with JST speaker connector (default) or 3.5mm headphone jack (optional, see below)

---

## Prerequisites

### secrets.yaml

This config uses ESPHome secrets. Make sure your `/config/esphome/secrets.yaml` contains:

```yaml
wifi_ssid: "YourNetworkName"
wifi_password: "YourWiFiPassword"
ota_password: "YourOTAPassword"
```

### Home Assistant

- Home Assistant with the ESPHome integration installed
- Wyoming Satellite add-on (or external Whisper/Piper containers) for local voice processing
- Or Nabu Casa cloud for cloud-based voice processing

---

## Installation

1. Copy `respeaker-lite-2026.3-shareable.yaml` into your `/config/esphome/` directory
2. Rename it to whatever you like (e.g. `respeaker-bedroom.yaml`)
3. Update `name` and `friendly_name` at the top of the file for each device
4. Make sure your `secrets.yaml` is populated (see above)
5. Flash via the ESPHome dashboard — first flash must be done via USB serial, subsequent updates can be done OTA

---

## Wake Words

The config includes five wake word models out of the box:

| Wake Word | Default Sensitivity |
|-----------|-------------------|
| Okay Nabu | Slightly sensitive (0.85) |
| Kenobi | 0.90 |
| Hey Jarvis | Slightly sensitive (0.97) |
| Hey Mycroft | Slightly sensitive (0.99) |
| Stop *(timer cancel)* | 0.50 — internal only |

Wake word sensitivity can be adjusted at runtime from the Home Assistant device page using the **Wake word sensitivity** select entity (Slightly / Moderately / Very sensitive).

---

## XMOS Firmware Auto-Update (Optional)

The `respeaker_lite:` component supports automatic XMOS chip firmware updates on boot. This block is included in the config but **commented out by default**.

### Which scenario applies to you?

**Fresh device — never manually updated:**
Uncomment the `firmware:` block in the `respeaker_lite:` section. The device will automatically update the XMOS chip to v1.1.0 on first boot. The LED will pulse gray during the update, then flash green on success. This may take a minute and will reboot the device once complete.

**Manually updated following Seeed Studio's instructions:**
Leave the `firmware:` block commented out. If you uncomment it after a manual update, you may get a firmware version mismatch error on boot. This is a known issue — the version string or checksum reported by the manually updated chip does not always match what the auto-update block expects, even if the firmware version is the same. Commenting it out resolves this immediately.

> **Note:** Full testing of the auto-update path on a fresh out-of-box device is pending. If you use it successfully on a fresh device, please open an issue or leave a comment so others know it works.

---

## 3.5mm Headphone Jack Output (Optional)

By default, audio is output through the **JST speaker connector**. If your JST connector is damaged or you want to use the **3.5mm headphone jack** instead, uncomment the `priority: 600` block in the `on_boot` section of the YAML:

```yaml
  on_boot:
    # Uncomment the block below to route audio to the 3.5mm jack instead of JST
    # - priority: 600
    #   then:
    #     - lambda: |-
    #         i2c::I2CDevice dev;
    #         ...
```

This works by writing directly to the TLV320AIC3204 codec registers over I2C at boot, before the audio pipeline initializes. Do not uncomment this if you are using the JST connector — it will redirect audio away from it.

---

## What Was Changed From the Original

The original formatBCE configuration does not compile on ESPHome 2026.3.x. The following changes were required:

| # | Change | Reason |
|---|--------|--------|
| 1 | `min_version` bumped to `2026.3.0` | Targets current ESPHome version |
| 2 | `sensor.template.publish` action replaced with `lambda` calling `publish_state()` | Action was removed in 2026.x |
| 3 | `!lambda` tags inside `script.execute` parameter blocks replaced with C++ lambda calls | Stricter YAML parser in 2026.3.x rejects `!lambda` as a tag in this position |
| 4 | `!lambda` tags inside `homeassistant.event` data blocks replaced with global string buffers | Same YAML parser issue |
| 5 | `light.turn_on` with `brightness: !lambda` replaced with C++ light call API in lambdas | Same YAML parser issue |
| 6 | `media_player name: None` changed to `name: "None"` | Bare `None` parsed as YAML null, causing validation error |
| 7 | `select.state` replaced with `current_option()` | Deprecated in 2026.3.x, will be an error in 2026.7.0 |
| 8 | `Mute/unmute sound` renamed to `Mute-unmute sound` | Forward slash reserved as URL path separator, will be an error in 2026.7.0 |
| 9 | `"Factory Reset Coming Up"` LED effect added to `led_internal` | Referenced in button multi-click handler but never defined — compile error |
| 10 | Two string globals added (`tts_uri_buf`, `stt_text_buf`) | Required for the `homeassistant.event` dynamic data workaround |
| 11 | 3.5mm jack routing via `i2c::I2CDevice` (optional, commented out) | `get_i2c_bus()` was removed in 2026.x — `I2CDevice` is the correct API |
| 12 | XMOS firmware auto-update block added (commented out by default) | Ported from Seeed Studio's reference config; `on_begin/end/error` callbacks converted from `brightness: !lambda` to C++ light call API for 2026.3.x compatibility |

---

## Relationship to formatBCE's Updated Config

**formatBCE released a significantly updated configuration in April 2026** that introduces a redesigned audio pipeline using experimental `sendspin` and `speaker_source` components from pre-merge ESPHome pull requests, along with a new `datetime` entity for alarm time and a cleaner `audio_file` component for sound management.

However, his updated config depends on specific git commits of experimental components that are not yet in mainline ESPHome, and uses `brightness: !lambda` syntax that does not compile on stock ESPHome 2026.3.x without his custom external component fork.

**If you want the latest cutting-edge config from formatBCE**, see his repo directly at [github.com/formatBCE/Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration).

**If you want a config that compiles and runs on stock ESPHome 2026.3.x today** with no experimental dependencies, this repo is the right choice.

---

## Known Issues / Notes

- **formatBCE's i2s_audio fork** (`respeaker_microphone` branch) has not been officially updated for ESPHome 2026.3.x as of April 2026. This config works with the current state of that branch, but may break if ESPHome makes further changes before formatBCE updates their fork.
- **XMOS firmware auto-update** may cause a version mismatch error if the device was previously updated manually via Seeed Studio's process. Leave the `firmware:` block commented out in that case. See the XMOS Firmware section above for details.
- **Noise encryption** is not configured on the API by default. If you want encrypted communication between the device and Home Assistant, add an `encryption:` key to the `api:` section.
- **GPIO3 strapping pin warning** is expected — this is the user button and is fine for this use case.

---

## Credits

- [formatBCE](https://github.com/formatBCE) — original ReSpeaker Lite ESPHome integration
- [Seeed Studio](https://www.seeedstudio.com) — ReSpeaker Lite hardware and reference configuration
- [ESPHome](https://esphome.io) — firmware framework

---

## License

This configuration is based on formatBCE's open source work. Please refer to their repository for licensing terms. Modifications documented in this repo are shared freely for community use.
