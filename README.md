# Floor Heating Controller

ESPHome firmware for custom floor heating controller using ESP32-S3 Super Mini with TB6612FNG motor drivers.

## Features

- **Hardware**: ESP32-S3 Super Mini + 2x TB6612FNG motor drivers
- **8 motor channels** with current-based endstop detection
- **Modular architecture**: Separate files for zones, control profiles, and modes
- **4 control profiles**: Tanh (recommended), Linear, PID, Remote
- **2 operating modes**: Standard or Hydraulic Balancing
- **Multiple temperature sources**: Home Assistant, Dallas/OneWire, DHT, BLE

## Hardware

### GPIO Pinout (ESP32-S3 Super Mini)

| GPIO | Function | Description |
|------|----------|-------------|
| GPIO1 | Mux | HIGH=N1 (odd motors), LOW=N2 (even motors) |
| GPIO2 | Enable 0 | Motors 1 & 2 (TB6612#1 PWMA) |
| GPIO3 | Enable 1 | Motors 3 & 4 (TB6612#1 PWMB) |
| GPIO4 | Enable 2 | Motors 5 & 6 (TB6612#2 PWMA) |
| GPIO5 | Enable 3 | Motors 7 & 8 (TB6612#2 PWMB) |
| GPIO6 | Direction | All motors (HIGH=open, LOW=close) |
| GPIO7 | Standby | Driver enable (LOW=active) |
| GPIO8 | I2C SDA | I2C Data |
| GPIO9 | I2C SCL | I2C Clock |
| GPIO10 | Current ADC | Motor current sensing |
| GPIO11 | Tacho | Revolution counting |
| GPIO12 | Ref ADC | Reference voltage |
| GPIO13 | 1-Wire | Dallas temperature sensors |
| GPIO43 | UART TX | UART0 Transmit |
| GPIO44 | UART RX | UART0 Receive |
| GPIO48 | Status LED | Standard LED + WS2812 RGB |

### Motor Mapping

| Motor | Enable Pin | Mux State | Output |
|-------|------------|-----------|--------|
| 1 | GPIO2 | HIGH (N1) | TB6612#1 AO1 |
| 2 | GPIO2 | LOW (N2) | TB6612#1 AO2 |
| 3 | GPIO3 | HIGH (N1) | TB6612#1 BO2 |
| 4 | GPIO3 | LOW (N2) | TB6612#1 BO1 |
| 5 | GPIO4 | HIGH (N1) | TB6612#2 AO1 |
| 6 | GPIO4 | LOW (N2) | TB6612#2 AO2 |
| 7 | GPIO5 | HIGH (N1) | TB6612#2 BO2 |
| 8 | GPIO5 | LOW (N2) | TB6612#2 BO1 |

**Note**: Only ONE motor can run at a time due to the multiplexed design.

### Status LED Color Codes (WS2812)

| Color | Effect | Status |
|-------|--------|--------|
| 🟢 Green | Solid | Normal, connected to Home Assistant |
| 🟢 Green | Breathing | Normal, standalone mode |
| 🔵 Blue | Flashing | Booting / WiFi connecting |
| 🔵 Cyan | Solid | Motor running |
| 🟡 Yellow | Solid | Warning (sensor offline) |
| 🟠 Orange | Flashing | No 1-Wire sensors found |
| 🔴 Red | Solid | Error (API disconnected, standalone OFF) |
| 🔴 Red | Flashing | Critical error (motor stall) |
| 🟣 Purple | Solid | Calibration in progress |
| ⚪ White | Flash | Command received |

## Folder Structure

```
floor-heating-controller/
├── boards/          # ESP32-S3 Super Mini only
├── core/            # Core infrastructure
│   ├── tb6612.yaml  # Motor driver control
│   ├── settings.yaml
│   └── ...
├── zones/           # Zone templates (base + control combined)
│   ├── tanh.yaml    # Tanh control (recommended for UFH)
│   ├── linear.yaml  # Linear control
│   ├── pid.yaml     # PID control
│   └── remote.yaml  # External/Home Assistant control
├── control/         # Control profiles (used internally)
├── modes/           # Operating modes
│   └── hydraulic.yaml
├── sensors/         # Sensor templates
├── optional/        # Pump control, MQTT integration
└── docs/            # Documentation
```

## Quick Start

1. **Clone** the repository:
   ```bash
   cd /mnt/data/esphome/config
   git clone https://github.com/birkemosen/floor-heating-controller.git floor-heating
   ```

2. **Setup secrets**:
   ```bash
   cp floor-heating/secrets.yaml secrets.yaml
   # Edit secrets.yaml with your WiFi credentials
   ```

3. **Create your config** (copy and modify `config.yaml`):
   - Set zone names and temperature sensors
   - Set Dallas sensor addresses for supply/return
   - Adjust motor mapping (`motor_number`) for each zone

4. **Compile & Upload**:
   ```bash
   esphome run config.yaml
   ```

## Configuration

### Control Profiles

| Profile | Algorithm | Best For |
|---------|-----------|----------|
| `zones/tanh.yaml` | Non-linear S-curve | UFH (recommended) |
| `zones/linear.yaml` | Linear interpolation | Simple systems |
| `zones/pid.yaml` | PID controller | Precise control |
| `zones/remote.yaml` | External control | Home Assistant |

### Zone Configuration

```yaml
zone_1: !include
  file: floor-heating/zones/tanh.yaml
  vars:
    zone_number: "1"
    motor_number: "1"          # Physical motor (1-8)
    id: living
    friendly_name: "Living Room"
    temperature_sensor: living_temp
    current_factor: "1.7"      # Endstop detection sensitivity
    zone_area_default: "25"
    zone_max_opening_default: "90"
    # ... temperature presets ...
```

### Hydraulic Balancing

Enable for pipe-length based flow balancing:

```yaml
packages:
  hydraulic: !include floor-heating/modes/hydraulic.yaml
```

## First Time Upload

1. Connect ESP32-S3 via USB
2. Press BOOT button for 2-3 seconds before flashing
3. After OTA update, press EN (reset) button
4. Future uploads will be wireless (OTA)

## Documentation

- [Quick Start Guide](docs/QUICK_START.md)
- [Hydraulic Balancing](docs/HYDRAULIC_BALANCING.md)

## License

See [LICENSE](LICENSE) file.
