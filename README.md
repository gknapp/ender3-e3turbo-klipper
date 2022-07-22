# Ender3 BTT E3 Turbo Klipper config

This repo serves as a backup for my klipper configs.

The [E3 Turbo pin diagram][1] is a useful document to reference.

### 3D Printer specs

- Ender 3
- BLTouch (using z-probe pins NOT z-stop)
- Dual Z axis
- BTT E3 Turbo motherboard
- Fysetc Sherpa Mini extruder

Possibly useful macros

## BL Touch

```yaml
[bltouch]
sensor_pin: ^P1.22
control_pin: P1.23
stow_on_each_sample: False
probe_with_touch_mode: False
pin_up_touch_mode_reports_triggered: False
set_output_mode: 5V
x_offset: <USE_YOURS>
y_offset: <USE_YOURS>
speed: 5.0
samples: 2
sample_retract_dist: 4
samples_result: average
samples_tolerance: 0.100
samples_tolerance_retries: 3
```

## Dual Z axis

You'll need `stepper_z1` in addition to `stepper_z` and a `safe_z_home` section.

> **NOTE**: My second stepper (RHS) is inverted (!) using the `dir_pin` field.

```yaml
[stepper_z1]
step_pin: P2.11
dir_pin: !P2.12
enable_pin: !P0.21
microsteps: 16
rotation_distance: 2.018
endstop_pin: probe:z_virtual_endstop

[tmc2209 stepper_z1]
uart_pin: P0.22
diag_pin: P1.25
run_current: 0.580
stealthchop_threshold: 999999
```

```yaml
[safe_z_home]
# Center of print bed (115, 115) + probe offsets
home_xy_position: 158, 123
speed: 90
z_hop: 7
z_hop_speed: 5
```

If you want auto-align Z that you might have had in Marlin (aka G34), it's called `z_tilt` in klipper. These `z_positions` are for the lead screws and stand outside the build surface. It's important to get the X coords spot on - as your probe only moves along X. Change these values if your printer isn't an Ender 3. `points` are where the probe will measure on the build surface.

```yaml
[z_tilt]
z_positions:
 -25,123
 261,123
points:
 45,123
 250,123
speed: 180
retries: 6
retry_tolerance: 0.02
```

My bed mesh stands in from the front and rear edges of the build surface, providing room for clips.

```yaml
[bed_mesh]
probe_count: 5
speed: 80
mesh_min: 15, 20
mesh_max: 207, 200
```

If you want to tram your bed, you'll need to specify where your build surface screws are positioned.

```yaml
[bed_screws]
screw1_name: Front left
screw1: 31,32
screw2_name: Rear right
screw2: 201,202
screw3_name: Front right
screw3: 201,32
screw4_name: Rear left
screw4: 31,202
```

# Macros

Occasionally my BLTouch doesn't deploy and flashes. To clear this I pull it to deploy it then run this macro to reset it.

#### BLTouch reset
```yaml
[gcode_macro bltouch_reset]
description: Reset BLTouch
gcode:
  BLTOUCH_DEBUG COMMAND=reset
  BLTOUCH_DEBUG COMMAND=pin_up
  BLTOUCH_DEBUG COMMAND=pin_down
  BLTOUCH_DEBUG COMMAND=pin_up
```

#### Bed tramming
```yaml
[gcode_macro bed_tramming]
gcode:
  G28
  BED_SCREWS_ADJUST

[gcode_macro tram_ok]
gcode:
  ACCEPT

[gcode_macro tram_adjust]
gcode:
  ADJUSTED

[gcode_macro tram_done]
gcode:
  ABORT
```

#### Load / unload filament

These macros are for a direct drive extruder that is close to the hot-end.  
You will need to specify the following in your `[extruder]` section for these to work:

```yaml
[extruder]
max_extrude_only_distance: 150
```

```yaml
[gcode_macro filament_load]
description: Load 140mm of filament
gcode:
  M104 T0 S215 ; heat hot-end to 210C
  G28 ; auto home
  G91 ; relative positioning
  G1 X-40 F6000 ; centre (accommodate probe)
  G1 Z+20 F1000 ; raise hot-end
  M109 T0 S215 ; wait for hot-end to reach 215C
  M83 ; relative extruder movement
  G1 E140 F300 ; extrude 140mm filament
  G4 P2000 ; 2s purging time
  G1 E30 F10 ; slowly pull in 30mm
  G90 ; absolute positioning
  {action_respond_info("Purging filament 10s")}
  G4 P10000 ; 10s purging time
  M104 T0 S0; hot-end cooldown
  {action_respond_info("Load complete")}

[gcode_macro filament_unload]
description: Unload 80mm of filament
gcode:
  M104 T0 S210 ; heat hotend to 210C
  G28 ; home toolhead
  G91 ; relative positioning
  G1 X-40 F6000 ; centre (accommodates my probe offset)
  M109 T0 S210 ; wait until temp reached
  M83 ; relative extruder movement
  {action_respond_info("Retracting filament")}
  G1 E7 F300 ; push into meltzone
  G4 P1000 ; wait 1 second
  G1 E-80 F400 ; retract 80mm
  G90 ; absolute positioning
  M104 T0 S0 ; turn off hotend
```

[1]: https://github.com/bigtreetech/BIGTREETECH-SKR-E3-Turbo/blob/master/Hardware/BTT%20SKR%20E3%20Turbo-Pin.pdf
