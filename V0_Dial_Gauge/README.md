# What is this for?

Before getting into the details on how this works, I think it's helpful to explain why it was created in the first place.  The critereria I tried to satisfy in creating this were:

1. Repeatable results - not relying on keeping resistance constant with a piece of paper
2. Able to center exactly over each screw without difficulty
3. No (or minor) modifications to printer
4. Could be built cheaply with materials on hand

The last two are important.  There are going to be benefits in my approach to levelling the bed, but this may not be the most accurate solution or using the best tools.

# Disclaimer

Since the mount is not rigidly fixed to the frame or toolhead and we are relying on a friction fit, we can't use the dial gauge as it's intended to be used.  Ideally we would roughly level the bed, adjust the dial gauge probe until it's touching the bed, zero the dial and then adjust all three screws until they match that zero.

The only method I found to get consistent and accurate results within ~0.02mm was to find the exact spot the the probe touches the bed.  The point where you could barely see the dial move, or where it would appear to be zero but moving the bed away 0.1mm would cause the needle to jump before settling back at 0.  If the probe was already touching the bed, even just removing and re-attaching the mount would show up to a 0.05mm difference in readings.

This means that we can get fairly accurate results with this project, but we have to follow a very specific process to compensate for these shortcomings.

Finally, if anything in this document is over-explained and seems patronizing, it's because I've already made some stupid mistake and want to save anyone else from doing it too.

# BOM

* Dial_Gauge_Mount.stl
* (optional) Dial_Test_Setup_Guide.stl
* 4x - 6mm x 3mm magnets
* Dial Test Indicator - https://www.aliexpress.com/item/2251832796520415.html
* Metric Feeler Gauge Set
* Superglue

The main printed part can be printed without supports and is already oriented correctly.  The optional print is to help adjust the angle of the Indicator's probe and the position of the dovetail clamp.  The indicator itself doesn't need to be the exact one linked, a search for `Lever` or `Dovetail` should bring up the same yellow Chinese dial indicator.  Feeler gauge will be used to measure distances after the bed has been levelled. Superglue is to keep the magnets in place, there's extra clearance on the magnets to make sure they can be placed easily which also means they can be pulled out.

## Assembly

The only assembly for the printed part is installing the magnets with a drop of superglue.  Make sure the polarity is correct.  If you set them flat next to each other on a screwdriver/calipers none of the magnets should try to push away.

## Adjusting the Dial Indicator

THe mount was designed to have the tip of the probe lined up with the back edge of the Indicator, and to rest 1mmm lower than the nozzle to ensure we don't crash into the bed while levelling.  

The optional setup guide is designed to set the Indicator in with the back against the bottom, and the top hanging over the edge.  Adjust the probe so that it's almost touching the the corner.  Then attach the dovetail to the exposed back as far down as it will go without letting the probe touch the corner.

Before using the indicator for anything, make sure the dial is also rotated to 0 where it rests naturally.

Don't stress too much about accuracy here, the process laid out later on will adjust for any inconsistencies here.  As long as the angle is close and it's lower than the nozzle, you'll be fine.

# Macro

The last piece of setup is to add macros to the klipper configuration that will simplify the levelling process.  Here are the macros that will be added:

* `MANUAL_BED_LEVEL` - preps some of the data for the next macro, sets `position_endstop` to 0.0, then moves to the first bed screw.
* `NEXT_Z` - Cycles through each of the bed level screws.  Uses the configured screw positions with offsets that will let you slide the mount right up to the toolhead and have it sit above the screw.  Sets the Z height to the `indicator_offset` value
* `BOOP` - Moves the Z axis +0.1mm, pauses for 1 second, then moves -0.1mm. Used to determine the exact point the probe touches the bed.  Mostly here since it will add a button to the dashboard for easy access.
* `END_BED_LEVEL` - Resets the data for `NEXT_Z`. This isn't not strictly necessary but will stop that macro from running.
* `UPDATE_Z_OFFSET` - Combines the calculated `hotend_offset` and `squish_offset` and updates `position_endstop` to the new value.

At the top of your `printer.cfg` add this line
```py
[include dial_indicator.cfg]
```

