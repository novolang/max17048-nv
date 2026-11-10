# max17048-nv

A driver for the MAX17048 and the MAX17049, battery fuel gauges from
Analog Devices (formerly Maxim Integrated), specified in the data sheet
"MAX17048/MAX17049: 3µA 1-Cell/2-Cell Fuel Gauge with ModelGauge",
document 19-6171, Rev 7, 11/16
([analog.com/en/products/max17048](https://www.analog.com/en/products/max17048)).
The driver speaks I2C and implements the battery side of the
`PowerManager` trait and the `Identifiable` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv).

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this chip, so the driver has run only against a double of the part, in
the host suite. The device program below builds for the nRF52840-DK and
the nRF54LM20-DK and has not been flashed.

## The device program

`examples/battery_monitor.nv` reads a lithium cell once a second and prints
its voltage, its state of charge and its charge rate over the board's
UART:

```
cell 3874 mV  charge 87.500 %  rate -1.248 %/h
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_power.{ PowerManager }
use dev_bus
use max17048.{ Max17048 }

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The gauge answers at 0x36 on the board's first I2C bus.
    let gauge = Max17048.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 },
                                dev_bus.wiring_i2c(0, 0x36))
    while true
        match gauge.battery()
            Some(b) =>
                // b.voltage.mv is the cell voltage in millivolts and
                // b.percent the whole percent of charge, 0 to 100.
                hal.uart.write("cell ")
                hal.uart.write(str.from_int(b.voltage.mv))
                hal.uart.write(" mV\n")
            None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(1000)
```

The full program, with the charge in thousandths of a percent and the
rate, is `examples/battery_monitor.nv`. A board package that depends on
this package builds it as its `main`. `tests/device_build.sh` does so
from a scratch package for the two boards below and for the emulated
board, where it runs the image and checks that the program reports that
the gauge does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 14,084 B | 2,390 B | 2,644 B |
| `nrf54lm20-dk` | 15,412 B | 2,710 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena, 65,536 B on both boards, is the runtime's heap and is reserved
whether or not the program uses it. The program performs no I/O beyond
the board: its effect is `[hw]`, for the I2C bus, the UART and the timer.

`BoardI2c` reaches the bus that the board's package names with
`@hal(i2c)`. On a board that names none, the bus is absent hardware, every
transfer is refused, and the program prints `no answer`. The gauge sits
at 0x36 and has no address pin.

## What it is

A fuel gauge reports how full a lithium cell is. The MAX17048 measures
the cell's voltage and runs the ModelGauge algorithm on it, which models
the cell's open-circuit voltage and its relaxation, to give the
**state of charge** (SOC) as a percent of the cell's capacity. It has
no current-sense resistor and measures no current. It also reports the
**charge rate** (CRATE), an estimate of how fast the state of charge is
changing, in percent of capacity per hour. The part draws 23 µA when
active and 3 µA in hibernate mode.

The MAX17049 is the same part for a pack of two cells in series. It has
the same registers, the same address and the same commands. Its voltage
register counts twice the voltage per step. The driver has one type for
each, `Max17048` and `Max17049`, and they differ in that conversion
only.

The registers of the part, as 16-bit words with the most significant
byte first (data sheet Table 2):

| Address | Register | Step | Power-up value |
|---|---|---|---|
| 0x02 | VCELL, the cell voltage | 78.125 µV per cell | |
| 0x04 | SOC, the state of charge | 1/256 percent | |
| 0x06 | MODE: quick start, sleep enable, hibernate status | | 0x0000 |
| 0x08 | VERSION | | 0x001_ |
| 0x0A | HIBRT: hibernate and active thresholds | 0.208 %/hr and 1.25 mV | 0x8030 |
| 0x0C | CONFIG: RCOMP, sleep, alerts | | 0x971C |
| 0x14 | VALRT: voltage alert window | 20 mV | 0x00FF |
| 0x16 | CRATE, the charge rate | 0.208 %/hr | |
| 0x18 | VRESET/ID: reset voltage, factory identifier | 40 mV | 0x96__ |
| 0x1A | STATUS: alert flags | | 0x01__ |
| 0x40 to 0x7F | TABLE, the battery model | | |
| 0xFE | CMD, the reset command | | |

## Install

```sh
novo pkg add max17048-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `max17048` | The register map, the conversions between a register word and a voltage, a charge, a rate or a threshold, the register access over any `I2cBus`, and the types `Max17048` and `Max17049`, which hold a board's bus |

## How to choose an entry point

A program on a board uses `Max17048` or `Max17049`, as the board's device
table builds it:

```novo norun:fragment
    devices
        gauge  Max17048  from "max17048-nv"  on i2c0  addr 0x36  irq gauge_alrt
```

and reads it as `bsp.board.gauge()`. A program that holds its own bus, or
a test with a double of the part, calls the module's functions, which take
the bus and the wiring as arguments.

| Method of `Max17048` | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` on the board's bus; nothing is sent |
| `battery()` | `PowerManager`: the cell voltage in millivolts, the whole percent of charge, and the direction of the charge |
| `device_id()` | `Identifiable`: the VERSION word masked with 0xFFF0, 0x0010, with the low nibble as the revision |
| `begin()` | whether the part answers as a MAX1704x; nothing is written |
| `soc_milli_percent()`, `charge_rate()` | the charge in thousandths of a percent, and the rate in thousandths of a percent per hour, signed |
| `status()`, `alert_service(flags)` | the STATUS flags; clear the flags of a mask and release the ALRT pin |
| `alert_level()` | the level of the ALRT line |
| `set_soc_alert(percent)`, `set_voltage_alert(min_mv, max_mv)` | the low-SOC threshold, 1 to 32 percent, and the voltage window |
| `quick_start()`, `reset()` | restart the fuel-gauge calculation; reset the part |
| `set_hibernate(rate, mv)`, `sleep()`, `wake()` | the hibernate thresholds; sleep mode |
| `compensate(temp_milli_c)` | update RCOMP for the battery temperature |

The module's functions of the same names take `(bus, w, ...)` and add
`hibernate_never`, `hibernate_always`, `set_reset_voltage`,
`set_reset_alert`, `set_soc_change_alert`, `rcomp_read`, `rcomp_write`,
`chip_id` and the conversions.

## The rules a user needs

1. A register is a 16-bit word and is written whole. The part ignores a
   write of one byte (data sheet, "Register Summary"). The driver writes
   two bytes every time and reads a register before it changes some of
   its bits, so setting one field does not undo another.
2. One step of VCELL is 78.125 µV per cell. A word of 0xB400 is 3.6 V.
   The `PowerManager` reading is rounded to the nearest millivolt. A
   MAX17049 reports the two cells together, so the same word is 7.2 V.
3. One step of SOC is 1/256 percent, so 0x6400 is 100 percent. The
   register can read above 100 percent. `battery` holds the percent at
   100, and `soc_milli_percent` answers the register as it is.
4. CRATE is a signed 16-bit word of 0.208 %/hr steps. It is positive
   while the cell charges. The part has no current measurement, so
   `battery` answers a current of 0, and a positive rate answers the
   charge phase `ChargeConstantCurrent`; any other rate answers
   `ChargeIdle`. The part cannot tell the phases of a charger apart.
5. `battery` reads VCELL, SOC and CRATE in three transactions. The
   registers are not latched together, so the three values can come from
   two updates of the part, 250 ms apart in active mode.
6. The part has no rails. `rail_count` answers 0, `rail_state` answers
   `RailOff`, `rail_voltage` and `rail_set_voltage` answer `None`,
   `rail_enable` answers `false`, and `rail_current` and `power_source`
   keep the trait's defaults.
7. A reset, command 0x5400 to CMD, is not acknowledged: the part resets
   when the last bit is clocked in (data sheet, "CMD Register"). The
   driver reads VERSION first, sends the command, does not read its
   answer, and returns one second, the time to the first state of charge.
   A reset includes a quick start, so it belongs to a cell that has
   relaxed, and it clears every register, the battery model included.
8. The reset indicator, STATUS.RI, is set at power-up. The data sheet
   says the part is then not configured. `reset_indicator` reads it and
   `alert_service(STATUS_RI)` clears it.
9. An alert holds the ALRT pin low until CONFIG.ALRT is written to 0.
   `alert_service` clears the STATUS flags first, then the pin. It keeps
   EnVR, the enable bit of the reset alert, which is in the same
   register.
10. RCOMP, the compensation of the model for temperature, is set from
    the battery temperature by the data sheet's formula. With RCOMP0,
    TempCoUp and TempCoDown the constants of the battery model (0x97,
    -0.5 and -5.0 by default):
    above 20 °C, `RCOMP = RCOMP0 + (T - 20) x TempCoUp`; at or below it,
    `RCOMP = RCOMP0 + (T - 20) x TempCoDown`. The result is held to 0 to
    255 and the adjustment is truncated toward zero. The data sheet asks
    for the update at least once a minute. `compensate` writes it and
    keeps the rest of CONFIG.
11. The least significant byte of CONFIG holds the sleep bit, the SOC
    change alert, the alert bit and the five-bit low-SOC threshold. The
    threshold is (32 - ATHD) percent, 1 to 32.
12. Sleep needs two steps: MODE.EnSleep, then CONFIG.SLEEP. In sleep mode
    the part halts and does not see the cell change, so wake it before
    charging or discharging. `Max17048.sleep()` does both steps.
13. The identity is the VERSION word, 0x001_. The driver accepts any
    word whose upper twelve bits are 0x001, and answers 0x0010 as the
    identity and the low nibble as the revision.

## Running on a microcontroller

Every function of the module builds at the embedded tier, and the
manifest declares that tier, so a program of the embedded, real-time or
system tier may call it. The driver's own code builds no list: its bytes
move through devices-nv's `DevBytes`, a 32-byte buffer held in the
caller's frame. devices-nv's register access allocates one frame per
transaction and releases it before it returns, because `I2cBus` takes and
answers lists. The registry shows the package on the host layer, the layer
of the trait packages it depends on.

## What is not included

- The ModelGauge battery model. The TABLE registers (0x40 to 0x7F) take
  the model of one cell type, and the data sheet says to contact Maxim
  for it. The part runs its default model, which "is appropriate for some
  Li+ batteries". NOT IMPLEMENTED.
- The hardware quick-start pin, QSTRT. It is a pin of the board; the
  driver starts a quick start through MODE.
- The async form of `PowerManager`: `battery_measure` needs a value type
  that an `async fn` cannot take.
- Any use of the charge rate as a time to empty or to full. The data sheet
  calls the rate approximate and "not for conversion to ampere".
- VALRT and VRESET steps for the MAX17049 other than the 20 mV and 40 mV
  of the data sheet, which gives no separate figure for it.

## Reference drivers read

Each of these was read for its fixes, and none was ported. The fix is
named at its site in `src/max17048.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/fuel_gauge/max17048` | the VERSION check with the 0xFFF0 mask; the 16-bit overflow of its CRATE product, which this driver avoids |
| Adafruit_MAX1704X | the reset command is not acknowledged; the MODE and STATUS bit positions; a driver that writes VALRT and HIBRT as single bytes writes nothing, which the data sheet says |
| Linux `max17040_battery.c` | RCOMP and ATHD are changed with a masked update of CONFIG; the threshold range 1 to 32; the MAX17049 step is twice the MAX17048's |
| eldruin `max170xx` | the signed CRATE; the MAX17049's 5/32000 V step |

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `PowerManager` and `Identifiable` traits, the wiring record a board's
  device table builds, and the register access this driver uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles `BoardI2c` and
  `BoardGpio`.
- A power-management chip with rails and a charger, such as the nPM1300,
  implements the whole of `PowerManager`; this part implements the battery
  measurement only.

## Tests

`novo test tests` runs the register logic on the host over a double of
the part. The double keeps a register file of bytes and does what the
data sheet says: a register is a word, a write of one byte has no effect,
the quick-start bit clears itself, and the reset command resets every
register and is not acknowledged. It records each write, so a test reads
back the exact bytes the driver sent.

The suite checks the conversions against the data sheet's steps (78.125 µV,
1/256 percent, 0.208 %/hr, 20 mV, 40 mV, 1.25 mV), the power-up values of
Table 2, the temperature-compensation formula at 85, 60, 30, 20, 0 and
-10 °C, every flag of STATUS and its clearing, and each path of a part
that stops answering. `bash tests/coverage.sh` merges the suites' line
coverage over `src/`.

## Licence

Apache-2.0.
