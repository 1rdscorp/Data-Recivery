# Prompt 02: Interactive recovery lab (DR Lab)

Use with `00-master.md`. Builds on the website from `01-website.md`. Read `docs/PLAN.md` section 7 for the lab design and the full curriculum.

## Goal

Build the engine for a guided, interactive, step-by-step lab that teaches people to recover data from storage devices, then write the triage wizard and the first lab modules with it.

The lab has two kinds of reader: a person with a failed device who needs the right procedure now, and a learner working through the tracks to build skill. Both must be able to tell at every step what to do, what they should see, and when to stop.

## Part A: the lab engine

Lab modules are MDX files in a content collection. Build the components and schema that make them interactive:

- **Front matter schema:** id, track, title, summary, risk rating (low / medium / high / professional only), estimated time, equipment list, prerequisites, review status (`draft` or `reviewed`, with reviewer and date).
- **Components:**
  - `Step`: a numbered step with a completion checkbox.
  - `Checklist`: equipment or preparation items.
  - `Decision`: "What do you see?" with options that branch to different steps or to another lab.
  - `Stop`: a stop-and-escalate gate. Explains why continuing is dangerous and what to do instead.
  - `Warning` and `Note`.
  - `Quiz`: a short knowledge check with explanations for wrong answers.
  - `Pinout` and `Figure`: annotated SVG diagrams and photos with labelled hotspots.
  - `TerminalReplay`: a recorded terminal session the learner steps through.
  - `Sim`: an embedded simulator task (see Part C).
- **Progress:** saved in the browser (local storage), per lab and per step, with resume and reset. No accounts.
- **Print view:** every lab prints as a clean checklist to use at the bench.
- **Validation in CI:** schema check, every `Decision` target exists, no lab without a risk rating, no unreviewed lab displayed without its "draft, not yet reviewed by a technician" banner.

All reading content must work without JavaScript; interactivity is an enhancement.

## Part B: the triage wizard

The lab's front door. A short question flow:

1. What kind of device? (hard drive, SSD, USB stick, memory card, phone, NAS or RAID, other)
2. What happened? (deleted or formatted, not detected, clicking or other noise, very slow or freezing, dropped, liquid, fire, power surge, asks to format, wrong capacity, encrypted or password)
3. What does it do now when powered?

The result page gives a risk level, an immediate do and do-not list, what is realistically recoverable, and a link to the right lab, or a clear recommendation to use a professional lab. Dangerous combinations (for example: hard drive + clicking) show the stop instruction first, before any further questions.

Define the wizard as a data file (a decision graph), not as code, so that it can be reviewed and translated.

## Part C: simulator tasks

The imaging engine (`libdr`, built in prompt 03) compiles to WebAssembly and runs against virtual failing drives. The `Sim` component loads a scenario, lets the learner choose imaging settings, shows the sector map filling in, and reports the result: data recovered, time taken, and stress put on the simulated drive.

If the WebAssembly build of `libdr` is not available yet, build the `Sim` component against a defined interface with a simple stand-in implementation, and document the interface so the real engine can be dropped in.

## Part D: first modules

Write these as complete drafts:

- **T0 Foundations:** how hard drives, SSDs and flash store data; the patient safety rules and why each exists; setting up a safe workbench.
- **T1 Logical recovery:** recovering deleted files; recovering a formatted or lost partition; why deleted data on a TRIM-enabled SSD is usually gone.
- **T2 Imaging, first module:** reading SMART and deciding whether a drive is safe to image; making a first image with `drctl` and reading the sector map, with a simulator task.

## Content rules

- Every instruction that touches a real device says what the reader should observe and what to do if they see something else.
- Never instruct readers to open a hard drive, freeze it, hit it, or swap parts without the matching lab and its warnings. Mechanical work is described for understanding and always ends in "this needs a professional lab".
- Do not include vendor-specific firmware commands in these first modules.
- Be honest about limits: encryption without the key, TRIM, physical damage.
- Plain language for non-native English readers. Define each technical term once, and add it to a shared glossary.
- Everything you write is a draft. Set review status to `draft`. You are not the reviewing technician.

## Done when

- The components work, are keyboard accessible, and are documented with an example lab that uses every one.
- The triage wizard runs from its data file and covers every device and symptom listed above.
- The Part D modules are complete drafts using the components, with at least one `Decision`, one `Stop` and one `Quiz` where they fit naturally.
- CI validation passes and fails correctly on a deliberately broken lab file.
- `web/README.md` explains how to write a new lab module.

## Start with

The front matter schema and the `Step`, `Decision` and `Stop` components, proven with one short example lab. Then the triage wizard.
