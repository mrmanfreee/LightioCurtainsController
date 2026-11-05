# Smart Curtains Controller — User Guide

## 1. About the project

Smart Curtains Controller turns an ESP32-S3 into a controller for motorized curtains and dimmable lighting. It supports a stepper motor with position feedback and end switches, so you can open, close, or move curtains to a chosen position.

You can operate it through a browser, MQTT, Home Assistant, an NEC-compatible infrared remote, or a daily schedule. A physical setup switch lets you enable the configuration access point even without a remote.

Main features:

- Curtain positioning: **0% = fully open; 100% = fully closed**.
- Adjustable motor speed, homing, travel calibration, and motor-error reporting.
- Lighting on/off and brightness control.
- Daily timed actions for curtains and lighting.
- Local WebUI with settings, schedules, logs, and firmware updates.
- MQTT commands, status reporting, and Home Assistant discovery.
- Device diagnostics: uptime, WiFi signal, memory usage, and filesystem space.

## 2. Hardware requirements

### Current controller options and memory requirements

The firmware is currently built for the two **ESP32-S3** controller options listed below. It can also be built for other compatible controllers with **at least 8 MB of flash** (required for OTA updates) and **at least 1 MB of RAM**. Building for another controller requires adapting the firmware configuration, memory layout, and GPIO assignments to that hardware; the supplied ESP32-S3 images cannot be used directly on other controller families.

| Module model | Physical flash | PSRAM | Firmware selection |
|---|---|---|---|
| ESP32-S3 **N8R2** | 8 MB | 2 MB, Quad SPI | N8R2 / Quad-PSRAM image |
| ESP32-S3 **N16R8** | 16 MB | 8 MB, Octal SPI | N16R8 / Octal-PSRAM image |

Match the complete model marking, including the PSRAM type. Having more flash does not make different firmware variants interchangeable. The current firmware layouts use 8 MB of flash, including on N16R8 hardware; the full 16 MB is not automatically available as file storage. Filesystem space is also smaller than the flash capacity because firmware occupies part of it.


### Connected hardware

For curtain movement, use:

- A stepper motor and a compatible **STEP/DIR/EN motor driver**.
- An **AS5600 magnetic angle sensor**, with a magnet coupled to the curtain shaft.
- Two normally open end switches: fully open and fully closed. 
- Suitable motor power and a regulated supply for the ESP32-S3 board.
- Ir receiver TL1838 VS1838B 1838 38Khz

The reference mechanism uses **3,200 motor steps per shaft revolution** and a **17.5 mm curtain shaft diameter**. Match the driver microstepping and mechanics to the firmware supplied for your installation.

For additional functions:

- An NEC-compatible IR receiver and remote for remote control and travel calibration.
- A normally open switch or jumper for configuration mode.
- A suitable light connected to the lighting PWM output.
- Optional status LEDs or a WS2812 indicator.
- A USB data cable and a computer for initial flashing or recovery.
- A 2.4 GHz WiFi network for normal network operation; an MQTT broker for MQTT/Home Assistant control.

GPIOs provide **3.3 V logic signals**, not motor or lighting power. Use appropriate drivers and a common signal ground. GPIO numbers below are the chip's GPIO labels, not connector pin positions.

## 3. Flashing the firmware

### First installation or USB recovery

Obtain a **complete firmware package for your memory model**. The following instructions use these three files from the same release:

| File | Flash address |
|---|---|
| `bootloader.bin` | `0x0` |
| `partition-table.bin` | `0x8000` |
| `micropython.bin` | `0x10000` |

An application-only update `.bin` is not a complete first-installation package. Do not mix bootloader, partition-table, or application files from different releases.

1. Install Python and Espressif's flashing tool:

   ```sh
   python -m pip install --upgrade esptool
   ```

