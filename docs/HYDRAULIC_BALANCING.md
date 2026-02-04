# Hydraulic Balancing Algorithm Documentation

## Overview

The hydraulic balancing system automatically adjusts valve positions across all underfloor heating (UFH) zones to ensure:

1. **Equal flow distribution** - Zones with longer pipes get proportionally more valve opening
2. **Thermal comfort** - Rooms reach their setpoint temperatures efficiently
3. **Heat pump efficiency** - System-wide valve average stays in optimal range (50-80%)
4. **Stable operation** - Hysteresis and damping prevent oscillation

This algorithm combines **hydraulic balancing** (based on pipe length ratios) with **thermal demand** (based on room temperature vs setpoint) to create a unified control strategy.

---

## Core Concepts

### Zone States

Each zone is classified into one of three states based on the difference between setpoint and current temperature:

| State | Condition | Behavior |
|-------|-----------|----------|
| **OVERHEATED** | temp > setpoint + comfort_band | Valve closes to 0% |
| **SATISFIED** | temp within ±comfort_band of setpoint | Maintenance flow (preload) |
| **DEMAND** | temp < setpoint - comfort_band | Full heating with boost |

```
         DEMAND          |    SATISFIED    |   OVERHEATED
    (full heating)       | (maintenance)   |   (close)
                         |   "preload"     |
<--------------------[setpoint-0.5]---[setpoint]---[setpoint+0.5]---->
         needs heat      |  comfort band   |   too warm
```

### Comfort Band (Thermal Lag Compensation)

Underfloor heating has significant thermal mass, meaning:
- Heat takes time to transfer from pipes to room air
- Closing valves when room reaches setpoint causes **undershoot**
- Opening valves only when below setpoint causes **slow response**

The **comfort band** (default ±0.5°C) creates a "preload" zone where the floor maintains stored heat even when at setpoint. This prevents the common UFH problem of rooms oscillating around the setpoint.

### Reference Zone

The **reference zone** is the zone with the longest pipe length among all non-overheated zones. It serves as the baseline for hydraulic calculations:

- Reference zone runs at its `max_opening` (typically 90%) when in DEMAND
- All other zones are scaled relative to this reference
- When the reference zone becomes overheated, a new reference is selected dynamically

---

## Position Calculation Algorithm

### Step 1: Calculate Pipe Length Ratio

```
pipe_ratio[zone] = pipe_length[zone] / reference_pipe_length
```

Example: If reference zone has 200m of pipe and another zone has 100m:
- pipe_ratio = 100 / 200 = 0.5

### Step 2: Calculate Temperature Difference

```
temp_diff[zone] = setpoint[zone] - current_temp[zone]
```

- Positive = room is cold (needs heat)
- Negative = room is warm (may be overheated)

### Step 3: Classify Zone State

```python
if temp_diff < -comfort_band:
    state = OVERHEATED  # e.g., temp > setpoint + 0.5°C
elif temp_diff <= comfort_band:
    state = SATISFIED   # e.g., within ±0.5°C of setpoint
else:
    state = DEMAND      # e.g., temp < setpoint - 0.5°C
```

### Step 4: Calculate Final Position

**OVERHEATED zones:**
```
position = 0%
```

**SATISFIED zones (maintenance mode):**
```
position = maintenance_base × pipe_ratio
```
Example: maintenance_base=45%, pipe_ratio=0.75 → position=33.75%

**DEMAND zones (heating mode):**
```
hydraulic_base = max_opening × pipe_ratio
demand_factor = temp_diff / max_expected_diff  # normalized 0-1
position = hydraulic_base × (1 + demand_boost × demand_factor)
```
Example: max_opening=90%, pipe_ratio=0.75, demand_boost=30%, demand_factor=0.6
- hydraulic_base = 90% × 0.75 = 67.5%
- position = 67.5% × (1 + 0.3 × 0.6) = 67.5% × 1.18 = 79.6%

