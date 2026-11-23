# Prompt 04: DR Studio desktop app

Use with `00-master.md`. Requires a working `libdr` from `03-core-library-cli.md`. Read `docs/PLAN.md` section 6.

## Goal

Build DR Studio, the installable desktop application for Windows and Linux, written in C with GTK 4. A technician who has never used the command line can connect a patient (through the Stabilizer, or directly in software-only mode), see its condition, image it safely, browse the image and extract files.

DR Studio is a thin layer. Every operation it performs is a `libdr` call that `drctl` can also perform. If you find yourself writing recovery logic in the GUI, move it into the library.

## Screens

1. **Devices.** Connected Stabilizers and directly attached drives. For each: model, serial, capacity, connection, health summary. Clear label of the current protection level: "Stabilizer, write-blocked", or "Direct connection, reduced protection".
2. **Case.** Each job is a case folder (identity, SMART, image, map, logs, hashes, notes). Create, open, resume.
3. **Patient overview.** Identity, SMART attributes with plain-language meaning, and with a Stabilizer: live voltage and current per rail, the spin-up current curve, temperature, power controls.
4. **Imaging.** Start, pause, resume, stop. Presets (gentle, standard, fast) plus advanced settings. Live **sector map** as a custom-drawn grid that stays responsive for multi-terabyte drives, with zoom and a legend. Current position, speed, error counts, time estimates, event log.
5. **Virtual disk.** Turn the Stabilizer's virtual disk on or off, choose error behaviour (report error or fill pattern), see what the PC is requesting.
6. **Browse and extract.** Partitions found in the image; file tree; preview; select files and folders and extract to a destination, with a report of files affected by unread or failed sectors. Offer to prioritise imaging of selected folders.
7. **Hex view.** Sector viewer over the image, showing the map state of each sector.
8. **Log and report.** Full event log; export a case report (PDF or HTML) with device identity, what was done, map statistics and hashes.

Program-mode features (writing to drives, ROM tools, terminal, vendor plugins) are a later task. Leave a clear place for them in the navigation but do not build them now.

## Behaviour that matters

- The interface never freezes. Device and engine work runs off the main thread; the UI receives events.
- A dying drive is the normal case. Errors, hangs, disconnects and power cycles are shown calmly as states with a suggested next action, not as crash dialogs.
- Destructive or risky actions (anything that writes, direct mode on a failing drive, choosing a destination on the patient itself) are blocked or need explicit confirmation that names the consequence.
- The app must refuse to save an image or extracted files onto the patient.
- All user-visible strings go through gettext. Plain language; technical values are available but not required reading.
- Keyboard accessible, readable at high DPI, light and dark themes.

## Backends

- **Stabilizer** over the DRP protocol (USB and TCP) once `05-device-daemon.md` provides it. Until then, develop against the simulator backend and a mock DRP server.
- **Linux direct** (SG_IO) from prompt 03.
- **Windows direct**: add a `libdr` backend using SCSI and ATA pass-through. Requires administrator rights; explain this to the user when needed.

## Packaging

- Windows: an installer that bundles the GTK runtime and needs no separate driver installation for the Stabilizer.
- Linux: AppImage and a .deb, with a udev rule that grants access to the Stabilizer and to drives for a dedicated group.
- CI builds the installers for every tagged release.

## Done when

- Every screen above works against the simulator, including the dying-drive scenarios, without freezing.
- Imaging a simulated 4 TB drive keeps the sector map view smooth.
- A full flow works on a real drive on Linux and on Windows: create case, image, browse, extract, export report. The report states which drives were used.
- File browsing covers NTFS, exFAT / FAT and ext4 at minimum, and clearly marks files that overlap unread or failed sectors.
- Installers install and uninstall cleanly on a fresh Windows 11 and a current Ubuntu LTS.
- UI tests cover the main flows against the simulator.

## Start with

The application shell, the Devices and Case screens against the simulator backend, and the sector map widget. The map widget is the hardest custom component; prove its performance early.
