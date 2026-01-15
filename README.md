# Smart Garage Door Controller with ESP32

This repository contains the firmware and configuration to transform any Hörmann Promatic 3 garage door opener into a smart, Home Assistant-controllable device. The system uses an **ESP32 Relay x2 (ESP32-WROOM-32E)** smart relay module to relay commands to the garage door drive unit while monitoring door position through a **TOF400C-VL53L1X** time-of-flight distance sensor and closed-position detection via a 24V optocoupler.

---

##  Key Capabilities
- **Remote Control**: Open, close, and ventilate (partial open) your garage door from Home Assistant
- **Position Feedback**: Real-time door position tracking using ToF sensor with millimeter precision
- **Safety Detection**: Closed position confirmation via hardware optocoupler
- **Local Integration**: Fully local control—no cloud dependency
- **Automation Ready**: Trigger automations based on door state and position
- **Voice Control**: Works with Home Assistant voice integrations (Alexa, Google Home)

---

## Hardware Requirements

| Component | Model/Specification | Purpose | Picture |
|-----------|-------------------|---------|----------|
| Main controller | ESP32 Relay x2 (ESP32-WROOM-32E) | Main controller, WiFi, Relays (Open/close/airing garage door) | [ESP32 Relay x2 (ESP32-WROOM-32E)](docs/ESP32%20Relay%20x2%20(ESP32-WROOM-32E).jpg) |
| Position Sensor | 1-Channel 24V Optocoupler | Closed-state detection | [1-Channel 24V Optocoupler](docs/1%20Channel%20Way%20Optocoupler%2024V.jpg) |
| Distance Sensor | TOF400C-VL53L1X (I²C) | Position feedback, addr 0x29 | [TOF400C-VL53L1X](docs/TOF400C-VL53L1X.png) |

### Calibration Constants (from garage.yaml)
- **Closed position distance**: 2630 mm (door fully closed, sensor facing floor)
- **Open position distance**: 240 mm (door fully open)
- **Airing position**: 9% (0.09 × 100, triggered by Relay 2 + binary_airing flag)
- **Debounce time**: 50 ms (end_stop_bottom optocoupler)
- **Relay pulse duration**: 800 ms (impulse command activation)