### Step 5: Apply Constraints

```python
if state != OVERHEATED:
    position = max(position, minimum_valve_opening)
position = clamp(position, 0%, 100%)
```

---

## Hysteresis for Overheated Zones

To prevent rapid on/off cycling when a zone is near the overheated threshold:

```
Zone closes when:  temp > setpoint + 0.5°C  (enters OVERHEATED)
Zone reopens when: temp <= setpoint          (exits OVERHEATED)
```

This creates a **0.5°C hysteresis gap**:
- A zone at 21.6°C with setpoint 21°C will close (21.6 > 21.5)
- It won't reopen until temp drops to 21.0°C or below

---

## Configurable Parameters

All parameters are adjustable via the ESPHome web interface and stored in flash.

### Minimum Valve Opening

| Setting | `minimum_valve_opening` |
|---------|------------------------|
| Default | 25% |
| Range | 10-50% |
| Purpose | Ensures flow in all pipes to prevent stagnation and heat pump short-cycling |

**When to adjust:**
- Increase if heat pump short-cycles frequently
- Decrease if system runs too hot when all zones satisfied

### Maintenance Base

| Setting | `maintenance_base` |
|---------|-------------------|
| Default | 45% |
| Range | 30-60% |
| Purpose | Reference zone opening when at setpoint (SATISFIED state) |

**Key relationship:**
```
maintenance_base × (shortest_pipe / longest_pipe) ≥ minimum_opening
```

**When to adjust:**
- Increase for better hydraulic ratio preservation
- Decrease to reduce flow when all rooms satisfied

### Demand Boost

| Setting | `demand_boost` |
|---------|---------------|
| Default | 30% |
| Range | 0-50% |
| Purpose | Extra opening for cold rooms above hydraulic base |

**When to adjust:**
- Increase for faster heating response
- Decrease for more gradual, stable heating

### Comfort Band

| Setting | `comfort_band` |
|---------|---------------|
| Default | 0.5°C |
| Range | 0.3-1.0°C |
| Purpose | Defines the "at setpoint" tolerance zone |

**When to adjust:**
- Increase if rooms oscillate around setpoint
- Decrease for tighter temperature control (may cause more valve movement)

### Balancing Interval

| Setting | `balancing_interval` |
|---------|---------------------|
| Default | 5 minutes |
| Range | 1-30 minutes |
| Purpose | How often the balancing algorithm runs |

**When to adjust:**
- Decrease for faster response (more valve movements)
- Increase for more stable operation

### Balancing Step

| Setting | `balancing_step` |
|---------|-----------------|
| Default | 3% |
| Range | 1-10% |
| Purpose | Maximum position change per balancing cycle (for fine-tuning) |

---

## Zone Physical Parameters

### Zone Area

| Setting | `zone_X_area` |
|---------|--------------|
| Default | 15 m² |
| Range | 1-100 m² |
| Purpose | Floor area covered by the zone |

Used to calculate pipe length:
```
pipe_length = zone_area / (pipe_spacing / 1000)
```

### Zone Max Opening

| Setting | `zone_X_max_opening` |
|---------|---------------------|
| Default | 90% |
| Range | 50-100% |
| Purpose | Normal maximum valve position for balanced operation |

**Note:** This is a soft limit. During high demand, valves can temporarily exceed this to 100%.

### Pipe Spacing

| Setting | `pipe_spacing` |
|---------|---------------|
| Default | 150 mm |
| Options | 100, 150, 200, 300 mm |
| Purpose | Center-to-center distance between pipe runs |

### Pipe Type

| Setting | `pipe_type` |
|---------|------------|
| Default | PEX 16x2 |
| Purpose | Determines water volume per meter |

