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
- **Set up without touching YAML.** Pick your lights and sensors from
  dropdowns in Home Assistant. Entity IDs are not compiled into the firmware,
  so you can change them later without reflashing.
- **Round dashboard.** Clock, two light toggles, a climate arc, and media
  transport controls, laid out for a circular panel rather than cropped from a
  rectangular one. Swipe or tap the chevron to move between pages.
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
*Settings → Devices & Services* and adopt the ESPHome device, then assign it an
Assist pipeline under *Settings → Voice assistants*.

## Choosing what appears on screen

The firmware has no entity IDs compiled into it. Instead it exposes empty
slots that Home Assistant writes into, and a blueprint wires those slots to
whatever you pick:

1. Import [`blueprints/halo_dashboard.yaml`](blueprints/halo_dashboard.yaml)
   under *Settings → Automations & Scenes → Blueprints → Import blueprint*.
2. Create an automation from it.
3. Pick your satellite, its two `Tile` switches, and the lights, sensors, and
   weather entity you want shown.

That's it — the screen fills in within a few seconds. Change your mind later
and just edit the automation; no rebuild, no reflash.

Until you do this, the climate page shows `--` and the tiles read
`Tile 1` / `Tile 2`, because nothing has sent it any values yet.

### How it works

| Direction | Mechanism |
| --- | --- |
| HA → screen | Blueprint calls `number.set_value` / `text.set_value` on the device's slots |
| Screen → HA | Tapping a tile flips the device's `Tile` switch; the blueprint sees it and toggles the real light |

The blueprint finds the write-only slots itself via `device_entities()`, so it
keeps working regardless of the MAC suffix in the device name. Only the two
`Tile` switches are picked by hand, because Home Assistant does not allow
templated entity IDs in automation triggers.

## Building it yourself

Only needed if you want to change the layout or add pages:

```bash
esphome run esphome/ha-amoled-satellite.yaml
```

## Layout

| Path | What's in it |
| --- | --- |
| `esphome/ha-amoled-satellite.yaml` | Entry point: naming, Wi-Fi, API, OTA, Improv |
| `esphome/boards/waveshare-esp32-s3-touch-amoled-1.32.yaml` | Hardware: pins, display, touch, audio, power, battery |
| `esphome/packages/ha_bridge.yaml` | The writable slots Home Assistant pushes values into |
| `esphome/packages/voice.yaml` | Wake word, Assist pipeline, speaker mixing, media player |
| `esphome/packages/ui.yaml` | LVGL pages and the voice overlay |
| `blueprints/halo_dashboard.yaml` | Home Assistant blueprint that wires the slots to your entities |
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