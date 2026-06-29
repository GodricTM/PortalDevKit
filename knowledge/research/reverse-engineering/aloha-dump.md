# Aloha dump inventory

- Status: observed
- Source: `Original APKS/` in the Portal+ workspace plus the public `facebook/aloha` dump
- Scope: original Portal firmware APKs that are useful for reverse engineering and behavior mapping

## Summary

This is the inventory of stock APKs and bundled apps that showed up in the Portal dump work.
Some are user-facing apps, some are system components, and some are only useful as breadcrumbs for
how Portal booted or routed features internally.

## Inventory

| Artifact | Likely package / role | Notes |
|---|---|---|
| `Alexa_falcon.apk` | `com.amazon.alexa.multimodal.falcon` | Alexa app bundle from the dump. Useful for voice-assistant behavior comparison. |
| `com.facebook.aloha.spotifystandalone.apk` | Spotify standalone build | Interesting as a stock Spotify path from Portal-era firmware. |
| `com.plexapp.android.apk` | `com.plexapp.android` | Stock Plex build from the dump. |
| `com.aspiro.tidal.apk` | `com.aspiro.tidal` | Stock Tidal build from the dump. |
| `org.chromium.chrome.apk` | `org.chromium.chrome` | Chromium-based browser artifact from the dump. |
| `photobooth.apk` / `PhotoBooth_photobooth.apk` | PhotoBooth / camera experience | Multiple artifacts likely tied to Portal camera or bundled app variants. |
| `Photos_superframe.apk` | Photos / Superframe experience | Likely a stock gallery / ambient-photo component. |
| `Storytime.apk` | Storytime | Portal content experience / companion app. |
| `spotify.apk` | Spotify | Standalone Spotify artifact from the dump set. |
| `us.zoom.zoompresence.apk` | Zoom Presence | Portal-era Zoom integration artifact. |

## Why this matters

These APKs help answer questions like:

1. Which third-party apps Meta shipped or preloaded on Portal.
2. Whether a feature came from a bundled app, a system service, or the launcher.
3. What can be compared against current sideloaded replacements.

## Follow-up work

- Pull package metadata from each APK and record versionCode / manifest quirks.
- Note which ones are launcher-visible versus service-only.
- Compare the bundled app behavior against Immortal and PortalDevKit guidance.

