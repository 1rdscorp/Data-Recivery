# Data Recovery Platform — Master Plan

Draft v0.2 · 2026-11-24 · Working names (Stabilizer, DR Studio, DR Lab) can be changed at any time.

## 1. What we are building

A complete, modular data recovery system. A failing storage device (the **patient**) connects through an adapter board to a hardware unit called the **Stabilizer**. The Stabilizer reads the patient gently and under strict control, stores what it reads, and shows the PC a clean, read-only "external HDD". Desktop software written in C controls the hardware, images drives, extracts files and (when deliberately armed) writes to drives. A website presents the project and hosts a guided, interactive lab that teaches recovery step by step.

```
 PATIENT                       STABILIZER (base unit)                          YOUR PC
+---------------+  adapter  +--------------------------------------+  USB-C  +-----------------------+
| HDD / SSD     |<--------->| patient ports  |  recovery engine    |<------->| sees an external HDD  |
| USB stick     |  switched | switched power |  image + sector map | USB 3   | (virtual, read-only)  |
| SD / eMMC     |<----------| current sense  |  virtual disk       |         | + DR Studio (C app)   |
| NAND chip     |  power    | supervisor MCU |  control protocol   |         | + any recovery tool   |
+---------------+           +--------------------------------------+         +-----------------------+
                                     |
                                     v
                            target storage (internal NVMe, or a SATA drive for full clones)
```

| # | Deliverable | What it is |
|---|-------------|------------|
| 1 | **Stabilizer base unit** | An embedded Linux computer (RK3568) plus an FPGA drive interface (Cyclone V), with patient ports, switched and monitored power, and a USB link to the PC |
| 2 | **Adapter boards** | A family of small PCBs that connect each kind of device (SATA, IDE, M.2, NVMe, USB, SD, eMMC, NAND, ROM chips, diagnostic UART) |
| 3 | **Device firmware and gateware** | The recovery daemon on the Stabilizer, the FPGA design for the drive interface, and a small supervisor microcontroller firmware |
| 4 | **PC software in C** | Core library, command-line tool and desktop app (DR Studio) for Windows and Linux, macOS later |
| 5 | **Website and DR Lab** | Project site, documentation, downloads, and the interactive step-by-step recovery lab |

The closest commercial products are the DeepSpar USB Stabilizer and Disk Imager, and at the high end ACE Lab PC-3000. The closest open-source software is GNU ddrescue and OpenSuperClone (the open-sourced HDDSuperClone). Our difference: open hardware and software in one system, a wide adapter family, and teaching built in.

### Non-goals

- Mechanical repair tooling (head swaps, platter work, clean room equipment). The lab explains these so people know when to stop and send a drive to a professional.
- Breaking encryption. BitLocker, FileVault, LUKS, self-encrypting drives and modern phones need the key. No tool changes that.
- Matching PC-3000 firmware repair for every drive family. Vendor firmware work is added family by family as plugins, late in the roadmap.

## 2. Patient safety rules

These rules apply to every component. They exist because the patient is usually the only copy of someone's data, and every operation on failing hardware can make it worse.

1. **Read-only by default.** Nothing writes to a patient unless a physical "write arm" switch on the Stabilizer is on *and* the user confirms in software. On the FPGA patient port the switch is wired into the logic itself, so no software fault can write.
2. **Image first.** All analysis and file extraction happens on the image. The patient is read once per sector where possible.
3. **Every read is recorded.** A sector map tracks each sector: unread, good, failed, skipped, with timing. Work always resumes from the map.
4. **Every command has a time limit.** On timeout the system escalates: abort, soft reset, hard reset, power cycle. It never waits forever.
5. **Power is watched.** Voltage and current are monitored on every patient rail. Over-current, failed spin-up or repeated spin-up attempts stop the job.
6. **Back up before changing.** Any ROM, firmware module or sector that will be overwritten is saved and verified first.
7. **Fail safe.** If the PC disconnects or the software crashes, the Stabilizer keeps the patient in a safe state and keeps the image and map intact.
8. **Honest limits.** The software and the lab say plainly when something cannot be recovered or needs a professional lab.

## 3. How the Stabilizer works

### The problem it solves

When a failing drive is connected straight to a PC, the operating system makes things worse. It retries bad sectors for 20 to 40 seconds each, reads the same damaged areas again and again, writes to the disk in the background, and often hangs or drops the device. Each of these stresses weak heads and damaged platters.

