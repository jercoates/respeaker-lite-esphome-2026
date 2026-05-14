# ReSpeaker Lite ESPHome — ESPHome 2026.3.x and 2026.4.x Compatible

This repo contains working ESPHome YAML configurations for the **Seeed Studio ReSpeaker Lite** voice satellite.

| File | Target Version | Status |
|------|---------------|--------|
| `respeaker-lite-2026.3-shareable.yaml` | ESPHome 2026.3.x | ✅ Tested and working |
| `respeaker-lite-2026.4-shareable.yaml` | ESPHome 2026.4.x | ✅ Tested and working |

Both files are based on the excellent work by [formatBCE](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration).

---

## What's New in v2 (2026.4.x)

The v2 config targets ESPHome 2026.4.5 and adds several user-facing improvements on top of the 2026.3.x fixes.

### Home Assistant UI Sliders

Three voice assistant tuning parameters that were previously hardcoded are now exposed as adjustable sliders on the HA device page:

| Slider | Range | Default | What it does |
|--------|-------|---------|-------------|
| Mic auto gain | 0-31 dBFS | 31 | Boosts mic input level sent to Whisper. Increase if HA struggles to hear commands. |
| Noise suppression level | 0-4 | 3 | Filters background noise. Higher = cleaner but may sound robotic. |
| Volume multiplier | 1.0-4.0 | 2.0 | Scales mic volume to the speech recognition pipeline. |

All slider values persist across reboots via `restore_value: true` and take effect immediately without reflashing.

### Headphone Jack Substitution

Routing audio to the 3.5mm headphone jack instead of the JST speaker connector is now a single line change at the top of the file:

```yaml
substitutions:
  use_headphone_jack: "false"   # change to "true" for 3.5mm jack
```

Previously this required finding and uncommenting a block deep in the `on_boot` section.

### Device Name Substitutions

`device_name` and `friendly_name` are now substitutions at the top of the file, making multi-unit deployment cleaner:

```yaml
substitutions:
  device_name: respeaker-lite
  friendly_name: "ReSpeaker Lite"
```

### Wake Word Model Changes

The v2 config uses only two wake word models -- **Okay Nabu** and **Hey Jarvis** -- both pinned to v2 model URLs with matching 10ms feature step sizes. The v1 config's model set (kenobi, hey_mycroft, stop) was removed due to step size compatibility issues on ESPHome 2026.4.x:

- `kenobi` -- v2 model conflicts with v1 okay_nabu step size
- `hey_mycroft` -- no valid URL found in the official model repo
- `stop` -- v2 version does not exist; removed from models and all `micro_wake_word.enable/disable_model` calls

The **Stop** timer cancel feature previously handled by the stop wake word is now handled exclusively by button press. Timer functionality is otherwise unchanged.

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

1. Copy the appropriate YAML file into your `/config/esphome/` directory
2. Rename it to whatever you like (e.g. `respeaker-bedroom.yaml`)
3. Update `device_name` and `friendly_name` in the substitutions block at the top
4. Make sure your `secrets.yaml` is populated (see above)
5. First flash must be done via USB serial -- subsequent updates can be done OTA
6. On first flash after an ESPHome version upgrade, use **Clean Build**

---

## Wake Words

### v2 (2026.4.x)

| Wake Word | Default Sensitivity |
|-----------|-------------------|
| Okay Nabu | Slightly sensitive (0.85) |
| Hey Jarvis | Slightly sensitive (0.97) |

### v1 (2026.3.x)

| Wake Word | Default Sensitivity |
|-----------|-------------------|
| Okay Nabu | Slightly sensitive (0.85) |
| Kenobi | 0.90 |
| Hey Jarvis | Slightly sensitive (0.97) |
| Hey Mycroft | Slightly sensitive (0.99) |
| Stop *(timer cancel)* | 0.50 -- internal only |

Wake word sensitivity can be adjusted at runtime from the Home Assistant device page using the **Wake word sensitivity** select entity (Slightly / Moderately / Very sensitive).

---

## XMOS Firmware Auto-Update

The `respeaker_lite:` component supports automatic XMOS chip firmware updates on boot. This block is **active by default** in both configs -- the device will check the firmware version on boot and update to v1.1.0 if needed.

### Which scenario applies to you?

**Fresh device -- never manually updated:**
Leave the `firmware:` block active. The device will automatically update the XMOS chip to v1.1.0 on first boot. The LED will pulse gray during the update, then flash green on success.

**Manually updated following Seeed Studio's instructions:**
Comment out the entire `firmware:` block in the `respeaker_lite:` section. If you leave it active after a manual update you may get a hash mismatch error on boot.

---

## 3.5mm Headphone Jack Output (Optional)

### v2 (2026.4.x)

Change the substitution at the top of the file:

```yaml
substitutions:
  use_headphone_jack: "true"
```

### v1 (2026.3.x)

Uncomment the `priority: 600` block in the `on_boot` section:

```yaml
  on_boot:
    # Uncomment the block below to route audio to the 3.5mm jack instead of JST
    # - priority: 600
    #   then:
    #     - lambda: |-
    #         i2c::I2CDevice dev;
    #         ...
```

Both methods write directly to the TLV320AIC3204 codec registers over I2C at boot. Do not enable this if you are using the JST connector.

---

## What Was Changed From the Original (v1 -- 2026.3.x)

| # | Change | Reason |
|---|--------|--------|
| 1 | `min_version` bumped to `2026.3.0` | Targets current ESPHome version |
| 2 | `sensor.template.publish` replaced with `lambda` calling `publish_state()` | Action removed in 2026.x |
| 3 | `!lambda` in `script.execute` parameter blocks replaced with C++ lambda calls | Stricter YAML parser in 2026.3.x |
| 4 | `!lambda` in `homeassistant.event` data blocks replaced with global string buffers | Same YAML parser issue |
| 5 | `light.turn_on brightness: !lambda` replaced with C++ light call API | Same YAML parser issue |
| 6 | `media_player name: None` changed to `name: "None"` | Bare `None` parsed as YAML null |
| 7 | `select.state` replaced with `current_option()` | Deprecated, error in 2026.7.0 |
| 8 | `Mute/unmute sound` renamed to `Mute-unmute sound` | Forward slash reserved, error in 2026.7.0 |
| 9 | `"Factory Reset Coming Up"` LED effect added to `led_internal` | Referenced but never defined -- compile error |
| 10 | Two string globals added (`tts_uri_buf`, `stt_text_buf`) | Required for `homeassistant.event` dynamic data workaround |
| 11 | 3.5mm jack routing via `i2c::I2CDevice` (optional, commented out) | `get_i2c_bus()` removed in 2026.x |
| 12 | XMOS firmware auto-update block added | Ported from Seeed reference config; callbacks converted to C++ light call API |

---

## What Was Changed From v1 (v2 -- 2026.4.x)

| # | Change | Reason |
|---|--------|--------|
| 1 | `min_version` bumped to `2026.4.0` | Targets ESPHome 2026.4.x |
| 2 | `hey_jarvis` and `okay_nabu` pinned to v2 model URLs (`/models/v2/`) | v1/v2 models have different feature step sizes and cannot be mixed |
| 3 | `kenobi`, `hey_mycroft`, `stop` models removed | Step size conflicts or missing v2 URLs |
| 4 | `activate_stop_word_once` script gutted | Referenced removed `stop` model ID |
| 5 | `micro_wake_word.enable/disable_model: stop` calls removed | `stop` model no longer exists |
| 6 | `use_headphone_jack` substitution added | Single line toggle replaces buried comment block |
| 7 | `device_name` and `friendly_name` moved to substitutions | Cleaner multi-unit deployment |
| 8 | Three HA UI sliders added | `noise_suppression_level`, `auto_gain`, `volume_multiplier` now runtime-adjustable |
| 9 | Slider values re-applied on boot | Ensures saved slider values survive restarts |
| 10 | Unicode characters removed from comments | ESPHome 2026.4.x YAML parser rejects non-ASCII even in comments |
| 11 | Wake word sensitivity select updated | Removed kenobi and hey_mycroft cutoffs |

---

## Known Issues / Notes

- **formatBCE's i2s_audio fork** (`respeaker_microphone` branch) has not been officially updated for ESPHome 2026.4.x. Both configs work with the current state of that branch but may break if ESPHome makes further changes before formatBCE updates their fork.
- **Stop wake word removed in v2** -- no v2 model exists. Timer cancel via voice ("stop") is unavailable in v2. Button press still works.
- **XMOS firmware auto-update** may cause a hash mismatch error if the device was previously updated manually. Comment out the `firmware:` block in that case.
- **GPIO3 strapping pin warning** is expected -- this is the user button and is fine for this use case.
- **Noise encryption** is not configured on the API by default. Add an `encryption:` key to the `api:` section if you want encrypted HA communication.

---

## Relationship to formatBCE's Updated Config

**formatBCE released a significantly updated configuration in April 2026** that introduces a redesigned audio pipeline using experimental components from pre-merge ESPHome pull requests, along with a new `datetime` entity for alarm time and a cleaner `audio_file` component for sound management.

However, his updated config depends on specific git commits of experimental components that are not yet in mainline ESPHome.

**If you want the latest cutting-edge config from formatBCE**, see his repo directly at [github.com/formatBCE/Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration).

**If you want a config that compiles and runs on stock ESPHome 2026.3.x or 2026.4.x today** with no experimental dependencies, this repo is the right choice.

---

## Credits

- [formatBCE](https://github.com/formatBCE) -- original ReSpeaker Lite ESPHome integration
- [Seeed Studio](https://www.seeedstudio.com) -- ReSpeaker Lite hardware and reference configuration
- [ESPHome](https://esphome.io) -- firmware framework

---

## License

This configuration is based on formatBCE's open source work. Please refer to their repository for licensing terms. Modifications documented in this repo are shared freely for community use.