### Optional Components
- 10 kΩ pull-up resistors (for ToF I²C SDA/SCL lines)
- **3D-printed enclosure and sensor mount** — See [3D Models](#3d-models) section for STL files and printing guide

---

## Installation

### 1. Hardware Assembly

#### Wiring Diagram

![Wiring Diagram](docs/Schematic_Garage_2025-12-16%20(ESP32-WROOM-32E%20Relay%20x2).svg)

```
Connections:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GPIO16  → Relay 1 (Open/close command)
GPIO17  → Relay 2 (Ventilation command)
GPIO21  → TOF400C I²C SDA (optional with 10kΩ pull-up)
GPIO22  → TOF400C I²C SCL (optional with 10kΩ pull-up)
GPIO23  → LED anode (330Ω resistor to GND)
GPIO32  → 24V Optocoupler input (INPUT_PULLUP)
GND     → Common ground (all modules)
5V      → Relay module VCC, ESP32 5V
Vin     → Optional 5V USB power

Hörmann Promatic 3 Control:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Relay 1 NO contacts  → Hörmann "Open" impulse terminals
Relay 1 NC contacts  → Hörmann "Close" impulse terminals
Relay 2 NO contacts  → Hörmann "Ventilation" impulse terminals
24V common           → Optocoupler cathode
```

---

## 3D Models

The project includes 3D-printable box in [`3D_model/`](3D_model/) directory.

---

## Firmware Upload

### Prerequisites
- **Home Assistant** instance running with ESPHome add-on installed
- **ESPHome Dashboard** web interface accessible (typically port 6052)
- **ESP32-WROOM-32E** connected via USB-C to the computer running Home Assistant
- **Secrets file** (`secrets.yaml`) with required credentials:
  ```yaml
  # secrets.yaml
  wifi_ssid: "Your_WiFi_Name"
  wifi_password: "Your_WiFi_Password"
  fallback_ap_password: "ApFallbackPassword"
  web_server_user: "admin"
  web_server_password: "YourWebPassword"
  garage_api: !secret api_encryption_key  # Generate via ESPHome
  ota_password: "YourOTAPassword"
  ```

### Step-by-Step Upload

1. **Prepare configuration file**
   - Copy the provided [`garage.yaml`](garage.yaml) to your Home Assistant configuration
   - Update GPIO pin assignments if different from your hardware
   - Edit substitution variables (friendly names, sensor calibration values)

2. **Access ESPHome Dashboard**
   - Open Home Assistant → Settings → Add-ons → ESPHome
   - Click "Open Web UI"
   - Ensure you see the ESPHome dashboard interface

3. **Create New Device**
   - Click "+ NEW DEVICE" button
   - Select "Continue"
   - Choose board type: **"ESP32 Dev Kit v1"** or similar
   - Enter device name: `garage` (or your preference)
   - Click "Next" → "Skip"

4. **Upload Configuration**
   - In dashboard, click the device card or **"Edit"** button
   - ⚠️ Copy `ap:` `password:` from auto-generated YAML to your `secrets.yaml` ⚠️
   - Replace the auto-generated YAML with your `garage.yaml` configuration
   - Click **"Save"**
   - ESPHome will validate the configuration

5. **Build & Upload Firmware**
   - Connect ESP32 via USB-C to your Home Assistant host
   - Click **"···" menu** → **"Upload"**
   - Select connection type: **"USB"** (or choose serial port)
   - Wait for compilation (~2–3 minutes) and upload
   - You should see: **`✓ Installed Successfully`**
   - Monitor serial logs during upload for any errors

6. **Verify Installation**
   - Disconnect USB cable (device will connect via WiFi)
   - In ESPHome dashboard, confirm device shows **"Online"** status
   - Go to Home Assistant → Settings → Devices & Services → ESPHome
   - You should see:
     - **Cover entity**: `cover.garage` (or `cover.tor`)
     - **Binary sensors**: `binary_sensor.geöffnet` (opened), `binary_sensor.lüftung` (airing)
     - **Sensors**: Position %, distance (mm), range status
   - Click the device to view detailed metrics

### Alternative: Manual Flashing (Advanced)

If USB connection is unavailable on your Home Assistant instance:

```bash
# Install esphome CLI
pip install esphome

# Compile firmware
esphome compile garage.yaml

# Flash via USB serial adapter
esphome run garage.yaml --device /dev/ttyUSB0 --upload-speed 460800

# View logs
esphome logs garage.yaml
```

---

## Usage

### Home Assistant Interface

Once discovered and configured, the following entities appear:

| Entity ID | Type | Function |
|-----------|------|----------|
| `cover.tor_tor` (or `tor`) | Cover | Main door control with position slider (0–100%) |
| `button.tor_tor` | Button | Trigger open/close impulse |
| `button.tor_luftung` | Button | Trigger ventilation/airing impulse |
| `button.tor_restart` | Button | Restart device |
| `button.tor_safe_mode_boot` | Button | Safe mode boot |
| `binary_sensor.tor_geoffnet` | Binary Sensor | True when door is **open** (optocoupler inactive) |
| `binary_sensor.tor_luftung` | Binary Sensor | True when **ventilation mode** is active |
| `sensor.tor_position` | Sensor | Current door position in % (0=closed, 100=open) |
| `sensor.tor_distanz` | Sensor | Calibrate ToF distance in mm |
| `sensor.tor_raw_distanz` | Sensor | Raw ToF distance in mm |
| `sensor.tor_bereichsstatus` | Text Sensor | Range status (e.g., "Range Valid", "Signal Fail") |
| `sensor.tor_bereichsstatuswert` | Sensor | Range status (e.g., "1", "2") |
| `update.tor_firmware` | Update sensor | True when update is available |

### Control Methods

**Via Home Assistant UI:**
- Click the cover card to **open/close** the door
- Use the **position slider** for precise control:
  - Drag left (<5%) → **Close command**
  - Drag middle (10–20%) → **Ventilation** (9% opening)
  - Drag right (>95%) → **Open command**
- The **stop button** pauses movement

**Via Automations:**
```yaml
automation:
  - alias: "Close garage at night"
    trigger:
      platform: time
      at: "22:00:00"
    condition:
      - condition: state
        entity_id: cover.garage
        state: "open"
    action:
      - service: cover.close_cover
        target:
          entity_id: cover.garage

  - alias: "Open garage when arriving home"
    trigger:
      platform: state
      entity_id: person.john_doe
      to: "home"
    action:
      - service: cover.open_cover
        target:
          entity_id: cover.garage
```

**Via Scripts:**
```yaml
script:
  garage_ventilate:
    sequence:
      - service: cover.set_cover_position
        target:
          entity_id: cover.garage
        data:
          position: 15  # Triggers ventilation (9-20% range)

  garage_open:
    sequence:
      - service: cover.open_cover
        target:
          entity_id: cover.garage
```

**Via Voice Commands** (if Alexa/Google Home linked):
```
"Alexa, open my garage"
"Alexa, close my garage"
"Hey Google, open the garage door"
```

**Via MQTT** (if configured):
```bash
mosquitto_pub -h homeassistant.local -t home/garage/command -m "OPEN"
mosquitto_pub -h homeassistant.local -t home/garage/command -m "CLOSE"
mosquitto_pub -h homeassistant.local -t home/garage/command -m "VENTILATE"
```

---

## Configuration

### ESPHome YAML Configuration (garage.yaml)

The project includes a comprehensive [`garage.yaml`](garage.yaml) configuration file. Key sections:

#### 1. Substitutions (Customizable Variables)
```yaml
substitutions:
  name: garage
  room: Garage
  friendly_name: "Tor"  # German: "Gate"
  project_version: "2025.12.0"
  
  # GPIO Pin Assignments
  relay1_pin: GPIO16       # Open command
  relay2_pin: GPIO17       # Ventilation command
  status_led: GPIO23       # Status LED
  optocoupler_pin: GPIO32  # Closed-position sensor
  sda_pin: GPIO21          # I²C Data (ToF sensor)
  scl_pin: GPIO22          # I²C Clock (ToF sensor)
```

#### 2. WiFi Configuration
```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: true       # Connect quickly to known networks
  ap:
    ssid: "Garage Fallback Hotspot"
    password: !secret fallback_ap_password
```

#### 3. I²C Bus (TOF400C Sensor)
```yaml
i2c:
  - id: bus_a
    sda: GPIO21
    scl: GPIO22
    scan: true
    frequency: 400kHz  # Standard I²C frequency
```

#### 4. Cover Entity (Main Control)
```yaml
cover:
  - platform: template
    id: garage_door_cover
    name: "Tor"  # Display name
    device_class: garage
    has_position: true
    
    # Current position (0.0=closed, 1.0=open)
    lambda: |-
      return id(pos_sensor).state / 100.0;
    
    # Open command (Relay 1 pulse)
    open_action:
      - if:
          condition: lambda: 'return id(pos_sensor).state < 99.0;'
          then:
            - button.press: button_gate
    
    # Close command (Relay 1 pulse)
    close_action:
      - if:
          condition: binary_sensor.is_on: end_stop_bottom
          then:
            - button.press: button_gate
    
    # Position-based control
    position_action:
      - lambda: |-
          if (pos < 0.05) {
            // Close command
            if (id(end_stop_bottom).state)
              id(button_gate).press();
            id(binary_airing).publish_state(false);
          }
          else if (pos > 0.95) {
            // Open command
            if (id(pos_sensor).state < 95.0)
              id(button_gate).press();
            id(binary_airing).publish_state(false);
          }
          else if (pos > 0.10 && pos < 0.20) {
            // Ventilation (9% target)
            id(button_airing).press();
            id(binary_airing).publish_state(true);
          }
```

#### 5. Position Sensor (Lambda Calculation)
The position is calculated with priority logic:
```lambda
if (x != 0) return NAN;  // Validate range status

// 1. HARDWARE END-STOP (highest priority)
if (!id(end_stop_bottom).state)
  return 0.0;  // Door is closed

// 2. VENTILATION FLAG
if (id(binary_airing).state)
  return 9.0;  // Fixed ventilation position

// 3. DISTANCE-BASED CALCULATION
float max_dist = 2630.0;   // Closed (mm)
float min_dist = 240.0;    // Open (mm)
float calculated = ((max_dist - current_dist) / (max_dist - min_dist)) * 100.0;
return constrain(calculated, 0.0, 100.0);
```

### Configuration Customization

**Change GPIO pins:**
```yaml
substitutions:
  relay1_pin: GPIO4      # Change from GPIO16
  relay2_pin: GPIO5      # Change from GPIO17
  optocoupler_pin: GPIO35 # Change from GPIO32
```

**Adjust sensor calibration:**
```yaml
# In the position lambda function:
float max_dist = 2750.0;   # If your closed distance differs
float min_dist = 200.0;    # If your open distance differs
```

**Modify relay pulse duration:**
```yaml
button:
  - platform: output
    id: button_gate
    output: relay1
    duration: 1000ms  # Change from 800ms
```

**Enable debug logging:**
```yaml
logger:
  level: DEBUG  # More verbose output
  
text_sensor:
  - platform: template
    id: range_value
    name: "Range Status"
    disabled_by_default: false  # Show in UI
```

---

## Troubleshooting

### Device Not Connecting to WiFi
- **Symptom**: Fallback hotspot appears, no home network connection
- **Solution**:
  - Verify SSID and password in `secrets.yaml` are correct
  - Check WiFi signal strength in garage (RSSI > -75 dBm ideal)
  - Restart ESPHome device via Home Assistant → Devices → Restart
  - Check logs: ESPHome dashboard → Logs

### Distance Sensor Returns NAN or Invalid Readings
- **Symptom**: ToF sensor shows "NAN" in position sensor, range status is not 0
- **Solution**:
  - Verify I²C connections (GPIO21 SDA, GPIO22 SCL)
  - Check for 10kΩ pull-up resistors on SDA/SCL lines
  - Clean lens of TOF400C sensor (dust reduces accuracy)
  - Ensure sensor is perpendicular to garage door
  - Confirm sensor address is 0x29: ESPHome dashboard → Logs → "i2c: Found device at 0x29"
  - Test I²C with: `esphome logs garage.yaml | grep -i i2c`

### Relay Not Triggering Door
- **Symptom**: Door doesn't move when pressing open/close button
- **Solution**:
  - Verify relay wiring to Hörmann control terminals
  - Test relays independently with multimeter:
    - Touch relay coil pins → should show 5V when active
    - Measure NO/NC contacts → should toggle continuity
  - Increase relay pulse duration (currently 800ms):
    ```yaml
    duration: 1200ms  # Try longer pulse
    ```
  - Verify GPIO16 and GPIO17 are not used elsewhere
  - Check Hörmann manual for impulse timing requirements

### Optocoupler Not Detecting Closed Position
- **Symptom**: Door closed but binary_sensor.geöffnet shows "ON" (open)
- **Solution**:
  - Verify 24V optocoupler wiring:
    - Cathode (-) to 24V common
    - Anode (+) to Hörmann closed-position circuit
  - Check GPIO32 continuity with multimeter
  - Verify INPUT_PULLUP mode: `inverted: false` in YAML
  - Increase debounce time if signal is unstable:
    ```yaml
    filters:
      - delayed_on_off: 100ms  # Change from 50ms
    ```

### Door State Not Updating in Home Assistant
- **Symptom**: Position slider doesn't reflect actual door movement
- **Solution**:
  - Increase sensor update interval:
    ```yaml
    update_interval: 500ms  # Change from 1s
    ```
  - Verify `pos_sensor` is receiving values (not NAN)
  - Check Home Assistant logs: Settings → System → Logs
  - Force entity refresh: Developer Tools → States → Restart device

---

## Contributing

Contributions are welcome! Please follow these guidelines:

1. **Fork** the repository
   ```bash
   git clone https://github.com/YOUR_USERNAME/garage.git
   cd garage
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-improvement
   # e.g., git checkout -b feature/multi-sensor-fusion
   ```

3. **Make your changes**
   - Modify YAML configuration or add new features
   - Update documentation with clear comments
   - Test thoroughly with real Hörmann Promatic 3 hardware

4. **Commit with clear messages**
   ```bash
   git commit -am "Add multi-sensor redundancy for position detection"
   ```

5. **Push and submit pull request**
   ```bash
   git push origin feature/your-improvement
   ```
   - Provide detailed description of changes
   - Include before/after testing results
   - Reference any related issues

### Testing Requirements
Before submitting a PR:
- [ ] Verify door opens and closes correctly
- [ ] Test ventilation mode (9% position)
- [ ] Confirm optocoupler detects closed position
- [ ] Check WiFi reconnection after dropout
- [ ] Test OTA firmware update functionality
- [ ] Verify no sensor NAN errors in normal operation

---

## License

This project is licensed under the **MIT License**. See [LICENSE](LICENSE) file for details.

You are free to use, modify, and distribute this software for personal and commercial purposes, with attribution to the original author (Barma-lej).

---

## Support & Community

- **GitHub Issues**: [Report bugs](https://github.com/Barma-lej/garage/issues) or request features
- **Discussions**: [GitHub Discussions](https://github.com/Barma-lej/garage/discussions) for Q&A
- **ESPHome Docs**: [Official Documentation](https://esphome.io)
- **Home Assistant Community**: [Community Forums](https://community.home-assistant.io)
- **Hörmann Promatic 3 Manual**: [Manual DE](docs/hoermann-promatic-3-anleitung.pdf)

---

## Safety Disclaimer

⚠️ **IMPORTANT SAFETY INFORMATION**

This system interfaces with **moving machinery**. Proper installation and operation are critical:

### Before Installation
- [ ] Read Hörmann Promatic 3 manual thoroughly
- [ ] Ensure you have electrical installation competence (or hire professional)
- [ ] Verify local electrical and building codes compliance
- [ ] Check garage door is in good mechanical condition

### During Operation
- [ ] All original safety features must remain **fully functional**
- [ ] Emergency manual override must **always be accessible**
- [ ] Perform regular testing of:
  - [ ] Door movement and reversal
  - [ ] Closed-position detection
  - [ ] Obstacle detection (if present)
  - [ ] Emergency stop functionality
- [ ] Never allow children unsupervised near the door
- [ ] Do not modify relay impulse timing without testing

### Liability
The authors assume **no liability** for:
- Injury or property damage
- Improper installation or electrical work
- Failure of safety features
- Unauthorized modifications

---

**Project Status**: Active Maintenance  
**Last Update**: 2026-01-15  
**Contributors**: Barma-lej
