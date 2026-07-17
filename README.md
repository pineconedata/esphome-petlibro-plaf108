# Petlibro PLAF108 ESPHome Firmware

ESPHome firmware for the Petlibro Air WiFi Feeder (Model Number PLAF108).

This project replaces the feeder's stock firmware with locally-managed [ESPHome](https://esphome.io/) firmware. It exposes the feeder's controls and sensors without requiring the Petlibro app, a Petlibro account, or an ongoing connection to Petlibro's servers. It also integrates natively with [Home Assistant](https://www.home-assistant.io/).

## Warning
Installing this firmware requires opening the feeder and flashing its ESP32-C3 microcontroller. This can disable the Petlibro cloud functionality and can potentially make the feeder unusable if something goes wrong. Make a complete factory firmware backup before writing anything to the device. Do not rely on this firmware as the only way to feed your animal without thorough testing.

## Supported Hardware

This repository is specifically for the Petlibro Air WiFi feeder (model number PLAF108). The configuration was developed and tested on my own PLAF108 hardware. Petlibro may make internal hardware revisions without changing the external model number, so confirm your microcontroller and board layout before flashing.

## Why Install ESPHome?

The stock PLAF108 firmware is designed around Petlibro's app, accounts, cloud services, and internet connectivity. ESPHome moves control of the feeder into a local ESPHome (or Home Assistant) environment.

Benefits include:

* Local Home Assistant integration
* No Petlibro account required for everyday operation
* No dependency on Petlibro's cloud for local control
* Direct access to the motor, lights, and feeder sensors
* Flexible feeding, alerting, and diagnostic automations
* Local over-the-air firmware updates after the initial flash
* Clear entity names based on observed hardware behavior
* Better visibility into feeder behavior and failure states

This project is best suited to people who are comfortable opening the feeder, working with low-voltage electronics, and accepting the risks of replacing the factory firmware.

## Current Features

The included configuration exposes:

### Connectivity

* ESPHome native API
* Password-protected ESPHome OTA updates
* Password-protected local ESPHome web server
* WiFi fallback access point
* Captive portal
* UART configuration for the board's serial pins

### Lights

* Status light on GPIO3
* Dimmable red alarm LED on GPIO10

### Motor and Feeder Controls

* Motor-right output on GPIO6
* Motor-left output on GPIO7
* Motor interlocking to prevent both directions from activating together
* 250 ms interlock delay when changing motor direction
* Food-sensor power control on GPIO9
* Restart control

### Sensors

* Food-presence sensor on GPIO18
* Motor-rotation sensor on GPIO8
* Wall-power detection on GPIO5
* Battery-connection detection on GPIO0
* Battery-sense voltage on GPIO4
* Estimated battery percentage
* Calculated current power source
* Reset-button state on GPIO19
* Uptime
* Diagnostic entities for currently unidentified GPIO1 and GPIO2 signals

The battery percentage is based on voltage values measured from my feeder:

```yaml
const float empty_v = 1.650;
const float full_v = 2.095;
```

These values may require calibration for your specific device.

## Repository Files

- `plaf108.yaml` contains the ESPHome device configuration.
- `secrets.yaml` contains placeholder values for the required secrets.
  - Your private `secrets.yaml` should not be shared with anyone.

## Attribution