| Pipe Type | Inner Diameter | Volume (L/m) |
|-----------|---------------|--------------|
| PEX 12x2 | 8mm | 0.050 |
| PEX 14x2 | 10mm | 0.079 |
| PEX 16x2 | 12mm | 0.113 |
| PEX 17x2 | 13mm | 0.133 |
| PEX 18x2 | 14mm | 0.154 |
| PEX 20x2 | 16mm | 0.201 |

---

## Worked Examples

### Example 1: High Demand (Cold Rooms)

Settings: maintenance_base=45%, demand_boost=30%, min_opening=25%, comfort_band=0.5°C

| Zone | Pipe Length | Setpoint | Current | Diff | State | Calculation | Final |
|------|-------------|----------|---------|------|-------|-------------|-------|
| Bedroom | 200m (ref) | 21°C | 17°C | +4°C | DEMAND | 90% × 1.0 × 1.24 | **100%** |
| Living | 150m | 23°C | 20°C | +3°C | DEMAND | 90% × 0.75 × 1.18 | **80%** |
| Bathroom | 80m | 22°C | 21.3°C | +0.7°C | DEMAND | 90% × 0.4 × 1.04 | **37%** |

### Example 2: All Rooms at Setpoint (Maintenance Mode)

| Zone | Pipe Length | Setpoint | Current | Pipe Ratio | Calculation | Final |
|------|-------------|----------|---------|------------|-------------|-------|
| Bedroom | 200m (ref) | 21°C | 21°C | 1.0 | 45% × 1.0 | **45%** |
| Living | 150m | 23°C | 23°C | 0.75 | 45% × 0.75 | **34%** |
| Bathroom | 80m | 22°C | 22°C | 0.4 | 45% × 0.4 = 18% | **25%** (min) |

### Example 3: Mixed States

| Zone | Pipe Length | Setpoint | Current | State | Calculation | Final |
|------|-------------|----------|---------|-------|-------------|-------|
| Bedroom | 200m (ref) | 21°C | 21°C | SATISFIED | 45% × 1.0 | **45%** |
| Living | 150m | 23°C | 20°C | DEMAND | 67.5% × 1.18 | **80%** |
| Bathroom | 80m | 22°C | 22.8°C | OVERHEATED | Close valve | **0%** |

### Example 4: Overheated Reference Zone (Sunlight)

When the longest zone overheats, a new reference is selected:

| Zone | Pipe Length | Setpoint | Current | State | Reference | Final |
|------|-------------|----------|---------|-------|-----------|-------|
| Bedroom | 200m | 21°C | 21.8°C | OVERHEATED | excluded | **0%** |
| Living | 150m (new ref) | 23°C | 22°C | DEMAND | reference | **90%** |
| Bathroom | 80m | 22°C | 21.8°C | SATISFIED | scaled | **35%** |

---

## Multi-Controller Coordination

When multiple floor heating controllers operate in the same system, they coordinate via MQTT:

### Published Topics

| Topic | Content |
|-------|---------|
| `floor_heating/controller_{id}/online` | Boolean availability (LWT) |
| `floor_heating/controller_{id}/heating_demand` | Any zone actively heating |
| `floor_heating/controller_{id}/avg_valve_position` | 0-100% average |
| `floor_heating/controller_{id}/zone_count` | Number of active zones |
| `floor_heating/controller_{id}/flow_temp_request` | -2 to +2 °C adjustment |

### System-Wide Average

Controllers calculate a weighted system average:
```
system_avg = (ctrl1_avg × ctrl1_zones + ctrl2_avg × ctrl2_zones) / total_zones
```

### Flow Temperature Adjustment

Based on system average, controllers request flow temperature changes:

| System Average | Request | Reasoning |
|---------------|---------|-----------|
| < 30% | -2°C | All zones nearly satisfied |
| 30-50% | -1°C | Low demand |
| 50-80% | 0°C | **Optimal range** |
| 80-90% | +1°C | High demand |
| > 90% | +2°C | Valves maxed out |

### Minimum Opening Coordination

