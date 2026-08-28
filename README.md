# Halo

A Home Assistant voice satellite with a round touch dashboard, for the
[Waveshare ESP32-S3-Touch-AMOLED-1.32](https://www.waveshare.com/esp32-s3-touch-amoled-1.32.htm).

On-device wake word, Home Assistant Assist for speech, and a 466×466 AMOLED
that shows a clock, your lights, climate, and media controls when nobody is
talking to it.

**[→ Flash it from your browser](https://motamed.github.io/ESP-S3-1.32/)** — no
software to install, works in Chrome, Edge, and Opera.

## What it does

- **Wake word on-device.** microWakeWord runs on the ESP32-S3, so audio only
  leaves the device once you address it. Run Whisper and Piper in Home
  Assistant and nothing leaves your network at all.
- **Round dashboard.** Clock, two light toggles, a climate arc, and media
  transport controls, laid out for a circular panel rather than cropped from a
  rectangular one.
- **Real audio.** ES8311 codec drives the onboard mic and the speaker header,
  so you get spoken replies, timers, and Home Assistant announcements.
- **Doubles as a media player.** It appears in Home Assistant as a media
  player; announcements mix over whatever is already playing.
- **Battery aware.** Reports battery voltage and holds the power latch so it
  keeps running unplugged.
- **OTA after the first flash.** You only need the cable once.

## Flashing

Open the [web flasher](https://motamed.github.io/ESP-S3-1.32/), plug the board
in with a USB-C **data** cable, and click *Connect & install*. When the flash
finishes, the page offers to send your Wi-Fi credentials over the same cable
via Improv.

If no serial port shows up, hold **BOOT**, tap **RESET**, then release **BOOT**
to force the bootloader.

## Adopting it in Home Assistant

Home Assistant discovers the device automatically once it joins Wi-Fi. Go to
*Settings → Devices & Services*, adopt the ESPHome device, then assign it an
Assist pipeline under *Settings → Voice assistants*.

To let the dashboard's light buttons work, enable **Allow the device to perform
Home Assistant actions** on the device page.

## Making it yours

The dashboard is wired to entity IDs at the top of
[`esphome/ha-amoled-satellite.yaml`](esphome/ha-amoled-satellite.yaml):

```yaml
substitutions:
  light_1_entity: light.living_room
  light_1_name: Living room
  light_2_entity: light.kitchen
  light_2_name: Kitchen
  temperature_entity: sensor.outside_temperature
  humidity_entity: sensor.living_room_humidity
  weather_entity: weather.forecast_home
```

Change those, then either adopt the config in the ESPHome add-on and install
over the air, or build it yourself:

```bash
esphome run esphome/ha-amoled-satellite.yaml
```

## Layout

| Path | What's in it |
| --- | --- |
| `esphome/ha-amoled-satellite.yaml` | Entry point: naming, Wi-Fi, API, OTA, Improv |
| `esphome/boards/waveshare-esp32-s3-touch-amoled-1.32.yaml` | Hardware: pins, display, touch, audio, power, battery |
| `esphome/packages/voice.yaml` | Wake word, Assist pipeline, speaker mixing, media player |
| `esphome/packages/ui.yaml` | LVGL pages and the voice overlay |
| `docs/` | The web flasher, published to GitHub Pages |
| `.github/workflows/build.yml` | Builds firmware and deploys the flasher |

## Hardware notes

Pin assignments come from Waveshare's own sources, cross-checked between
[the BSP](https://github.com/waveshareteam/Waveshare-ESP32-components/blob/main/bsp/esp32_s3_touch_amoled_1_32/include/bsp/esp32_s3_touch_amoled_1_32.h)
and
[the demo code](https://github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.32/blob/main/Example/ESP-IDF/08_FactoryProgram/main/user_config.h).

A few things worth knowing if you fork this:

- **PSRAM is octal (OPI), flash is QIO 8MB.** Configuring PSRAM as quad will
  not boot.
- **GPIO18 is a power latch.** It must be driven high or the board powers
  itself off shortly after the PWR button is released on battery.
- **The panel is offset by 6 columns.** The CO5300 has 480 columns of RAM but
  only 466 are visible, starting at column 6. Without `offset_width: 6` the
  whole image sits shifted.
- **There is no backlight GPIO.** Brightness is a controller register, exposed
  as the display's `brightness` option.
- **`board_cfg.txt` in the vendor demo has the wrong I2C pins** for the codec
  (it lists GPIO7/8, which are actually the touch and LCD reset lines). The
  real bus is GPIO47/48.

## CI

Every push builds the firmware with the pinned
[`esphome/build-action`](https://github.com/esphome/build-action). Pushes to
`main` and tags also publish the flasher and the freshly built binary to GitHub
Pages, so the site always offers a firmware that matches the repository.

To enable it on a fork, set *Settings → Pages → Source* to **GitHub Actions**.

Tagging `v1.2.3` stamps that version into the firmware and the manifest.