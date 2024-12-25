Arduino Port Expander
=====================

.. seo::
    :description: Instructions on using an Arduino board, like the Pro Mini for expanding ports of an ESPHome node
    :image: arduino_logo.svg
    :keywords: Arduino port expander extender ESPHome

With this sketch you can control pins of a remote Arduino board through ESPHome. The Arduino acts as a port
expander, allowing you to use more pins than a standard ESP8266/ESP32 has.

.. figure:: images/arduino_pro_mini.jpg
    :align: center
    :width: 75.0%

The Arduino is connected to the ESP via I²C. Most Arduinos use the ``A4`` and ``A5`` pins for the I²C bus
so those pins are not available to read from ESPHome.
It is recommended to use a 3.3V I/O level Arduino, however using 5V Arduinos seems to work too. In the latter
case you should power your 5V Arduino with 3.3V otherwise you will need a level converter for the
I²C bus.

Currently it is supported:

    - reading digital inputs
    - reading analog inputs
    - writing digital outputs

The Arduino sketch can be retrieved from `here <https://gist.github.com/glmnet/49ca3d6a9742fc3649f4fbdeaa4cdf5d#file-arduino_port_expander_sketch-ino>`__
you can rename it to ``.ino`` and use the Arduino IDE to program it.

To use the device add the external component from `here <https://github.com/glmnet/esphome-components>`__.

.. code-block:: yaml

    external_components:
      source: github://glmnet/esphome-components
      components: [arduino_port_expander]

Setup your :ref:`I²C Bus <i2c>` and assign it an ``id``:

.. code-block:: yaml

    i2c:
      id: i2c_component

By default ESP8266 uses ``SDA`` pin ``GPIO4`` which you need to connect to Arduino's ``A4`` and the ``SCL``
is ``GPIO5`` which goes to Arduino's ``A5``.

Then configure the component:

.. code-block:: yaml

    arduino_port_expander:
      id: ape1 # identifier to use in the individual components
      address: 0x08
      analog_reference: DEFAULT

By default the I²C address is ``0x08`` but you can change it on the Arduino sketch so you can have more devices
on the same bus.

