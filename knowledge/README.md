# Portal knowledge base

This folder is for community-submitted Portal field notes and compatibility data. Prefer facts that
someone can reproduce on real hardware.

## Structure

- `devices/` - model-specific notes, firmware behavior, hardware quirks.
- `apps/` - app compatibility reports and sideloading notes.
- `tools/` - host tooling, drivers, ADB, metavr, SDK setup findings.
- `research/` - leads that are not yet verified.

## Entry template

```markdown
# Short finding title

- Status: verified | observed | reported | hypothesis
- Device: Portal model and generation
- Firmware/API: build number or Android API if known
- Date tested: YYYY-MM-DD
- App/package: optional

## Summary

One or two sentences.

## Steps

1. Exact step or command.
2. Exact step or command.

## Result

What happened.

## Evidence

Logs, screenshots, command output, links, or notes on how many times it was reproduced.
```
