# Data-Recivery

An open data recovery platform: hardware, software and teaching material for recovering data from failing hard drives, SSDs, USB sticks, memory cards and other storage.

- **Stabilizer**: a hardware unit that sits between a failing drive and the PC. It controls the drive's power, reads it with strict time limits, saves everything it reads, and shows the PC a clean read-only external disk. Inside: an RK3568 Linux computer and a Cyclone V FPGA that talks to the drive.
- **Adapter boards**: PCBs that connect SATA, IDE, M.2, NVMe, USB, SD, eMMC, NAND chips, ROM chips and diagnostic ports.
- **Software in C**: a core library, a command-line tool and a desktop app to read, image, extract and program drives.
- **Website and lab**: an interactive, step-by-step guide to recovering data from each kind of device.

**Status:** planning. Nothing is built yet.

**Licence:** open source. Software GPL-2.0-or-later, hardware and gateware CERN-OHL-S-2.0, lab content and documentation CC BY-SA 4.0. The licence files are added with the first code.

## Documents

- [Master plan](docs/PLAN.md): architecture, hardware, software, lab curriculum, roadmap, risks, open decisions.
- [Prompts](docs/prompts/README.md): ready-to-use prompts for building each part with an AI coding agent.
