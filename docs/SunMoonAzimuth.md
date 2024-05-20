Sun-Moon Azimuth Tracking with SG90 servos, Home Assistant and ESPHome
======================================================================
by Phil Pownall

This is a variant of the Sun Altitude-Azimuth Tracker that tracks both the Sun and Moon Azimuths using:
- a Home Assistant Automation, 
- an ESPHome device driving two servo motors, and 
- a 3D-printed mechanism of two cogs, two platforms, and a base.

You could track any astronomical object - and make an Electronic Antikythera mechanism.

This device is astonishing in its simplicity - but it replicates something that in the mechanical age required years/decades of painstaking astronomical observation, brain-freezing cog ratio mathematics, and extraordinary skill in building devices.

### Ingredients
- A Home Assistant server
- An ESP32 device running ESPHome
- A 608ZZ bearing
- Two SG90 or MG90 180-degree servo motors
- A 3D-printed mount consisting of a base, two cogs, and two platforms

### Optional ingredients
- An ESP32 Expansion board to supply power and additional pins
- A 9V power adapter for the Expansion board
- An SSD1306 128x64 OLED display
- LEDs to illustrate when the Sun or Moon is visible above the horizon

### Derivative ingredients
- A compass rose for the base to show the azimuth

You load up and adopt an ESP32 module as you would normally using ESPHome.

The servo motors are connected to the ESP32, which is where an Expansion board comes in handy as it has lots of pin rows that match the servo 3-pin connector, and you can configure the power row to supply 5V using the expansion board jumper.  The board also has a 9V supply barrel connector and a power regulator so you can supply enough power for the 2 servos and the ESP32.

The Mount
---------

The mount was designed in TinkerCAD, providing a base with two SG90 cases for the servos placed at right angles to each other.
STL files for the design are available in this repository.

The platforms consist of a 1X cog and a pointer.  One platform has a hole in the centre; the other has a centre shaft. 
The two platforms are stacked on top of the bearing which is placed in the top of the base.
The 2X cogs are placed on top of each servo, and interlock with the 1X cogs.

To set up the tracker, 
- Set each servo to its centre position (180) using the number slider, 
- Position the base with the top (Moon) servo pointing North (360), and 
- Move the platforms to the 180-degree point 
- Tighten the 2X cog screw into the servo.
- Start the Sun Moon Tracker Automation in Home Assistant

[Sun Moon Tracker](SunMoonTracker.png)

The ESPHome code
================

The ESPHome yaml code simply consists of Number template and Servo definitions for two servos: the Sun Azimuth servo and the Moon Azimuth servo.  Note how the formula for the Servo maps the Azimuth (from 0 to 360 degrees) onto the servo position (-1 to +1).  Since the servo motor can rotate approximately 180 degrees, the 2X Cog increases the platform range to 360 degrees.

Number Templates
----------------

### Example Definition of number templates to drive the Sun and Moon Azimuth servos

```
number:
  # Define a number template to control servo 1
  # servo 1 is sun azimuth: 0 to 360: map to -1 to +1
  - platform: template
    name: Sun Azimuth Number
    id: number_sun_azimuth
    optimistic: True
#    internal: True
    mode: slider
    min_value: 0
    initial_value: 180
    max_value: 360
    step: 0.1
    set_action:
      then:
        - servo.write:
            id: servo_sun_azimuth
            level: !lambda 'return ((x / 360.0)* 2.0) - 1.0;'

  # Define a number template to control servo 2
  # servo 2 is moon azimuth: 0 to 360: map to -1 to +1
  - platform: template
    name: Moon Azimuth Number
    id: number_moon_azimuth
    optimistic: true
#    internal: True
    mode: slider
    min_value: 0
    initial_value: 180
    max_value: 360
    step: 0.1
    set_action:
      then:
        - servo.write:
            id: servo_moon_azimuth
            level: !lambda 'return ((x / 360.0)* 2.0) - 1.0;'
```

Servo Definitions
-----------------

The servos are defined as usual with two outputs, and two servo definitions:

### Example Definition of outputs and servos for Azimuth and Altitude
```
substitutions:
  # pin assignments
  servo_sun_azimuth: GPIO19
  servo_moon_azimuth: GPIO21

output:

  # Servo output 1   
  - platform: ledc
    pin: ${servo_sun_azimuth}
    id: ledc_1
    channel: 0
    frequency: 50 Hz

  # Servo output 2
  - platform: ledc
    pin: ${servo_moon_azimuth}
    id: ledc_2
    channel: 1
    frequency: 50 Hz

servo:
    # servo 1
  - id: servo_sun_azimuth
    output: ledc_1
    auto_detach_time: 1s
    transition_length: 8s

    # servo 2
  - id: servo_moon_azimuth
    output: ledc_2
    auto_detach_time: 1s
    transition_length: 8s

```


The Home Assistant Automation
=============================

The Home Assistant Automation runs every 5 minutes.  If the Sun is above the horizon, the ESPHome Sun Azimuth servo is updated with the Sun Azimuth position. 
Likewise, if the Moon is above the horizon, the ESPHome Moon Azimuth servo is updated with the Moon Azimuth position.
The Sun Azimuth and Elevation sensors are provided by the Sun Integration.
The Moon Azimuth and Elevation (Altitude) sensors are provided by the AstroWeather custom Integration via HACS.

The Sun Moon Tracker Automation
-------------------------------

```
alias: Sun Moon Tracker
description: ""
trigger:
  - platform: time_pattern
    minutes: /5
condition: []
action:
  - choose:
      - conditions:
          - condition: numeric_state
            entity_id: sensor.sun_solar_elevation
            above: 0
        sequence:
          - service: number.set_value
            metadata: {}
            data:
              value: "{{ states('sensor.sun_solar_azimuth') | float(0) }}"
            target:
              entity_id: number.espnode10_sun_azimuth_number
  - choose:
      - conditions:
          - condition: numeric_state
            entity_id: sensor.astroweather_moon_altitude
            above: 0
        sequence:
          - service: number.set_value
            metadata: {}
            data:
              value: "{{ states('sensor.astroweather_moon_azimuth') | float(0) }}"
            target:
              entity_id: number.espnode10_moon_azimuth_number
mode: single

```
