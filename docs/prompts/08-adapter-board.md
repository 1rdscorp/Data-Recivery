# Prompt 08: Adapter board (template)

Use with `00-master.md`. Read `docs/PLAN.md` sections 4.2 to 4.4 and, for Lab Port adapters, `docs/specs/lab-port.md`.

**Fill in before use:**

- Adapter ID and name: `<for example: A03 PATA / IDE>`
- Devices it must support: `<list>`
- Anything already decided: `<bridge chip, connector, constraints, or "nothing yet">`

## Goal

Design the adapter board named above: a small PCB that connects one class of storage device to the Stabilizer base unit, safely and repeatably. Your output is a design package for review by the project owner, who is the electronics engineer, plus the software description the Stabilizer needs to recognise and use the adapter.

## Requirements common to all adapters

- **ID EEPROM** on I2C holding adapter type, hardware revision, serial number, capability flags and the maximum voltage allowed on each rail. The Stabilizer reads it and refuses unsafe settings. Follow the EEPROM layout in `docs/specs/lab-port.md`; if it is not defined yet, propose it as part of this task.
- **Patient power only from the Stabilizer's switched, monitored rails.** No always-on power to the patient.
- **Safe when plugged or unplugged** with patient power off; state clearly whether hot-plugging the patient is allowed.
- **ESD protection** on patient-facing signals, suitable for the signal speed.
- **Wear parts replaceable:** sockets, clips and connectors that wear out are on the adapter or on a cheap sub-board, never on the base unit.
- **Labelled:** adapter ID, revision, pin 1 marks, voltage warnings on the silkscreen.
- Passive adapters still carry the ID EEPROM where a connector with I2C is available; where it is not (a pure in-line port saver), say so.

## Notes per adapter class

- **Passive SATA family (A01, A02, A10):** keep SATA pairs short, impedance-controlled, no stubs. Check each proprietary connector pinout against a verified source before drawing it.
- **PATA / IDE (A03):** the IDE host is implemented in the Stabilizer's FPGA and reaches the adapter through the Lab Port, so the adapter carries 5 V level shifting, termination and protection, not a bridge chip. Support 40-pin 3.5", 44-pin 2.5" with 5 V, CompactFlash, and ZIF / LIF through sub-boards; master / slave configuration. A SATA-to-PATA bridge variant is the fallback if the native host is not ready.
- **UART terminal (A04):** selectable 1.8 V and 3.3 V levels, series protection, a way for the software to measure the drive's idle line voltage before driving it, probe options for different drive connectors.
- **ROM programmer (A05):** SPI NOR and I2C EEPROM at 1.8 / 2.5 / 3.3 V, SOIC-8 clip and socket. In-circuit reading must not back-power the drive's board; document the limits.
- **SD / eMMC (A06, A07):** native host mode, adjustable VCC and VCCQ, low-clock and 1-bit fallback; eMMC through replaceable BGA sockets and an in-system header with short leads.
- **NVMe (A08):** PCIe reference clock, reset and presence control, switched 3.3 V (and 12 V for U.2), surprise-removal tolerance.
- **USB patient port (A09):** switched and monitored VBUS with adjustable current limit, data-line protection.
- **NAND reader (A11, A12):** adjustable VCC / VCCQ, signal integrity at the chosen read speed, replaceable sockets per package. Scope is reading a raw dump with its spare area; reconstruction is a separate software project.

## Deliverables (`hardware/adapters/<ID>-<name>/`)

1. One-page requirement and block diagram: devices supported, connectors, voltages, currents, speeds.
2. Part selection with datasheet references and alternates.
3. KiCad schematic passing ERC, with design notes.
4. Layout: complete for low-speed adapters; for boards carrying SATA, PCIe or USB 3.0 signals, a constraints document and a draft labelled for engineer review.
5. BOM, fabrication notes (layers, thickness, finish, impedance if needed).
6. Bring-up and test procedure, including a short-circuit test on each patient rail and a test with a known-good device.
7. **Adapter descriptor** for the software: the EEPROM contents and a data file telling `libdr` which backend and limits this adapter uses.
8. A short user guide page for the website: what it connects, how to plug it, warnings.

## Constraints

- Pinouts, voltages and part numbers come from sources you consulted; cite them. Mark anything unchecked `UNVERIFIED:`. A wrong pinout here can destroy a patient.
- Keep it simple and cheap: two layers where the signals allow it, common parts, hand-solderable where practical.
- Do not claim the board works. State what was checked by rule checks or simulation, and what must be measured on the first built unit.

## Done when

- All deliverables exist and agree with each other.
- ERC and DRC pass in CI, which also exports schematic PDF, Gerbers and BOM.
- The descriptor loads in `libdr`'s adapter registry test.
- Open questions and unverified items are listed for the reviewing engineer.

## Start with

The requirement page and the connector pinouts with their sources. Most adapter failures are pinout mistakes, so settle those before drawing the schematic.
