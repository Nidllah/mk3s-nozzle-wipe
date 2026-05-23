# 🧹 MK3S+ Klipper Nozzle Wipe

**Modified `PRINT_START` and `PRINT_END` macros for the Prusa MK3S+ running Klipper — adds automatic nozzle cleaning, adaptive bed mesh and smart cooldown.**

---

## ✨ What does this do?

Gives your Prusa MK3S+ a more modern startup and shutdown routine:

- 🔥 Heats nozzle to 250°C and runs a full nozzle wipe before every print
- ❄️ Starts cooling mid-wipe to save time — parts fan kicks in immediately
- 🌡️ Parts fan turns off automatically as soon as cal temp is reached
- 🛏️ Adaptive bed mesh calibration on every print
- 🧹 Cleaner nozzle = better first layer adhesion
- 💨 `PRINT_END` uses the parts fan to cool the nozzle down to 170°C before shutting off

---

## 🛒 Requirements

**You need to print and install the physical nozzle wiper mod before using this.**

👉 [Nozzle Wiper for MK3S / MK3.5 / MK3.9 / MK4S by Apheli0n_265140 on Printables](https://www.printables.com/model/1394873-nozzle-wiper-mk3s-mk35-mk39-mk4s)

---

## 📦 Based on

Klipper MK3S+ config and settings by **charminULTRA**:
👉 https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade

These are modified versions of the `PRINT_START` and `PRINT_END` macros from that config. Replace the originals with the files here.

---

## 📋 Changes from original

**PRINT_START:**
| What | Change |
|---|---|
| Nozzle preheat | Heated to 250°C instead of cal temp |
| Nozzle cleaning | Full wipe routine added — sweeps X252–254 at two Z levels |
| Cooldown | Parts fan on full after wipe, turns off when cal temp is reached |
| Cooldown timing | Cooling starts mid-wipe to save time |

**PRINT_END:**
| What | Change |
|---|---|
| Cooldown | Parts fan cools nozzle to 170°C before shutting down |

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

## ⚙️ Slicer setup

In your slicer's **Start G-code**, replace everything with:

**PrusaSlicer / OrcaSlicer / SuperSlicer:**
```gcode
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature]
```

**Cura:**
```gcode
PRINT_START BED_TEMP={material_bed_temperature_layer_0} EXTRUDER_TEMP={material_print_temperature_layer_0}
```

---

## 📝 Notes

- Tested on Prusa MK3S+ running Klipper via Raspberry Pi
- `PRUSA_LINE_PURGE` is used by default — swap for `LINE_PURGE` if you use KAMP
- The 3mm retract after cleaning is compensated automatically by the purge line
