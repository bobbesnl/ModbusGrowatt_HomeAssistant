# Growatt SPH Modbus Control
**Verified Behaviour for Battery First / Grid First / AC Charge via Local Modbus RTU**  
Tested on Growatt SPH inverter using Home Assistant and pymodbus debug logs.

This document provides a confirmed and practical reference for controlling Growatt SPH hybrid inverters locally without cloud dependency.

---

## 1. Register Types

Growatt SPH inverters expose two useful Modbus register types:

### **Input Registers (Function Code 04)**  
Read-only. Provide real-time inverter data:

- Battery voltage, SOC  
- PV power  
- Grid flow  
- Load power  
- Temperatures  

These cannot be written.

---

### **Holding Registers (Function Code 03/06/16)**  
Writable configuration registers:

- Battery First  
- Grid First  
- AC Charge  
- Charge/discharge limits  
- Time windows  
- SoC limits  
- Inverter enable  

Not all holding registers are writable.  
All registers referenced here are confirmed to work without password.

---

## 2. Time Window Format

Growatt encodes timestamps in a single 16-bit register:

- High 8 bits = hour (0–23)  
- Low 8 bits = minute (0–59)

Example:  
`09:30` → `(9 << 8) + 30` → `0x091E`  

A window of `00:00–00:00` disables the feature.

---

## 3. Battery First

Battery First uses three registers:

| Register | Function |
|---------|----------|
| **1100** | Start time |
| **1101** | Stop time |
| **1102** | Enable (1=on, 0=off) |

Behaviour:

- Takes effect immediately.
- Only operates inside the configured time window.
- No password is required.

---

## 4. Grid First

Grid First requires additional configuration:

| Register | Function |
|---------|----------|
| **1080** | Start time |
| **1081** | Stop time |
| **1082** | Enable (1=on, 0=off) |
| **1070** | Discharge power limit (required) |
| **1071** | Stop-discharge SoC (required) |

Behaviour:

- Will not activate unless both 1070 and 1071 are set.
- Priority mode switches instantly when 1082 is set to 1.

---

## 5. AC Charge

| Register | Function |
|---------|----------|
| **1090** | Start time |
| **1091** | Stop time |
| **1092** | Enable (1=on, 0=off) |

Behaviour:

- Works only during an active time window.
- Confirmed working even when Battery First is active.
- No detectable lock state on SPH.

---

## 6. Priority Mode Logic

Growatt uses three modes:

- Battery First  
- Grid First  
- Load First  

Rules:

- Battery First and Grid First cannot be active at the same time.
- If both are disabled, system defaults to Load First.
- Modbus writes override cloud/app settings immediately.

---

## 7. Home Assistant YAML Examples

### Set a time window

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1100
  values: "{{ [(9 << 8) + 30] }}"
```

### Enable Battery First

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1102
  values: [1]
```

### Full Grid First activation

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1080
  values: "{{ [(22 << 8) + 0] }}"
```

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1081
  values: "{{ [(6 << 8) + 0] }}"
```

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1070
  values: [3000]
```

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1071
  values: [20]
```

```yaml
service: modbus.write_registers
data:
  hub: growatt
  slave: 1
  address: 1082
  values: [1]
```

---

## 8. Example Switch in Home Assistant (Battery First)

```yaml
- platform: modbus
  scan_interval: 30
  switches:
    - name: "Battery First"
      hub: growatt
      slave: 1
      address: 1102
      write_type: holding
      command_on: 1
      command_off: 0
      verify:
        input_type: holding
        address: 1102
        state_on: 1
        state_off: 0
```

---

## 9. Troubleshooting

- Battery First enabled but no charging → time window not set.  
- Grid First not working → registers 1070 and 1071 missing.  
- AC charge inactive → 1090/1091 still at 00:00.  
- Modbus write ignored → wrong register type or model-specific behaviour.

---

## 10. Conclusions

- SPH supports full energy-priority control via Modbus.  
- No unlock/password required for Battery First, Grid First, or AC Charge.  
- Time windows are essential.  
- Inverter responds instantly to local Modbus control.