Then create a new file named `dial_indicator.cfg` and paste this code
```py
[save_variables]
filename: ~/klipper_config/variables.cfg

[gcode_macro UPDATE_Z_OFFSET]
gcode:
  {% set svv = printer.save_variables.variables %}
  SET_GCODE_OFFSET Z={((svv.hotend_offset + svv.squish_offset) * -1) + printer.configfile.config["stepper_z"]["position_endstop"]|float}
  Z_OFFSET_APPLY_ENDSTOP
  SAVE_CONFIG  

[gcode_macro MANUAL_BED_LEVEL]
gcode:
  {% if "xyz" not in printer.toolhead.homed_axes %}
      G28
  {% endif %}
  SET_GCODE_OFFSET Z={printer.configfile.config["stepper_z"]["position_endstop"]|float}
  Z_OFFSET_APPLY_ENDSTOP
  SET_GCODE_OFFSET Z=0 MOVE=1
  SET_GCODE_VARIABLE MACRO=NEXT_Z VARIABLE=screw VALUE=1
  NEXT_Z
  

[gcode_macro NEXT_Z]
variable_screw: 0
gcode:
  {%if screw == 0 %}
    {action_raise_error("Must use CALIBRATE_Z to begin")}
  {% endif %}
  
  {% set coords = printer.configfile.config["bed_screws"]["screw" + screw|string].split(',') %}
  {% set x = coords[0]|int %}
  {% set y = coords[1]|int - 4 %}
  {% if x|int >= 60 %}
    {% set x = x - 40 %}
  {% else %}
    {% set x = x + 40 %}
  {% endif %}

  G90
  G0 X{x} Y{y} Z1.0 F3000

  {% if screw >= 3 %}
    {% set screw = 1 %}
  {% else %}
    {% set screw = screw + 1 %}
  {% endif %}
  SET_GCODE_VARIABLE MACRO=NEXT_Z VARIABLE=screw VALUE={screw}

[gcode_macro END_BED_LEVEL]
gcode:
  SET_GCODE_VARIABLE MACRO=NEXT_Z VARIABLE=screw VALUE=0

[gcode_macro BOOP]
gcode:
  G91
  G0 Z0.1
  G4 P1000
  G0 Z-0.1
  G90

```

Create another new file named `variables.cfg`.  If you already have a file to store variables, add the values below to it and remove the reference at the top of your new `dial_indicator.cfg`
```python
[Variables]
indicator_offset = 1.00
hotend_offset = 0.20
squish_offset = 0.03
```

# Levelling Process

As with any other bed levelling, make sure to preheat the hotend and bed to the temperature you most commonly print at. If there's anything on the nozzle, do a small retraction to stop oozing and clean off stuck on filament.

## Finding `indicator_offset`

The first step is determing what our `indicator_offset` should be.  This is distance between the 0 from the Z Endstop and the tip of the indicator's probe.

1. If the bed is not already roughly level, level the bed using `BED_SCREWS_ADJUST` and paper/feeler gauge as you normally would.
2. Run `MANUAL_BED_LEVEL` to move to the front bed screw, then attach the mount above it.  Make sure it's fitting securly and not resting on the toolhead.
3. Increase the Z Height in +0.1mm increments until the dial reads 0.  It may take a few revolutions before it stops moving.
4. Lower the Z Height in -0.1mm increments until you see movement on the dial.
5. Update `indicator_offset` in `variables.cfg` to whatever the current Z Height is, then save and reboot.

## Levelling the Bed

Now that we have a rough idea of where the probe is in relation to the Z Endstop, we can start making adjustments

1. Run `MANUAL_BED_LEVEL` to start the levelling process.
2. Use `BOOP` to check the height of the screw
    * If the dial goes up but doesn't return near 0 or doesn't drop at all, the screw is too high
    * If there's no movement on the dial at all, the screw is too low
3. Adjust the screw if needed. Make sure to remove your hand from the screw since the pressure you're putting on the bed can lead to a false reading.
4. Repeat steps 2 and 3 until `BOOP` shows a sharp increase and drop to 0 on the first motion, AND either no motion or barely perceptible movement between 0 and 1.
5. Continue to the next screw with `NEXT_Z` and repeat 2-4 until you are satisfied with all three bed screws.

## Determining Final Offset

Now that we've got a level bed, there's still one problem.  The `indicator_offset` has no relationship with our nozzle, so we don't actually know how far the hotend is from our freshly level bed.

1. Run the `BED_SCREWS_ADJUST` to begin cycling the hotend between the beds screws at 0 Z Height
2. From lowest to highest thickness, check feeler gauge blades underneath the hotend until one touches.
    * Tip: If you want to check a thickness between to blades, you may be able to combine two thinner blades to get an intermediate value
3. Save the thickness of the thinnest blade that touch the hotend
4. Repeat steps 2 and 3 for the remaining screws
    * The value should be the same or very close, but this helps double check that the bed is actually level
5. Update the `hotend_offset` in `variables.cfg` with the thickness from step 3
6. Run `UPDATE_Z_OFFSET` to update `printer.cfg` with your new value

Are you wondering "why are we using feeler gauge here, wasn't the whole point to use something better?" The problem with "paper" bed levelling (even with a more precise thickness) is the bed is only as level as our ability to feel consistent drag across all three screws. 

Checking depth as a simple pass/fail (either it touches the nozzle or it doesn't) eliminates that ambiguity.  And if we are confident that our bed is near-perfectly leveled, even if the true depth falls in the .01mm-0.03mm gap between some of the blades, we know that mistake will be the same across the bed and can be corrected by adjusting our offset.

## Squish Offset
 Setting the `squish_offset` in `variables.cfg` is completely optional.  It's an additional offset in case you're confident that you have the right `hotend_offset` but still want to make some small adjustments to help with first layer adhesion.  If you make a change to `squish_offset`, you just need to run `UPDATE_Z_OFFSET` to recalculate the Z `position_endstop` with the new combined value.