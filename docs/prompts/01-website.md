# Prompt 01: Project website

This prompt can be used alone in a website builder, or with `00-master.md` in a coding agent.

## Goal

Build the public website for the Data Recovery Platform: an open hardware and software system for recovering data from failing hard drives, SSDs, USB sticks and memory cards. The site explains the project, documents the hardware and software, offers downloads, and hosts an interactive recovery lab (the lab's engine is built separately; this task creates its landing page and the place it plugs in).

## Who visits

- **Someone with a broken drive right now.** Stressed, not technical. Needs to know within seconds what to do and what not to do.
- **Repair technicians and hobbyists.** Want to know what the hardware does, which devices it supports, and how to build or get one.
- **Developers and electronics people.** Want source code, schematics and specifications.

## What the product is (for the copy)

A failing drive connects to a box called the **Stabilizer**. The Stabilizer controls the drive's power, reads it with short time limits so it never hangs, saves everything it reads, and shows the PC a clean read-only external disk. Adapter boards connect many device types: SATA, IDE, M.2, NVMe, USB, SD, eMMC, NAND chips. A desktop app (DR Studio, Windows and Linux) controls it, images drives and extracts files. Everything is open: hardware, software, and teaching material.

## Pages

1. **Home.** One clear sentence of what this is, a diagram of patient → Stabilizer → PC, three paths ("My drive failed: start here", "See the hardware", "Get the software"), project status.
2. **Emergency first aid** (linked from the top of every page). Short do and do-not list: power off a clicking drive, do not run repair tools on the failing disk, do not open the drive, do not freeze it, stop using an SSD after accidental deletion. Then a link into the lab's triage wizard.
3. **Stabilizer.** How it works, in steps; operating modes; specifications; build status.
4. **Adapters.** A filterable compatibility matrix: device type → adapter → status (planned, prototype, available).
5. **Software.** What DR Studio and the `drctl` command-line tool do, screenshots when they exist, downloads per operating system, changelog.
6. **Lab.** Landing page with the triage wizard entry and the list of tracks; lab modules live under `/lab/...`.
7. **Docs.** User manual, hardware build guides, specifications. Sidebar navigation and search.
8. **Updates.** Dated progress posts.
9. **Community and contributing**, **Legal and safety** (authorisation to recover data, disclaimers, licences).

## Technical requirements

- **Astro** static site, content in **MDX**, TypeScript, small interactive components only where needed. No backend. Deployable to GitHub Pages or Cloudflare Pages. Code lives in `web/`.
- Content collections with validated front matter for docs, updates, adapters and labs, so that new content is added by writing a file, not by editing page code. The adapter matrix is generated from data files.
- Client-side search (Pagefind or similar).
- Structured for translation from the start: no hard-coded interface strings, locale-prefixed routes ready, English as the first language.
- Fast and accessible: works without JavaScript for all reading content, WCAG 2.1 AA contrast and keyboard navigation, light and dark themes, good on a phone (people read this next to a broken computer).
- CI: build, link check, front matter validation, accessibility check.

## Design direction

Calm, precise and trustworthy, like well-made test equipment. Clear typography, generous spacing, real diagrams drawn as SVG. Warnings must be unmistakable but used sparingly, so they keep their force. Avoid the usual "hacker" look, stock photos of people at laptops, and exaggerated promises. Write copy in plain language that a non-native English reader understands; explain a technical term the first time it appears.

## Honesty in the copy

Do not promise recovery. Say what the system can do, what it cannot (encrypted devices without the key, TRIMmed SSD data, mechanical repair), and show the real status of each part. Planned features are labelled planned.

## Done when

- All pages above exist with real draft copy, not placeholder text.
- The adapter matrix renders from data files and can be filtered.
- A new doc page, update post or adapter can be added by creating one content file.
- The site builds in CI, passes link and accessibility checks, and scores well on Lighthouse for performance and accessibility on mobile.
- A short `web/README.md` explains how to run, build, add content and deploy.

## Start with

The content model (collections and their schemas) and the site layout with navigation. Then Home and Emergency first aid, since they matter most to the most stressed visitor.
