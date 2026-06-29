# Reverse-engineering notes

Use this folder for Portal dump analysis, APK inventories, decompiled package notes, and firmware
breadcrumbs that are useful for understanding how Portal worked internally.

Keep this separate from `knowledge/apps/`: app compatibility notes are for things we can install and
run; reverse-engineering notes are for original firmware artifacts, dump contents, and components
that may never be user-installable.

## What belongs here

- Package inventories from stock Portal dumps.
- Decompiled APK observations.
- System-service and boot-path notes.
- Forum-derived notes that point to reproducible behavior or useful historical clues.
- Hypotheses that need validation on device.

## What to include in each note

- Status: `verified`, `observed`, `reported`, or `hypothesis`.
- Source: dump path, APK filename, logcat line, screenshot, or forum post.
- Device or firmware context when known.
- Exact package name if identified.
- A short summary of why the item matters.
