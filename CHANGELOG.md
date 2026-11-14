# Changelog

## 1.1.0 – 2026-11-14

- New patterns `whiteblink` (color and white alternating, no dark phase) and `duowhiteblink`
  (color, white, second color, white)
- New action `notify2`: like `notify`, plus a second color (`red2`/`green2`/`blue2`, used by
  `duowhiteblink` and `dot`) and its own notification name (`event`) for the status sensor and
  `clear_if`

## 1.0.0 – 2026-11-13

First public release.

- ESPHome package for ESP8266 (D1 mini, tested) and ESP32 (compile-tested) with example device files
- Notification presets: 3D printer done / problem, washer done, timer done
- Free notifications with any color and seven patterns
- Previous light state restored after a notification; selective clearing by event prefix
- Home Assistant outage detection (red blink), no reboot while Home Assistant is down
- Party effect tuned for diffusers
- Home Assistant blueprint and example scripts and automations (printer, washer, timer, party, wake-up)
