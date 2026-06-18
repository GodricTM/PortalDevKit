# snippets

Ready-to-edit starting points. Replace `com.example.app` / `YOU` / `YOURAPP` placeholders.

| File | Use |
|---|---|
| [`app-readme-template.md`](app-readme-template.md) | Starting README with screenshot gallery, install path, permissions, release notes, and credits. See [../07](../07-documentation-and-previews.md). |
| [`release-notes-template.md`](release-notes-template.md) | GitHub release notes template with screenshot slots and install commands. |
| [`meta-portal.json`](meta-portal.json) | Local app preview manifest for package id, supported Portal models, minSdk, and screenshots. |
| [`version.json`](version.json) | OTA update manifest — host on your repo's `main` branch. See [../04](../04-store-and-discovery.md). |
| [`manifest-fragments.xml`](manifest-fragments.xml) | Copy the permission + launcher + service tag blocks into your `AndroidManifest.xml`. See [../02](../02-manifest-tags.md). |

## New app docs checklist

Start every new Portal app with:

- `README.md` from `app-readme-template.md`
- `CHANGELOG.md`
- `meta-portal.json`
- `version.json`
- `screenshot.png`
- `screenshots/01_main.png`, `screenshots/02_core_workflow.png`, `screenshots/03_settings.png`
- `release/RELEASE_NOTES.md` from `release-notes-template.md`

## Live files to copy verbatim (don't duplicate — point at these)

These already exist and work; lift them directly rather than re-keying:

- **Silent install daemon:** [`PortalOverlays/installd.sh`](https://github.com/GodricTM/PortalOverlays/blob/main/installd.sh)
  (pin to LF via `.gitattributes`).
- **Permission-grant helper:** [`PortalOverlays/enable_portal_permissions.ps1`](https://github.com/GodricTM/PortalOverlays/blob/main/enable_portal_permissions.ps1)
  and [`.bat`](https://github.com/GodricTM/PortalOverlays/blob/main/enable_portal_permissions.bat).
- **Patched Windows USB driver:** [`../drivers/android_winusb.inf`](../drivers/android_winusb.inf).
- **OTA updater client:** [`PortalOverlays/.../UpdateChecker.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/UpdateChecker.kt)
  (+ `InstallDaemon.kt`, `UpdateInstallReceiver.kt`).
- **Accessibility config xml:** [`PortalOverlays/.../nav_accessibility.xml`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/res/xml/nav_accessibility.xml).
