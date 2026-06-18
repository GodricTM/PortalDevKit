# Your App

One sentence explaining what this app does on Meta Portal.

![Your App on Portal](screenshot.png)

## Features

- Main user-visible feature
- Second user-visible feature
- Portal-specific feature or constraint handled by the app

## Screenshots

| Main | Workflow | Settings |
|---|---|---|
| ![](screenshots/01_main.png) | ![](screenshots/02_core_workflow.png) | ![](screenshots/03_settings.png) |

## Requirements

- Meta Portal with ADB enabled
- Node.js 20+ for `metavr`
- JDK 17 + Android SDK if building from source

## Build & install

```powershell
$env:JAVA_HOME='C:\Program Files\Android\Android Studio\jbr'
.\gradlew.bat assembleRelease
npx -y metavr app install -r app\build\outputs\apk\release\app-release.apk
npx -y metavr adb shell am start -n com.example.app/.MainActivity
```

## Permissions

List only the permissions this app actually needs.

```bash
npx -y metavr adb shell appops set com.example.app SYSTEM_ALERT_WINDOW allow
```

## Release

The in-app updater reads `version.json` from `main`. Release APKs are attached to GitHub releases as
`YourApp-vX.Y-release.apk`.

## Credits

Name services, datasets, libraries, or projects the app depends on.

## License

MIT.
