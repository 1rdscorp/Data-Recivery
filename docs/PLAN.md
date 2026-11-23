# Data Recovery Platform — Master Plan

Draft v0.1 · 2026-11-23 · Working names (Stabilizer, DR Studio, DR Lab) can be changed at any time.

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
| 1 | **Stabilizer base unit** | Embedded Linux hardware with patient ports, switched and monitored power, and a USB link to the PC |
| 2 | **Adapter boards** | A family of small PCBs that connect each kind of device (SATA, IDE, M.2, NVMe, USB, SD, eMMC, NAND, ROM chips, diagnostic UART) |
| 3 | **Device firmware** | The recovery daemon running on the Stabilizer, plus a small supervisor microcontroller firmware |
| 4 | **PC software in C** | Core library, command-line tool and desktop app (DR Studio) for Windows and Linux, macOS later |
| 5 | **Website and DR Lab** | Project site, documentation, downloads, and the interactive step-by-step recovery lab |

The closest commercial products are the DeepSpar USB Stabilizer and Disk Imager, and at the high end ACE Lab PC-3000. The closest open-source software is GNU ddrescue and OpenSuperClone (the open-sourced HDDSuperClone). Our difference: open hardware and software in one system, a wide adapter family, and teaching built in.

### Non-goals

- Mechanical repair tooling (head swaps, platter work, clean room equipment). The lab explains these so people know when to stop and send a drive to a professional.
- Breaking encryption. BitLocker, FileVault, LUKS, self-encrypting drives and modern phones need the key. No tool changes that.
- Matching PC-3000 firmware repair for every drive family. Vendor firmware work is added family by family as plugins, late in the roadmap.

## 2. Patient safety rules

These rules apply to every component. They exist because the patient is usually the only copy of someone's data, and every operation on failing hardware can make it worse.

1. **Read-only by default.** Nothing writes to a patient unless a physical "write arm" switch on the Stabilizer is on *and* the user confirms in software.
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

**Compute module.** Proposed: a Rockchip RK3568 system-on-module. It has native SATA III, a USB 3.0 port that works in device mode, and PCIe, at low cost. Proposed use of its high-speed lanes:

| SoC resource | Use |
|--------------|-----|
| Combo PHY 0 as USB 3.0 OTG (device mode) | Link to the PC |
| Combo PHY 1 as SATA | Patient SATA port |
| Combo PHY 2 as SATA | Target SATA port (clone destination) |
| PCIe 3.0, 2 lanes | Internal NVMe image store, and a patient NVMe port |
| USB 2.0 host | Patient USB port (sticks, card readers, USB-native drives) |
| SD/MMC controller | Patient SD and eMMC adapters |
| Gigabit Ethernet | Network control, imaging to a NAS |
| UART, SPI, I2C, GPIO | Lab Port (section 4.3) |

The combo PHY sharing is confirmed by the mainline kernel driver. Three points are **not yet verified** and are Phase 0 tasks on a development board: splitting the two PCIe 3.0 lanes between two devices, USB 3.0 device-mode throughput and stability, and direct register-level control of the SATA controller. The patient USB port is USB 2.0 in revision A; this is acceptable because failing flash devices are read slowly anyway.

**Patient power.** This is the part that makes the unit a stabilizer and not just a bridge.

- Input: 12 V DC, at least 5 A.
- Patient rails: 12 V, 5 V, 3.3 V (for M.2 and mSATA), and two adjustable rails 1.2 V to 3.3 V (VCC and VCCQ for flash chips).
- Each rail has an electronic fuse with a programmable current limit and fast short-circuit trip, a current and voltage monitor sampled at 1 kHz or faster, and an active discharge circuit so a power cycle fully resets the drive.
- Candidate parts, to be checked against datasheets: TI TPS2595x / TPS25947 eFuses, TI INA228 monitors.

**Supervisor microcontroller.** A small MCU (STM32G0 class or RP2040) owns the rail enables, power sequencing, fan, buttons, LEDs, the write-arm switch and a watchdog over the Linux module. If Linux crashes, the supervisor still protects the patient.

**IO coprocessor.** A second MCU with programmable IO (candidate: RP2350B) or later a small FPGA drives the Lab Port: SPI, UART, JTAG/SWD and parallel NAND timing.

**Front panel.** Small display, a few buttons, status LEDs including a red WRITE ARMED light, a physical write-arm switch, and a patient power-off button.

**Protection and mechanics.** ESD protection on every patient-facing line, reverse-polarity protection, replaceable "port saver" cables on the patient connectors (they wear out), a fan aimed at the patient with a temperature probe, and a drive tray with vibration damping.

### 4.2 Adapter boards

