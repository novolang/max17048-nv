# Changelog

## 0.1.0

The first release: a driver for Analog Devices' MAX17048 and MAX17049
battery fuel gauges on an I2C bus.

- `Max17048` and `Max17049`, the types a board's device table builds
  from the gauge's wiring, implementing the battery side of devices-nv's
  `PowerManager` and `Identifiable`. The rails of the trait are answered
  as a part with none: no rail, `RailOff`, `None` and `false`.
- The conversions of the data sheet: 78.125 µV per step of VCELL, 1/256
  percent per step of SOC, 0.208 %/hr per step of CRATE, and the steps of
  HIBRT, VALRT and VRESET; the MAX17049 reads twice the voltage per step.
- The alerts: the STATUS flags and their clearing, the ALRT pin, the
  low-SOC threshold, the SOC-change alert, the voltage window and the
  reset alert.
- Quick start, hibernate thresholds, sleep and wake, and the reset, whose
  command the part does not acknowledge.
- The RCOMP update against temperature, by the data sheet's formula.
- The device program `examples/battery_monitor.nv`, a battery monitor. Built
  for the nRF52840-DK it is 18,144 B of flash and for the nRF54LM20-DK
  19,504 B. When the package was written the two Nordic boards answered
  for I2C as absent hardware and the images were 14,084 B and 15,412 B,
  without the reading path; the TWIM drivers landed on 2026-11-10.
- Not verified on hardware: no board on the bench carries the chip. A reply
  from a MAX17048 over the Nordic boards' TWIM buses is NOT VERIFIED until
  the driver rig.
- Not implemented: loading the ModelGauge battery model into the TABLE
  registers, and the async form of `PowerManager`.
- `novo pkg build --bin <name> --target=<board>` hands the board's slug to
  clang as a target triple (`bugs/pkg/build-bin-board-target.md` in the
  novo-lang repository), and a `[[bin]]` entry makes the registry list a
  library as a program. The device program is therefore built by
  `tests/device_build.sh`, as the main of a scratch package, and the
  manifest has no `[[bin]]` entry.