`analog_reference` can be `DEFAULT` or `INTERNAL`. Default is `INTERNAL`. See
Arduino [analogReference()](https://www.arduino.cc/reference/en/language/functions/analog-io/analogreference/) function for details.

`reference_voltage` is the maximum value that can be measured.
If the `analog_reference` is `DEFAULT`, then this should be the voltage that your Arduino board runs at, either `5` (default) or `3.3`.
If the `analog_reference` is `INTERNAL`, then this will be `1.1` (default) or `2.56`.  See the `analogReference()` link for details.

`gpio` pins
-----------

Use pins as any other port expander in ESPHome, you can use it in almost every place a pin is needed:

.. code-block:: yaml

    binary_sensor:
      - platform: gpio
        pin:
          arduino_port_expander: ape1
          number: 2
          mode:
            input: true # defaults to False
            output: false # defaults to False
            pullup: true # defaults to False

    switch:
      - platform: gpio
        name: Switch pin 3
        pin:
          arduino_port_expander: ape1
          number: 3


You can use any PIN from 0 to 13 or `A0` to `A3` (`A4` and `A5` are used for
I²C and `A6` and `A7` do not support internal pull up).
For A0 use 14, A1, 15 and so on.

.. note::

    Arduino PIN 13 usually has a LED connected to it and using it as digital input with the built in internal
    pull up might be problematic, using it as an output is preferred.

Sensor
------

Sensors allows for reading the analog value of an analog pin, those are from ``A0`` to ``A7`` except for
``A4`` and ``A5``.

Arduino analog inputs measures voltage. By default the sketch is
configured to use the Arduino internal VREF reference setup to 1.1 volt, so
higher voltages are read as 1023. You can configure Arduino to compare
the voltage to VIN voltage, this voltage might be 5 volts or 3.3 volts,
depending on how you are powering it.  See the main component config to declare
which voltage your board uses.

`raw: true` will return the raw measured value from 0-1023 (the value returned by the Arduino `analogRead` function) instead of the calculated voltage.

.. code-block:: yaml

    sensor:
      - platform: arduino_port_expander
        id: sx
        pin: A0
        name: Ardu A0
        update_interval: 10s

Full Example
------------

Let's connect a 4 channel relay board and 2 push buttons to toggle the relays, a PIR sensor, a window and a door
a LM35 temperature sensor and a voltage sensor. Seems a bit too much for an ESP8266? You'll still have some
spares I/Os.

.. code-block:: yaml

    esphome:
      name: test_arduino

    esp8266:
      board: nodemcu

    wifi:
      ssid: !secret wifi_ssid
      password: !secret wifi_password

    api:

    ota:
      platform: esphome

    external_components:
      source: github://glmnet/esphome-components
      components: [arduino_port_expander]

    # define i2c device
    # for an ESP8266 SDA is D2 and goes to Arduino's A4
    #                SCL is D1 and goes to Arduino's A5
    i2c:
      id: i2c_component

    logger:
      level: DEBUG

    # define the port expander hub, here we define one with id 'expander1',
    # but you can define many
    arduino_port_expander:
      id: expander1
      analog_reference: DEFAULT

    # define binary outputs, here we have 4, as the relays are inverse logic
    # (a path to ground turns the relay ON), we defined the inverted: true
    # option of ESPHome outputs.
    output:
      - platform: gpio
        id: relay1
        inverted: true
        pin:
          arduino_port_expander: expander1
          number: 2
      - platform: gpio
        id: relay2
        inverted: true
        pin:
          arduino_port_expander: expander1
          number: 3
      - platform: gpio
        id: relay3
        inverted: true
        pin:
          arduino_port_expander: expander1
          number: 4

    # connect a pump to the 4th relay
    switch:
      - platform: gpio
        name: Tank pump
        id: relay4
        inverted: true
        pin:
          arduino_port_expander: expander1
          number: 5

    # connect lights to the first 2 relays
    light:
      - platform: binary
        id: ceiling_light
        name: Ceiling light
        output: relay_1
      - platform: binary
        id: room_light
        name: Living room light
        output: relay_2

    # connect a fan to the third relay
    fan:
    - platform: binary
      id: ceiling_fan
      output: relay_3
      name: Ceiling fan

    # define binary sensors, use the Arduino PIN number for digital pins and
    # for analog use 14 for A0, 15 for A1 and so on...
    binary_sensor:
      - platform: gpio
        id: push_button1
        pin:
          arduino_port_expander: ape1
          number: 7
          mode:
            input: true
            pullup: true
        on_press:
          - light.toggle: ceiling_light
      - platform: gpio
        id: push_button2
        pin:
          arduino_port_expander: ape1
          number: 8
          mode:
            input: true
            pullup: true
        on_press:
          - light.toggle: room_light
      - platform: gpio
        id: pir_sensor
        name: Living PIR
        device_class: motion
        pin:
          arduino_port_expander: ape1
          number: 9
          mode:
            input: true
            pullup: true
      - platform: gpio
        id: window_reed_switch
        name: Living Window
        device_class: window
        pin:
          arduino_port_expander: ape1
          number: 10
          mode:
            input: true
            pullup: true
      - platform: gpio
        id: garage_door
        name: Garage garage
        device_class: garage_door
        pin:
          arduino_port_expander: ape1
          number: 14  # 14 = A0
          mode:
            input: true
            pullup: true

    # define analog sensors
    sensor:
      - platform: arduino_port_expander
        name: LM35 Living room temperature
        id: lm35_temp
        update_interval: 60s
        filters:
          # LM35 outputs 0.01v per ºC
          - lambda: return x / 100;
      - platform: arduino_port_expander
        name: Analog A2
        id: analog_a2
        update_interval: 2s


See Also
--------

- :ghedit:`Edit`
