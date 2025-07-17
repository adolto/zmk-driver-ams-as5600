# ams AS5600 Input Driver for ZMK Keyboard Firmware

A driver for the [ams AS5600](https://ams-osram.com/products/sensor-solutions/position-sensors/ams-as5600-position-sensor) magnetic rotary position sensor that converts rotary motion into scroll input events for the [ZMK](https://github.com/zmkfirmware/zmk) keyboard firmware.

## Features

- Converts rotary motion into smooth scroll input events
- Comprehensive sensor configuration options
- Monitors sensor status register and reports errors
- Logs AGC values to help optimize the air gap between sensor and magnet

## Installation

Add the driver module to your `config/west.yaml` by adding the new entries to the `remote` and `projects` sections:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: adolto # <-- new entry
      url-base: https://github.com/adolto
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-driver-ams-as5600 # <-- new entry
      remote: adolto
      revision: main
  self:
    path: config
```

## Integration

Enable the AS5600 driver in your config file:

```ini
CONFIG_ZMK_INPUT_AMS_AS5600=y
```

Integrate the AS5600 driver and listener into your `*.overlay` file.
Here is an example for a Seeed Studio XIAO nRF52840:

```dts
&xiao_i2c {
    compatible = "nordic,nrf-twi";
    pinctrl-0 = <&i2c0_default>;
    pinctrl-1 = <&i2c0_sleep>;
    pinctrl-names = "default", "sleep";
    status = "okay";

    as5600: as5600@36{
        compatible = "zmk,input-ams-as5600";
        reg = <0x36>;
        status = "okay";
    };
};

/ {
    
    ...

    as5600_listener: as5600_listener {
        compatible = "zmk,input-listener";
        device = <&as5600>;
    };
};
```

Scaling and reversing of scroll values can be done using the `zip_scroll_scaler` input processor:

```dts
as5600_listener: as5600_listener {
        compatible = "zmk,input-listener";
        device = <&as5600>;
        input-processors = <&zip_scroll_scaler (-2) 1>;
    };
```

## Setting up the distance between sensor and magnet

The sensor requires proper positioning relative to the magnet to function correctly. While the datasheet recommends 0.5–3 mm, the optimal distance varies depending on your specific magnet.

**The driver will not trigger scroll events if the magnetic field is:**

- Too weak (magnet too far)
- Too strong (magnet too close)
- Not detected (magnet misaligned or absent)

**Troubleshooting:** Check the terminal log for sensor status error messages and enable AGC logging to find the optimal distance:

```ini
CONFIG_ZMK_INPUT_AMS_AS5600_LOG_AGC=y
```

Optimal AGC value as described in the datasheet:

> The AGC register indicates the gain. For
the most robust performance, the gain value should be in the
center of its range. The airgap of the physical system can be
adjusted to achieve this value.\
In 5V operation, the AGC range is 0-255 counts. The AGC range
is reduced to 0-128 counts in 3.3V mode.

It's recommended to disable the logging after finding the optimal magnet distance.

## Configuration

The driver supports multiple configuration options.
See the [AS5600 datasheet](https://look.ams-osram.com/m/7059eac7531a86fd/original/AS5600-DS000365.pdf) for detailed technical information.

### Poll interval

The poll interval specifies the time period between sensor reads by the controller (in milliseconds).
The default value is 20 ms.

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_POLL_INTERVAL=20
```

It is recommended to match the poll interval with a suitable power mode.

### Power mode

The power mode configures the internal polling frequency of the sensor.
A higher polling time reduces the power consumption of the sensor.

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_POWER_MODE_NOM=y # NOM: Always on (default)
CONFIG_ZMK_INPUT_AMS_AS5600_POWER_MODE_LPM1=y # LPM1: Polling time = 5ms
CONFIG_ZMK_INPUT_AMS_AS5600_POWER_MODE_LPM2=y # LPM2: Polling time = 20ms
CONFIG_ZMK_INPUT_AMS_AS5600_POWER_MODE_LPM3=y # LPM3: Polling time = 100ms
```

Enabling the [watchdog](#watchdog) can help minimize power consumption even further.

### Hysteresis

The hysteresis setting helps avoid output fluctuations when the magnet is not moving:

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_HYSTERESIS_OFF=y # OFF (default)
CONFIG_ZMK_INPUT_AMS_AS5600_HYSTERESIS_1LSB=y # 1 LSB
CONFIG_ZMK_INPUT_AMS_AS5600_HYSTERESIS_2LSB=y # 2 LSB
CONFIG_ZMK_INPUT_AMS_AS5600_HYSTERESIS_3LSB=y # 3 LSB
```

### Step response and filter settings

The sensor is equipped with a programmable digital filter that consists of both slow and fast speed filters:

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_SLOW_FILTER_16X=y # 16x (default)
CONFIG_ZMK_INPUT_AMS_AS5600_SLOW_FILTER_8X=y # 8x
CONFIG_ZMK_INPUT_AMS_AS5600_SLOW_FILTER_4X=y # 4x
CONFIG_ZMK_INPUT_AMS_AS5600_SLOW_FILTER_2X=y # 2x
```

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_SLOW_ONLY=y # Slow filter only (default)
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_6LSB=y # 6 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_7LSB=y # 7 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_9LSB=y # 9 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_18LSB=y # 18 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_21LSB=y # 21 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_24LSB=y # 24 LSBs
CONFIG_ZMK_INPUT_AMS_AS5600_FAST_FILTER_THRESHOLD_10LSB=y # 10 LSBs
```

### Watchdog

Enable the watchdog timer to save power by automatically switching to LPM3 mode when the angle stays within the watchdog threshold of 4 LSB for at least one minute:

``` ini
CONFIG_ZMK_INPUT_AMS_AS5600_WATCHDOG_TIMER=y
```

## Troubleshooting

### Sensor is not triggering any scroll events

- Check wiring and I²C address of the sensor
- Magnet distance might not be set up properly. Please check for error logs and [set up the magnet distance](#setting-up-the-distance-between-sensor-and-magnet)

## Limitations

Analog output and PWM output modes are not supported.
The sensor values are always read via I²C polling.
