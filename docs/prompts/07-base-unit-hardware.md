# Prompt 07: Stabilizer base unit hardware

Use with `00-master.md`. Read `docs/PLAN.md` section 4. Start this only after the development-board prototype (`05-device-daemon.md`) works, and use its feasibility results.

## Goal

Design the Stabilizer base unit, revision A: a carrier board for an RK3568 system-on-module with patient ports, switched and monitored patient power, a supervisor microcontroller, an IO coprocessor with the Lab Port, and a USB 3.0 link to the PC.

Your output is a design package that an electronics engineer can review, complete and send to fabrication. You are not expected to produce a validated multi-gigabit layout unaided, and you must not present one as ready to build.

## Requirements

**Interfaces**

- USB-C, USB 3.0 device mode, to the PC.
- Patient SATA port (data and power connector, drive-facing).
- Target SATA port for clone destinations.
- M.2 M-key slot for the internal NVMe image store.
- Patient PCIe connection for the NVMe adapter (A08), if Phase 0 confirmed the lane split.
- Patient USB host port (USB 2.0) with switched, monitored VBUS.
- SD/MMC host connection for the SD and eMMC adapters.
- Gigabit Ethernet.
- Lab Port (see below).
- Debug: serial console, recovery / boot-mode button.

**Patient power**

- Input 12 V DC, at least 5 A, with reverse-polarity and surge protection.
- Rails to the patient: 12 V / 3 A, 5 V / 3 A, 3.3 V / 3 A, and two adjustable rails 1.2 V to 3.3 V / 0.5 A.
- Per rail: electronic fuse with programmable current limit and fast short-circuit response, current and voltage monitor able to sample at 1 kHz or faster, active discharge.
- Patient power must be independent of the compute module's own supply, so a shorted patient cannot brown out the system.
- Default state with no firmware running: all patient rails off.

**Supervisor microcontroller**

- Controls rail enables, discharge, fan, LEDs, buttons; reads the write-arm switch and fault lines; UART to the compute module; can reset the compute module.
- Powered whenever the unit has input power.

**IO coprocessor and Lab Port**

- A microcontroller with programmable IO (candidate RP2350B), connected to the compute module by USB or SPI.
- Lab Port: about 24 bidirectional level-shifted signals at an adjustable IO voltage (1.2 V to 3.3 V), switched 3.3 V and 5 V, I2C for the adapter ID EEPROM, an adapter-present signal. Define the connector, pin map and electrical limits as a written specification (`docs/specs/lab-port.md`), because every low-speed adapter depends on it.

**Protection and mechanics**

- ESD protection on every external and patient-facing line, chosen for the signal speed.
- Patient connectors mounted so that a replaceable port-saver cable or board takes the wear.
- Fan header aimed at the patient, temperature probe input.
- Front panel: small display, buttons, LEDs (power, activity, fault, WRITE ARMED), write-arm switch, patient power-off button.
- Mounting holes and connector placement suitable for a simple enclosure with a drive tray.

## Deliverables (`hardware/base-unit/`)

1. **Requirements and block diagram**, including a power tree with worst-case currents and a thermal estimate.
2. **Compute module selection:** compare at least two RK3568 modules (availability, connector, documentation, price) and recommend one. Complete pin-multiplexing table showing every interface above fits.
3. **Part selection** for each function with reasoning, key datasheet parameters, an alternative part, and the datasheet reference. Check that every chosen part is in production and available from mainstream distributors.
4. **Schematics** as a KiCad project with hierarchical sheets, passing ERC. Every non-obvious circuit has a note explaining the design choice and the datasheet section it follows.
5. **Layout constraints document:** stack-up, impedance targets (90 Ω USB, 100 Ω SATA, 85 Ω PCIe), length matching, placement rules for the high-speed and power sections, and the compute module vendor's routing guidelines.
6. **BOM** with manufacturer part numbers and alternates.
7. **Bring-up and test procedure:** step by step from first power (no module fitted, current-limited supply) to full function, with expected measurements, plus fault injection tests for each rail.
8. **Design review checklist** and a list of every open question and unverified assumption.

## Constraints

- Every electrical value, pin assignment and part number comes from a datasheet or the module's hardware guide that you consulted; cite it. Mark anything you could not check `UNVERIFIED:`. A plausible-looking guess on a schematic is worse than a blank marked "to be decided".
- Design for hand debugging: test points on every rail and key signal, zero-ohm links to isolate sections, a way to run the board with the compute module removed.
- Prefer parts that a common assembly service stocks.
- Do not route the high-speed sections and call them finished. If you produce a layout, label it a draft for engineer review.

## Done when

- All eight deliverables exist and agree with each other (pin table, schematic, BOM).
- ERC passes; CI exports schematic PDF and BOM.
- The lab-port specification is complete enough to design an adapter against.
- The open-questions list is explicit, so the reviewing engineer knows exactly what to check.

## Start with

The requirements, block diagram, power tree and the pin-multiplexing table for the candidate modules. These show early whether everything fits on one module.