2. Connect the board using a USB **data** cable. Close any program using its serial port. Identify the port, for example `COM5` on Windows, `/dev/ttyACM0` or `/dev/ttyUSB0` on Linux, or `/dev/cu.usbmodem…` on macOS.
3. If the board does not enter download mode automatically, hold **BOOT**, press and release **RESET/EN**, then release **BOOT**. This is the board's BOOT button, separate from the GPIO21 configuration switch.
4. Open a terminal in the folder containing the three firmware files. The erase command removes **all existing settings, schedules, and stored files**:

   ```sh
   python -m esptool --chip esp32s3 --port COM5 erase-flash
   python -m esptool --chip esp32s3 --port COM5 write-flash 0x0 bootloader.bin 0x8000 partition-table.bin 0x10000 micropython.bin
   ```

   Replace `COM5` with your actual port. On systems using `python3` instead of `python`, substitute it in both commands. These commands use current esptool command names; older esptool versions use `erase_flash` and `write_flash`.

5. Wait for flashing and verification to finish, then press **RESET/EN** if the board does not restart automatically. Allow a few seconds for startup and preparation of device files.
6. Enter configuration mode and set up WiFi as described below.

The firmware package must include the controller application and its WebUI/default files. A generic MicroPython image alone does not provide Curtain Controller. Follow any additional file-installation instructions supplied with your release package.

