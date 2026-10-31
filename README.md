# ESPHome Clack WS1 softener

ESPHome firmware for a Clack WS1 water softener, with Home Assistant over the native ESPHome API (encrypted).

An ESP32-C3 reads the two auxiliary relays of the Clack controller (water meter pulse and regeneration),
measures the salt level in the brine tank with an ultrasonic sensor, and reads a BME280 inside the box.

## Credits

This project is based on [aver-ua/clack-ws1-meter](https://github.com/aver-ua/clack-ws1-meter) (itself a
fork of [fonske/clack-reader](https://github.com/fonske/clack-reader)). The valve setup and the wiring
below come from its README. The config here was ported to an ESP32-C3 with a BME280, Italian entity
names, a dual optocoupler instead of the relay modules, and ESPHome 2026.x.

## Set up Clack WS1 "RELAY 1 / RLY 1", "RELAY 2 / RLY 2"

*From [aver-ua/clack-ws1-meter](https://github.com/aver-ua/clack-ws1-meter#set-up-clack-ws1-relay-1--rly-1-relay-2--rly-2).*

You need to configure Clack WS1 Valve relays:  
- Relay 1 (RELAY 1 / rLY1) activates after a set number of water liters have been used  
- Relay 2 (RELAY 2 / rLY2) activates at the beginning of a regeneration cycle  

[EN Manual 1](https://github.com/aver-ua/clack-ws1-meter/blob/main/docs/Clack%20WS1%20manual.pdf) pages 13-14  
[EN Manual 2](https://github.com/aver-ua/clack-ws1-meter/blob/main/docs/Clack%20WS1%20manual2.pdf) pages 12-13  

Buttons: `NEXT` - next step | `REGEN` - previous step | `ARROWS` - change value  

1.	Press `NEXT` + `ARROW DOWN` and hold 5 seconds  
2.	Press `NEXT` until `RELAY 1` / `rLY1` is visible  
    a.	Use arrows to change value. Set it on: `VOLUME` / `Softening On L`  
    b.	Press `NEXT` again  
    c.	Set it on 10 L (Watermeter pulse. Must match the *Water liters pulse* setting of this firmware, default 10 L)  
    d.	Press `NEXT` again  
    e.	Set time: 0.01 min  
3.	Press `NEXT` until `RELAY 2` / `rLY2` is visible  
    a.	Use arrows to change value. Set it on: `REGEN TIME` / `Time to On`  
    b.	Press NEXT again  
    c.	Set time: 0.01 min  

## Schematic

*From [aver-ua/clack-ws1-meter](https://github.com/aver-ua/clack-ws1-meter#schematic) (its GPIOs are for an
ESP32; the ones used here are in [Wiring of this device](#wiring-of-this-device)).*  

[Clack WS1 PCB Relay terminal block](https://github.com/aver-ua/clack-ws1-meter/blob/main/images/ws1-pcb.png) has 3 outs (VCC is central). Output voltage 12-18V, so, a step-down DC-DC modules is required. Then you must use also two 12V or 5V relay modules connected to each DC-DC module out. Relays connected to GPIO19 (watermeter pulse), GPIO18 (regeneration pulse) of ESP32.  
You can also use solid state relays like this: [HY-M285 (OMRON G3MB-202P **24VDC**)](https://www.aliexpress.com/item/1005005364510250.html). In this case you don't need to use DC-DC modules.  

Ultrasonic sensor HC-SR04 (to measure the salt level) connected to GPIO25 (trigger) and GPIO26 (echo). The sensor is [placed under the lid of the salt tank](https://github.com/aver-ua/clack-ws1-meter/blob/main/images/HC-SR04.jpg).  
[Schematic diagram of connections](https://github.com/aver-ua/clack-ws1-meter/blob/main/images/Schematic_clack-ws1-meter.png)  
[Board Example](https://github.com/aver-ua/clack-ws1-meter/blob/main/images/board_example.jpg)  

## Wiring of this device

Board: ESP32-C3 DevKitM-1 (`esp32-c3-devkitm-1`, Arduino framework).

Instead of the DC-DC modules and relay modules of the schematic above, the two Clack relay outputs go
through a dual optocoupler module (12 V input side, 3.3 V output side), which also isolates the ESP from
the valve.

```
Clack WS1 ── relay 1 (meter pulse)  ─┐  dual optocoupler  ┌─ GPIO7 ─┐
          └─ relay 2 (regeneration) ─┴─  12 V -> 3.3 V  ──┴─ GPIO6 ─┤
HC-SR04 (salt tank lid) trig GPIO2, echo GPIO3 ─────────────────────┼── ESP32-C3 ──WiFi── Home Assistant
BME280 (box) I2C 0x76, SDA GPIO0, SCL GPIO1 ────────────────────────┘
```

| GPIO | Connected to | Config |
| --- | --- | --- |
| GPIO7 | Clack relay 1, water meter pulse (optocoupler channel 1) | `binary_sensor` *Watermeter pulse*, active low, no internal pull-up, 50 ms debounce |
| GPIO6 | Clack relay 2, regeneration start (optocoupler channel 2) | `binary_sensor` *Regeneration pulse*, active low, no internal pull-up, 50 ms debounce |
| GPIO2 | HC-SR04 trigger | `sensor` *Saltlevel cm* (ultrasonic) |
| GPIO3 | HC-SR04 echo | same sensor |
| GPIO0 | BME280 SDA | `i2c`, BME280 at address `0x76` |
| GPIO1 | BME280 SCL | same bus |

The relay inputs have no internal pull-up (`pullup: false`): when a Clack relay closes, the optocoupler
output pulls the pin to GND, and the pull-up to 3.3 V is on the optocoupler module. The HC-SR04 sits under
the lid of the brine tank, `sensor_height` (20 cm) above the top of a `tank_height` (65 cm) tank. The BME280 measures temperature, pressure and humidity inside the box.

All pins and sizes are substitutions at the top of `esp-clack.yaml`.

## How it works

- **Water meter**: every pulse of relay 1 adds *Water liters pulse* liters (default 10 L) to the counter,
  which is restored after a reboot. A pulse of relay 2 (regeneration) resets it to zero and stores the
  date of the regeneration.
- **Capacity**: *Capacity in liters* minus the counter gives the liters and m³ left before the next
  regeneration. *Time 2 Regeneration* estimates when it will happen: last regeneration + *Capacity in
  days*, or the next occurrence of the usual regeneration hour once the liters are used up.
- **Salt**: salt height = `tank_height + sensor_height - distance`, also as a percentage of
  `tank_height`. *Fill Salt* says "Yes" below *Salt warning percent*.

## Entities

| Entity | What |
| --- | --- |
| Water meter | liters since the last regeneration |
| Water softener ltr left / m3 left | capacity left |
| Regenerated on | date of the last regeneration |
| Time 2 Regeneration | estimate, e.g. `3d 5h` |
| Saltlevel cm / percent, Fill Salt | brine tank |
| Watermeter pulse, Regeneration pulse | raw relay states |
| Temperatura / Pressione / Umidita box | BME280 |
| Capacity in liters / days, Water liters pulse, Salt warning percent | settings (sliders, restored) |
| Uptime, Uptime HR, WiFi Signal dB / Percent, IP, SSID, Restart | diagnostics |

Two Home Assistant actions fix the counters by hand:

- `esphome.esp_clack_meterstand_clack` with `meter_value` (liters used);
- `esphome.esp_clack_meterstand_clack_reg_date` with `meter_value`, the date of the last regeneration in
  the format of *Regenerated on* (`2026 Thu 29 Oct 02:00`).

Pins, tank size, timezone and update intervals are substitutions at the top of `esp-clack.yaml`.

## Files

| Path | What |
| --- | --- |
| `esp-clack.yaml` | the device (node `esp-clack`, 192.168.1.39) |
| `secrets.yaml` | WiFi, API key, OTA password (gitignored; template `secrets.example.yaml`) |

## Build and flash

No Python on the host: ESPHome runs in Docker with a pinned image. `.esphome/` belongs to root.

```sh
cd /mnt/dietpi_userdata/esphome-clack
ESPHOME_IMG=ghcr.io/esphome/esphome:2026.10.1
sudo docker run --rm -v "$PWD":/config $ESPHOME_IMG config  esp-clack.yaml
sudo docker run --rm -v "$PWD":/config $ESPHOME_IMG compile esp-clack.yaml
sudo docker run --rm --network host -v "$PWD":/config $ESPHOME_IMG upload esp-clack.yaml --device 192.168.1.39
sudo docker run --rm --network host -v "$PWD":/config $ESPHOME_IMG logs   esp-clack.yaml --device 192.168.1.39
```
