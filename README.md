# Hamulight-L2446-remote-control-in-ESPHome (ESP32 + CC1101)

This project lets you control a wireless patio roof lighting that originally only works with its own 433 MHz remote control. Using **ESPHome** and **Home Assistant**, you get on/off and brightness control. An ESP32 with a CC1101 radio module imitates the remote's signal and also listens, so Home Assistant follows along when someone uses the original remote.

> ## ⚠️ Important compatibility note
>
> **This project has only been tried with a single patio roof lighting** (remote control "Hamulight Apex L2446" with a Holtek BS84B08C chip). Whether other models, other manufacturers, or even other units of the same series work is **untested**.
>
> In particular, the brightness calculation was worked out from one remote only. The learning mode (see [Learn the remote](#learn-the-remote-synchronize)) has **not been tested with a second, different remote**.
>
> Use at your own risk. This project is not affiliated with the manufacturer; all names and trademarks belong to their respective owners.
>
> This Code was developed with AI.

## Features

- **Switch** "Terrassenbeleuchtung" (on/off) in Home Assistant
- **Brightness control** (levels 0–36)
- **Feedback from the original remote:** if someone uses the remote, the switch and brightness in Home Assistant update accordingly
- **Learning mode via button:** the 24-bit ID of your own remote is detected at the press of a button, stored, and used for both sending and receiving
- The ID and the brightness level survive reboots

## Hardware

| Part | Note |
|---|---|
| ESP32 development board (`esp32dev`) | other boards work with adjusted pins |
| CC1101 radio module (433 MHz) | powered with **3.3 V, not 5 V** |
| Antenna | about 17 cm of wire (λ/4 for 433 MHz) or the antenna that comes with the module |

### Wiring

| CC1101 | ESP32 | Note |
|---|---|---|
| VCC | 3V3 | **not 5 V!** |
| GND | GND | |
| SCK | GPIO18 | `clk_pin` |
| MOSI (SI) | GPIO23 | `mosi_pin` |
| MISO (SO) | GPIO19 | `miso_pin` |
| CSN | GPIO5 | `cs_pin` (a strapping pin: ESPHome prints a warning, which was harmless in the tested setup) |
| GDO0 | GPIO26 | transmitter (`remote_transmitter`) |
| GDO2 | GPIO27 | receiver (`remote_receiver`) |

In two-pin mode the transmitter is connected to GDO0 and the receiver to GDO2. The pins can be chosen freely, but then they must be changed in the YAML.

## Software requirements

- **ESPHome 2025.11 or newer** with the built-in `cc1101` component. Please enter the version you actually tested with here: `<version>`
- Home Assistant with the ESPHome integration (for the switch, the brightness control, and the learning button)

## Installation

1. Download or clone this repository.
2. Copy `terrassenbeleuchtung.yaml` into your ESPHome folder.
3. Add these entries to your `secrets.yaml`:

   ```yaml
   wifi_ssid: "MyWiFi"
   wifi_password: "MyPassword"
   terrassenbeleuchtung__api_encryption_key: "<32-byte key in Base64>"
   ```

   You can generate an API key with, for example, `openssl rand -base64 32`.
4. If you wired the CC1101 differently, adjust the pins in the YAML. The configuration uses 2-FSK at 433.96 MHz with a 200 kHz filter bandwidth, which worked with the tested lighting.
5. Flash the file to the ESP32 (via USB the first time).
6. Add the new device in Home Assistant.

## Learn the remote (synchronize)

The lamps are already paired with their **original remote**. So the ESP32 does not need to register with the lamps. Instead, it needs to know the **ID of the remote** (24 bits) that is contained in every radio telegram. With this ID it sends commands the lamps accept, and it recognizes commands from the original remote.

By default, the YAML contains the ID of the developer's remote (`0xB1A8BA`). **For a different remote you need to learn your own ID:**

1. Flash the configuration and wait until the device is connected in Home Assistant.
2. Open the device page and press the **"Fernbedienung anlernen"** button (learn remote). The **"Anlern-Status"** sensor (learning status) now shows "Warte auf Taste der Fernbedienung (30 s)" (waiting for a remote button).
3. Hold the **original remote** about 30–50 cm from the ESP and press **any button within 30 seconds**, for about one second. The remote sends several identical telegrams per button press.
4. As soon as two consecutive telegrams carry the same ID, the ESP accepts it. The status shows "gelernt: XXXXXX" (learned), and the **"Fernbedienungs-ID"** sensor (remote ID) shows the new ID.
5. The ID is stored in flash and kept after a reboot. All commands (sending) and receiving use this ID from now on.
6. Test the switch and the brightness control in Home Assistant. Then test the other way round: press a button on the remote and check that Home Assistant follows.

Notes on learning:

- During learning mode, only **your** remote should be used. Otherwise the ESP may learn the ID of another remote within range.
- If nothing is received within 30 seconds, the status shows "Zeitüberschreitung – nichts empfangen" (timeout). In that case check the distance, the antenna, the frequency, and the modulation.
- In normal operation, telegrams with a **foreign ID are ignored**.
- To use a different remote later, repeat the steps with the button.

## Entities in Home Assistant

| Entity | Type | Function |
|---|---|---|
| Terrassenbeleuchtung | Switch | on/off |
| Terrasse Helligkeitsstufe | Number (0–36) | sets the brightness, sends only while the light is on |
| Fernbedienung anlernen | Button | starts the learning mode (30 s) |
| Fernbedienungs-ID | Text sensor (diagnostic) | currently stored ID |
| Anlern-Status | Text sensor (diagnostic) | status of the learning mode |