Thanks to the creators and maintainers of [ESPHome](https://esphome.io/) for making user-friendly, open-source firmware possible. 

This project is based on and inspired by: [`taylorfinnell/Petlibro-esphome`](https://github.com/taylorfinnell/Petlibro-esphome). Special thanks to the creators and contributors to that project for investigating the Petlibro hardware, identifying GPIO functions, documenting the programming interface, and publishing the original ESPHome work that made this project possible.

This repository focuses exclusively on the **PLAF108** and includes additional testing and changes related to:

* Battery and power-source reporting
* Food sensing
* Motor interlocking
* Diagnostic entities
* LED control
* OTA and web authentication
* Device-specific calibration

This project is not affiliated with, endorsed by, or supported by Petlibro.

## Safety

Before working on the feeder:

1. Disconnect the wall adapter.
2. Disconnect the internal battery once the feeder is open.
3. Do not connect the USB-to-TTL adapter's 5V pin to the feeder.
4. Use only regulated 3.3V if powering the ESP32 board directly.
5. Connect all serial and external-power grounds together.
6. Do not connect multiple power sources (unless you know they are safe to combine).
7. Back up the full factory flash before installing ESPHome.
8. Keep the factory backup private.
9. Test motor behavior while the feeder is open and empty.
10. Keep hands, tools, clothing, food, and pets away from moving parts.

A factory firmware dump may contain device-specific certificates, credentials, identifiers, or other private data. Do not share it with anyone.

## Requirements

You will likely need:

- Petlibro Air WiFi Feeder (model PLAF108)
- A 3.3V USB-to-TTL serial adapter
  - I used [this CH340 adapter](https://www.amazon.com/dp/B00LZV1G6K). 
- Jumper wires or test leads
  - I used three female-to-male and one male-to-male [jumper wires](https://www.amazon.com/dp/B07GD2PGY4).
- A machine capable of running Python tools (I used Ubuntu 24 LTS)
    - Python 3
    - `esptool`
    - `esphome`
        - I used ESPHome version `2025.2.2` , but a newer version with potentially different behavior may be available in the future. If anything in this guide conflicts with ESPHome's official documentation, follow the instructions in ESPHome.
- A long, narrow Phillips-head screwdriver
- A regulated 3.3V supply if the board cannot communicate reliably while powered by the feeder

Helpful tools include:

- Multimeter with DC voltage and continuity modes
- A way to take clear photos of the board before disconnecting anything
- Labels or tape for marking cables
- Fine tweezers for disconnecting cables
- [Home Assistant](https://www.home-assistant.io/) (or similar hub) with the [ESPHome](https://esphome.io/) integration

Install the required Python tools using your preferred environment. For example:

```bash
python3 -m pip install --user esphome esptool
```

Confirm that both are available:

```bash
python3 -m esptool version
python3 -m esphome version
```

## Configure YAML Files

Download `plaf108.yaml` and `secrets.yaml`. Edit the secrets file to use your WiFi network credentials and use unique, strong passwords. You can also replace the `!secret` lines in `plaf108.yaml` with your credentials and forgo using `secrets.yaml` entirely if you prefer.

The current configuration in `plaf108.yaml` will identify your feeder as `plaf108-a`:

```yaml
esphome:
  name: plaf108-a
  friendly_name: Pet Feeder A
```

Change these values before compiling if you'd like to refer to your feeder differently. Every ESPHome device on the same network must have a unique node name.

## Flashing Overview

The following is the ideal path that I used. It is not a substitute for understanding the risks or verifying the wiring on your own board.

### 1. Open the Feeder

1. Remove the food, bowl, hopper, cylindrical upper shell, and clear food plate.
2. Remove the four screws from the bottom.
3. Lift off the bottom cover.
4. Disconnect the two-pin battery connector.
5. Follow the eight-pin cable toward the food chute.
6. Remove the food chute.
7. Release the six clips holding the upper electronics panel.
8. Expose the ESP32 board.

Confirm that your feeder uses an `ESP32-C3-WROOM-02` board before continuing.

The programming pads are located beneath the ESP32 module and are labeled:

* `VCC`
* `RX`
* `TX`
* `GND`

Two nearby unlabeled pads are used to enter the ROM bootloader.

### 2. Connect the Serial Adapter

Use a USB-to-TTL adapter configured for 3.3V.

Wiring: 

```text
USB-TTL GND       -> feeder GND
USB-TTL TXD       -> feeder RX
USB-TTL RXD       -> feeder TX
unlabelled pad    -> other unlabeled pad
8-pin cable       -> connected
wall adapter      -> disconnected (until power-on)
battery           -> disconnected 
```

On Linux, identify the serial device:

```bash
ls /dev/ttyUSB*
```

The examples below use `/dev/ttyUSB0`. If your system identifies an alternate path, use that instead.

### 3. Enter Bootloader Mode

Power the board on while the two unlabelled boot pads are bridged. You can check the serial output with:

```bash
python3 -m serial.tools.miniterm /dev/ttyUSB0 115200
```

A successful bootloader startup should include:

```text
--- Miniterm on /dev/ttyUSB0  115200,8,N,1 ---
--- Quit: Ctrl+] | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H ---
ESP-ROM:esp32c3-api1-20210207
Build:Feb  7 2021
rst:0x1 (POWERON),boot:0x5 (DOWNLOAD(USB/UART0/1))
waiting for download
```

Exit miniterm with `Ctrl+]` before running `esptool`. Next, confirm esptool can communicate with your adapter:

```bash
python3 -m esptool --port /dev/ttyUSB0 chip_id
```

The result should identify an ESP32-C3:

```text
esptool.py v4.7.0
Serial port /dev/ttyUSB0
Connecting....
Detecting chip type... ESP32-C3
Chip is ESP32-C3 (QFN32) (revision v0.4)
Features: WiFi, BLE
Crystal is 40MHz
MAC:
```

### Power Troubleshooting

On the feeder used to develop this configuration, the board printed serial logs while powered normally but would not communicate reliably with `esptool`. For some reason, the chip was at 3.8V instead of 3.3V when powered by the wall adapter.

Reliable communication required isolating the ESP32 board and powering it from a regulated external 3.3V supply:

```text
External 3.3V +   -> feeder VCC
External GND      -> feeder GND
USB-TTL GND       -> feeder GND
USB-TTL TXD       -> feeder RX
USB-TTL RXD       -> feeder TX
unlabelled pad    -> other unlabelled pad
8-pin cable       -> disconnected
wall adapter      -> disconnected 
battery           -> disconnected
```

When using an external supply:

* Do not connect the USB-to-TTL adapter's VCC pin.
* Do not connect wall power.
* Do not connect the battery.
* Ensure the external supply and serial adapter share ground.

### 4. Back Up the Factory Firmware

You can name the backup file anything you want, but I went with `petlibro_factory_dump.bin`: 

```bash
python3 -m esptool \
  --chip esp32c3 \
  --port /dev/ttyUSB0 \
  read_flash 0x0 0x400000 petlibro_factory_dump.bin
```

Verify that the resulting file exists and is approximately 4MB. Store it somewhere private and secure. Do not add it to this repository or upload it publicly.

### 5. Validate and Compile

From the directory containing `plaf108.yaml` (and `secrets.yaml`, if you're using it), run:

```bash
python3 -m esphome compile plaf108.yaml
```

A successful compilation should take just over a minute and have an output ending in: 

```text
===== [SUCCESS] Took 86.34 seconds =====
```

ESPHome may warn that GPIO2, GPIO8, and GPIO9 are strapping pins. These pins are used by the feeder hardware and should not be modified without understanding their effect during boot. Fix any configuration and compilation errors before proceeding.

### 6. Flash ESPHome

Note: This step actually replaces the firmware on the device. Once you run the commands in this section, your device will no longer work with Petlibro's app and will instead be using ESPHome. Double-check that the factory firmware is safely backed up and that you are ready to proceed. 

To flash ESPHome to the device, run (from the folder containing your `plaf108.yaml` file, or provide the full filepath):

```bash
python3 -m esphome upload plaf108.yaml --device /dev/ttyUSB0
```

During a successful upload, the logs should show several segments, including:

```
bootloader at 0x00000000
partition table at 0x00008000
OTA data at 0x0000e000
application at 0x00010000
```

Once you receive those messages, your pet feeder's firmware has been replaced!

### 7. Boot Normally

1. Disconnect all power.
2. Remove the bridge between the unlabelled boot pads.
3. Disconnect the USB-to-TTL adapter.
4. Remove any external power-supply wiring.
5. Reconnect the feeder's internal eight-pin cable, if it was removed.
6. Power the feeder normally with the wall adapter.

The feeder should automatically connect to WiFi using the credentials in `secrets.yaml` (or `plaf108.yaml`).

Test network connectivity with:

```bash
ping plaf108-a.local
```

You can also visit `plaf108-a.local` (or use your device's name / IP address) in your web browser. Confirm that the feeder boots, connects to WiFi, and has all entities before fully reassembling it.

### Reassemble the Feeder

1. Remove all power sources from the device (wall adapter, battery, external power supply)
2. Ensure all jumper wires and the adapter are unplugged
3. Reconnect the 8-pin cable, if previously disconnected
4. Reconnect the 2-pin battery cable
5. Clip the feeder board back into the top shell
6. Place the food chute (carefully not crushing or catching any wires) and screw it in place
7. Slide the shell with the food chute back over the base of the feeder 
8. Reattach the clear food plate (aligning the central gear with the motor shaft)
9. Slide on and clip down the upper cylindrical food-containment shell
10. Attach the food hopper and dish
11. Refill with food and replace the button-lid
12. Connect to wall power

## Add the Feeder to Home Assistant

In Home Assistant, ensure you have the [official ESPHome integration](https://www.home-assistant.io/integrations/esphome/). You can add it with:

```
Settings -> Devices & services -> Add integration -> ESPHome
```

Once you have ESPHome, you can add the device specifically with: 

```
Settings -> Devices & services -> Devices -> Add Device -> ESPHome
```

You should see a screen with a `Host` and a `Port` field. Enter the feeder's hostname or IP address into the `Host` field and click Submit. 

Note that the YAML must contain the `api:` component for Home Assistant to connect. It is enabled by default in the included configuration. 

Once connected, Home Assistant should show the device's status, sensors, and controls. 

## Initial Testing

Test all feeder  functionality while it is empty and directly supervised. I don't recommend building unattended feeding automations until motor control and sensor behavior have been tested repeatedly.

At minimum, check:

* The feeder remains online
* The status LED behaves as expected
* The red alarm LED can be controlled and dimmed
* Food detection changes when the chute is blocked and cleared
* Wall-power state changes correctly
* Battery-connection state changes correctly
* Battery voltage is plausible
* The motor-rotation sensor changes while the motor moves
* Each motor direction operates correctly
* Enabling one motor direction disables the other
* Both motor outputs turn off after a restart

## Battery Calibration

The voltage reported by the ESP32 is not necessarily the battery's direct terminal voltage. The included percentage calculation is based on observed readings from my feeder specifically.

To calibrate another feeder:

1. Compare ESPHome's reported value with known battery states.
2. Record the ADC reading when the feeder considers the battery empty.
3. Fully charge the battery and record the corresponding ADC reading.
4. Replace `empty_v` and `full_v` in `plaf108.yaml`.
5. Test the result across several charge and discharge cycles.

The displayed percentage should be treated as an estimate.

## Web Server Security

The configuration currently enables ESPHome's local web server:

```yaml
web_server:
  auth:
    username: !secret web_server_username
    password: !secret web_server_password
```

The web server exposes controls for hardware including the feeder motor. Use strong credentials and do not expose it directly to the internet. Consider removing or disabling `web_server:` after setup if browser-based local control is not needed.

## Over-the-Air Updates

After the initial serial installation, later firmware updates can normally be installed over WiFi:

```bash
python3 -m esphome run plaf108.yaml
```

ESPHome will prompt for the OTA password stored in `secrets.yaml` (if configured).  If the device cannot connect to Wifi in the future (such as if WiFi credentials change without updating the firmware beforehand), then the device should create a Fallback Access Point. Connect to the fallback AP and use the fallback AP password stored in `secrets.yaml` (if configured), to update the firmware OTA with new WiFi credentials. If a future update prevents the device from booting, you may have to reopen the device and update the firmware using the serial connection. 

## Configuration Notes

### Unknown GPIOs

GPIO1 and GPIO2 are included as diagnostic entities because their exact functions have not yet been conclusively identified. Their configuration and names may change in the future as the hardware is investigated further.

### Motor Safety

To prevent both left and right motor directions from being triggered simultaneously, the motor outputs are configured to use ESPHome interlocks with a delay when switching directions: 

```yaml
interlock: [motor_left]
interlock_wait_time: 250ms
```

and:

```yaml
interlock: [motor_right]
interlock_wait_time: 250ms
```

This does not guarantee protection from every firmware, electrical, or mechanical failure. I'd recommend using additional safeguards in Home Assistant scripts and automations.

## Troubleshooting

### No `/dev/ttyUSB*` device appears

- Check the USB cable and adapter.
- Verify that the adapter is recognized by the operating system.
- Test the adapter using a serial loopback.
- On Linux, inspect `dmesg` while connecting it.
- Some CH340 adapters may conflict with `brltty`.

### Factory logs appear instead of `waiting for download`

The boot pads were not bridged properly during power-on. Disconnect power, bridge the two unlabelled pads, and power the board again.

### `esptool` reports “No serial data received”

Check:

- TX and RX are crossed correctly
- Grounds are connected
- The test leads are making reliable contact
- The board entered download mode
- The adapter uses 3.3V logic
- The board has a stable 3.3V power source

### The feeder boot-loops after flashing

Verify the core ESP32 configuration matches the board in your device, especially:

```yaml
esp32:
  board: esp32-c3-devkitm-1
  variant: ESP32C3
  flash_size: 4MB
```

and:

```yaml
platformio_options:
  board_build.flash_mode: dio
  board_build.f_flash: 40000000L
```

Also check the power supply, compiled configuration, and serial logs.

### The feeder connects to WiFi but not Home Assistant

Confirm that:

* `api:` is present in the YAML
* Home Assistant and the feeder are connected to the same WiFi network
* The hostname or IP address is correct
* Local network isolation is not blocking ESPHome traffic

### The web interface does not accept the login

Confirm that `web_server_username` and `web_server_password` exist in `secrets.yaml`, then recompile and upload the configuration. Beware of certain special characters that are not YAML-friendly.

### Battery percentage is inaccurate

Calibrate `empty_v` and `full_v` for your feeder. The included values are estimates based on my hardware and not factory specifications.

## Disclaimer

This firmware and documentation are provided without any warranty. I am not responsible for:

* Damage to the feeder or other equipment
* Loss of Petlibro app or cloud access
* Failed, missed, delayed, or repeated feedings
* Data or credential exposure
* Property damage
* Injury
* Harm to animals

Use this project at your own risk and maintain an independent, tested feeding plan.