### The solution: image on demand

The PC never talks to the patient. It talks to a **virtual disk** created by the Stabilizer.

1. The PC asks the virtual disk for some sectors.
2. If those sectors are already in the image, they are returned from the image. The patient is not touched.
3. If not, the Stabilizer reads them from the patient with a short time limit (configurable, for example 250 ms to 8 s) and with drive retries, read-ahead and background scans turned off.
4. If the read succeeds, the data is saved to the image, marked good in the map, and returned to the PC.
5. If it fails, the Stabilizer runs the escalation ladder, marks the sectors as failed, and answers the PC quickly with a read error or a fill pattern. The PC never hangs.
6. Writes from the PC are refused (the disk reports write-protected), or optionally go to a separate overlay so that tools which insist on "repairing" can do so without touching the patient.

The Stabilizer can also run a full imaging job by itself in the background, with no PC connected, to the internal NVMe or to a target SATA drive.

### Operating modes

| Mode | What happens | Writes to patient |
|------|--------------|-------------------|
| **Stabilizer** | PC sees a read-only external disk; image on demand | Never |
| **Imaging** | Standalone multi-pass clone to image or target drive | Never |
| **Expert** | Direct ATA / SCSI / NVMe / MMC commands, SMART, diagnostics, UART terminal | Only if armed |
| **Program** | Write images, ROM chips, firmware modules, drive settings | Yes, armed and confirmed |
| **Software-only** | DR Studio or the CLI on a PC with the drive attached directly, no Stabilizer | Reduced protection, clearly labelled |

### Imaging engine features

- Multi-pass strategy: fast forward pass with skipping, reverse pass, trimming, scraping, limited retries.
- Per-command timeouts and the reset / power-cycle ladder.
- Drive settings applied before reading: no NCQ, read look-ahead off, short error recovery time (SCT ERC), background SMART scans off, optional link speed limit.
- Head map: find which areas belong to a weak head and leave them for last.
- File-system-aware imaging: read file system metadata first, then only used space, or only the folders the user chose.
- Thermal pauses and fan control from drive temperature.
- Power signature diagnostics: the spin-up current curve shows a stuck motor, stiction or a shorted board before any command is sent.
- Map files import and export in ddrescue format, so work can move between tools.

## 4. Hardware

### 4.1 Base unit

The base unit has two main chips. Each does what it is best at.

**Computer: Rockchip RK3568 system-on-module.** Runs Linux and the recovery daemon. It provides the USB 3.0 link to the PC, the NVMe image store, Ethernet and one native SATA port.

**Drive interface: Altera Cyclone V FPGA.** Talks to the patient. Our own SATA host core in the FPGA gives control that a standard SATA controller cannot:

- timeouts enforced in hardware, with immediate abort and reset;
- forced link speed and exact control of the reset sequence;
- data delivered and flagged even when a frame arrives with a CRC error, instead of being thrown away;
- a write blocker built into the logic and wired to the physical write-arm switch;
- exact timing and status for every command.

The same FPGA drives the Lab Port (native IDE, NAND, SPI, UART, JTAG), so no separate IO coprocessor is needed. The two chips are linked by PCIe: the Cyclone V's hard PCIe block is the endpoint and the RK3568 is the root.

| Resource | Use |
|----------|-----|
| RK3568 combo PHY 0 as USB 3.0 OTG (device mode) | Link to the PC |
| RK3568 combo PHY 1 as SATA (6 Gbps) | Target drive port; also a basic patient port |
| RK3568 combo PHY 2 as PCIe ×1 | Link to the Cyclone V |
| RK3568 PCIe 3.0, 2 lanes | Internal NVMe image store; patient NVMe port if the lanes can be split |
| RK3568 USB 2.0 host | Patient USB port (sticks, card readers, USB-native drives) |
| RK3568 SD/MMC controller | Patient SD and eMMC adapters |
| RK3568 Gigabit Ethernet | Network control, imaging to a NAS |
| Cyclone V transceiver 1 | PCIe endpoint to the RK3568 |
| Cyclone V transceiver 2 | Patient SATA port with full low-level control (1.5 and 3.0 Gbps) |
| Cyclone V transceiver 3 | Spare, routed to a second SATA connector |
| Cyclone V general IO | Lab Port (section 4.3) |

**Why not a Cyclone V alone.** A Cyclone V SoC (SX or ST, with its two ARM cores) could run the whole unit. This was evaluated:

| Need | Cyclone V alone | Verdict |
|------|-----------------|---------|
| SATA to the patient | The transceivers support SATA at 1.5 and 3.0 Gbps. No open-source SATA host core exists for Cyclone V and the commercial ones are closed, so we write our own. | Works, after development |
| "External HDD" link to the PC | The ARM side has USB 2.0 only. Mass storage reaches roughly 10 to 35 MB/s, so reading one terabyte through the virtual disk takes about 8 to 28 hours. USB 3.0 would need an extra controller chip (for example Infineon FX3) with its own firmware. | Works, but slow |
| Image store | NVMe through the Cyclone V PCIe root port is reported at 50 to 90 MB/s, with driver problems | Weak |
| Processor | Two Cortex-A9 cores | Enough for control, slow for hashing and file system work |

So the Cyclone V is the stronger chip for the drive side and the weaker one for everything else. Using both also lowers schedule risk: the first prototype needs only the RK3568 and its native SATA port, and the FPGA patient port is added when its SATA core works. The daemon sees both ports through the same backend interface.

**Which Cyclone V.** It must be a variant with transceivers: GX, GT, SX or ST, with at least three channels. E and SE devices have no transceivers and cannot do SATA or PCIe; several popular boards (DE10-Nano, DE1-SoC) carry SE devices. A GX is enough, because the RK3568 is the processor.

**A limit to know.** Cyclone V does not support SATA at 6 Gbps. SATA III drives fall back to 3 Gbps on the FPGA port, which is enough for hard drives and acceptable for SATA SSDs. The RK3568 port runs at 6 Gbps.

**Not yet verified (Phase 0 tasks on development boards):**

1. SATA link bring-up on a Cyclone V transceiver: out-of-band signalling, 1.5 and 3.0 Gbps, drives that use spread-spectrum clocking.
2. PCIe between the RK3568 (root) and the Cyclone V (endpoint), with DMA under Linux.
3. USB 3.0 device-mode throughput and stability on the RK3568.
4. Splitting the two PCIe 3.0 lanes, needed only for the patient NVMe port.
5. That the free Quartus edition covers the transceiver and PCIe blocks we use.

The combo PHY sharing on the RK3568 is confirmed by the mainline kernel driver. The patient USB port is USB 2.0 in revision A; this is acceptable because failing flash devices are read slowly anyway.

**Patient power.** This is the part that makes the unit a stabilizer and not just a bridge.

- Input: 12 V DC, at least 5 A.
- Patient rails: 12 V, 5 V, 3.3 V (for M.2 and mSATA), and two adjustable rails 1.2 V to 3.3 V (VCC and VCCQ for flash chips).
- Each rail has an electronic fuse with a programmable current limit and fast short-circuit trip, a current and voltage monitor sampled at 1 kHz or faster, and an active discharge circuit so a power cycle fully resets the drive.
- Candidate parts, to be checked against datasheets: TI TPS2595x / TPS25947 eFuses, TI INA228 monitors.

**Supervisor microcontroller.** A small MCU (STM32G0 class or RP2040) owns the rail enables, power sequencing, fan, buttons, LEDs, the write-arm switch and a watchdog over the Linux module. If Linux crashes, the supervisor still protects the patient.

**Front panel.** Small display, a few buttons, status LEDs including a red WRITE ARMED light, a physical write-arm switch, and a patient power-off button.

**Protection and mechanics.** ESD protection on every patient-facing line, reverse-polarity protection, replaceable "port saver" cables on the patient connectors (they wear out), a fan aimed at the patient with a temperature probe, and a drive tray with vibration damping.

### 4.2 Adapter boards