If one controller has high demand (>50% avg), other controllers can allow zones below minimum:
- This ensures total system flow while allowing fine control
- Requires MQTT connectivity; falls back to local minimum if disconnected

---

## Return Temperature Feedback

Per-zone return temperature sensors enable fine-tuning:

1. **Expected behavior:** Equal return temperatures = equal flow rates
2. **Feedback adjustment:**
   - Return temp higher than expected → reduce opening slightly
   - Return temp lower than expected → increase opening slightly
3. **Corrects for:** Actual pipe lengths, fittings, installation variations

---

## Troubleshooting

### Rooms Oscillate Around Setpoint

**Symptoms:** Temperature swings ±1-2°C around target

**Solutions:**
- Increase `comfort_band` to 0.7-1.0°C
- Increase `balancing_interval` to 10-15 min
- Decrease `demand_boost` to 15-20%

### One Room Always Cold

**Symptoms:** Room never reaches setpoint while others are warm

**Solutions:**
- Check if zone area is set correctly
- Verify pipe spacing matches actual installation
- Increase `zone_X_max_opening` to 100%
- Check for air in the pipes or stuck valve

### Heat Pump Short-Cycling

**Symptoms:** Heat pump starts/stops frequently

**Solutions:**
- Increase `minimum_valve_opening` to 30-40%
- Increase `maintenance_base` to 50-60%
- Check if all zones are overheated (may need lower flow temp)

### All Zones at Minimum, Still Too Warm

**Symptoms:** System average <30% but rooms above setpoint

**Solutions:**
- Enable flow temperature requests to Ecodan
- Reduce heat curve on heat pump
- Decrease `minimum_valve_opening` (with caution)

### Hydraulic Imbalance Despite Algorithm

**Symptoms:** Return temperatures vary significantly between zones

**Solutions:**
- Verify zone areas are accurate
- Check pipe spacing is consistent
- Let return temperature feedback correct over time
- Consider manual pre-balancing if severe

---

## Algorithm Flowchart

```
┌─────────────────────────────────────────┐
│         Start Balancing Check           │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│      For each zone, calculate:          │
│      temp_diff = setpoint - current     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Classify zone state:            │
│    OVERHEATED / SATISFIED / DEMAND      │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│    Find reference zone (longest pipe    │
│       among non-overheated zones)       │
└─────────────────┬───────────────────────┘
                  │
          ┌───────┴───────┐
          │               │
          ▼               ▼
┌─────────────────┐ ┌─────────────────────┐
│  No reference   │ │ Reference found     │
│  (all overheat) │ │                     │
└────────┬────────┘ └──────────┬──────────┘
         │                     │
         ▼                     ▼
┌─────────────────┐ ┌─────────────────────┐
│ Close all valves│ │ Calculate positions │
│ Request lower   │ │ based on state and  │
│ flow temp       │ │ pipe ratios         │
└─────────────────┘ └──────────┬──────────┘
                               │
                               ▼
                  ┌─────────────────────────┐
                  │ Apply minimum opening   │
                  │ constraint (except      │
                  │ OVERHEATED zones)       │
                  └────────────┬────────────┘
                               │
                               ▼
                  ┌─────────────────────────┐
                  │ Move valves to target   │
                  │ positions               │
                  └─────────────────────────┘
```

---

## File Reference

| File | Purpose |
|------|---------|
| `hydraulic_balancing.yaml` | Core algorithm, globals, configurable parameters |
| `zone_parameters.yaml` | Zone physical properties (area, max opening, pipe type) |
| `sensor_return_temperature.yaml` | Per-zone return temperature sensors |
| `heating_channel_tanh.yaml` | Zone control logic with balancing integration |
| `standalone_control.yaml` | Standalone operation, status monitoring |
| `mqtt_ecodan_integration.yaml` | Multi-controller coordination, flow temp requests |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-01 | Initial hydraulic balancing implementation |
