# Prompt 06: Supervisor microcontroller firmware

Use with `00-master.md`. Read `docs/PLAN.md` sections 4.1 and 5.

## Goal

Write the firmware for the Stabilizer's supervisor microcontroller. The supervisor is the last line of protection for the patient: it switches and watches the patient's power rails, and keeps the patient safe even when the Linux module has crashed. It must be small, simple and reviewable.

## What it controls

- **Patient power rails:** 12 V, 5 V, 3.3 V, and two adjustable rails (1.2 V to 3.3 V). Each rail has an electronic fuse (enable, fault output, programmable limit), a current and voltage monitor, and an active discharge switch.
- **Front panel:** status LEDs including WRITE ARMED, buttons, the physical write-arm switch, the patient power-off button.
- **Fan** (PWM) and a patient temperature probe.
- **Watchdog** over the Linux module.
- **UART link** to the Linux module.

The exact microcontroller and pin map come from the hardware design (`07-base-unit-hardware.md`). If that is not finished, write the firmware against a hardware abstraction layer and a development board, and list the assumptions you made.

## Required behaviour

- **Safe at reset.** All patient rails off, discharge on, until commanded. This holds through brown-out, watchdog reset and firmware update.
- **Sequencing.** Rails turn on in a commanded order with controlled timing, and turn off in reverse. A power cycle holds the rails off with discharge active until the measured voltage is below a threshold, then waits a configurable time, so the drive truly resets.
- **Fault response without Linux.** Over-current, short circuit, over-voltage, under-voltage and over-temperature are handled locally and immediately: rails off, fault latched, LED on, event queued for Linux. The electronic fuses trip in hardware; the firmware confirms and latches.
- **Spin-up supervision.** Record the current curve on power-on at 1 kHz or faster into a buffer that Linux can fetch. Count spin-up attempts; if the drive keeps retrying spin-up, cut power and latch a fault, because repeated attempts damage a stuck drive.
- **Write-arm truth.** The supervisor reports the physical switch position. Linux can read it but can never set it. The WRITE ARMED LED is driven from the switch by the supervisor, not by Linux.
- **Patient power-off button** always works, whatever Linux is doing.
- **Linux watchdog.** If the Linux module stops sending heartbeats, the supervisor applies a configured policy: keep the patient powered and idle, or power it down. Default: power down after a grace period.
- **Voltage safety for adjustable rails.** The supervisor enforces the maximum voltage declared by the connected adapter's ID EEPROM and refuses a higher setting.

## Protocol to Linux

Specify it in `docs/specs/supervisor-protocol.md` before implementing: framed binary messages over UART with CRC, sequence numbers, request / response, and asynchronous events (fault, button, switch change). Commands: get status, set rail, sequence on / off, power cycle, set limits, fetch current trace, fan control, heartbeat, firmware version, firmware update entry.

## Constraints

- Bare-metal C11, no dynamic memory, no RTOS unless you can show it makes the firmware simpler to review.
- Fault handling paths are short, have no blocking calls, and do not depend on the UART link.
- Every state and transition of the power state machine is written down in the specification and covered by a test.
- The logic is separated from the hardware layer so it runs in host-side unit tests with simulated rails and faults.
- Part numbers and register maps come from datasheets you consulted; mark anything else `UNVERIFIED:`.

## Done when

- Host-side tests cover the power state machine: normal sequencing, each fault type on each rail, faults during sequencing, repeated spin-up, heartbeat loss, write-arm reporting, adapter voltage limits.
- Fault-to-rails-off time is measured on real hardware (development board with a dummy load is acceptable) and stated in the report.
- A protocol test tool on the Linux side exercises every command.
- The firmware builds reproducibly in CI, and the build reports flash and RAM use.

## Start with

The power state machine as pure logic with host-side tests. Then the protocol specification, then the hardware layer.
