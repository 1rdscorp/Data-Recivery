# Prompt 03: Core library, command-line tool and simulator

Use with `00-master.md`. Read `docs/PLAN.md` sections 3 and 6.

## Goal

Build the heart of the project: `libdr`, a C11 library containing the drive imaging engine; `drctl`, a command-line tool over it; and a drive simulator for testing. At the end, a user on Linux can image a real failing SATA or USB drive with `drctl`, stop, resume, and get an image file plus a sector map, with no Stabilizer hardware.

This library is later reused unchanged inside the Stabilizer daemon, the desktop app and (compiled to WebAssembly) the website's lab simulator. Design it for that: no global state, no direct file or device access outside the backend and storage interfaces, no dependency on a GUI or an operating system outside the platform layer.

## What to build

### 1. Specifications first (`docs/specs/`)

- **Sector map format.** States per block range: unread, good, failed, skipped, non-trimmed, non-scraped, plus timing and pass information. Journaled so that a crash or power loss never loses completed work. Import and export of the GNU ddrescue map file format.
- **Backend interface.** The small set of operations the engine needs from a device: identify, read with timeout, abort, soft reset, hard reset, power cycle (optional capability), set drive parameters, get status and timing.

### 2. `libdr` modules (`software/libdr/`)

- `map`: the sector map, its journal, and ddrescue import / export.
- `image`: raw image store with sparse files, flush policy, optional SHA-256 over completed ranges.
- `engine`: the imaging strategy. Passes: forward copy with skip-on-error (growing skip size), reverse pass, trimming, scraping, limited retries. Configurable read size, timeouts and limits. Pause, resume, cancel. Progress and event callbacks.
- `ladder`: the escalation ladder on timeout or error: abort, soft reset, hard reset, power cycle if the backend can; with counters and limits that end the job instead of looping.
- `ata`, `scsi`: command builders and response parsers for IDENTIFY, SMART, read commands (non-queued, 48-bit), SCT error recovery control, read look-ahead and cache settings, and SCSI / ATA pass-through.
- `backend/sim`: the simulator (below).
- `backend/linux_sg`: Linux SG_IO backend for SATA (ATA pass-through) and USB / SCSI devices. Opens devices read-only. Refuses to open a device with mounted partitions unless forced.
- `log`: structured event log of every command, result and timing.

Write paths are out of scope for this prompt. The library does not contain any function that writes to a patient yet.

### 3. Simulator (`software/sim/` and `backend/sim`)

Virtual drives described by scenario files: size, sector size, head and zone layout, bad sector lists, a weak head whose error rate grows with use, slow sectors, commands that hang until reset or power cycle, a drive that degrades when hammered with retries. Fully deterministic from a seed, with a virtual clock so tests run fast. The simulator reports the stress the engine put on the drive, so tests can assert that the engine is gentle, not only that it recovers data.

Provide at least these scenarios: healthy drive; scattered bad sectors; one dead head; slow-responding drive; drive that hangs and needs a power cycle; drive that dies after a read budget is used up.

### 4. `drctl` (`software/drctl/`)

Commands: list devices; show identity and SMART; `image` (start or resume, with all engine options); show a map summary and a text rendering of the map; export and import ddrescue maps; run against a simulator scenario. Clear exit codes and a `--json` output mode for scripting. Safe defaults: the user should get a gentle imaging run without knowing any options.

## Constraints

- C11, CMake, warnings as errors, sanitizers in tests, per `00-master.md`.
- Dependencies: the C standard library and the platform API. A small vendored SHA-256 and a small test framework are fine. Ask before adding anything else.
- Portable core: everything except `backend/linux_sg` must compile on Linux, Windows (MSYS2) and with Emscripten. CI builds at least Linux GCC, Linux Clang and Windows.
- Every parser of external data (maps, ddrescue files, device responses, scenario files) has a fuzz target.
- GPL-licensed projects (ddrescue, OpenSuperClone) may be studied for behaviour. Whether their code may be copied depends on the licence decision in `docs/PLAN.md` section 12; until that is decided, write original code.

## Done when

- Unit tests cover map, image, ladder and parsers; all pass under sanitizers.
- Each simulator scenario has an engine test asserting recovered data, map correctness, bounded time and bounded drive stress.
- Killing `drctl` at random points during a simulated run, then resuming, never loses a completed block and never re-reads a block marked good. This is an automated test.
- A map exported to ddrescue format is accepted by GNU ddrescue, and the reverse.
- `drctl image` has been run against at least one real drive on Linux; the report says which drive, what happened, and what was not tested.
- `software/README.md` covers building, testing and basic use.

## Start with

The sector map specification and the backend interface. Then `map`, the simulator with the "healthy" and "scattered bad sectors" scenarios, and a minimal single-pass engine, so there is a working end-to-end slice early. Grow the engine and the scenarios together from there.
