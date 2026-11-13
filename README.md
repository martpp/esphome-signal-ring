# ESPHome Signal Ring

[![CI](https://github.com/martpp/esphome-signal-ring/actions/workflows/ci.yml/badge.svg)](https://github.com/martpp/esphome-signal-ring/actions/workflows/ci.yml)

ESPHome firmware that turns a small WS2812 LED ring behind a diffuser into a notification light
for Home Assistant: the 3D printer is done or has a problem, the washer has finished, a timer
went off. Each notification has its own color and animation, ends on its own or when the
situation is resolved, and the ring then returns to whatever it showed before. It also notices
on its own when Home Assistant is unreachable.

Build one per room; they all share the same package, so a change to an animation reaches every
ring with the next OTA update.

It is made for the 3D printed
[WLED Status Lamp – Smart Home Notification Light](https://makerworld.com/en/models/3417805-wled-status-lamp-smart-home-notification-light#profileId-3893971)
on MakerWorld, which is designed for WLED. This project replaces WLED with ESPHome so the lamp
becomes a regular Home Assistant device with the notifications built in.

## Contents

- [Features](#features)
- [Hardware](#hardware)
- [Installation](#installation)
- [Home Assistant](#home-assistant)
- [Configuration](#configuration)
- [How it works](#how-it-works)
- [Development](#development)
- [License](#license)

## Features

- **Notification presets**, tuned to be readable behind a white diffuser:

  | Event | Look |
  |---|---|
  | `print_done` | green, slow breathing pulse (30–100 %, 2.5 s) |
  | `print_error` | green with a red block running around the ring |
  | `washer_done` | blue, slow breathing pulse |
  | `timer_done` | white blink (300 ms) |
  | `ha_down` | red blink, raised by the ring itself when Home Assistant is unreachable |

- **Free notifications** with any color and one of seven patterns: `solid`, `blink`,
  `fastblink`, `pulse`, `slowpulse`, `spin`, `dot`
- **Restores the previous state** after a notification (on/off, color, brightness, effect),
  so a notification can play on top of a running effect
- **Selective clearing**: end only the matching notification (`clear_if: print` leaves a washer
  notification alone)
- **Home Assistant outage detection**: after 3 minutes without API connection the ring blinks
  red, and stops as soon as Home Assistant is back
- **Party effect**: a rotating, saturated half color wheel with sparks in the complementary color
- A text sensor that shows the active notification, for dashboards and automations
- Home Assistant blueprint and example automations: 3D printer, washer, timers, party mode,
  a wake-up color fade after the alarm

## Hardware

Tested with the [WLED Status Lamp](https://makerworld.com/en/models/3417805-wled-status-lamp-smart-home-notification-light#profileId-3893971)
build:

- Wemos/LOLIN **D1 mini** (ESP8266)
- **16-LED WS2812 ring** (5 V, GRB)
- a white diffuser in front of the ring

Wiring (D1 mini):

| Ring | D1 mini |
|---|---|
| 5V | 5V |
| GND | G |
| DIN | D4 (GPIO2) |

GPIO2 drives the LEDs through the hardware UART1, which stays flicker-free while WiFi is busy.
It is also the pin of the blue on-board LED, so that one flashes briefly at boot. If the first
LED shows wrong colors or flickers, put a 330 Ω resistor in the data line or use a 3.3 V → 5 V
level shifter.

An ESP32 board package is included as well (data on GPIO16 by default). It compiles in CI but
has not been tested on hardware yet.

## Installation

The firmware is an ESPHome configuration split into a reusable package and a small device file
per ring.

1. Download the repository (`git clone https://github.com/martpp/esphome-signal-ring.git` or
   *Code → Download ZIP*) and copy the contents of the [`esphome/`](esphome) folder into your
   ESPHome configuration directory, so that `signal-ring/` sits next to your device files:

   ```
   config/
   ├── hallway-signal-1.yaml   (copy of signal-ring-esp8266.yaml)
   ├── secrets.yaml
   └── signal-ring/
       ├── core.yaml
       ├── esp8266.yaml
       └── esp32.yaml
   ```

2. Add the entries from [`secrets.yaml.example`](esphome/secrets.yaml.example) to your
   `secrets.yaml` (WiFi, API key, OTA and hotspot password). With several rings, give each its
   own keys (`hallway_signal_1_api_key`, ...) and adjust the `!secret` names in the device file.
3. Set `name` and `friendly_name` in the device file. A naming scheme that scales well is
   `<room>-signal-<n>`, e.g. `hallway-signal-1`.
4. Flash once over USB (ESPHome dashboard → Install → Plug into this computer, or
   `esphome run hallway-signal-1.yaml`). Later updates go over the air.

   Already running other firmware with a web updater (WLED, Tasmota, ...)? Download the
   ESPHome binary (Install → Manual download → *OTA format*) and upload it there instead.

5. Home Assistant discovers the ring; add it under *Settings → Devices & services*.

To pull the package straight from GitHub instead of copying it, use a remote package:

```yaml
packages:
  signal_ring: github://martpp/esphome-signal-ring/esphome/signal-ring/esp8266.yaml@main
```

## Home Assistant

### Actions

Every ring registers these actions (the device name with `-` replaced by `_`):

| Action | Data | Effect |
|---|---|---|
| `esphome.<name>_event` | `event`, `duration` | show a preset; `duration` in seconds, `0` until cleared |
| `esphome.<name>_notify` | `red`, `green`, `blue`, `pattern`, `duration` | show any color with a pattern |
| `esphome.<name>_clear_if` | `event` | end the notification if the active one starts with this prefix |
| `esphome.<name>_clear` | – | end any notification |

```yaml
action: esphome.hallway_signal_1_event
data:
  event: print_error
  duration: 0
```

```yaml
action: esphome.hallway_signal_1_notify
data: {red: 255, green: 120, blue: 0, pattern: spin, duration: 30}
```

The light entity can be used as a normal RGB light with the built-in effects (`Party`,
`Rainbow`, ...) in between notifications.

### Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fmartpp%2Fesphome-signal-ring%2Fblob%2Fmain%2Fhomeassistant%2Fblueprints%2Fautomation%2Fsignal_ring%2Fnotification.yaml)

[`notification.yaml`](homeassistant/blueprints/automation/signal_ring/notification.yaml):
pick an entity and a target state (e.g. the printer status and `finish`), a preset and the
rings. The notification ends when the entity leaves that state again, or after the maximum
duration.

### Examples

[`homeassistant/examples/`](homeassistant/examples) contains what the blueprint does not cover:

- [`scripts.yaml`](homeassistant/examples/scripts.yaml): `script.signal_event` and
  `script.signal_clear_if` send to all rings at once and skip notifications while nobody is
  home or everyone is asleep
- [`automations.yaml`](homeassistant/examples/automations.yaml):
  - 3D printer (Bambu Lab): done, failed or paused; ends when the door opens or the next print starts
  - washer (SmartThings): done; ends when the machine switches off or the next wash starts
  - kitchen timer
  - party mode: party effect while a helper is on, previous state restored afterwards
  - wake-up: yellow → turquoise → red over 45 minutes after the alarm, off when the blinds close again

Entity IDs in the examples are placeholders; replace them with yours.

## Configuration

Substitutions in the device file override the defaults:

| Substitution | Default | Meaning |
|---|---|---|
| `num_leds` | `16` | number of LEDs on the ring |
| `color_order` | `GRB` | channel order of the LEDs (`RGB` if red and green are swapped) |
| `led_pin` | `GPIO2` / `GPIO16` | data pin (ESP8266: keep GPIO2 for the UART1 driver) |
| `light_name` | `Light` | name of the light entity |
| `status_name` | `Notification` | name of the text sensor with the active notification |
| `notify_brightness` | `1.0` | brightness of notifications (0.0–1.0) |
| `dot_width` | `5` | width of the running block of the `dot` pattern |
| `dot_gap` | `1` | dark LEDs in front of the block, keeps the colors apart behind a diffuser |
| `ha_down_after_s` | `180` | seconds without Home Assistant before the ring blinks red |

The preset colors and patterns are defined in the `event` action at the end of
[`core.yaml`](esphome/signal-ring/core.yaml); add your own presets there.

## How it works

- A notification remembers the light state (on/off, color, brightness, effect) unless another
  notification is already showing, switches to its effect and restores the state when it ends.
  A new notification replaces the current one; the state from before the first one comes back.
- The text sensor reports the active event, which `clear_if` compares by prefix. That way an
  automation for the printer cannot end a washer notification by accident.
- The ring checks the API connection every 10 s. After `ha_down_after_s` without Home Assistant
  it shows `ha_down`. The API reboot timeout is disabled so the ring keeps running during an
  outage instead of restarting every 15 minutes. A WiFi outage shows the same signal.
- Brightness steps are smoothed by keeping pulses above 30 %: with ESPHome's gamma correction
  the lowest 8-bit levels would otherwise be visible as steps.

## Development

CI ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)) validates and compiles both example
device files with the ESPHome container and uploads the firmware binaries as artifacts.

To build locally:

```sh
cd esphome
cp secrets.yaml.example secrets.yaml   # then put a real API key in
esphome compile signal-ring-esp8266.yaml
```

## License

[MIT](LICENSE)
