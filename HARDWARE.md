# WatchGuard XTM 330 hardware notes

This document records what has been verified on the XTM 330 used to develop
this project, along with published specifications, visual observations, and
remaining unknowns. It is intentionally explicit about evidence: a device-tree
description or a reading from one unit is not proof that every hardware
revision is identical.

Last verified on the test unit: 2026-10-11.

## Evidence levels

- **Runtime verified**: read from the test unit running OpenWrt.
- **Source verified**: described by the upstream OpenWrt device tree or the
  manufacturer's hardware guide.
- **Visual observation**: visible in a photograph of the test unit's mainboard,
  but not electrically traced.
- **Unknown**: not established well enough to rely on.

## Identification

| Item | Value | Evidence |
| --- | --- | --- |
| Product | WatchGuard XTM 330 | Runtime and source verified |
| Model identifier | NC5AE7 | Runtime and source verified |
| OpenWrt board name | `watchguard,xtm330` | Runtime verified |
| Mainboard marking | `MB-XTM330 V1.0` | Visual observation |
| OpenWrt target | `mpc85xx/p2020` | Runtime verified |
| Tested OpenWrt release | 25.12.5, revision `r33051-f5dae5ece4` | Runtime verified |
| Tested kernel | Linux 6.12.94 | Runtime verified |

The [OpenWrt XTM 330 device tree][openwrt-dts] is the most useful public
description of the board-level connections. The [WatchGuard hardware
guide][watchguard-guide] describes the enclosure, interfaces, environmental
requirements, and original LCD hardware monitor.

## Processor and memory

- Freescale/NXP P2020 SoC.
- Two e500v2 PowerPC cores, revision 5.1.
- Runtime clock: approximately 1066.67 MHz per core.
- The CPU is passively cooled by the large central heatsink.
- OpenWrt reports 768 MiB of platform memory and `MemTotal` of approximately
  743 MiB after kernel reservations.
- The WatchGuard guide specifies 1 GB RAM. The reason only 768 MiB is exposed
  to OpenWrt is currently unknown; possible bootloader or reserved-address-space
  explanations have not been verified.
- No direct CPU junction-temperature channel has been identified. The two valid
  temperature readings are external thermistors, so processor maximum-junction
  specifications must not be used as their operating targets.

## Flash and boot layout

The test unit exposes a 2 MiB parallel NOR device and a 512 MiB NAND device.

### NOR

| Partition | Offset | Size |
| --- | ---: | ---: |
| `cfg0` | `0x000000` | 128 KiB |
| `cfg1` | `0x020000` | 64 KiB |
| `mfg-data` | `0x030000` | 64 KiB |
| `bootopt` | `0x040000` | 704 KiB |
| `u-boot-env` | `0x0f0000` | 64 KiB |
| `u-boot` | `0x100000` | 512 KiB |
| `u-boot-failsafe` | `0x180000` | 512 KiB |

### NAND

| Partition | Offset | Size |
| --- | ---: | ---: |
| `dtb` | `0x00000000` | 128 KiB |
| `kernel` | `0x00020000` | 5 MiB |
| `ubi` | `0x00520000` | 506.875 MiB |

The original configuration areas were inspected for plain-text fan, PWM, and
temperature configuration strings; none were found. This does not prove that
the original firmware lacked such settings.

## Ethernet architecture

The appliance has seven external 10/100/1000BASE-T ports, numbered 0 through 6.
OpenWrt describes two cascaded Marvell DSA switches, both using the
`marvell,mv88e6085` compatible binding:

- Switch at MDIO address `0x10` serves external ports 0-3 and has a fixed
  1 Gbit/s RGMII-ID CPU link to the P2020 `enet2` controller.
- Switch at MDIO address `0x11` serves external ports 4-6 and has a fixed
  1 Gbit/s RGMII-ID CPU link to the P2020 `enet0` controller.
- The switches expose `internal1` and `internal2` ports for the inter-switch
  connection.
