# Reddit launch post draft

Title:

Building PortalDevKit: a community knowledge base for Meta Portal sideloading, quirks, and app dev

Post:

Hi everyone,

I have been working on a small project called PortalDevKit: a public developer kit and knowledge base
for Meta Portal devices.

The goal is simple: a lot of Portal knowledge is scattered, fragile, or has to be rediscovered by
trial and error. These devices are discontinued, but they are still useful Android-based smart
displays, and now that ADB/sideloading is possible there is room for a small community around keeping
them useful.

PortalDevKit collects practical information for building and sideloading apps on Portal, including:

- device/API constraints across Portal, Portal+, Portal Mini, Portal Go, and Portal TV
- manifest tags needed for apps to appear in the launcher
- no-Google-Services alternatives for push, weather, updates, etc.
- Windows USB/ADB driver notes
- build, signing, install, and release workflow
- reusable patterns from working apps
- screenshots and preview/documentation standards for new apps
- gotchas like overlay ghost trails, keyboard behavior, Recents differences, and accessibility resets

What I would really like is for other Portal owners and developers to contribute field reports:

- What works or fails on your specific Portal model?
- Which APKs launch, crash, or need special setup?
- What firmware/API behavior have you verified?
- What ADB commands, logs, screenshots, or fixes helped you?
- Are there camera, mic, Bluetooth, TV, launcher, or permission quirks we should document?

The repo is set up with issue templates for field reports, app compatibility reports, and doc fixes,
plus a `knowledge/` folder where verified findings can become part of the database.

Repo:

https://github.com/GodricTM/PortalDevKit

If you still have a Portal and have learned anything weird or useful about it, please share it. Even a
small note like "this APK works on Portal Mini API 29 but not Portal TV" can save someone else hours.

I am especially interested in reproducible details: model, firmware/API, package name/version, exact
commands, screenshots, and logs with private info removed.

Hopefully this can become a central place for keeping these devices alive and useful.