| ID | Adapter | Devices | Method | Difficulty | Phase |
|----|---------|---------|--------|------------|-------|
| A01 | SATA port saver | 2.5" and 3.5" SATA HDD / SSD | Passive | Low | 3 |
| A02 | mSATA and M.2 SATA | SATA SSD modules | Passive, switched 3.3 V | Low | 3 |
| A03 | PATA / IDE | 40-pin and 44-pin IDE, CompactFlash, 1.8" ZIF / LIF | Native IDE host in the FPGA through 5 V level shifters; a SATA-to-PATA bridge chip as fallback | Medium | 3 |
| A04 | UART terminal | HDD diagnostic ports (Seagate and others) | Lab Port, selectable 1.8 V / 3.3 V levels | Low | 3 |
| A05 | ROM programmer | SPI NOR and I2C EEPROM on drive PCBs, SOIC-8 clip and socket | Lab Port, 1.8 / 2.5 / 3.3 V | Low | 3 |
| A06 | SD / microSD / MMC | Memory cards | Native SD host, adjustable voltage and clock, 1-bit mode | Medium | 5 |
| A07 | eMMC socket and ISP | BGA eMMC chips, in-system flying leads | Native MMC host, adjustable VCC / VCCQ | Medium | 5 |
| A08 | M.2 NVMe and U.2 | NVMe SSD | PCIe lane, switched power, reset control | Medium to high | 5 |
| A09 | USB patient port | USB sticks, card readers, USB-native HDD | USB host with switched, monitored VBUS | Low to medium | 5 |
| A10 | Proprietary SSD passives | Apple 2013 to 2017 blade SSDs and similar | Passive to M.2 | Low | 5 |
| A11 | NAND reader | Chip-off: TSOP48, BGA152 / 132, LGA52 / 60 | Parallel NAND interface in the FPGA | High | 6 |
| A12 | Monolith breakout | Monolithic microSD and USB sticks via test pads | NAND reader plus fine-pitch leads | High | 6 |
| A13 | JTAG / SWD probe | SSD controllers, drive MCUs (research) | Lab Port | Medium | 6 |
| A14 | SAS | SAS HDD | PCIe SAS controller | High | 7, optional |
| A15 | UFS | Phone storage | UFS host bridge | High | 7, optional |

Notes that keep expectations honest:

- Reading raw NAND (A11, A12) is the easy half. Turning the dump back into files needs ECC correction, descrambling and rebuilding the controller's block mapping, which differs per controller. That is a research-heavy software project of its own.
- Chip-off does not work on modern SSDs and phones, because the controller encrypts the NAND. For those, recovery means getting the original controller to work again.
- Seagate diagnostic ports use 1.8 V on some models and 3.3 V on others, and newer models lock the port. A04 must be level-selectable and the software must measure before driving.

### 4.3 Lab Port and adapter identification

High-speed interfaces (SATA, PCIe, USB) use their own standard connectors for signal quality. Everything slower goes through one common connector, the **Lab Port**, driven by the FPGA:

- Adjustable IO voltage rail and switched 3.3 V / 5 V.
- About 40 level-shifted, bidirectional signals (a 40-pin IDE interface alone needs close to 30).
- I2C for an **ID EEPROM on every adapter**: adapter type, revision, serial number, capabilities and voltage limits.

The software reads the EEPROM, loads the matching FPGA function, and refuses to apply a voltage the adapter does not allow. The exact connector and pin map are a Phase 3 design output.

### 4.4 PCB workflow

- KiCad, current stable release. One project folder per board, one shared symbol and footprint library.
- Base unit: 6 layers or more expected, controlled impedance (90 Ω USB, 100 Ω SATA, 85 Ω PCIe differential pairs). Adapters: 2 or 4 layers.
- CI runs ERC and DRC and produces schematic PDF, Gerbers, BOM, pick-and-place files and a 3D render for every change.
- Every board has test points, a version on the silkscreen, a bring-up checklist and a test procedure.
- The project owner is the electronics engineer: they design, review and sign off every board. AI-drafted schematics and part choices are input to that review, never a substitute for it. Budget for two or three revisions of the base unit.
- **Prototype first on development boards**: an RK3568 board with a simple power-switch board for the Stabilizer concept, and a Cyclone V transceiver board for the SATA core. The custom base unit PCB comes only after both are proven.

## 5. Device firmware and gateware

**Linux image (RK3568).** Buildroot, a recent long-term kernel, read-only root file system, signed A/B updates. The kernel must never touch the patient on its own: no partition scanning, no auto-mount, no file system drivers bound to patient ports.

**Recovery daemon `drd`** (C, built on the same core library as the PC software):

| Module | Job |
|--------|-----|
| Patient backends | FPGA SATA, native SATA, USB / SCSI, NVMe, MMC, NAND, SPI |
| Engine | Imaging strategy, timeouts, escalation ladder |
| Store | Image file and journaled sector map, safe against power loss |
| Virtual disk | A ublk userspace block device that implements image-on-demand |
| USB gadget | Composite device: mass storage backed by the virtual disk, plus a vendor interface for control |
| Power manager | Talks to the supervisor MCU |
| Protocol server | The control protocol (DRP) over USB and TCP |
| Black box | Full command and event log for every job |