See [Espressif's ESP32-S3 flashing instructions](https://docs.espressif.com/projects/esptool/en/latest/esp32s3/esptool/basic-commands.html) for the flashing tool's command reference.

### Later updates through the WebUI

1. Back up your settings and schedule using **Configuration → Settings → Edit JSON** and **Configuration → Schedule → Edit JSON** before updating.
2. Open **System → Choose firmware**.
3. Select the application update `.bin` supplied for your board and memory model. Do not select a bootloader or partition-table image.
4. Start the upload and wait for the success message.
5. Choose **System → Restart** and confirm.
6. Keep power connected while the controller installs the update and restarts. It can require up to 5 minutes. Reconnect to its WebUI afterwards and check your settings.

If the page still shows an older layout after an update, hard-refresh the browser.

## 4. Pinout and connections

| GPIO | Connection | Description |
|---|---|---|
| **1** | CONNECTION indicator, blue | PWM LED output; network/broker status |
| **2** | STATE indicator, red | PWM LED output; movement/error status |
| **4** | Motor driver STEP | Step-pulse output |
| **5** | Motor driver DIR | Direction output; low = opening, high = closing |
| **6** | Motor driver EN | Active-low enable; high releases the motor |
| **7** | Lighting | Dimmable lighting PWM output |
| **8** | Closed-position end switch | Input with internal pull-up; switch connects to GND |
| **15** | AS5600 PWM output, optional | Reserved sensor input; normal position feedback uses I2C |
| **16** | AS5600 SDA | I2C data; sensor address `0x36` |
| **17** | AS5600 SCL | I2C clock |
| **18** | Open-position end switch | Input with internal pull-up; switch connects to GND |
| **21** | Configuration-mode switch | Input with internal pull-up; connect to GND during startup |
| **41** | AUTO indicator, green | PWM LED output; driven low during initial startup |
| **42** | IR receiver signal | NEC-compatible IR input |
| **48** | WS2812 data | RGB indicator output |
| **GND** | Common signal ground | Shared ground for controller, driver signals, sensor, and switches |


The motor driver, motor, light driver, and ESP32 board may need different supply voltages. Use each component's specified supply; do not connect motor power to a GPIO.

## 5. First setup and configuration mode

### Enter configuration mode

1. Connect a normally open switch or jumper between **GPIO21 and GND** OR hold open/close button on remote.
2. Power on or resetting the controller.
3. Hold it closed until the status LEDs blink quickly.
4. Connect your phone or computer to the WiFi network **`CURTAINS`**.
5. Open **`http://192.168.0.1/`**.

Configuration mode remains active until restart. Opening the switch alone does not exit it.

### Save network settings

Under **Configuration → Settings → Network**, enter:

| Setting | Purpose |
|---|---|
| Wi-Fi SSID / password | Your 2.4 GHz network credentials |
| Admin login / password | Optional WebUI credentials |
| MQTT broker IP / user / password | Broker connection, using port **1883** |
| NTP server | Server used to synchronize the device clock |

Set a unique **Device name** under General; this also identifies the controller's MQTT topics and Home Assistant device. Set the timezone as a whole-hour UTC offset for the device clock display and daily schedule. Adjust it when daylight-saving time changes; there is no automatic regional timezone selection.

Press **Save** and confirm. Open/remove the GPIO21-to-GND connection, release remote buttons, and restart the device.

Find its normal network address in your router's DHCP client list or the serial startup log, then open **`http://<device-ip>/`** from the same network.

### WebUI login

The browser asks for credentials when **both** Admin login and Admin password are non-empty. Authentication is disabled if either is missing or empty. The same credentials apply in configuration/AP mode.

Saved passwords appear masked. Leave the masked value unchanged to keep the password; enter an empty value to clear it. Setting both admin fields enables login immediately, so the browser may ask you to sign in again. Restart after saving settings to apply all startup-dependent changes.

The WebUI uses HTTP and browser Basic authentication; the connection does not encrypt credentials.

## 6. Using the WebUI

### Overview → Device info

Information is grouped into Device, Runtime, Connectivity, and Resources:

- Device name, identifier, hardware, and firmware version.
- Uptime displayed in seconds, minutes, or hours; device time and reset information.
- WiFi and MQTT connection status, plus WiFi signal strength in **dBm**.
- Allocated/total memory, filesystem **free / total** space, and any available sensor readings.

Hover over, keyboard-focus, or tap the memory or WiFi-signal item for details and a history chart. Both charts use samples from the normal status refreshes. The browser stores up to **120 samples**, with a **10-minute expiry** at the normal 5-second refresh interval. History survives a page reload in the same tab, is separate for each device, and resets when a device restart is detected. No additional controller requests are made for the charts.

Memory refers to the reported Python heap, not the board's entire physical RAM. Sensor fields show `—` when no reading is available.

### Overview → Control

- **Open / Close**: move fully open or fully closed.
- **Position**: choose a position from 0% open to 100% closed.
- **Motor speed**: choose 20–100%.
- **Automatic schedule**: enable or disable timed actions.
- **Light on / Brightness**: switch lighting and adjust its level.

A position command received while the curtains are moving stops the current movement. You can therefore press Open or Close again to stop from this page. MQTT also provides an explicit `STOP` command.

A first position command after startup can home the curtains before moving to its target. Keep the path clear during initial setup.

### Configuration → Settings

Settings are grouped as General, Network, Indication, and IR remote. Light brightness is in General; admin credentials are in Network. Additional configuration fields appear in General.

**Save** applies and stores your changes; **Reload** discards unsaved edits and reads the stored settings. Feedback appears above the footer buttons without moving them. **Edit JSON** provides a text editor for advanced configuration and copying a backup.

### Configuration → Schedule

1. Select **Add entry**.
2. Set the time and choose Position, Light, or Position + Light.
3. Set the position, speed, and/or brightness.
4. Press **Save schedule** and confirm.
5. Enable **Automatic schedule** under Control or General settings.

Times are local device times, calculated using the configured UTC offset. The clock must be synchronized before timed actions run. Entries repeat daily and can run up to about two minutes late. Review the schedule after changing the timezone.

Edits, additions, and deletions display an unsaved-change message. **Reload** discards unsaved changes. Save after deleting entries, including when clearing the whole schedule.

Example schedule for the JSON editor or MQTT:

```json
[
  { "time": "06:30", "position": 0, "speed": 60 },
  { "time": "07:00", "light": 20 },
  { "time": "21:00", "position": 100, "speed": 50, "light": 30 }
]
```

Position and lighting use 0–100%; use 20–100% for speed. **Current release limitation:** a scheduled brightness of `0` is ignored; use the WebUI light switch or MQTT lighting state `OFF` when you need to turn the light off. The timezone is a fixed offset, so adjust it for seasonal clock changes where needed.

### Schedule data fields and operation

The JSON editor and MQTT use the same list of entries. Each entry is an object with:

| Field | Data type | Meaning |
|---|---|---|
| `time` | String, `"HH:MM"` | Required daily time, from `"00:00"` to `"23:59"`, in the configured local device time |
| `position` | Integer, 0–100 | Optional curtain target; 0 opens and 100 closes |
| `speed` | Integer, normally 20–100 | Optional speed for that position action; omission uses 100%, so include it when a slower action is wanted |
| `light` | Integer, 0–100 | Optional lighting level; use nonzero values for an effective scheduled action in this release |

Every entry needs `time` and at least one of `position` or `light`. An entry can control both together. The built-in scheduler has no weekday, date, sunrise, or presence conditions.

Saving a schedule stores it, but does not enable AUTO automatically. Disabling AUTO preserves the entries. Manual curtain or light commands also leave AUTO unchanged, so a later scheduled entry can still change the output. Disable AUTO when you want manual control without subsequent scheduled actions.

An entry runs once per local day while AUTO is enabled and the device clock is synchronized. If the device becomes ready within about two minutes after an entry's time, the entry can still run; missed actions outside that window are not replayed. Restarting the controller or replacing its schedule resets the record of executed entries, so an entry still inside that window may run again. The WebUI Reload button only reloads the editor and does not itself restart scheduled jobs.

Changes to a schedule replace its full list. Keep all entries you want when sending `/schedule/set`. To remove every entry, delete them in the WebUI and save; the current MQTT handler ignores a bare `[]` payload. For recurring rules more complex than daily times, use Home Assistant automations.

### Logs and System

- **Logs → Main log** shows recent activity since startup; **Error log** shows saved records. Press Refresh to load records.
- **System → Restart** restarts the controller after confirmation.
- **System → Choose firmware** uploads an update; restart afterwards to install it.

## 7. MQTT topics and data formats

Configure the broker address and credentials in Network settings, then restart. Each controller uses this topic prefix:

```text
dev/<device-name>
```

For a device named `LivingRoom`, its position command topic is `dev/LivingRoom/position/value/set`. Topic names and uppercase command payloads below are case-sensitive. Publish commands without retaining them, so reconnecting does not replay old actions.

### Connection and payload rules

The controller connects to an MQTT broker over TCP port **1883**. WebUI admin credentials and MQTT credentials are separate. Use a unique device name for each controller and restart after changing its name or broker settings.

Payloads are UTF-8 text:

- Numbers are decimal integer text, such as `50`, without `%` or units.
- Action commands are plain text, such as `OPEN`, not a JSON string containing quotation marks.
- Boolean commands accept `ON`/`OFF`, `TRUE`/`FALSE`, or `1`/`0`; reported Boolean states use `ON`/`OFF`.
- Settings use a JSON **object**, while schedules use a JSON **array**. JSON numbers and Boolean values must have their proper types: `60` and `true`, not `"60"` and `"true"`.
- Use double quotes for JSON keys and string values. In terminal examples, surrounding single quotes protect the payload from the shell and are not sent.

### Command topics: client → controller

The table lists suffixes to append to `dev/<device-name>`.

| Topic suffix | Data format and example | Effect |
|---|---|---|
| `/position/value/set` | Text: `OPEN`, `CLOSE`, `STOP`; integer text: `50` | Open, close, stop, or move to 0–100% closed. A new position command during movement stops it. |
| `/speed/value/set` | Integer text: `60`; use `20`–`100` | Save the speed for movement commands. |
| `/position/auto/set` | Boolean text: `ON` or `OFF` | Enable/disable the daily schedule without deleting its entries. |
| `/light/top/state/set` | Boolean text: `ON` or `OFF` | Turn on at the saved brightness, or turn off without clearing it. |
| `/light/top/value/set` | Integer text: `40` | Save brightness and apply it immediately. Values are clamped to 5–100%; send state `OFF` for off. |
| `/config/set` | JSON object: `{"speed":60,"timezone":1}` | Update the supplied settings and save them. Omitted settings are preserved. Restart for startup-dependent settings. |
| `/schedule/set` | JSON array: `[{"time":"21:00","position":100,"speed":50}]` | Replace and save the complete schedule. This is not an append operation. |
| `/schedule/rq` | Any text, for example `1` | Request the current schedule on `/schedule`. |

Boolean commands also accept `TRUE`/`FALSE` and `1`/`0`. To clear all schedule entries, use the WebUI; the current MQTT handler ignores an empty `[]` payload.

Examples using Mosquitto command-line clients:

```sh
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/position/value/set -m OPEN
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/position/value/set -m 50
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/position/value/set -m STOP
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/position/auto/set -m ON
mosquitto_sub -h 192.168.1.10 -t 'dev/LivingRoom/#' -v
```

Replace the broker address and device name. Add your client's username/password options if your broker requires them.

### State topics: controller → client

| Topic suffix | Data format | Example / meaning |
|---|---|---|
| `/status` | Lowercase text | `online` or `offline`; device availability |
| `/position/current` | Integer text | `35` = 35% closed; `-1` can indicate an unhomed position |
| `/position/value` | Integer text | Action/target notification, for example `100`; use `/position/current` for measured position |
| `/position/moving` | Uppercase text | `OPENING`, `CLOSING`, or `STOPPED` |
| `/position/attributes` | JSON object | `{"error":"ON","error_message":"Motor stuck"}`; `error` is an `ON`/`OFF` string |
| `/speed/value` | Integer text | `60`; saved motor speed percentage |
| `/position/auto` | Uppercase text | `ON` / `OFF`; daily schedule enabled/disabled |
| `/light/top/state` | Uppercase text | `ON` / `OFF`; lighting state |
| `/light/top/value` | Integer text | `40`; current lighting brightness percentage |
| `/config` | JSON object | `{"name":"LivingRoom","wifiPass":"***"}`; settings with passwords masked; example abbreviated |
| `/schedule` | Array-like text | `[{'time': '21:00', 'position': 100}]`; see the format note below |
| `/system/uptime` | Integer text | `3600`; seconds since startup |
| `/system/rssi` | Signed integer text | `-62`; WiFi signal strength in dBm |
| `/system/ip` | IPv4 address text | `192.168.1.50` |
| `/system/mac` | MAC address text | `aa:bb:cc:dd:ee:ff` |
| `/system/ram` | Integer text | `24`; allocated heap memory percentage |

State messages are normally retained, so a subscription can receive the most recently published value immediately. System diagnostics are published about every 30 seconds during normal operation; measured curtain position is published when it changes, approximately once per second while moving. An empty retained value is not a command acknowledgement; observe the state and logs to verify an action.

Schedule messages currently may contain single-quoted entry text rather than strict JSON. For a portable JSON backup, use **Configuration → Schedule → Edit JSON**.

The broker reports an unexpected disconnect through the controller's offline will message after detecting the lost connection; detection is not instantaneous.

### More MQTT examples

These examples use a POSIX shell; adapt quoting if using a different terminal. Replace the broker address and device name, and supply broker credentials when required.

```sh
# Switch the curtain light on at its saved brightness, then set it to 40%.
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/light/top/state/set -m ON
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/light/top/value/set -m 40

# Change only the maximum travel setting. Restart afterwards.
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/config/set -m '{"max_distance":300}'

# Replace the daily schedule, then enable it.
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/schedule/set -m '[{"time":"06:30","position":0,"speed":60},{"time":"21:00","position":100,"light":30}]'
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/position/auto/set -m ON

# Request the current schedule; listen for its reply on dev/LivingRoom/schedule.
mosquitto_pub -h 192.168.1.10 -t dev/LivingRoom/schedule/rq -m 1
```

## 8. Home Assistant integration

### Connect through MQTT discovery

1. Have an MQTT broker available, such as Mosquitto, and create credentials the controller can use.
2. In Home Assistant, open **Settings → Devices & services → Add integration → MQTT** and connect it to that broker. Keep MQTT discovery enabled with the default discovery prefix, **`homeassistant`**. The [official MQTT integration guide](https://www.home-assistant.io/integrations/mqtt) describes these settings.
3. In the controller's **Configuration → Settings → Network**, enter the same broker's address and the controller's MQTT credentials. Give the controller a unique device name, save, and restart.
4. Once the controller connects, look for its device under the Home Assistant MQTT integration. It announces its entities automatically when the release's discovery data is installed; manual entity YAML is not required.
5. Add the discovered controls and sensors you want to a dashboard.

The controller re-announces discovery when it reconnects to the broker and when Home Assistant announces that its MQTT connection is online. If discovery is missing, check the broker credentials and access permissions, the `homeassistant` discovery prefix, and the device's MQTT connection. Restart the controller to request a fresh announcement.

### Available entities

| Entity | Everyday use |
|---|---|
| Curtains / cover | Open, close, stop, and select a position |
| Top Light | Switch the curtain lighting and adjust brightness |
| Speed | Set motor speed, 20–100% |
| Auto | Enable or disable the controller's built-in daily schedule |
| Schedule | View schedule text or send a complete replacement schedule |
| Schedule Request | Request an updated schedule value |
| Motor Error | Show a motor fault and its associated message |
| Online | Show controller availability |
| Uptime, WiFi Signal, RAM, IP, MAC | Inspect device diagnostics |

Home Assistant uses **100% = fully open**, whereas direct MQTT/WebUI positions use **0% = fully open**. The supplied discovery configuration translates between them automatically. For example, Home Assistant 75% open corresponds to a direct controller position of 25% closed.

Use the cover entity's Stop control when you want to stop movement explicitly. If you instead send another position while moving, the controller stops that movement; send the new target again after it stops.

### Home Assistant automations or device schedules

Choose the controller's built-in schedule when you want daily actions to continue without Home Assistant running. WiFi/NTP must still be available when the controller needs to establish its clock after startup. Use Home Assistant automations for conditions such as presence, sunrise, or weekdays; those conditions are not part of the built-in daily scheduler.

The Auto switch controls only the built-in schedule. Turning it off does not disable Home Assistant automations. Avoid overlapping automations and built-in entries, because a second position command during motion acts as a stop.

The discovered Schedule text can use single quotes in its reported value. Submit valid JSON when editing it; use the WebUI for convenient entry editing and clearing all entries.

## 9. Curtain lighting

The controller has one dimmable lighting output on **GPIO7**, called **Light** in the WebUI and **Top Light** in Home Assistant/MQTT. Connect it through a suitable light driver. Lighting operates independently of the curtain position; it does not automatically follow opening or closing.

| Control | How to use it |
|---|---|
| WebUI | Open **Overview → Control → Light**. Use **Light on** for on/off and **Brightness** for 0–100%. Moving the slider applies its brightness immediately. |
| Home Assistant | Use the discovered **Top Light** entity's switch and brightness control. |
| MQTT | Send `ON`/`OFF` to `/light/top/state/set`; send a brightness integer to `/light/top/value/set`. |
| IR remote | Tap LIGHT to toggle; hold it to dim. The dimming direction changes between holds. |
| Daily schedule | Add a Light or Position + Light entry and set a nonzero brightness. |

Switching off through the WebUI or MQTT state command keeps the saved brightness for the next switch-on. Changing brightness through WebUI or MQTT saves it; remote dimming saves the setting when LIGHT is released. A brightness command can turn the light on immediately, even if it was previously off.

MQTT brightness commands are limited to **5–100%**, and remote dimming stops at about **10%**. Use an explicit off command when you want darkness. A scheduled nonzero brightness applies to the output but does not change the saved manual switch-on brightness. In this release a scheduled value of `0` is ignored, so scheduled light-off needs an external automation that sends an explicit `OFF` command.

The **Light brightness %** setting under General is the saved manual brightness. The separate LED brightness controls under **Indication** adjust the status indicators, not the curtain light. The lighting output starts off after a restart until a control or schedule action changes it.

## 10. IR remote, homing, and maximum travel

### Remote buttons

| Button | Short press | Long press |
|---|---|---|
| OPEN | Open; stop if already moving | Home the open position |
| CLOSE | Close; stop if already moving | Calibrate travel, after homing |
| AUTO | Toggle the daily schedule | — |
| LIGHT | Toggle lighting | Adjust brightness while held |

For homing or calibration, hold the button for roughly two seconds or more, then release it. Lighting dimming direction alternates between holds.

In **Settings → IR remote**, set the NEC remote address and the decimal codes for each action. Multiple codes for the same action are comma-separated strings, for example `10,2`. Defaults are address `0`, Open `10,2`, Close `8,1`, Auto `6`, and Light `5,5`; missing or invalid action-code values fall back to those defaults. Save and restart after changing remote settings.

### Homing: establish the fully open reference

Homing finds the open-side end switch on **GPIO18**. It sets the reference used for percentage positions; it does **not** measure or replace the maximum travel setting.

1. Check motor direction, encoder/magnet alignment, and both end-switch connections.
2. After normal startup, hold OPEN on the configured remote for about two seconds or more, then release it.
3. Wait for the open-side homing movement to finish before sending another command.
4. If the open switch was already pressed, the controller can first back away. Release the switch through that movement, then issue homing again if necessary.

The controller needs to establish its reference after a restart. A normal Open/Close/position command from WebUI, MQTT, or Home Assistant can initiate homing automatically when needed. Allow it to finish and confirm the position before sending the desired target again. For an explicit re-home, use the remote's long OPEN action.

### Measure and save the maximum length automatically

Maximum length is the curtain's full travel in **millimetres**, represented by **Max travel (mm)** in Settings and `max_distance` in JSON. This is the distance represented by a 100% closed command, not a percentage or a motor speed.

1. Complete homing first; travel calibration requires an already homed controller.
2. Hold CLOSE on the remote for about two seconds or more, then release it.
3. Wait while the curtains move towards the close-side switch on **GPIO8**.
4. When that switch is reached, the controller saves the measured maximum travel with a small closing clearance, then returns towards open.
5. Reload **Configuration → Settings** and check **Max travel (mm)**. Test intermediate positions and full closure at a moderate speed.

The firmware leaves approximately **10 mm** before the measured closed limit, so the saved travel can be shorter than the raw switch-to-switch measurement. Calibrate again after changing the mechanism, magnet coupling, or end-switch positions.

### Set maximum length manually, including without a remote

If you already know the correct travel distance for the installed mechanism:

1. Open **Configuration → Settings → General**.
2. Set **Max travel (mm)** to the required positive distance, for example `300` for 300 mm. Never use zero: position percentages depend on this value.
3. Save and restart the controller. A manual change to this setting is used after restart.
4. Home the curtains using the remote or let a normal position command initiate homing.
5. Check a short movement first, then intermediate positions and full closure.

Alternatively, publish a partial settings update to `dev/<device-name>/config/set`:

```json
{"max_distance":300}
```

Restart after this MQTT update as well. Entering a maximum length does not home the device and does not correct an incorrect shaft diameter, microstepping setting, or sensor installation.

Homing can run automatically without a remote, and a known maximum length can be entered through WebUI or MQTT. The current firmware does not expose a dedicated homing or automatic travel-calibration command through those interfaces; the remote's long CLOSE action is required to start the automatic full-travel measurement.

## 11. Status indicators and troubleshooting

| Indicator | Meaning |
|---|---|
| STATE/red steady | Motor moving |
| STATE/red blinking | Motor error |
| CONNECTION/blue steady | WiFi and MQTT connected |
| CONNECTION/blue pulsing | WiFi connected; MQTT not connected |
| CONNECTION/blue off | No WiFi connection |
| AUTO/green on | Daily schedule enabled |
| STATE and CONNECTION blinking quickly together after startup | Configuration/AP mode |

The WS2812 indicator mirrors these states. Indicator brightness is adjustable in Settings → Indication and can be reduced when the curtains are near the configured closed position.

| Problem | What to check |
|---|---|
| Flashing tool cannot connect | USB data cable, correct serial port, closed serial monitor, and BOOT/RESET download sequence |
| Controller does not boot after flashing | Firmware variant, complete matching flash files, and power supply |
| Normal WiFi network is unavailable | Use the GPIO21 setup switch, reconnect to `CURTAINS`, and correct WiFi settings |
| Controller repeatedly starts its access point | Open the GPIO21 switch and release remote buttons before restarting |
| Browser asks for a login | Use the configured admin credentials; AP mode uses them too |
| WebUI looks outdated | Hard-refresh or clear cached website files after updating |
| Curtains do not move or move incorrectly | Motor power, driver wiring, encoder/magnet, end switches, homing, and travel calibration |
| Motor stops with an error | Remove a mechanical obstruction and check the encoder and switches; a new move command clears the error before retrying |
| Timed actions do not run | Save the schedule, enable AUTO, and check device time, UTC offset, and NTP connectivity |
| Home Assistant device is missing | Check MQTT connectivity, discovery settings, and that the release included discovery data |
| Memory/RSSI chart is empty | Leave the page open for status updates; old samples expire and a restart resets history |
