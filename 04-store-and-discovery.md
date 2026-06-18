# 04 — Store & discoverability

> *"How other people find your app in their projects."*

A sideloaded Portal app is invisible unless it's **listed somewhere people look**. There are three
discovery channels, in order of reach for the Portal community:

1. **The Immortal App Store catalog** — the main way Portal owners browse + install apps on-device.
2. **F-Droid** — for open-source apps; resolved live so listings never go stale.
3. **A GitHub Release** — the canonical home of your APK + the `version.json` that drives OTA updates.

## 1. Get listed in the Immortal Store

Immortal ships an on-device **App Store** backed by a hosted [`immortal/catalog.json`](https://github.com/GodricTM/immortal/blob/main/catalog.json)
(schema v2). Submitting your app is how every Immortal user can find and one-tap install it.
Full rules: [`immortal/SUBMISSIONS.md`](https://github.com/GodricTM/immortal/blob/main/SUBMISSIONS.md).

Before submitting, make sure the app has the preview package from
[07 - Documentation & previews](07-documentation-and-previews.md): a root `screenshot.png`, a
`screenshots/` gallery, `meta-portal.json`, release notes, and a README that shows the app running.

**Submit** by opening an issue with the "App submission" template, or a PR adding your entry to
`catalog.json`. CI validates every change (schema shape, duplicate packages, https URLs, the F-Droid
id resolving, and that the icon + APK URLs actually serve).

### Catalog entry — your own app (direct APK)

```jsonc
{
  "name": "Your App",
  "packageName": "com.example.yourapp",
  "source": "url",
  "apkUrl": "https://github.com/you/yourapp/releases/latest/download/yourapp.apk",
  "minSdk": 29,                       // drives the store's "Needs Android X+" badge
  "description": "One clear sentence, under ~120 chars (one line in the list).",
  "longDescription": "A paragraph for the detail page.",   // optional
  "iconUrl": "https://…/icon.png",    // optional https PNG; omit → monogram tile
  "author": "You / Your Project",     // optional
  "homepage": "https://github.com/you/yourapp",  // optional
  "submittedBy": "your-handle",       // optional, community credit
  "devices": ["touch"]               // optional — only if restricted to "touch" or "tv"
}
```

### Catalog entry — an existing F-Droid app

```jsonc
{
  "name": "VLC",
  "packageName": "org.videolan.vlc",
  "source": "fdroid",
  "fdroidId": "org.videolan.vlc",
  "versionCode": 13070106,            // pin the arm64-v8a build (Portal is arm64)
  "minSdk": 21,
  "description": "Plays virtually any video or audio file or network stream.",
  "iconUrl": "https://f-droid.org/repo/org.videolan.vlc/en-US/icon.png"
}
```

Key fields / rules:
- **`source`** is `"url"` (your hosted APK) or `"fdroid"` (resolved live at install time — never stale).
- **`minSdk`** gates the compatibility badge: 1st-gen Portal/Portal+ and Portal TV are **API 28**, the
  Go/Mini/gen-2 are **API 29**. Apps with `minSdk ≥ 30` can't run on any Portal and won't be listed.
- **Multi-ABI apps:** pin the **arm64-v8a** APK with `versionCode`.
- **`devices`:** omit when it runs on every Portal; set `["tv"]` or `["touch"]` only if restricted.
- All v2 fields are **additive** — v1 clients keep working.

## 2. F-Droid

If your app is open-source, getting it into F-Droid means it can be listed in the Immortal catalog as
`source: "fdroid"` (auto-updating) and reaches the wider FOSS audience. Use the standard F-Droid
inclusion process; nothing Portal-specific beyond the API-29/arm64/no-GMS constraints.

## 3. GitHub Release conventions (these power OTA self-update)

Our apps self-update from a hosted `version.json` on the `main` branch (see
[05 § auto-updater](05-reusable-patterns.md)). The release setup that makes that work:

- **Tag = `vX.Y`**, e.g. `v1.4`.
- **Asset filename is stable and matches `apkUrl`** in `version.json` — e.g.
  `PortalOverlays-v1.4-release.apk`. The updater downloads exactly that URL, so a mismatch breaks
  in-app updates.
- **`version.json` lives on `main`** and is bumped *as part of cutting the release* (raw URL:
  `https://raw.githubusercontent.com/<you>/<repo>/main/version.json`).
- Optionally use the GitHub `…/releases/latest/download/<asset>.apk` redirect for the catalog `apkUrl`
  so it always points at the newest build.
- Reuse the screenshot and release-note standards from
  [07 - Documentation & previews](07-documentation-and-previews.md) so GitHub, local tooling, and app
  catalogs all show the same preview.

### `version.json` schema (the OTA manifest)

```json
{
  "versionCode": 5,
  "versionName": "1.4",
  "apkUrl": "https://github.com/you/yourapp/releases/download/v1.4/yourapp-v1.4-release.apk",
  "notes": "What changed in this build — shown in the in-app update dialog and the notification."
}
```

The client compares `versionCode` (a `long`) to the installed build and offers the update if greater.
Template: [`snippets/version.json`](snippets/version.json).

## Release checklist

1. Bump `versionCode` (+`versionName`) in `build.gradle.kts`. **Monotonic** — the updater compares it.
2. Update `CHANGELOG.md` and `version.json` (versionCode/Name/apkUrl/notes).
3. Refresh `README.md`, `release/RELEASE_NOTES.md`, `meta-portal.json`, `screenshot.png`, and the
   `screenshots/` gallery.
4. Build the **signed** release APK (same key as before).
5. Commit and push the release source/docs to `main`.
6. `gh release create vX.Y <asset>.apk --target main` — asset filename matching `apkUrl`.
7. Verify the raw `version.json`, the `apkUrl`, and the documented screenshot paths all return HTTP 200
   when hosted.
8. (If listed) bump/confirm the Immortal `catalog.json` entry.