**SATA control.** The patient can be reached two ways, behind the same backend interface:

1. **RK3568 native SATA port**, with standard Linux libata and ATA pass-through. It works on day one, but the kernel's and the controller's own error handling can still take many seconds on a hung drive. Used for the first prototype and kept as the basic port.
2. **FPGA SATA port**, with our own host core. Full control. This is the port for failing drives.

If the FPGA core is late, a small custom kernel driver for the native port is the fallback. OpenSuperClone shows that direct control of a standard SATA controller is possible.

**Gateware (Cyclone V).**

| Block | Job |
|-------|-----|
| SATA PHY wrapper | Transceiver setup, out-of-band signalling, speed negotiation, forced speed |
| SATA link and transport | Frames, CRC, scrambling, flow control; PIO, DMA and non-data commands. No NCQ |
| Command engine | Hardware timeout per command, immediate abort, reset on demand, exact timing and status, data kept and flagged on CRC error |
| Write blocker | Passes only commands on a fixed read-only list unless the write-arm switch input is on. Software cannot change the list |
| PCIe endpoint | Registers, DMA and interrupts toward the RK3568 |
| Lab Port functions | Native IDE host, parallel NAND, SPI, UART, I2C, JTAG / SWD, added over time |
| Capture buffer | Records link events for debugging difficult drives |

A Linux driver on the RK3568 exposes the FPGA to `drd`, and `libdr` gets a matching backend. Starting points for the SATA core are LiteSATA (its link and transport layers are vendor-neutral; its PHY layer is Xilinx-only) and an open-source SATA core for Stratix V published in 2016. Closed or encrypted IP cores are not used. The design is simulated in CI with open-source simulators against a SATA device model, and built with Quartus Prime.

**Phase 0 checks for the USB side:** the mass storage gadget backed by a ublk device, disks larger than 2 TiB, 4K-sector patients, write-protect reporting, and behaviour on Windows, Linux and macOS when a read returns an error. Fallback if the kernel gadget is too limiting: implement mass storage ourselves in userspace through FunctionFS.

**Supervisor firmware.** Bare-metal C. Rail sequencing, current limits, fast fault shutdown, fan, buttons, LEDs, write-arm state, watchdog. Simple UART protocol to the Linux module. Kept deliberately small so it can be reviewed and trusted.

**Control protocol (DRP).** One versioned protocol between PC and Stabilizer, the same over USB and TCP. Length-prefixed frames with CBOR payloads, request / response plus an event stream, capability negotiation at connect. Written down as a specification with test vectors before it is implemented.

## 6. PC software (C)

```
              +------------------------------------------+
              |  DR Studio (GTK 4 desktop app, C)        |
              +------------------------------------------+
              |  drctl (command-line tool, C)            |
              +------------------------------------------+
              |  libdr (core library, C11)               |
              |  engine, map, image, fs, vendor plugins  |
              +------------------------------------------+
   backends:  | Stabilizer (DRP over USB / TCP)          |
              | Linux SG_IO / NVMe / MMC                 |
              | Windows SPTI / ATA pass-through          |
              | Simulator (virtual failing drives)       |
              +------------------------------------------+
```

- **Language and build.** C11, CMake, GCC / Clang, Windows builds through MSYS2. Warnings as errors, sanitizers, static analysis, fuzzing of every parser (disk images are untrusted input).
- **One engine everywhere.** The same `libdr` engine runs on the PC, inside the Stabilizer daemon, and in the browser simulator (compiled to WebAssembly).
- **GUI toolkit.** GTK 4, because it is a mature toolkit with a native C API. The GUI is a thin layer; all logic lives in `libdr` and is fully usable from `drctl`.
- **Scripting.** Embedded Lua for vendor-specific procedures, so support for a new drive family can be added without rebuilding the app.
- **Cases.** Every job is a case folder: device identity, SMART, image, map, logs, hashes, notes. This supports resuming and gives a chain of custody.
- **Installers.** Windows: installer with bundled runtime; the Stabilizer uses Microsoft OS 2.0 descriptors so no driver install is needed. Linux: AppImage and .deb with a udev rule. macOS: later.

### Feature tiers

