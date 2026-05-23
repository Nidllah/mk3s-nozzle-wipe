# 🧹 MK3S+ Klipper Nozzle Wipe

**Modified `PRINT_START` and `PRINT_END` macros for the Prusa MK3S+ running Klipper — adds automatic nozzle cleaning and smart cooldown.**

---

## ✨ What does this do?

Gives your Prusa MK3S+ a more modern startup and shutdown routine:

- 🔥 Heats nozzle to 250°C and runs a full nozzle wipe before every print
- ❄️ Starts cooling mid-wipe to save time — parts fan kicks in immediately
- 🌡️ Parts fan turns off automatically as soon as cal temp is reached
- 🧹 Cleaner nozzle = better first layer adhesion
- 💨 `PRINT_END` uses the parts fan to cool the nozzle down to 170°C before shutting off

---

## 🛒 Requirements

**You need to print and install the physical nozzle wiper mod before using this.**

👉 [Nozzle Wiper for MK3S / MK3.5 / MK3.9 / MK4S by Apheli0n_265140 on Printables](https://www.printables.com/model/1394873-nozzle-wiper-mk3s-mk35-mk39-mk4s)
He has instructions on how to add the instructions in your start gcode incase you run it stock and not klipper.

---

## 📦 Based on

Klipper MK3S+ config and settings by **charminULTRA**:
👉 https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade

These are modified versions of the `PRINT_START` and `PRINT_END` macros from that config. Drop in replacement.

---

## 📋 Changes from original

**PRINT_START:**
| What | Change |
|---|---|
| Nozzle preheat | Heated to 250°C instead of cal temp |
| Nozzle cleaning | Full wipe routine added — sweeps X252–254 at two Z levels |
| Cooldown | Parts fan on full after wipe, turns off when cal temp is reached |
| Cooldown timing | Cooling starts mid-wipe to save time |

This is what my PRINT_START looks like (drop in replacement if you are using **charminULTRAS** configuration
    
    [gcode_macro PRINT_START]
    gcode:
    {% set BED_TEMP = params.BED_TEMP|default(60)|float %}
    {% set BED_CAL_TEMP = params.BED_CAL_TEMP|default(60)|float %}
    {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(215)|float %}
    {% set EXTRUDER_CAL_TEMP = params.EXTRUDER_CAL_TEMP|default(170)|int %} ; 170*C gets common filaments like PETG, PLA, and ASA/ABS nice and soft without oozing.
    {% set PRINTER_MODEL = params.PRINTER_MODEL|default("MK3S+")|string %}
    {% set FILAMENT_TYPE = params.FILAMENT_TYPE|default("PLA")|string %}

    ; Make sure the MK3S+ uses the bed temp as the calibration temp since SuperPINDA has internal compensation
    {% if PRINTER_MODEL == "MK3S+" %}
        {% set BED_CAL_TEMP = BED_TEMP %}
    {% endif %}

    ; --- PREHEAT ---
    M117 Preheating...
    M140 S{BED_CAL_TEMP}             ; Start bed heating, don't wait
    M104 S250                        ; [ADDED] Heat nozzle to 250C for cleaning instead of cal temp
    SET_GCODE_OFFSET Z=0.0           ; Reset Z offset
    BED_MESH_CLEAR                   ; Clear bed mesh
    G28                              ; Home all axes
    G90                              ; Absolute positioning

    ; --- WAIT FOR NOZZLE CLEANING TEMP --- [ADDED]
    ; Wait for nozzle to reach 250C before cleaning
    M117 Heating for cleaning...
    TEMPERATURE_WAIT SENSOR=extruder MINIMUM=248 MAXIMUM=255

    ; --- NOZZLE CLEANING --- [ADDED]
    ; Requires a physical nozzle cleaner/brush mounted at X252-254 Y-4
    ; Adjust X and Y coordinates to match your nozzle cleaner position
    M117 Cleaning nozzle...
    G1 Z15 F4000
    G1 X252 Y-4 F8000                ; Move to cleaning position
    G1 Z5                            ; Lower to bottom level
    G1 Y40 F6000                     ; Forward - bottom
    G1 Z7
    G1 X252 Y-4 F6000                ; Back - top
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X253 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X253 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X254 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    ; [ADDED] Start cooling mid-wipe to save time
    M104 S{EXTRUDER_CAL_TEMP}        ; Start cooling nozzle immediately
    M106 S255                        ; Parts fan on full to help cool
    G1 E-3 F1800                     ; Retract 3mm to stop oozing
    G1 X254 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X253 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X252 Y-4 F6000
    G1 Z5
    G1 Y40 F6000
    G1 Z7
    G1 X253 Y-4 F6000

    ; --- LEAVE CLEANING POSITION --- [ADDED]
    G1 Z15 F4000                     ; Lift up over cleaning plate first
    G1 X125 Y100 F8000               ; Move to center while cooling
    G0 Z50 F4000                     ; Raise up (keeps PINDA cool)

    ; --- WAIT FOR BED AND NOZZLE TO REACH CAL TEMPS ---
    ; [CHANGED] Wait for extruder first so fan turns off as soon as possible
    M117 Waiting for temps...
    TEMPERATURE_WAIT SENSOR=extruder MINIMUM={EXTRUDER_CAL_TEMP-1} MAXIMUM={EXTRUDER_CAL_TEMP+5}
    M106 S0                          ; [ADDED] Extruder at cal temp - turn off fan
    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={BED_CAL_TEMP-1} MAXIMUM={BED_CAL_TEMP+3}

    ; --- PINDA HEATSOAK (MK3/MK3S only) ---
    {% if PRINTER_MODEL == 'MK3S' or PRINTER_MODEL == "MK3" %}
        G0 X20 Y60 Z0.15             ; Move to PINDA preheat position
        {% for timer in range(45,0,-1) %}
            M117 PINDA Heatsoak...{timer|int}
            G4 P1000
        {% endfor %}
    {% endif %}

    ; --- BED MESH ---
    M117 Probing bed...
    BED_MESH_CALIBRATE ADAPTIVE=1    ; Adaptive bed mesh

    ; --- HEAT TO FINAL PRINT TEMPS ---
    M117 Heating...
    M140 S{BED_TEMP}                 ; Heat bed to final temp, don't wait
    M104 S{EXTRUDER_TEMP}            ; Heat nozzle to final temp, don't wait
    G0 X0 Y-4 Z15 F4000             ; Move to purge start position
    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={BED_TEMP-1} MAXIMUM={BED_TEMP+3}
    TEMPERATURE_WAIT SENSOR=extruder MINIMUM={EXTRUDER_TEMP-1} MAXIMUM={EXTRUDER_TEMP+5}

    ; --- PURGE AND GO ---
    M117
    PRUSA_LINE_PURGE                 ; Purge line, then ready to print
    #LINE_PURGE                      ; Alternative if using KAMP
    #SKEW_PROFILE LOAD=CaliFlower

---




**PRINT_END:**
| What | Change |
|---|---|
| Cooldown | Parts fan cools nozzle to 170°C before shutting down |
The printer already retracts 3-4mm when print end, this helps cool the nozzle faster and reduce oozing.

This is how my PRINT_END look like:

    # Print End Added cooldown
    [gcode_macro PRINT_END]
    gcode:
    #SET_SKEW CLEAR=1
    M83 ; extruder relative mode
    G1 E-4 ; retract filament a bit for minimizing ooze
    _TOOLHEAD_PARK_PAUSE_CANCEL    ; Park
    G90 ; absolute positioning
    G0 Z{[printer.toolhead.position.z, 100]|max} F300 ; park at least 100mm off the bed
    M117 Cooling down...
    M106 S255                        ; Parts fan on full to help cool nozzle
    TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=170  ; Wait until nozzle is at 170C
    M84 X Y E    ; Disable steppers
    TURN_OFF_HEATERS    ; Disable heaters
    M106 S0     ; Disable fans
    BED_MESH_CLEAR    ; Clear bed mesh


---

## 🔧 Installation

1. Open your Klipper config (in Mainsail or Fluidd)
2. Find your existing `PRINT_START` and `PRINT_END` macros
3. Replace them with the ones from this repo
4. **Adjust the nozzle cleaner coordinates** to match your physical install:
```ini
G1 X252 Y-4 F8000   ; adjust X and Y to center on your wiper
```
5. Save and restart Klipper

---

## 📝 Notes

- Tested on Prusa MK3S+ running Klipper via Raspberry Pi
- `PRUSA_LINE_PURGE` is used by default — swap for `LINE_PURGE` if you use KAMP
- The 3mm retract after cleaning is compensated automatically by the purge line
