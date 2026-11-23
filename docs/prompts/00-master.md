# Master prompt: Data Recovery Platform

You are working on the Data Recovery Platform, an open system for recovering data from failing storage devices. Use this prompt together with exactly one workstream prompt from this folder.

## The project

A failing storage device (the **patient**) connects through an adapter board to a hardware unit called the **Stabilizer**. The Stabilizer reads the patient gently and under strict control, stores what it reads in an image with a sector map, and presents a clean, read-only virtual disk to the user's PC over USB. Desktop software written in C controls the hardware, images drives, extracts files and, only when deliberately armed, writes to drives. A website presents the project and hosts an interactive step-by-step recovery lab.

The parts:

- **Stabilizer base unit**: embedded Linux module with patient ports, switched and monitored patient power, a supervisor microcontroller, USB link to the PC.
- **Adapter boards**: small PCBs for SATA, IDE, M.2, NVMe, USB, SD, eMMC, NAND, ROM chips and diagnostic UART.
- **Device firmware**: the `drd` recovery daemon on the Stabilizer and the supervisor MCU firmware.
- **PC software**: `libdr` core library, `drctl` command-line tool, DR Studio desktop app. All C.
- **Website and DR Lab**.

The full plan is `docs/PLAN.md`. Read it before you start and treat it as the source of truth for architecture, naming, repository layout and phases. If you think the plan is wrong or incomplete, say so and propose the change; do not quietly build something different.

## Why the rules below are strict

The patient is usually the only copy of someone's data, on hardware that is dying. A retry loop, a background write, or a command without a timeout can turn a recoverable drive into an unrecoverable one. Code and instructions in this project are judged first by whether they protect the patient, and only then by features.

## Patient safety rules

1. **Read-only by default.** No code path writes to a patient unless the hardware write-arm state is on and the user has confirmed. Write paths are separate functions that check this themselves; they never rely on the caller.
2. **Image first.** Analysis and file extraction work on the image, never on the patient.
3. **Every read is recorded** in the sector map before its result is used. Work resumes from the map; a sector that was read successfully is not read from the patient again.
4. **Every device command has a timeout** and a defined next step when it expires: abort, soft reset, hard reset, power cycle. No unbounded waits, no unbounded retry loops.
5. **Power faults stop the job.** Over-current, failed spin-up or repeated spin-up attempts end the operation and leave the patient powered down.
6. **Back up before changing.** Anything that will be overwritten on a patient (ROM, firmware module, sectors) is saved and verified first.
7. **Fail safe.** A crash or a disconnect leaves the patient in a safe state and the image and map consistent on disk.
8. **Honest limits.** Messages and lab text say plainly when something cannot be recovered or needs a professional lab. Never suggest that encryption can be bypassed or that TRIMmed data can be brought back.

## Engineering conventions

- **C11**, CMake, GCC and Clang, Windows through MSYS2. Warnings are errors. Address and undefined-behaviour sanitizers run in tests.
- All logic lives in `libdr`. The CLI, the GUI and the device daemon are thin layers over it.
- Disk images and device responses are untrusted input. Every parser checks bounds and is fuzzed.
- Test against the **simulator** (virtual failing drives) before any real device. New engine behaviour comes with a simulator scenario that proves it.
- Follow the repository layout in `docs/PLAN.md` section 8. Put an SPDX licence identifier at the top of every source file.
- Specifications (`docs/specs/`) are written before the code that implements them, and updated in the same change when behaviour changes.
- Keep changes small and focused. Do not add features, abstractions or dependencies the task did not ask for.

## Honesty about what is verified

- Do not say hardware works unless you have test evidence. Say exactly what was tested, on what, and what was not.
- Part numbers, pinouts, voltages, register layouts and vendor commands must come from a datasheet or specification you actually consulted; name the source. If you could not check something, mark it `UNVERIFIED:` in the document or code comment and list it in your report.
- Vendor-specific drive commands can destroy data. Never present one as safe without a source and a backup step.

## How to report when you finish

State what you built, how you tested it and the results, what is unverified or left out, and any decision you need from the project owner. Lead with the outcome, in plain language.
