# Prompts

Ready-to-use prompts for building each part of the project with an AI coding agent. They follow [`docs/PLAN.md`](../PLAN.md).

## How to use them

1. Always give the agent **`00-master.md`** first. It holds the project context and the rules every part must follow.
2. Add **one** workstream prompt. One prompt, one session, one goal.
3. When the agent finishes, check its report against the "Done when" list in the prompt before starting the next one.

In an agent that can read this repository (for example Claude Code), you do not need to paste anything. Say:

> Read `docs/prompts/00-master.md` and `docs/prompts/03-core-library-cli.md`, then do the work described.

In a tool that cannot read the repository, paste the master prompt followed by the workstream prompt. `01-website.md` also works alone in website builders.

## The prompts

| File | Builds | Plan phase |
|------|--------|-----------|
| [`00-master.md`](00-master.md) | Project context and rules (use with every other prompt) | all |
| [`01-website.md`](01-website.md) | Project website | 1 |
| [`02-lab.md`](02-lab.md) | Interactive lab engine, triage wizard and first lab modules | 1 |
| [`03-core-library-cli.md`](03-core-library-cli.md) | `libdr` engine, `drctl` command-line tool, drive simulator | 1 |
| [`04-desktop-app.md`](04-desktop-app.md) | DR Studio desktop app and installers | 4 |
| [`05-device-daemon.md`](05-device-daemon.md) | Stabilizer Linux image, `drd` daemon, virtual disk over USB | 0 and 2 |
| [`06-supervisor-firmware.md`](06-supervisor-firmware.md) | Supervisor microcontroller firmware | 2 and 3 |
| [`07-base-unit-hardware.md`](07-base-unit-hardware.md) | Stabilizer base unit hardware design | 3 |
| [`08-adapter-board.md`](08-adapter-board.md) | Any adapter board (template: fill in the adapter ID) | 3, 5, 6 |
| [`09-fpga-gateware.md`](09-fpga-gateware.md) | Cyclone V drive interface: SATA host core, hardware write blocker, PCIe endpoint, Linux driver (template: fill in the device and board) | 0 and 2F |

## Suggested order

`03` and `01` can start now and run in parallel. `02` needs the site from `01` and, for simulator tasks, the engine from `03`. `05` and `09` each start with a Phase 0 feasibility part as soon as their development board is on the bench; they are independent of each other. `06`, `07` and `08` follow once the prototype in `05` works and the SATA link in `09` is proven. `04` builds on a stable `03`.

## Two limits to keep in mind

- **Hardware and gateware.** An AI agent can write requirements, choose parts, draft schematics, pin maps, BOMs, HDL and test plans. It cannot validate a multi-gigabit PCB layout or prove a transceiver link without the hardware. The project owner, as the electronics engineer, reviews every board before fabrication and runs the hardware tests.
- **Lab content.** An AI agent can draft lab modules well, but a wrong instruction can destroy someone's data. Every module stays marked "draft" until an experienced recovery technician has reviewed it.
