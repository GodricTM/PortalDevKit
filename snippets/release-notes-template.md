# Your App v1.0

Short paragraph describing what changed and why Portal owners should care.

## What's in this release

- User-visible change
- Portal-specific fix or compatibility note
- Setup/update improvement

## Screenshots

| Main | Workflow | Settings |
|---|---|---|
| ![](https://raw.githubusercontent.com/YOU/YOURAPP/main/screenshots/01_main.png) | ![](https://raw.githubusercontent.com/YOU/YOURAPP/main/screenshots/02_core_workflow.png) | ![](https://raw.githubusercontent.com/YOU/YOURAPP/main/screenshots/03_settings.png) |

## Install

```bash
npx -y metavr app install -r YourApp-v1.0-release.apk
npx -y metavr adb shell am start -n com.example.app/.MainActivity
```

## Permissions

Re-run the permission helper after updating if the app uses overlays, accessibility, notification
access, or silent installs.

## License

MIT.
