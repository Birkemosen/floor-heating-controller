# Floor Heating Controller

ESPHome firmware for [esp32_8ch_motor_shield](https://github.com/nliaudat/esp32_8ch_motor_shield/)

## Features

- **Modular architecture**: Separate files for zones, control profiles, and modes
- **4 control profiles**: Tanh (recommended), Linear, PID, Remote
- **2 operating modes**: Standard or Hydraulic Balancing
- **Configurable zones**: 1-8 zones per controller
- **Multiple temperature sources**: Home Assistant, Dallas/OneWire, DHT, BLE

## Folder Structure

```
floor-heating-controller/
├── boards/          # ESP32, ESP32-S3, ESP32-C3
├── core/            # Core infrastructure
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
├── devices/         # Device config examples
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
   cd floor-heating
   cp secrets.yaml.example secrets.yaml
   # Edit secrets.yaml with your WiFi credentials
   ```

3. **Configure** `config.yaml`:
   - Set `control_profile` (tanh/linear/pid/remote)
   - Configure zones with your temperature sensors
   - Set Dallas sensor addresses for supply/return

4. **Compile & Upload**:
   ```bash
   esphome run config.yaml
   ```

## Configuration

### Global Settings

```yaml
substitutions:
  # Control profile for ALL zones (change here to switch)
  control_profile: "zones/tanh.yaml"
  # Options: zones/tanh.yaml, zones/linear.yaml, 
  #          zones/pid.yaml, zones/remote.yaml

packages:
  # Enable hydraulic balancing (optional)
  # hydraulic: !include modes/hydraulic.yaml
```

### Control Profiles

| Profile | Algorithm | Best For |
|---------|-----------|----------|
| `zones/tanh.yaml` | Non-linear S-curve | UFH (recommended) |
| `zones/linear.yaml` | Linear interpolation | Simple systems |
| `zones/pid.yaml` | PID controller | Precise control |
| `zones/remote.yaml` | External control | Home Assistant |

### Operating Modes

| Mode | Description |
|------|-------------|
| **Standard** | Temperature control only (default) |
| **Hydraulic** | + Pipe-length based balancing |

## Multi-Controller Setup

For ESPHome Docker with multiple boards:

```
/mnt/data/esphome/config/
├── floor-heating/          # This repo (git clone)
├── controller-1.yaml       # Device 1 config
├── controller-2.yaml       # Device 2 config
└── secrets.yaml            # Shared secrets
```

See `devices/` folder for example configurations.

## Documentation

- [Quick Start Guide](docs/QUICK_START.md)
- [Hydraulic Balancing](docs/HYDRAULIC_BALANCING.md)
- [Device Configuration](devices/README.md)

## First Time Upload

1. Detach ESP32 board from shield, connect via USB
2. Press BOOT button for 2-3 seconds before flashing
3. After OTA update, press EN (reset) button
4. Future uploads will be wireless (OTA)

## ESP-NOW Pump Control

Add to your pump relay ESP config:

```yaml
espnow:
  peers:
    - "{MAC ADDRESS OF THE FLOOR HEATING CONTROLLER}"
  auto_add_peer: False
  on_receive:
    - lambda: |-
        std::string received_data((const char*)data, size);
        if (received_data == "PUMP_ON") {
          id(relay).turn_on();
        } else if (received_data == "PUMP_OFF") {
          id(relay).turn_off();
        }
```

## License

See [LICENSE](LICENSE) file.
