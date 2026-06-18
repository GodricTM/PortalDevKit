# 07 - Documentation & app previews

Good Portal apps should ship with the same care as the APK: a useful README, release notes, updater
metadata, and screenshots that show the app on a real Portal screen. Do this while building the app,
not at the end, because screenshots catch UI and permission-flow problems early.

## The docs package every new app should have

Create these files as soon as the app has a name and package id:

```text
YourApp/
  README.md
  CHANGELOG.md
  LICENSE
  meta-portal.json
  version.json
  screenshot.png
  screenshots/
    01_main.png
    02_core_workflow.png
    03_settings.png
    04_permissions.png
  release/
    RELEASE_NOTES.md
```

Use the snippets in [`snippets/`](snippets):

- [`app-readme-template.md`](snippets/app-readme-template.md)
- [`release-notes-template.md`](snippets/release-notes-template.md)
- [`meta-portal.json`](snippets/meta-portal.json)
- [`version.json`](snippets/version.json)

## Screenshot standard

Use real device screenshots wherever possible. A screenshot is part of the product surface: it tells
Portal owners what will appear on their tabletop display before they install it.

**Required preview images:**

- `screenshot.png` at repo root: the best single preview. Use this in `meta-portal.json`, app catalogs,
  social previews, and GitHub READMEs.
- `screenshots/01_main.png`: the first useful screen after setup.
- `screenshots/02_core_workflow.png`: the app doing its main job.
- `screenshots/03_settings.png`: settings/customization if the app has any.
- `screenshots/04_permissions.png`: setup/onboarding only if permissions are a meaningful part of the
  user experience.

**Quality rules:**

- Capture on a Portal, not a desktop mockup, once the app can run on hardware.
- Hide secrets, private topics, tokens, account names, IPs, and personal photos.
- Prefer real data from safe sources over placeholder data. If you must stage data, make it plausible
  and label it honestly in docs.
- Avoid debug overlays, log toasts, pointer trails, and notification noise.
- Make the primary action or value visible in the first screenshot.
- Keep text legible at tabletop distance; screenshots often reveal phone-sized UI.
- Include Portal-specific surfaces when relevant: top system pills, TV banner, overlay permission flow,
  notification listener setup, or Recents fallback behavior.

## Capture workflow

From a clean-ish install:

```bash
metavr device list
metavr app install -r app/build/outputs/apk/release/app-release.apk
metavr adb shell am start -n com.example.app/.MainActivity
```

Grant the permissions the app needs, then capture:

```bash
mkdir -p screenshots
metavr capture screenshot -o screenshots/01_main.png
metavr capture screenshot -o screenshots/02_core_workflow.png
metavr capture screenshot -o screenshots/03_settings.png
cp screenshots/01_main.png screenshot.png
```

On Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force screenshots
npx -y metavr capture screenshot -o screenshots\01_main.png
npx -y metavr capture screenshot -o screenshots\02_core_workflow.png
npx -y metavr capture screenshot -o screenshots\03_settings.png
Copy-Item screenshots\01_main.png screenshot.png
```

Portal screenshot caveat: `metavr capture screenshot` falls back to Android `screencap`, which does
not composite the soft keyboard surface. If you are documenting keyboard behavior, pair the screenshot
with an on-device note or verify IME state through `dumpsys input_method`.

## README structure

The README should answer four questions quickly:

1. What is this app?
2. What does it look like on Portal?
3. How do I install and grant permissions?
4. What Portal-specific constraints should I know?

Recommended order:

1. App name and one-sentence purpose.
2. Primary screenshot or short screenshot gallery.
3. Feature list.
4. Requirements.
5. Install and launch commands.
6. Permissions/setup commands.
7. Update/release notes link.
8. Credits/license.

Use a compact gallery for multiple screenshots:

```markdown
| Main | Workflow | Settings |
|---|---|---|
| ![](screenshots/01_main.png) | ![](screenshots/02_core_workflow.png) | ![](screenshots/03_settings.png) |
```

## Metadata for previews and stores

Keep `meta-portal.json` beside the README. It is not a replacement for the Immortal catalog entry, but
it gives every local app a consistent preview manifest that tools can read.

Minimum fields:

```json
{
  "name": "Your App",
  "description": "One sentence describing what the app does on Portal.",
  "package": "com.example.app",
  "category": "utility",
  "models": ["portal-plus", "portal-2019", "portal-mini", "portal-go", "portal-tv"],
  "screenshots": ["screenshot.png", "screenshots/01_main.png"],
  "minSdk": 28
}
```

For the Immortal catalog, reuse the same short description, homepage, icon, APK URL, minSdk, and
screenshot set where supported. See [04 - Store & discoverability](04-store-and-discovery.md).

## Release notes with visuals

Each release should include:

- The version title.
- A short paragraph explaining the release.
- A bullet list of user-visible changes.
- The install command for the release APK.
- Any permission re-grant note.
- Screenshots if the UI changed.

For GitHub release bodies, use absolute image URLs such as
`https://raw.githubusercontent.com/YOU/YOURAPP/main/screenshots/01_main.png`, or upload the images as
release assets. Relative screenshot paths are fine in `README.md`, but they do not reliably render
from `gh release create --notes-file`.

Use [`release-notes-template.md`](snippets/release-notes-template.md), then pass it to GitHub:

```bash
gh release create v1.0 YourApp-v1.0-release.apk --target main --title "Your App v1.0" --notes-file release/RELEASE_NOTES.md
```

## Definition of done for a new app

Before calling a new Portal app ready:

- `README.md` opens with a screenshot and install path.
- `screenshot.png` exists and is current.
- `screenshots/` has at least main, workflow, and settings/setup images.
- `meta-portal.json` names the screenshots and package id correctly.
- `version.json` points at the release APK URL.
- `release/RELEASE_NOTES.md` describes the current version.
- The README and release notes do not expose private data.
- The APK URL and screenshot paths work from a fresh clone.
