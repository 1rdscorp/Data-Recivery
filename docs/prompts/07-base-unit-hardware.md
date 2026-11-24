# Prompt 07: Stabilizer base unit hardware

Use with `00-master.md`. Read `docs/PLAN.md` section 4. Start this only after the development-board prototypes work (`05-device-daemon.md` and Part A of `09-fpga-gateware.md`), and use their feasibility results.

## Goal

Prepare the design package for the Stabilizer base unit, revision A: one board carrying an RK3568 system-on-module (the computer) and a Cyclone V FPGA (the drive interface), with patient ports, switched and monitored patient power, a supervisor microcontroller, the Lab Port, and a USB 3.0 link to the PC.

The project owner is the electronics engineer. They will review your work, do or finish the layout, and decide what goes to fabrication. Your job is to give them a complete, well-sourced starting point and to make every assumption visible. Do not present a multi-gigabit layout as ready to build.

## Requirements

**Interfaces on the RK3568**

- USB-C, USB 3.0 device mode, to the PC.
- SATA port (6 Gbps): target drive for clones, also usable as a basic patient port.
- M.2 M-key slot for the internal NVMe image store.
- PCIe ×1 to the Cyclone V.
- Patient PCIe connection for the NVMe adapter (A08), if Phase 0 confirmed the lane split.
- Patient USB host port (USB 2.0) with switched, monitored VBUS.
- SD/MMC host connection for the SD and eMMC adapters.
- Gigabit Ethernet.
- Debug: serial console, recovery / boot-mode button.

**Cyclone V drive interface**

- A Cyclone V with transceivers (GX, GT, SX or ST; the exact device is the owner's choice) and at least three channels: PCIe endpoint to the RK3568, patient SATA port, one spare routed to a second SATA connector.
- Reference clocks suitable for SATA and for PCIe, per the transceiver handbook.
- Configuration: flash for stand-alone start, JTAG header, and a way for the RK3568 to update the flash. The FPGA must be configured before Linux enumerates PCIe, or the link must support a rescan.
- The write-arm switch signal reaches an FPGA input pin directly (through the supervisor's debounce at most), never through Linux.
- Lab Port driven from FPGA IO banks through level shifters (see below).
- Default state while the FPGA is unconfigured: patient signals idle, Lab Port outputs disabled.

**Patient power**

- Input 12 V DC, at least 5 A, with reverse-polarity and surge protection.
- Rails to the patient: 12 V / 3 A, 5 V / 3 A, 3.3 V / 3 A, and two adjustable rails 1.2 V to 3.3 V / 0.5 A.
- Per rail: electronic fuse with programmable current limit and fast short-circuit response, current and voltage monitor able to sample at 1 kHz or faster, active discharge.
- Patient power must be independent of the computer's and the FPGA's own supplies, so a shorted patient cannot brown out the system.
- Default state with no firmware running: all patient rails off.

**Supervisor microcontroller**

- Controls rail enables, discharge, fan, LEDs, buttons; reads the write-arm switch and fault lines; UART to the RK3568; can reset the RK3568 and hold the FPGA's patient outputs disabled.
- Powered whenever the unit has input power.

**Lab Port**

- About 40 bidirectional level-shifted signals at an adjustable IO voltage (1.2 V to 3.3 V), switched 3.3 V and 5 V, I2C for the adapter ID EEPROM, an adapter-present signal.
- Must be able to carry a 40-pin IDE interface (through 5 V level shifting on the adapter), parallel NAND, SPI, UART and JTAG.
- Protection so that a fault on the adapter side cannot damage the FPGA.
- Define the connector, pin map and electrical limits as a written specification (`docs/specs/lab-port.md`), because every low-speed adapter depends on it.

**Protection and mechanics**

- ESD protection on every external and patient-facing line, chosen for the signal speed.
- Patient connectors mounted so that a replaceable port-saver cable or board takes the wear.
- Fan header aimed at the patient, temperature probe input.
- Front panel: small display, buttons, LEDs (power, activity, fault, WRITE ARMED), write-arm switch, patient power-off button.
- Mounting holes and connector placement suitable for a simple enclosure with a drive tray.

## Deliverables (`hardware/base-unit/`)

1. **Requirements and block diagram**, including a power tree with worst-case currents (RK3568 module, FPGA core, transceiver and IO rails, patient rails) and a thermal estimate.
2. **Module and device selection:** compare at least two RK3568 modules (availability, connector, documentation, price) and recommend one. For the Cyclone V, list the candidate devices and packages with transceiver count, hard IP location, IO count and price, and show the transceiver and PCIe hard IP placement rules are met.
3. **Pin tables:** RK3568 pin multiplexing, and Cyclone V pin and bank plan (bank voltages, transceiver channels, clocks, configuration pins).
4. **Part selection** for each function with reasoning, key datasheet parameters, an alternative part, and the datasheet reference. Check that every chosen part is in production and available from mainstream distributors.
5. **Schematics** as a KiCad project with hierarchical sheets, passing ERC. Every non-obvious circuit has a note explaining the design choice and the datasheet section it follows.
6. **Layout constraints document:** stack-up, impedance targets (90 Ω USB, 100 Ω SATA, 85 Ω PCIe), length matching, placement rules for the high-speed and power sections, decoupling per the FPGA and module vendors' guidelines.
7. **BOM** with manufacturer part numbers and alternates.
8. **Bring-up and test procedure:** step by step from first power (no module fitted, current-limited supply) to full function, with expected measurements, plus fault injection tests for each rail.
9. **Review notes for the owner:** every open question, every unverified assumption, and the places where you were least sure.

## Constraints

- Every electrical value, pin assignment and part number comes from a datasheet, handbook or the module's hardware guide that you consulted; cite it. Mark anything you could not check `UNVERIFIED:`. A plausible-looking guess on a schematic is worse than a blank marked "to be decided".
- Design for hand debugging: test points on every rail and key signal, zero-ohm links to isolate sections, a way to run the board with the RK3568 module removed and with the FPGA unconfigured.
- Prefer parts that a common assembly service stocks.
- If you produce any layout, label it a draft for the owner's review.

## Done when

- All nine deliverables exist and agree with each other (pin tables, schematic, BOM).
- ERC passes; CI exports schematic PDF and BOM.
- The Lab Port specification is complete enough to design an adapter against.
- The review notes are explicit, so the owner knows exactly what to check.

## Start with

The requirements, block diagram, power tree and the two pin tables. These show early whether everything fits on the chosen module and FPGA package.