| Tier | Features |
|------|----------|
| **1. Imaging (MVP)** | Device detection, identify and SMART, multi-pass imaging with sector map, pause and resume, power cycle on hang, virtual disk control, logs, simulator |
| **2. Extraction** | Desktop GUI, live sector map view, hex viewer, partition recovery, file-system-aware imaging, file browser and extraction from the image, carving, hashing and reports, restore image to a drive |
| **3. Program** | ATA tools (HPA / DCO, security unlock with a known password, error recovery settings), official firmware download, ROM read / write with parsers, UART terminal with script library, vendor firmware module backup and write for supported families, NVMe and eMMC admin tools, NAND dump analysis |

File system support in Tier 2 starts by calling proven open-source tools (The Sleuth Kit, TestDisk / PhotoRec) as separate programs, and adds our own parsers where targeted imaging needs them: NTFS, exFAT / FAT, ext4, then HFS+ and APFS. Every reused component's licence is checked first. The Sleuth Kit is mostly under the IBM and Common Public Licences, which are not GPL-compatible, so it must not be linked into `libdr` (to be confirmed per file before use).

### Testing

- Unit tests for all library code.
- **Simulator scenarios in CI**: deterministic virtual drives with bad sectors, a weak head, slow responses, hangs that need a power cycle. Every engine change is tested against them.
- **Patient emulator**: a second development board that pretends to be a failing drive, for hardware-in-the-loop tests without burning real drives.
- A shelf of real failed drives for acceptance tests before each release.

## 7. Website and DR Lab

**Stack.** Astro static site with MDX content and small interactive components, deployed to GitHub Pages or Cloudflare Pages. No backend needed at first; lab progress is stored in the browser. Built to be translated.

**Site map.** Home · Stabilizer and adapters (with a compatibility matrix) · Software (downloads, changelog) · **Lab** · Docs (user manual, build guides, specifications) · Updates · Community · Legal and safety.

### DR Lab design

**Entry point: the triage wizard.** "What device? What happened? What does it do now?" The answers lead to a risk level, an immediate do / do-not list, and the right lab. Dangerous situations get a clear stop sign first (for example: clicking drive, power it off now).

**Lab format.** Every lab has: goal, risk rating, time, equipment checklist, prerequisites, then numbered steps. Steps can contain checklists, "what do you see?" branches, annotated photos and pinout diagrams, terminal replays, simulator tasks, a knowledge check, and "stop and escalate" gates. Each lab shows a review status; nothing is marked verified until an experienced technician has checked it.

**In-browser simulator.** The real imaging engine compiled to WebAssembly, running against virtual failing drives. Learners choose settings, watch the sector map fill, and see how aggressive retries "damage" a simulated drive. They can practise before touching real hardware.

**Curriculum.**

| Track | Content |
|-------|---------|
| T0 Foundations | How HDD, SSD and flash store data; firmware and translators; the safety rules; setting up a bench; ESD; legal basics |
| T1 Logical recovery | Deleted files, formatted or lost partitions, corrupted file systems, carving; why TRIM makes deleted SSD data unrecoverable |
| T2 Imaging unstable drives | Reading SMART, bad sectors, multi-pass imaging, timeouts, head maps, using the Stabilizer, when to stop |
| T3 HDD electronics | PCB diagnosis, TVS diodes and fuses, current draw analysis, ROM transfer for a PCB swap |
| T4 HDD firmware | Service area concepts, diagnostic terminal basics, common firmware faults, commands that destroy data |
| T5 HDD mechanics (awareness) | Stiction, head crash, motor failure; what a clean room recovery involves; why not to attempt it at home |
| T6 SSD | Controller failures and safe modes, capacity and identity faults, TRIM and garbage collection, NVMe specifics |
| T7 USB sticks and memory cards | Logical faults, controller failure, connector repair, monoliths, chip-off overview |
| T8 Phones, eMMC, UFS | Why encryption changed everything, repair-to-recover, in-system access on older devices |
| T9 RAID and NAS | Image every member first, find the parameters, virtual rebuild; never rebuild on the originals |
| T10 Encryption | BitLocker, FileVault, LUKS, self-encrypting drives, encrypting USB bridges, Apple T2 and Apple Silicon |
| T11 Running a case | Intake, documentation, chain of custody, verifying and delivering data, wiping working copies |

The first modules to publish, alongside the software MVP: triage wizard, T0, T1 and the start of T2 with the simulator.

## 8. Repository layout

