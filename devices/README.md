# Device Configuration

## Architecture

Zone infrastructure and control profiles are **separated**:

```
zones/
└── base.yaml          # Cover, BEMF, GPIO, globals, parameters

control/
├── tanh.yaml          # Tanh control algorithm
├── linear.yaml        # Linear control algorithm
└── pid.yaml           # PID control algorithm
```

This separation makes it easy to:
- Debug issues in specific components
- Change control profile without touching zone infrastructure
- Maintain and extend each part independently

## Quick Setup

```yaml
# Define zone variables once using YAML anchors
.zone_1_vars: &zone_1_vars
  zone_number: "1"
  id: living
  friendly_name: "Living Room"
  # ... all variables

packages:
  # 1. Include base infrastructure
  zone_1_base: !include
    file: floor-heating/zones/base.yaml
    vars: *zone_1_vars

  # 2. Include control profile (same file for all zones)
  zone_1_ctrl: !include
    file: floor-heating/control/tanh.yaml    # ← Change here to switch profile
    vars: *zone_1_vars
```

## Control Profiles

| File | Algorithm | Best For |
|------|-----------|----------|
| `control/tanh.yaml` | Non-linear S-curve | UFH (recommended) |
| `control/linear.yaml` | Linear interpolation | Simple systems |
| `control/pid.yaml` | PID controller | Precise control |
| `control/remote.yaml` | External control | Home Assistant integration |

### Switching Profile

To change control profile for ALL zones, change the `file:` path:

```yaml
# From tanh to linear:
zone_1_ctrl: !include
  file: floor-heating/control/linear.yaml   # Changed
  vars: *zone_1_vars

zone_2_ctrl: !include
  file: floor-heating/control/linear.yaml   # Changed
  vars: *zone_2_vars
```

### Profile-Specific Variables

| Profile | Extra Variables |
|---------|-----------------|
| tanh | `tanh_steepness: "0.70"` |
| linear | (none) |
| pid | `target_temp_default: "21"`, `pid_kp`, `pid_ki`, `pid_kd` |
| remote | (none) |

### Remote Profile

The remote profile exposes the thermostat and valve cover for external control.
Valve position is controlled directly via:
- Home Assistant `cover` entity
- ESPHome web interface
- API calls

Useful when you want Home Assistant automations to control valve positions.

## Operating Modes

| Mode | Setup |
|------|-------|
| **Standard** | Default (no extra config) |
| **Hydraulic** | Add `modes/hydraulic.yaml` + enable switch |

```yaml
packages:
  # Hydraulic mode - uncomment to enable
  # hydraulic: !include floor-heating/modes/hydraulic.yaml
```

## Complete Folder Structure

```
floor-heating/
├── boards/
│   ├── esp32.yaml
│   ├── esp32-s3.yaml
│   └── esp32-c3.yaml
├── core/
│   ├── settings.yaml      # Global parameters
│   ├── logger.yaml
│   ├── wifi.yaml
│   ├── time.yaml
│   ├── sn74hc595.yaml
│   ├── sensor_adc.yaml
│   ├── sensor_others.yaml
│   ├── switch_others.yaml
│   ├── inputs.yaml
│   └── onewire.yaml
├── zones/
│   └── base.yaml          # Zone infrastructure
├── control/
│   ├── tanh.yaml          # Tanh profile
│   ├── linear.yaml        # Linear profile
│   ├── pid.yaml           # PID profile
│   └── remote.yaml        # Remote/external control
├── modes/
│   ├── standard.yaml      # Documentation only
│   └── hydraulic.yaml     # Hydraulic balancing
├── sensors/
│   └── return_temp.yaml   # Return temperature
└── optional/
    ├── pump_control.yaml
    └── mqtt_ecodan.yaml
```

## Component Responsibilities

### zones/base.yaml
- Valve cover (endstop-based)
- GPIO switches for motor
- BEMF sensor
- Zone globals (state, hydraulic_factor)
- Zone parameters (area, max_opening, BEMF trigger)
- Pipe length calculation
- Calibration script

### control/tanh.yaml, linear.yaml
- Climate thermostat
- Check script (TH_check)
- Periodic interval

### control/pid.yaml
- PID climate
- Heat output
- PID tuning parameters (Kp, Ki, Kd)
- PID diagnostic sensors
- Autotune button

### modes/hydraulic.yaml
- Hydraulic balancing engine
- Calculates hydraulic_factor per zone

## Hardware Mapping

| Zone | BEMF ADC | IA Pin | IB Pin |
|------|----------|--------|--------|
| 1 | BEMF_1_2_sensor_ADC | 0 | 1 |
| 2 | BEMF_1_2_sensor_ADC | 2 | 3 |
| 3 | BEMF_3_4_sensor_ADC | 4 | 5 |
| 4 | BEMF_3_4_sensor_ADC | 6 | 7 |
| 5 | BEMF_5_6_sensor_ADC | 8 | 9 |
| 6 | BEMF_5_6_sensor_ADC | 10 | 11 |
| 7 | BEMF_7_8_sensor_ADC | 12 | 13 |
| 8 | BEMF_7_8_sensor_ADC | 14 | 15 |
