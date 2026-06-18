# 02 — Manifest tags

The `AndroidManifest.xml` "tags" are what make a Portal app **appear on the home screen** and **wire
up system integrations** (overlays, accessibility, notification access, screensavers, device admin).
Get these wrong and the app either installs invisibly or silently can't do its job.

Real references:
- [`PortalOverlays/app/src/main/AndroidManifest.xml`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/AndroidManifest.xml) — overlay + accessibility + notification-listener app.
- [`immortal/app/src/main/AndroidManifest.xml`](https://github.com/GodricTM/immortal/blob/main/app/src/main/AndroidManifest.xml) — the full kitchen sink (launcher, home, Dream, device-admin, FGS, alias).

## 1. Launcher visibility — the tags that make the app show up

A launchable app needs **one** of these `intent-filter`s on its main activity.

```xml
<!-- Touch Portals (Portal, Mini, +, Go) -->
<activity android:name=".MainActivity" android:exported="true"
          android:icon="@mipmap/ic_launcher">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

```xml
<!-- Portal TV — Leanback launcher + a banner instead of an icon -->
<activity android:name=".MainActivity" android:exported="true"
          android:banner="@drawable/banner">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
    </intent-filter>
</activity>
```

**Support both families from one APK** by declaring both categories on one activity (what Portal
Overlays does):

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
    <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
</intent-filter>
```

Notes learned the hard way:
- **`CATEGORY_DEFAULT` is *not* required** for the launcher tile (verified on Portal).
- **Don't put `CATEGORY_HOME` on a plain app.** Portal TV's stock home (`ripleyhome`) *hides* apps
  that carry `CATEGORY_HOME`. Immortal solves the "I'm both a home app and a normal app" problem with
  an **`<activity-alias>`** that carries only `LAUNCHER`/`LEANBACK_LAUNCHER` (no `HOME`) and points at
  the real activity — see the `ImmortalAppEntry` alias. Use that trick if you ever ship a launcher.

## 2. Icon tags — or the tile is blank

- Declare `android:icon` (touch) / `android:banner` (TV, 320×180) on the launcher activity.
- **Ship a real PNG in `mipmap-xxxhdpi/`.** Adaptive-only icons (`mipmap-anydpi-v26/` XML) render
  **blank** on Portal's launcher; the PNG is the fallback it actually uses. You can ship adaptive
  icons *alongside* — Portal falls back correctly — but never PNG-less.

## 3. Permission tags (the ones that actually work)

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />   <!-- draw over other apps -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" /> <!-- self-update -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />
```

Run-on-Portal feature declarations (so the app installs on every model):

```xml
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />
<uses-feature android:name="android.software.leanback"    android:required="false" />
<uses-feature android:name="android.hardware.camera"      android:required="false" />
```

Privileged permissions (only effective if a provisioning step grants them over adb — harmless no-ops
otherwise): `WRITE_SECURE_SETTINGS`, device-admin. Immortal uses these for screensaver/lock features
and grants them with `pm grant` / `dpm set-active-admin`.

## 4. System-integration service tags

These are the high-value, reusable declarations. Each is a "tag block" you can lift wholesale.

### Accessibility service (global Back/Home/Recents, gestures)
Powers a sideloaded nav cluster — the only sanctioned way to send system-wide Back etc.

```xml
<service android:name=".NavAccessibilityService" android:exported="false"
         android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter><action android:name="android.accessibilityservice.AccessibilityService" /></intent-filter>
    <meta-data android:name="android.accessibilityservice" android:resource="@xml/nav_accessibility" />
</service>
```
The `@xml/nav_accessibility` config sets `canPerformGestures="true"` etc. Enabled by the user in
Settings → Accessibility, or over adb (see [03](03-build-and-sideload.md)). Code:
[`NavAccessibilityService.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/NavAccessibilityService.kt).

### Notification listener (read other apps' notifications / media session)

```xml
<service android:name=".NotifyListenerService" android:exported="true"
         android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter><action android:name="android.service.notification.NotificationListenerService" /></intent-filter>
</service>
```
Used for the now-playing widget / notification mirroring. Grant: `cmd notification allow_listener`.

### Foreground service (must declare a type on API 29+)

```xml
<service android:name=".OverlayService" android:exported="false"
         android:foregroundServiceType="dataSync|mediaProjection" />
```
Immortal's overlay back-gesture uses `specialUse` + a `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` property.

### Boot receiver (restart a service after reboot)

```xml
<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter><action android:name="android.intent.action.BOOT_COMPLETED" /></intent-filter>
</receiver>
```

### Screensaver / Dream (Immortal only, but reusable)

```xml
<service android:name=".PhotoDreamService" android:exported="true"
         android:permission="android.permission.BIND_DREAM_SERVICE">
    <intent-filter>
        <action android:name="android.service.dreams.DreamService" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</service>
```
Set as the active dream via `settings put secure screensaver_components`.

### "Open with…" APK install handler (Immortal's universal installer)
Lets the app handle any `.apk` the user taps (routes through the silent daemon):

```xml
<activity android:name=".ApkInstallActivity" android:exported="true" android:excludeFromRecents="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="content" android:mimeType="application/vnd.android.package-archive" />
        <data android:scheme="file"    android:mimeType="application/vnd.android.package-archive" />
    </intent-filter>
</activity>
```

## 5. Activity attributes worth knowing

- `android:excludeFromRecents="true"` for ambient/full-screen surfaces (previews, lamps, wake lights).
- `android:showWhenLocked="true"` + `android:turnScreenOn="true"` for alarm-style wake screens.
- `android:windowSoftInputMode="adjustResize"` if you have text fields — **do not** add
  `stateUnchanged`, it suppresses the soft keyboard on focus (a real bug we hit; see
  [05 § keyboard](05-reusable-patterns.md)).
- `android:configChanges="orientation|screenSize|keyboardHidden"` to avoid recreates on rotation.
