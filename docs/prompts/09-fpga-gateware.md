# Prompt 09: Cyclone V drive interface gateware

Use with `00-master.md`. Read `docs/PLAN.md` sections 4.1 and 5.

**Fill in before use:**

- Cyclone V device and development board: 5CGXFC5C6F27C7N on the Terasic Cyclone V GX Starter Kit (C5G). Six transceivers rated to 3.125 Gbps, PCIe hard IP, HSMC connector, no SATA or PCIe connector on the board.
- How the SATA drive connects to the board: `<Terasic SATA/SAS HSMC card P0053, or a custom HSMC card>`
- Hardware description language: `<SystemVerilog or VHDL>`

## Goal

Build the FPGA design that talks to the patient drive. A standard SATA controller hides the link behind its own state machines and error handling; when a failing drive stops responding, the controller, not our software, decides how long to wait. This design replaces it with our own SATA host core on a Cyclone V transceiver, so the recovery daemon controls every command, timeout and reset exactly, and so that writing to a patient is blocked in hardware.

The FPGA connects to the Stabilizer's computer (RK3568, Linux) as a PCIe endpoint.

## What you need to know

- Cyclone V transceivers (GX, GT, SX, ST) officially support SATA at 1.5 and 3.0 Gbps. They do not support 6 Gbps; SATA III drives must negotiate down, and the design must make sure they do.
- No open-source SATA host core exists for Cyclone V. Starting points: LiteSATA (link and transport layers are vendor-neutral; its PHY layer is Xilinx-only) and the open-source Stratix V SATA core published in 2016. Commercial encrypted cores are not allowed in this project.
- The PCIe hard IP block of the Cyclone V is used as the endpoint. Vendor hard IP and the transceiver PHY IP are acceptable; everything above them is our own open code.

## Part A: feasibility (Phase 0)

Prove the link before building the rest. Write method, results and evidence in `docs/specs/fpga-feasibility.md`.

0. Read the kit's manual and schematic first and record: which transceiver channels reach the HSMC connector, the AC-coupling capacitor values on them and on the SATA card (SATA expects about 10 nF), which reference clocks feed the transceiver bank and at what frequencies, and which channel the PCIe hard IP requires. Flag any conflict before writing HDL.
1. Bring up the transceiver for SATA: reference clock, out-of-band signalling (COMRESET, COMINIT, COMWAKE) using electrical idle and signal detect, speed negotiation.
2. Reach a stable link at 1.5 Gbps and at 3.0 Gbps, including with drives that use spread-spectrum clocking.
3. Send IDENTIFY DEVICE and read sectors with a PIO and a DMA read command.
4. Test with several drives: a 3 Gbps hard drive, a 6 Gbps hard drive, a 6 Gbps SSD. Record the negotiated speed and link error counters over a long read.
5. Confirm the free Quartus edition builds the design with the transceiver and PCIe blocks.

If the link cannot be made stable, say so plainly and report what was tried. That result decides whether the FPGA port goes ahead.

## Part B: the drive interface (Phase 2F)

**SATA host**

- PHY wrapper for the Cyclone V transceiver: reset sequencing, out-of-band signalling, speed negotiation, forced speed (1.5 or 3.0 Gbps), alignment, clock compensation.
- Link layer: primitives, scrambling, CRC, flow control.
- Transport layer: all frame types needed for non-data, PIO in / out and DMA in / out commands. No NCQ.

**Command engine (what a standard controller cannot do)**

- Timeout per command, counted in hardware, set by software for each command.
- Immediate abort of a running command, and soft reset, hard reset (COMRESET with programmable timing) on request, each with a guaranteed maximum time to return control.
- Data from a frame with a CRC error is delivered and flagged, not discarded.
- For every command: raw status and error registers, received frames, and timing of each stage.
- Counters for link events (CRC errors, retries, loss of sync, speed changes).
- A capture buffer that records link primitives and frames around an event, readable by software.

**Write blocker**

- A filter between the command engine and the link. Each outgoing command is checked against a fixed list of read-only and harmless commands. Anything else is rejected unless the write-arm input pin is high. That pin comes from the physical switch on the front panel.
- The list is part of the design, not a register. Software cannot change or bypass it.
- Unknown and vendor-specific commands count as writes.
- Document the list with the reason for each entry, from the ATA command set specification.

**PCIe endpoint and Linux side**

- Register interface, DMA for drive data, interrupts.
- The development kit has no PCIe connector. Until an HSMC-to-PCIe adapter exists, drive the same register interface through a JTAG or UART bridge, so that the PCIe wrapper is the only part left to prove on hardware later.
- A Linux kernel driver for the RK3568 (`firmware/device/`) that exposes the command interface to the `drd` daemon, and a `libdr` backend that implements the standard backend interface on top of it.
- The patient port is never presented to Linux as a normal disk.

**Structure for later functions**

Leave a clean place for the Lab Port functions (native IDE host, parallel NAND, SPI, UART, I2C, JTAG / SWD). Do not build them in this task.

## Constraints

- Only the PHY wrapper and the PCIe wrapper may contain vendor-specific code. Everything else is portable and simulates in open-source simulators.
- A testbench with a SATA device model covers: link bring-up, each command type, timeouts, abort and resets at every stage of a command, CRC errors, a device that stops responding at every stage, and every path through the write blocker. CI runs it.
- The write blocker has its own exhaustive test over all 256 command codes, for both states of the arm pin.
- Quartus builds are scripted and reproducible; the build reports timing and resource use, and a build with failed timing is a failed build.
- Licence: CERN-OHL-S-2.0. Reused code keeps its own licence and attribution; check compatibility first.
- State exactly which drives and which board were used for every hardware result. Mark anything not measured `UNVERIFIED:`.

## Done when

- `docs/specs/fpga-feasibility.md` answers the Part A points with evidence.
- The register map and driver interface are specified in `docs/specs/fpga-interface.md` and match the implementation.
- All testbenches pass in CI.
- On hardware: a healthy drive images correctly through the FPGA port with `drctl`, and the image matches one taken through a normal SATA port.
- On hardware: with a drive that hangs (or a device emulator that does), the time from "stops responding" to "control regained" is measured and reported, next to the same measurement on the RK3568 native port.
- On hardware: with the arm pin low, write commands sent by a test tool never appear on the link (shown by the capture buffer).

## Start with

Part A, steps 1 to 3, on the development board. In parallel, set up the simulation testbench with the device model, because the link and transport layers can be developed and tested there before the hardware link is stable.