```
docs/                plan, specifications, decision records, prompts
hardware/
  base-unit/         KiCad project
  adapters/A01-.../  one KiCad project per adapter
  lib/               shared symbols and footprints
  mech/              enclosure and tray
gateware/            Cyclone V design: SATA host, PCIe endpoint, Lab Port, testbenches
firmware/
  supervisor/        MCU firmware (C)
  device/            Buildroot tree, kernel driver for the FPGA, the drd daemon
software/
  libdr/             core library
  drctl/             command-line tool
  studio/            GTK desktop app
  sim/               drive simulator
  packaging/         installers
web/                 Astro site and lab content
tests/               integration tests, scenarios, hardware-in-the-loop
tools/               scripts and CI helpers
```

## 9. Roadmap

Effort figures are rough estimates for a team of one to three people and will change.

| Phase | Goal | Main deliverables | Done when | Rough effort |
|-------|------|-------------------|-----------|--------------|
| **0 Foundations** | Remove the biggest unknowns | Repo scaffold, licence files and CI; map format and DRP specifications; feasibility tests on an RK3568 board and a Cyclone V transceiver board (section 4.1) | A virtual disk served over USB from the RK3568 board reads on Windows and Linux; a Cyclone V transceiver brings up a SATA link and reads a drive's identity | 1 to 2 months |
| **1 Software MVP** | Useful without any hardware | `libdr` engine, `drctl`, simulator; website skeleton; triage wizard and first labs | `drctl` images a real failing drive on Linux with a resumable map; simulator scenarios pass in CI; site is live | 2 to 3 months |
| **2 Stabilizer prototype** | Prove the stabilizer concept | RK3568 board plus power-switch board, native SATA port; `drd`; virtual disk; DRP remote backend | A drive with bad sectors appears on the PC as a read-only disk; no read exceeds the set limit; unplugging the PC does not disturb the patient; image and map survive power loss | 2 to 3 months |
| **2F FPGA drive interface** | Full low-level control of the patient | SATA host core, hardware write blocker, PCIe endpoint, Linux driver, `libdr` backend, on development boards | The Phase 2 tests pass through the FPGA port; time to regain control of a hung drive is measured and far below the native port; blocked write commands are proven never to reach the drive | 3 to 5 months, in parallel |
| **3 Base unit rev A** | Real hardware | Base unit PCB with both chips, supervisor firmware, enclosure; adapters A01 to A05 | Rev A passes the bring-up and test procedure and runs the Phase 2 and 2F tests | 3 to 4 months, including fabrication |
| **4 DR Studio v1** | Usable by non-experts | GTK app, installers, file extraction, reports; lab tracks T2 and T3 | A new user installs the app, images a drive and extracts files following only the lab | 3 to 4 months |
| **5 Flash and SSD** | Beyond hard drives | Adapters A06 to A10; NVMe, SD / eMMC and USB patient support; lab tracks T6 to T8 | Each adapter images its device class through the same workflow | 3 to 5 months |
| **6 Firmware and chip-off** | Expert tools | Program-mode tools, vendor plugins, native IDE and NAND functions in the FPGA, NAND reader A11 / A12 and dump analysis | Per-plugin acceptance on named drive families | Ongoing |
| **7 Product** | Ready to sell or distribute | Compliance (CE / FCC, RoHS), manufacturing files, support docs, optional A14 / A15 | Certified, documented, repeatable build | 3 to 6 months |

Phases 1, 2 and 2F can overlap. If Phase 2F is not finished when the base unit is designed, rev A still carries the FPGA and works through the native SATA port until the gateware is ready. Website and lab content grow in every phase. A usable Stabilizer with desktop software and a published lab (Phases 0 to 4) is roughly 10 to 16 months of work for a small team.

## 10. Main risks

| Risk | Response |
|------|----------|
| The scope is very large | Strict phases; imaging first, because it solves most recoverable cases without firmware tools |
| A standard SATA controller does not give enough low-level control for some faults | The Cyclone V drive interface with our own SATA core (Phase 2F) |
| Writing the SATA host core takes longer than planned | Prove the link in Phase 0; start from existing open cores; the RK3568 native port keeps the Stabilizer working without it |
| Cyclone V cannot run SATA at 6 Gbps | Drives fall back to 3 Gbps; the target port stays on the RK3568 at 6 Gbps |
| Host operating systems behave badly with a disk that returns errors | Bounded response times, write-protect reporting, tested on each OS in Phase 0 |
| High-speed PCB with two BGA devices needs several revisions | Development-board prototypes first, budget for respins |
| Vendor firmware knowledge is proprietary and ports are locked on new drives | Plugin model, named supported families only, honest "not supported" messages |
| Encrypted or TRIMmed data cannot be recovered | Triage and labs set expectations early |
| A user damages a drive following our instructions | Safety rules, stop gates, technician review of lab content, clear disclaimers |
| A chosen module or chip goes out of stock | Board support kept separate from the daemon; keep a second candidate |
| Licence conflicts when reusing open-source code | Licences are decided (section 12); every reused component is checked against them |

