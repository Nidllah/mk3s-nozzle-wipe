# mk3s-nozzle-wipe
Klipper PRINT_START macro for Prusa MK3S+ with automatic nozzle cleaning, adaptive bed mesh and smart cooldown


I got the idea from @Apheli0n_265140 on printables, from his nozzle wiper mod, check it out here: https://www.printables.com/model/1394873-nozzle-wiper-mk3s-mk35-mk39-mk4s


Using that modified the PRINT_START and also PRINT_END macros in klipper, to get a more "modern feel".

This gets you faster cooling down, and a nozzle clean before you start printing.


My config files and settings to make klipper work are taken from: https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade/tree/main


Changes made se below: (REQUIRES PHYSICAL NOZZLE WIPER MOD DOWNLOAD FROM HERE: https://www.printables.com/model/1394873-nozzle-wiper-mk3s-mk35-mk39-mk4s
Added  PRINT_START

# =================================================================================
# PRINT_START macro for Prusa MK3S+ with Klipper
# Based on the Klipper MK3S+ config by charminULTRA:
# https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade
#
# Changes from original:
# - Nozzle heated to 250C before print for nozzle cleaning
# - Nozzle cleaning routine added (requires physical nozzle cleaner)
# - Parts fan used to speed up cooldown after cleaning
# - Cooldown starts mid-wipe to save time
# - Extruder fan turned off as soon as cal temp is reached
# =================================================================================

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

    
    #-----------------------------------------------------------------------------------------------------------------------------







    Changed Print_END:
    Added faster cooldown to minimize oozing to PRINT_END:
    [gcode_macro PRINT_END]
    gcode:
    #SET_SKEW CLEAR=1
    M83                                          ; Extruder relative mode
    G1 E-4                                       ; Retract filament to minimize ooze
    _TOOLHEAD_PARK_PAUSE_CANCEL                  ; Park
    G90                                          ; Absolute positioning
    G0 Z{[printer.toolhead.position.z, 100]|max} F300 ; Park at least 100mm off bed

    ; --- COOLDOWN WITH PARTS FAN ---
    M117 Cooling down...
    M106 S255                                    ; Parts fan on full to help cool nozzle
    TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=170 ; Wait until nozzle is below 170C
    M106 S0                                      ; Fan off
    ; --- END COOLDOWN ---

    M84 X Y E                                    ; Disable steppers
    TURN_OFF_HEATERS                             ; Disable heaters
    BED_MESH_CLEAR                               ; Clear bed mesh
    M117