- The P2020 `enet1` controller is disabled in the OpenWrt device tree.
- Linux presents the external interfaces as `port0` through `port6`, backed by
  `eth1` for ports 0-3 and `eth0` for ports 4-6.

The chip markings visible near the lower heatsinks appear consistent with the
Marvell 88E6092 family. This is only a visual observation; software relies on
the broader `mv88e6085` compatibility declaration.

## I2C devices

Runtime enumeration on I2C bus 0:

| Address | Driver/name | Function |
| ---: | --- | --- |
| `0x2d` | `w83793` | Hardware monitor and fan controller |
| `0x32` | `rs5c372a` | Real-time clock |
| `0x49` | `dummy` | W83793-related/reserved client; exact role unverified |
| `0x4d` | `dummy` | W83793-related/reserved client; exact role unverified |
| `0x54` | `24c04` | EEPROM containing the base MAC-address data |
| `0x55` | `dummy` | Reserved client; exact role unverified |

The SoC exposes two MPC I2C adapters at `i2c@3000` and `i2c@3100`. The known
board devices above are on the first adapter.

## USB, GPIO, and expansion

- One Freescale on-chip EHCI USB 2.0 host controller is active.
- A Genesys Logic USB 2.0 hub (`05e3:0608`) is present behind it.
- GPIO 12 is assigned to active-low `usb-hub-reset`.
- GPIO 13 is assigned to the active-low Ethernet-switch reset line.
- PCIe controllers 0 and 1 are disabled by the OpenWrt device tree. PCIe
  controller 2 remains described, but no useful installed PCIe function has
  been established during this investigation.
- The front LCD, buttons, and LEDs are associated with an external controller
  whose protocol is not implemented by the current OpenWrt support. No fan
  control relationship to that controller has been found.

## Cooling hardware and sensor topology

The W83793 provides the only verified software fan-control path.

| Channel | Observed state |
| --- | --- |
| `temp1`-`temp4` | `-128000`; invalid/unconnected on this unit |
| `temp5`, `temp6` | Valid type-4 external thermistors, 1 °C resolution |
| `pwm1` | Active fan-control output |
| `pwm2`, `pwm3` | Exposed and observed at 252, but wiring/function unknown |
| `fan1` | Only active tachometer input |
| `fan2`-`fan5` | Zero RPM with no asserted fan alarm |

Both valid temperature channels are mapped to PWM1. The W83793 selects the
safer, higher PWM request when more than one sensor controls the same output.
The service configures both channels identically in the tested driver's
SmartFanII mode.

Visual inspection shows one populated four-position fan header near the upper
left corner of the board and an adjacent apparently unused header marked
`FAN2`. The CPU itself has a passive heatsink. The large power harness enters
the board on the right.

The appliance physically produces sound and airflow from both the chassis and
power-supply areas. Both appeared to change together during PWM tests, but only
one tachometer signal is exposed. The following possibilities remain open:

1. both fan motors share one regulated supply/control path and only one returns
   a tachometer signal; or
2. PWM1 directly controls the case fan while the PSU fan reacts separately but
   similarly.

There is currently no verified second software control channel, so independent
fan control must not be assumed. The PSU must not be opened merely to establish
this; hazardous voltages can remain stored after disconnection.

## Fan calibration

The test temporarily disconnected both temperature channels from PWM1, applied
fixed PWM values under supervision, and restored automatic control afterward.
Temperatures remained between 44 and 46 °C.

| Written PWM | Effective PWM | Observed fan1 RPM | Notes |
| ---: | ---: | ---: | --- |
| 72 | 72 | 0 | Fan stopped |
| 76 | 76 | 0 | Fan stopped |
| 80 | 80 | 0 | Fan stopped |
| 84 | 84 | about 1,725 when approached from below | Lowest verified running value |
| 88 | 88 | about 2,000-3,300 | Strong dependence on previous speed |
| 92 | 92 | about 5,000-5,450 | Intermediate region |
| 96 | 96 | about 9,060 | Large step from PWM 92 |
| 112 | 112 | about 10,465-10,546 | Near maximum observed speed |
| 128 | 128 | about 10,465 | No meaningful increase over 112 |

