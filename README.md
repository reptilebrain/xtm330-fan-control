# WatchGuard XTM 330 fan control for OpenWrt

This repository contains one small OpenWrt init script that configures quieter,
temperature-controlled fan operation on a WatchGuard XTM 330. It programs the
on-board Winbond W83793 hardware-monitoring chip, so fan control continues in
hardware after startup and needs no background process.

This has been tested on **one physical XTM 330 only**. It is not a claim of
compatibility with every XTM 330 revision or any other appliance.

## Tested environment

- WatchGuard XTM 330
- OpenWrt 25.12.0
- Winbond W83793 at `/sys/bus/i2c/devices/0-002d`
- fan tachometer: `fan1_input`
- fan control: `pwm1`
- usable temperature sensors: `temp5_input` and `temp6_input`

On the tested unit, `temp1_input` through `temp4_input` report `-128000` and
must not be used. The original fan speed was about 10,975 RPM. PWM 84 produced
about 1,700 RPM, and idle temperature stabilized around 40–41 °C. These are
observations from one unit, not safety limits or guaranteed results.

## How it works

The W83793 supports SmartFanII, a hardware fan curve with seven temperature/PWM
points per temperature channel. The script sets `temp5_pwm_enable` and
`temp6_pwm_enable` to `3` (SmartFanII), then maps both sensors to PWM1 with
`temp5_auto_channels_pwm=1` and `temp6_auto_channels_pwm=1`. When both sensors
request a speed, the chip uses the more critical request.

W83793 PWM values are effectively quantized in steps of 4. The same tested
curve is installed for both temp5 and temp6:

| Temperature | PWM |
| ---: | ---: |
| 30 °C | 80 |
| 38 °C | 84 |
| 42 °C | 84 |
| 45 °C | 92 |
| 50 °C | 96 |
| 55 °C | 128 |
| 60 °C | 252 |

At startup, the script waits up to 30 seconds for the W83793 sysfs interface.
It verifies every required file, temporarily disconnects temp5 and temp6 from
PWM1, sets PWM1 to the known-good value 84, installs both curves, enables
SmartFanII, and reconnects both sensors. Running it again writes the same
configuration and is safe. It does not alter unrelated W83793 settings.

## Install

Copy the script to the router and enable it:

```sh
scp xtm330-fan root@192.168.1.1:/etc/init.d/xtm330-fan
ssh root@192.168.1.1
chmod +x /etc/init.d/xtm330-fan
/etc/init.d/xtm330-fan enable
/etc/init.d/xtm330-fan start
```

To prevent it from running on future boots:

```sh
/etc/init.d/xtm330-fan disable
```

Disabling the service does not undo settings already programmed into the chip.
Use the restore steps below or reboot after disabling it.

## Monitor

Watch PWM, fan speed, and both valid temperature inputs:

```sh
D=/sys/bus/i2c/devices/0-002d

while true; do
    echo "PWM: $(cat $D/pwm1)  RPM: $(cat $D/fan1_input)  T5: $(cat $D/temp5_input)  T6: $(cat $D/temp6_input)"
    sleep 5
done
```

Temperatures are reported in millidegrees Celsius, so `41000` means 41 °C.
You can also inspect the active mappings and modes:

```sh
D=/sys/bus/i2c/devices/0-002d
cat $D/temp5_auto_channels_pwm $D/temp6_auto_channels_pwm
cat $D/temp5_pwm_enable $D/temp6_pwm_enable
```

## Restore manual or boot-default behaviour

Stop the service to disconnect both temperature channels from PWM1 and set a
high manual value (252):

```sh
/etc/init.d/xtm330-fan stop
```

For the device's normal boot-time behaviour, disable the service and reboot:

```sh
/etc/init.d/xtm330-fan disable
reboot
```

## Safety warning

**Do not leave the appliance unattended for 24/7 use until you have verified
temperatures and fan operation under real CPU and network load.** Cooling needs
depend on ambient temperature, workload, dust, fan condition, and hardware
revision. Keep a way to recover the router if the fan does not respond as
expected.

Reports from other XTM 330 owners are welcome. Please include the hardware
revision, OpenWrt version, relevant sysfs paths, idle/load temperatures, PWM
values, and observed RPM.

## License

MIT. See [LICENSE](LICENSE).
