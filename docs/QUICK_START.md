# Quick Start Guide: Hydraulic Balancing Setup

## Prerequisites

- Floor heating controller hardware with ESP32/ESP32-S3
- DS18B20 temperature sensors installed at manifold returns (optional but recommended)
- ESPHome development environment

---

## Step 1: Configure Board Type

In `config.yaml`, uncomment the appropriate board:

```yaml
packages:
  # Board configuration (uncomment the one you're using)
  # board: !include board_esp32.yaml
  # board: !include board_esp32-C3.yaml
  board: !include board_esp32-S3.yaml  # For ESP32-S3 Mini
```

---

## Step 2: Scan for 1-Wire Sensors

1. Flash the firmware to your ESP32
2. Open the web interface or serial console
3. Press "Test onewire bus" button
4. Note the addresses of all DS18B20 sensors
5. Label which sensor is at which zone return

---

## Step 3: Configure Sensor Addresses

In `config.yaml`, fill in the Dallas addresses:

```yaml
substitutions:
  # Supply and manifold return (existing)
  dallasaddress_input: "0xb300000084aa6128"
  dallasaddress_output: "0xeb0000008492ad28"
  
  # Per-zone return sensors (new)
  dallasaddress_zone_1_return: "0x1234567890abcdef"
  dallasaddress_zone_2_return: "0xfedcba0987654321"
  # ... continue for all zones
  # Use "0x0000000000000000" for unused zones
```

---

## Step 4: Configure Zone Areas

After flashing, use the web interface to set each zone's floor area:

1. Navigate to the web interface (http://your-device-ip)
2. Find "Zone X Area" inputs
3. Enter the floor area in m² for each zone
4. Set "Pipe Spacing" (typically 150mm)
5. Select "Pipe Type" (typically PEX 16x2)

The system will automatically calculate pipe lengths.

---

## Step 5: Initial Settings (Recommended Defaults)

| Parameter | Recommended Start Value |
|-----------|------------------------|
| Minimum Valve Opening | 25% |
| Maintenance Base | 45% |
| Demand Boost | 30% |
| Comfort Band | 0.5°C |
| Balancing Interval | 5 min |
| Hydraulic Balancing Enabled | OFF (initially) |

---

## Step 6: Test Without Balancing First

1. Keep "Hydraulic Balancing Enabled" **OFF**
2. Verify all zones heat correctly with the standard tanh control
3. Monitor temperatures and valve positions
4. Confirm all sensors are reporting correctly

---

## Step 7: Enable Hydraulic Balancing

1. Turn "Hydraulic Balancing Enabled" **ON**
2. Monitor the system for 1-2 days
3. Check the "Balancing Status" text sensor for system state

---

## Step 8: Fine-Tune if Needed

### If rooms oscillate around setpoint:
- Increase Comfort Band to 0.7-1.0°C
- Increase Balancing Interval to 10 min

### If one room is always cold:
- Verify zone area is correct
- Check Zone X Max Opening (try 100%)
- Verify return temp sensor is working

### If heat pump short-cycles:
- Increase Minimum Valve Opening to 30-35%
- Increase Maintenance Base to 50%

---

## MQTT Setup (Optional)

To enable multi-controller coordination and flow temperature requests:

1. Add MQTT credentials to `secrets.yaml`:

```yaml
mqtt_broker: "homeassistant.local"
mqtt_username: "mqtt_user"
mqtt_password: "mqtt_password"
```

2. Uncomment MQTT in `config.yaml`:

```yaml
packages:
  # ...
  mqtt_ecodan: !include mqtt_ecodan_integration.yaml
```

3. Set controller ID for each device:

```yaml
substitutions:
  controller_id: "1"  # "1" for first, "2" for second controller
  total_controllers: "2"
```

---

## Monitoring

### Key Sensors to Watch

| Sensor | Healthy Range |
|--------|--------------|
| Average Valve Position | 50-80% (optimal) |
| Active Zones Count | Matches your setup |
| Zones in Demand | Varies with heating load |
| Reference Zone | Should be longest pipe zone |

### Status Messages

| Status | Meaning |
|--------|---------|
| "Optimal range (50-80%)" | System running efficiently |
| "Low demand" | Consider reducing flow temp |
| "High demand" | May need higher flow temp |
| "Disabled" | Balancing not active |

---

## Troubleshooting

### No valve movement
- Check "Hydraulic Balancing Enabled" is ON
- Verify zone states in logs
- Check minimum movement threshold

### All zones at 0%
- All zones may be overheated
- Check comfort band setting
- Verify temperature sensors working

### Erratic behavior
- Increase balancing interval
- Check for sensor noise/failures
- Review comfort band setting

---

## Next Steps

- Read the full [HYDRAULIC_BALANCING.md](HYDRAULIC_BALANCING.md) documentation
- Monitor system for several days before major adjustments
- Consider adding per-zone pipe spacing if zones have different layouts