| ID | Adapter | Devices | Method | Difficulty | Phase |
|----|---------|---------|--------|------------|-------|
| A01 | SATA port saver | 2.5" and 3.5" SATA HDD / SSD | Passive | Low | 3 |
| A02 | mSATA and M.2 SATA | SATA SSD modules | Passive, switched 3.3 V | Low | 3 |
| A03 | PATA / IDE | 40-pin and 44-pin IDE, CompactFlash, 1.8" ZIF / LIF | SATA-to-PATA bridge chip | Medium | 3 |
| A04 | UART terminal | HDD diagnostic ports (Seagate and others) | Lab Port, selectable 1.8 V / 3.3 V levels | Low | 3 |
| A05 | ROM programmer | SPI NOR and I2C EEPROM on drive PCBs, SOIC-8 clip and socket | Lab Port, 1.8 / 2.5 / 3.3 V | Low | 3 |
| A06 | SD / microSD / MMC | Memory cards | Native SD host, adjustable voltage and clock, 1-bit mode | Medium | 5 |
| A07 | eMMC socket and ISP | BGA eMMC chips, in-system flying leads | Native MMC host, adjustable VCC / VCCQ | Medium | 5 |
| A08 | M.2 NVMe and U.2 | NVMe SSD | PCIe lane, switched power, reset control | Medium to high | 5 |
| A09 | USB patient port | USB sticks, card readers, USB-native HDD | USB host with switched, monitored VBUS | Low to medium | 5 |
| A10 | Proprietary SSD passives | Apple 2013 to 2017 blade SSDs and similar | Passive to M.2 | Low | 5 |
| A11 | NAND reader | Chip-off: TSOP48, BGA152 / 132, LGA52 / 60 | Parallel NAND from coprocessor or FPGA | High | 6 |
| A12 | Monolith breakout | Monolithic microSD and USB sticks via test pads | NAND reader plus fine-pitch leads | High | 6 |
| A13 | JTAG / SWD probe | SSD controllers, drive MCUs (research) | Lab Port | Medium | 6 |
| A14 | SAS | SAS HDD | PCIe SAS controller | High | 7, optional |
| A15 | UFS | Phone storage | UFS host bridge | High | 7, optional |

Notes that keep expectations honest:

- Reading raw NAND (A11, A12) is the easy half. Turning the dump back into files needs ECC correction, descrambling and rebuilding the controller's block mapping, which differs per controller. That is a research-heavy software project of its own.
- Chip-off does not work on modern SSDs and phones, because the controller encrypts the NAND. For those, recovery means getting the original controller to work again.
- Seagate diagnostic ports use 1.8 V on some models and 3.3 V on others, and newer models lock the port. A04 must be level-selectable and the software must measure before driving.

### 4.3 Lab Port and adapter identification

High-speed interfaces (SATA, PCIe, USB) use their own standard connectors for signal quality. Everything slower goes through one common connector, the **Lab Port**:

- Adjustable IO voltage rail and switched 3.3 V / 5 V.
- About 24 level-shifted, bidirectional signals from the IO coprocessor.
- I2C for an **ID EEPROM on every adapter**: adapter type, revision, serial number, capabilities and voltage limits.

The software reads the EEPROM, configures itself, and refuses to apply a voltage the adapter does not allow. The exact connector and pin map are a Phase 3 design output.

### 4.4 PCB workflow

- KiCad, current stable release. One project folder per board, one shared symbol and footprint library.
- Base unit: 6 layers expected, controlled impedance (90 Ω USB, 100 Ω SATA, 85 Ω PCIe differential pairs). Adapters: 2 or 4 layers.
- CI runs ERC and DRC and produces schematic PDF, Gerbers, BOM, pick-and-place files and a 3D render for every change.
- Every board has test points, a version on the silkscreen, a bring-up checklist and a test procedure.
- Any board carrying multi-gigabit signals is reviewed by an experienced electronics engineer before fabrication. Budget for two or three revisions of the base unit.
- **Prototype first on an off-the-shelf RK3568 development board** plus a simple power-switch board. The custom base unit PCB comes only after the concept is proven.

## 5. Device firmware

**Linux image.** Buildroot, a recent long-term kernel, read-only root file system, signed A/B updates. The kernel must never touch the patient on its own: no partition scanning, no auto-mount, no file system drivers bound to patient ports.

**Recovery daemon `drd`** (C, built on the same core library as the PC software):

| Module | Job |
|--------|-----|
| Patient backends | SATA, USB / SCSI, NVMe, MMC, NAND, SPI |
| Engine | Imaging strategy, timeouts, escalation ladder |
| Store | Image file and journaled sector map, safe against power loss |
| Virtual disk | A ublk userspace block device that implements image-on-demand |
| USB gadget | Composite device: mass storage backed by the virtual disk, plus a vendor interface for control |
| Power manager | Talks to the supervisor MCU |
| Protocol server | The control protocol (DRP) over USB and TCP |
| Black box | Full command and event log for every job |

**SATA control.** Three options, in order of control and effort:

1. Standard Linux libata with ATA pass-through and tuned settings. Quick to start, but the kernel's own error handling can still take many seconds.
2. A small custom kernel driver that owns the patient SATA port and gives the daemon exact control of timeouts and resets. **Recommended target.**
3. A full userspace AHCI driver. OpenSuperClone shows this is possible on PCs.

Start with option 1 in Phase 2 to prove the whole chain, then move to option 2.

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