The hardware quantizes values in steps of four: requested values 89-91 read
back as 88, and 93-95 read back as 92. The motor/control path also exhibits
hysteresis. For example, PWM 84 produced roughly 1,725 RPM when approached from
below, but approximately 2,800 RPM after descending from a higher speed.

Values below 84 are unsafe for this unit's unattended curve because the fan can
stop and the unused fan alarm thresholds do not provide a verified restart or
alarm path.

## Active tested curve

The same seven-point table is programmed for temp5 and temp6:

| Temperature | PWM |
| ---: | ---: |
| 30 °C | 84 |
| 40 °C | 84 |
| 46 °C | 84 |
| 50 °C | 84 |
| 54 °C | 88 |
| 57 °C | 96 |
| 60 °C | 252 |

The global W83793 critical-temperature value observed on the test unit was
80 °C. It is a final hardware safety mechanism, not a suitable normal operating
target. The project curve reaches full output at 60 °C to retain substantial
margin.

## Thermal test results

With the active curve and PWM 84:

| Test | Duration | Peak temp5/temp6 | Fan speed |
| --- | ---: | ---: | ---: |
| Idle baseline | observed before load | 45/45 °C | about 2,800 RPM |
| One CPU core saturated | 5 minutes | 45/45 °C | about 2,830 RPM |
| Both CPU cores saturated | 5 minutes | 46/46 °C | about 2,830 RPM |

The two-core test reached a load average of approximately 2.1. Both load
processes were confirmed stopped afterward. This validates CPU-heavy operation
for the tested duration, but not maximum simultaneous Ethernet-switch,
storage, PSU, and high-ambient load.

The WatchGuard guide specifies an operating ambient range of 0-40 °C and warns
against obstructing ventilation. The internal thermistor values above must not
be confused with the ambient-temperature specification.

## Voltage monitoring

The W83793 exposed ten voltage inputs during inspection. None asserted an alarm,
but the board-specific rail labels and scaling have not been established.
Raw values should therefore not be documented as named supply rails without
additional tracing or original board documentation.

The kernel also reports that it cannot identify the CPU VRM version. The VRM
setting should not be changed based on guesswork.

## Remaining work and unknowns

- Locate temp5 and temp6 physically on the board.
- Trace whether one PWM/power path controls both physical fans.
- Identify which physical fan supplies `fan1_input`.
- Establish whether PWM2/PWM3 and the empty `FAN2` header are electrically
  usable on this board revision.
- Explain the difference between the documented 1 GB RAM and the 768 MiB
  platform memory exposed to OpenWrt.
- Identify and scale the voltage-monitoring inputs.
- Characterize temperatures under sustained routed traffic through both switch
  groups and at higher room temperature.
- Measure PSU exhaust temperature externally during load.
- Decode the external front-panel controller protocol, if LCD/buttons/LEDs are
  desired under OpenWrt.

## Sources

- [OpenWrt XTM 330 device tree][openwrt-dts]
- [OpenWrt XTM 330 hardware data page][openwrt-hw]
- [OpenWrt support pull request 21020][openwrt-pr]
- [Linux W83793 driver documentation][linux-w83793]
- [WatchGuard XTM 330 Hardware Guide][watchguard-guide]
- [NXP P2020 product page][nxp-p2020]

[openwrt-dts]: https://github.com/openwrt/openwrt/blob/main/target/linux/mpc85xx/files/arch/powerpc/boot/dts/xtm330.dts
[openwrt-hw]: https://openwrt.org/toh/hwdata/watchguard/watchguard_xtm330_nc5ae7
[openwrt-pr]: https://github.com/openwrt/openwrt/pull/21020
[linux-w83793]: https://docs.kernel.org/hwmon/w83793.html
[watchguard-guide]: https://www.boc.de/pub/media/documents/xtm330/25_Hardware-Guide-WatchGuard-XTM-330.pdf
[nxp-p2020]: https://www.nxp.com/products/P2020