## 11. Legal and safety

- Recover data only with the owner's authorisation. Password and security features are for owners and authorised technicians.
- If recovery is offered as a service, data protection law applies to customer data (in the EU, GDPR): access control, retention limits, secure wiping.
- Selling hardware requires electrical safety and EMC compliance (CE / UKCA / FCC), RoHS and WEEE.
- Vendor-specific commands come from public documentation and interoperability research; do not copy proprietary tools, firmware or user interfaces. Get legal advice for your jurisdiction before distributing vendor firmware tools.
- Electrical safety on the bench: ESD protection, no hot-plugging of unprotected boards, current-limited supplies.

## 12. Decisions

### Decided

| Decision | Outcome |
|----------|---------|
| Licence | Open source. Software: GPL-2.0-or-later, so it can be combined with ddrescue and OpenSuperClone code. Hardware and gateware: CERN-OHL-S-2.0. Lab content and documentation: CC BY-SA 4.0. |
| Electronics engineer | The project owner designs and reviews all boards. |
| Main chips | RK3568 as the computer, Cyclone V as the drive interface (section 4.1). A Cyclone V alone was evaluated and not chosen because of its USB 2.0 link and weak image store. |

### Still open

Each has a default that the plan assumes until decided.

| # | Decision | Default assumed |
|---|----------|-----------------|
| 1 | Which Cyclone V device and development board | A GX or SX with at least three transceivers; to be confirmed against what is on hand |
| 2 | Main audience | Repair technicians and serious hobbyists first, professional labs later |
| 3 | PC operating systems | Windows and Linux first, macOS later |
| 4 | GUI toolkit | GTK 4 |
| 5 | Hardware description language for the gateware | SystemVerilog, unless the owner prefers VHDL |
| 6 | Website languages, and whether the site sells hardware | English first, translation-ready; no shop at launch |
| 7 | Who reviews lab content | A recovery technician, to be named |
| 8 | Product names and branding | Working names for now |

## 13. Prior art and references

- GNU ddrescue: imaging strategy and the map file format.
- OpenSuperClone (GPL fork of HDDSuperClone): virtual disk mode, direct AHCI access, scripting. <https://github.com/ISpillMyDrink/OpenSuperClone>
- DeepSpar USB Stabilizer, Guardonix and Disk Imager: commercial stabilizer and imaging hardware.
- ACE Lab PC-3000, MRT, Atola: professional firmware-level tools.
- smartmontools, hdparm, sg3_utils, nvme-cli, mmc-utils, flashrom: reference implementations of device commands.
- The Sleuth Kit, TestDisk and PhotoRec: file system analysis and carving.
- Linux ublk userspace block driver: <https://docs.kernel.org/block/ublk.html>
- RK3568 combo PHY driver discussion (lane sharing between SATA, USB 3 and PCIe): <https://lore.kernel.org/all/YcRbST7GDTNiZAkn@matsya/T/>
- Cyclone V transceiver handbook, SATA and SAS protocols (1.5 and 3.0 Gbps): <https://www.intel.com/content/www/us/en/docs/programmable/683586/current/sata-and-sas-protocols.html>
- Cyclone V SoC design guidelines, USB interface (USB 2.0 OTG only): <https://www.intel.com/content/www/us/en/docs/programmable/683360/18-0/usb-interface-design-guidelines.html>
- Cyclone V hard IP for PCI Express (root port and endpoint, up to ×4): <https://intel.com/content/www/us/en/docs/programmable/683694/current/gen1-gen2-and-gen-3-and-gen2-hard-ip.html>
- LiteSATA open-source SATA core (Xilinx PHYs): <https://pypi.org/project/litesata>
- Open-source SATA III core for Stratix V (2016 paper): <https://caslab.io/publications/guha2016design.pdf>

Prompts for building each part with an AI coding agent are in [`docs/prompts/`](prompts/README.md).
