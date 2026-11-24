# Prompt 05: Stabilizer device software

Use with `00-master.md`. Requires `libdr` from `03-core-library-cli.md`. Read `docs/PLAN.md` sections 3, 4.1 and 5.

## Goal

Make an RK3568 development board behave as the Stabilizer: a failing SATA drive connected to the board appears on a PC, over USB, as a clean read-only external disk whose reads never hang. Behind that disk, the board images the patient on demand into a local image with a sector map.

This is the core idea of the whole project, so the work is split into a feasibility part that removes the unknowns, and a build part.

## What you need to know

- The board runs embedded Linux. In this task the patient connects to the RK3568's native SATA port. The board's USB 3.0 port works in device mode and connects to the PC.
- A second patient port, on a Cyclone V FPGA with our own SATA core (`09-fpga-gateware.md`), comes later and plugs in through the same `libdr` backend interface. Keep everything above the backend unaware of which port is in use.
- The virtual disk is a **ublk** userspace block device served by our daemon `drd`. The Linux **USB mass storage gadget** uses that block device as its backing store. So every read from the PC arrives in `drd`, which answers from the image or reads the patient through the `libdr` engine with a time limit.
- A second USB interface (vendor-specific, bound automatically to WinUSB on Windows through Microsoft OS 2.0 descriptors) carries the control protocol, DRP. The same protocol also runs over TCP.
- In this prototype, patient power is switched by a simple external power-switch board on GPIO. The real supervisor MCU comes later (`06-supervisor-firmware.md`); hide power control behind an interface so it can be swapped.

## Part A: feasibility (Phase 0)

Answer each question with a small experiment, and write the method, result and evidence in `docs/specs/feasibility.md`. If an answer is "no", propose the alternative.

1. Does the mass storage gadget work with a ublk device as backing store on this board and kernel, at USB 3.0 speed? What throughput?
2. Does it handle disks larger than 2 TiB and 4K-sector disks correctly?
3. Does the PC see the disk as write-protected, and do Windows, Linux and macOS leave it unmodified?
4. When `drd` returns an I/O error for a read, what does each host OS see and do? Does it stay responsive? What is the longest response time each OS tolerates before it resets or drops the device?
5. With standard Linux libata and ATA pass-through, what is the worst-case time from "drive stops responding" to "we have control again"? This number is the baseline the FPGA port must beat, and it decides how usable the native port is for failing drives in the meantime.
6. Can the kernel be configured so that it never scans partitions on, mounts, or otherwise touches the patient by itself?
7. Can the board provide PCIe to an external endpoint (the future FPGA link) and to the NVMe image store at the same time? Can the two PCIe 3.0 lanes be split to add a patient NVMe port?

If the kernel gadget proves too limiting in questions 1 to 4, the fallback is to implement USB mass storage in userspace through FunctionFS.

## Part B: build (Phase 2)

- **Linux image** (`firmware/device/`): Buildroot external tree, reproducible build, read-only root file system, the kernel options found in Part A. The patient port is never auto-probed.
- **`drd` daemon** in C, built on `libdr`:
  - Patient backend for the SATA port, with the level of control decided in Part A.
  - Store: image and journaled map on the board's NVMe. Consistent after sudden power loss.
  - Virtual disk: the ublk server implementing image-on-demand. Reads of imaged sectors never touch the patient. Unread sectors are read with the configured limit. Failed reads answer quickly with an error or a fill pattern, as configured. Host writes are refused.
  - Optional background imaging that yields to host requests.
  - Gadget manager: sets up the composite USB device; presents the patient's capacity and sector size; removes the disk cleanly when the patient is powered down.
  - Power interface: on, off, cycle, with the escalation ladder from `libdr`.
  - DRP server over USB and TCP.
  - Black-box log of every command, timing and event.
- **DRP specification** (`docs/specs/drp.md`) written before the implementation: framing, versioning, capability negotiation, requests, events, error model, test vectors. Length-prefixed frames with CBOR payloads.
- **`libdr` remote backend** so that `drctl` on the PC can control the Stabilizer.

## Constraints

- All patient safety rules in `00-master.md` apply, especially: bounded time for every patient command, fail safe on disconnect or crash, and no write path. `drd` contains no code that writes to the patient in this task.
- If `drd` crashes or is restarted, the patient is left idle or powered down and the image and map remain consistent; the PC sees the disk disappear, not hang.
- A stalled patient must never stall the USB side. The ublk server always answers within the limit.
- Kernel changes are kept as small as possible and documented; prefer configuration and userspace.

## Done when

- `docs/specs/feasibility.md` answers all seven questions with evidence.
- With a known-bad drive (bad sectors) as patient: Windows and Linux both see a read-only external disk; a recovery tool on the PC can scan it without the OS hanging; the longest observed read is within the configured limit; the black-box log proves it.
- Reading the same region twice from the PC touches the patient only once.
- Unplugging the PC cable mid-read, killing `drd`, and cutting board power each leave a consistent image and map; resuming loses nothing completed.
- `drctl` on the PC can connect over DRP, show status and the live map, change the read limit, and power-cycle the patient.
- The report lists the drives used, measurements taken, and anything not tested.

## Start with

Part A questions 1 and 4. They decide whether the simplest architecture works. Use a healthy drive and the simulator backend to inject errors before connecting any failing drive.
