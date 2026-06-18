# Contributing to PortalDevKit

PortalDevKit is a community knowledge base for Meta Portal devices and app development. The most useful
contributions are specific, reproducible, and tied to a device model or APK version.

## What to contribute

- Portal device behavior: launcher, overlays, permissions, input, camera, mic, sleep, networking, USB.
- Build and sideloading fixes: Gradle, SDK, signing, `metavr`, Windows drivers, ADB quirks.
- App compatibility reports: package name, version, source, what works, what fails, logs/screenshots.
- Documentation fixes: clearer setup steps, missing commands, better screenshots, broken links.
- Reusable patterns: snippets that avoid Google Mobile Services and work on API 28/29.

## Field report standard

When reporting a finding, include:

- Device model: Portal, Portal+, Portal Mini, Portal Go, or Portal TV.
- Generation/year if known.
- Android/API version or firmware/build number if available.
- App package name and version, if an app is involved.
- Exact commands or steps.
- Expected result and actual result.
- Logs, screenshots, or `dumpsys` output when relevant.
- Whether the behavior was seen once or reproduced multiple times.

## Evidence levels

Use these labels in notes or issues:

- `verified`: reproduced on real hardware by you or another contributor.
- `observed`: seen once on real hardware, needs confirmation.
- `reported`: credible third-party report, not yet reproduced by the project.
- `hypothesis`: useful lead, not proven yet.

## Where to put information

- Broad docs or stable guidance: update the numbered Markdown docs.
- Device-specific field notes: add to `knowledge/devices/`.
- App compatibility: add to `knowledge/apps/`.
- Commands, scripts, or templates: add to `snippets/` or link to a working app.
- One-off reports: open an issue using the relevant template.

## Style

- Prefer short, concrete notes over long speculation.
- Do not include private tokens, ntfy topics, IPs, personal photos, or account names.
- If you quote a source, link it.
- If something only works on one Portal model, say so clearly.
- If a command is destructive or wipes app data, mark it before the command.

## Pull request checklist

- Links resolve.
- Commands are copy/pasteable.
- Screenshots are safe to publish.
- Device/app versions are included for hardware-specific claims.
- New snippets use placeholders like `com.example.app`, `YOU`, and `YOURAPP`.