File system support in Tier 2 starts by reusing proven open-source code (The Sleuth Kit, TestDisk / PhotoRec) and adds our own parsers where targeted imaging needs them: NTFS, exFAT / FAT, ext4, then HFS+ and APFS.

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
firmware/
  supervisor/        MCU firmware (C)
  device/            Buildroot tree and the drd daemon
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
| **0 Foundations** | Remove the biggest unknowns | Repo scaffold and CI; map format and DRP specifications; feasibility tests on an RK3568 development board; decisions from section 12 | A virtual disk served over USB from the dev board reads on Windows and Linux; SATA command timeout and reset control demonstrated | 1 month |
| **1 Software MVP** | Useful without any hardware | `libdr` engine, `drctl`, simulator; website skeleton; triage wizard and first labs | `drctl` images a real failing drive on Linux with a resumable map; simulator scenarios pass in CI; site is live | 2 to 3 months |
| **2 Stabilizer prototype** | Prove the stabilizer concept | Dev board plus power-switch board; `drd`; virtual disk; DRP remote backend | A drive with bad sectors appears on the PC as a read-only disk; no read exceeds the set limit; unplugging the PC does not disturb the patient; image and map survive power loss | 2 to 3 months |
| **3 Base unit rev A** | Real hardware | Base unit PCB, supervisor firmware, enclosure; adapters A01 to A05 | Rev A passes the bring-up and test procedure and runs the Phase 2 tests | 3 to 4 months, including fabrication |
| **4 DR Studio v1** | Usable by non-experts | GTK app, installers, file extraction, reports; lab tracks T2 and T3 | A new user installs the app, images a drive and extracts files following only the lab | 3 to 4 months |
| **5 Flash and SSD** | Beyond hard drives | Adapters A06 to A10; NVMe, SD / eMMC and USB patient support; lab tracks T6 to T8 | Each adapter images its device class through the same workflow | 3 to 5 months |
| **6 Firmware and chip-off** | Expert tools | Program-mode tools, vendor plugins, NAND reader A11 / A12 and dump analysis, FPGA SATA research | Per-plugin acceptance on named drive families | Ongoing |
| **7 Product** | Ready to sell or distribute | Compliance (CE / FCC, RoHS), manufacturing files, support docs, optional A14 / A15 | Certified, documented, repeatable build | 3 to 6 months |

Phases 1 and 2 can overlap. Website and lab content grow in every phase. A usable Stabilizer with desktop software and a published lab (Phases 0 to 4) is roughly 9 to 14 months of work for a small team.

## 10. Main risks

| Risk | Response |
|------|----------|
| The scope is very large | Strict phases; imaging first, because it solves most recoverable cases without firmware tools |
| A standard SATA controller does not give enough low-level control for some faults | Custom kernel driver first; FPGA-based SATA host as a later research track |
| Host operating systems behave badly with a disk that returns errors | Bounded response times, write-protect reporting, tested on each OS in Phase 0 |
| High-speed PCB needs several revisions | Dev-board prototype first, engineer review, budget for respins |
| Vendor firmware knowledge is proprietary and ports are locked on new drives | Plugin model, named supported families only, honest "not supported" messages |
| Encrypted or TRIMmed data cannot be recovered | Triage and labs set expectations early |
| A user damages a drive following our instructions | Safety rules, stop gates, technician review of lab content, clear disclaimers |
| The chosen compute module goes out of stock | Board support kept separate from the daemon; keep a second candidate |
| Licence conflicts when reusing open-source code | Decide the licence in Phase 0 (section 12) |

## 11. Legal and safety

- Recover data only with the owner's authorisation. Password and security features are for owners and authorised technicians.
- If recovery is offered as a service, data protection law applies to customer data (in the EU, GDPR): access control, retention limits, secure wiping.
- Selling hardware requires electrical safety and EMC compliance (CE / UKCA / FCC), RoHS and WEEE.
- Vendor-specific commands come from public documentation and interoperability research; do not copy proprietary tools, firmware or user interfaces. Get legal advice for your jurisdiction before distributing vendor firmware tools.
- Electrical safety on the bench: ESD protection, no hot-plugging of unprotected boards, current-limited supplies.

## 12. Decisions needed

Each has a default that the plan assumes until you say otherwise.

| # | Decision | Default assumed |
|---|----------|-----------------|
| 1 | Open source or closed / commercial | Open: GPL for software (allows reuse of ddrescue and OpenSuperClone ideas and code), CERN-OHL for hardware, CC BY-SA for lab content. A closed product cannot reuse GPL code. |
| 2 | Main audience | Repair technicians and serious hobbyists first, professional labs later |
| 3 | PC operating systems | Windows and Linux first, macOS later |
| 4 | Compute module | RK3568, confirmed or replaced after Phase 0 |
| 5 | GUI toolkit | GTK 4 |
| 6 | Website languages, and whether the site sells hardware | English first, translation-ready; no shop at launch |
| 7 | Who reviews PCB layouts and lab content | To be named: one electronics engineer, one recovery technician |
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

Prompts for building each part with an AI coding agent are in [`docs/prompts/`](prompts/README.md).
