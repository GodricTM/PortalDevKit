# PortalDevKit

Everything we've learned building Android apps for **Meta Portal** — distilled from
[`immortal`](https://github.com/GodricTM/immortal) (the launcher) and [`PortalOverlays`](https://github.com/GodricTM/PortalOverlays) (the floating HUD),
so the next Portal app starts from working patterns instead of a blank page.

Portal is a discontinued, no-Google-services Android appliance. The platform has a handful of sharp
edges that bite every new app (no GMS, old SDK, a launcher that needs the right manifest "tags", an
unsigned-USB-driver dance on Windows). This kit captures those, plus the **drop-in building blocks**
we've already written and shipped — the OTA auto-updater, the silent installer, the draw-over-apps
overlay service, the accessibility nav cluster, keyless weather, ntfy push, and the on-device
permission onboarding.

## Community goal

PortalDevKit is meant to become a shared technical knowledge base for Meta Portal owners and
developers: device quirks, working app patterns, sideloading fixes, screenshots, compatibility notes,
and reproducible field reports. If you discover something on real hardware, please contribute it so
the next person does not have to rediscover it from scratch.

Good contributions include:

- Device and firmware behavior that differs between Portal, Portal+, Portal Mini, Portal Go, or Portal TV.
- ADB, driver, install, launcher, notification, overlay, camera, mic, or accessibility findings.
- App compatibility reports, including what works, what fails, and which APK/version was tested.
- Screenshots, logs, and exact commands that make a claim reproducible.
- New reusable snippets for common Portal app patterns.

Start with [CONTRIBUTING.md](CONTRIBUTING.md) and the [knowledge base](knowledge/README.md).

## Read in this order

| Doc | What it covers |
|---|---|
| [01 – Platform & constraints](01-platform-and-constraints.md) | Hardware matrix, no-GMS, SDK levels, design rules, the things that crash on launch |
| [02 – Manifest tags](02-manifest-tags.md) | The intent-filters, categories, icons/banner, and service declarations that make an app **appear** and **work**. This is the "tags" reference. |
| [03 – Build & sideload](03-build-and-sideload.md) | Toolchain (JDK + SDK + metavr), gradle config, signing, the install/debug loop, and the Windows USB-driver fix |
| [04 – Store & discoverability](04-store-and-discovery.md) | How **other people find your app**: the Immortal store `catalog.json` submission, F-Droid, GitHub-release conventions, banners |
| [05 – Reusable patterns](05-reusable-patterns.md) | Copy-this-pattern building blocks: **OTA auto-updater**, silent install daemon, overlay foreground service, accessibility nav, keyless weather, ntfy push, permission onboarding, screenshot capture |
| [06 – Gotchas & fixes](06-gotchas.md) | Device-verified field notes: overlay ghost-trails, the keyboard/touch-freeze fix, reinstall disabling accessibility, Recents on Portal Mini, no split-screen, USB Code-10 |
| [07 - Documentation & previews](07-documentation-and-previews.md) | The docs package for new apps: README, screenshots, `meta-portal.json`, release notes, and preview rules |
| [knowledge/](knowledge) | Community-submitted field notes, device reports, app compatibility notes, and research leads |
| [snippets/](snippets) | Ready-to-edit files: README/release templates, `meta-portal.json`, `version.json`, manifest fragments, and helper references |

## The six rules that matter most

If you remember nothing else:

1. **No Google Mobile Services.** No Play Services / Firebase / FCM / Maps SDK / ML Kit. Apps with a
   hard GMS dependency crash on launch. Pick keyless alternatives (Open-Meteo for weather, ntfy.sh
   for push, etc. — see [05](05-reusable-patterns.md)).
2. **`minSdk = 28`, `targetSdk = 29`.** Portal tops out at API 29. `compileSdk` can be current (35).
3. **The launcher needs the right tag.** Touch Portals want `MAIN + LAUNCHER`; Portal TV wants
   `MAIN + LEANBACK_LAUNCHER`. Without one, the app installs but is invisible. See [02](02-manifest-tags.md).
4. **Ship a PNG icon in `mipmap-xxxhdpi/`** (and `android:banner` for TV). Adaptive-only icons render
   blank on Portal's launcher.
5. **Design for a tabletop at arm's length, landscape-first.** Hit targets ≥ 64 dp, body ≥ 16 sp,
   and keep the top ~64 dp clear of Portal's white system pills.

6. **Document with screenshots while you build.** Every app should have `screenshot.png`,
   `screenshots/`, `meta-portal.json`, release notes, and a README that shows the app before asking
   users to install it. See [07](07-documentation-and-previews.md).

## Provenance

Two shipped apps back every claim here:

- **Immortal** (`com.immortal.launcher`) — a full home-screen replacement, screensaver/Dream, and an
  on-device **App Store** with a hosted catalog and self-update. The richest source of manifest and
  service patterns.
- **Portal Overlays** (`com.portal.overlays`) — a floating overlay HUD: draw-over-apps widgets,
  status strip, ntfy banners, an accessibility nav cluster, screenshot capture, and the OTA updater
  with a silent-install daemon.

When a doc says "see X", it points at the real file in one of those trees — that's the source of
truth; this kit is the map.
