# Portal TTS engine — multi-language Sherpa-ONNX text-to-speech

- Status: verified (builds, installs, and speaks on Portal+ arm64)
- Device: Meta Portal+ (gen-1, aloha, API 28); applies to all arm64 Portals
- App/package: `com.k2fsa.sherpa.onnx.tts.engine`
- Upstream: [k2-fsa/sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) `android/SherpaOnnxTtsEngine`
- Source: `portal-tts-src/` (our fork, changes are in the working tree)
- Date: 2026-06-26

## Summary

Portals ship **with no system text-to-speech engine** (no Google services, no Samsung/Pico TTS). To
get spoken output anywhere on the device we sideload a fully on-device neural TTS engine built on
[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (ONNX Runtime + Kokoro / Piper / VITS models).
Once installed and selected, **any** app can speak through the standard
`android.speech.tts.TextToSpeech` API — including Portal Overlays' breaking-news alerts.

The upstream demo app ships **one hard-coded model, one language**. The whole point of our fork is to
turn it into a **multi-model, multi-language engine that reports whatever voices are actually present
on disk** and hot-swaps between them per request. That "how we got all the languages working" story
is below.

## How it works (architecture)

A TTS engine on Android is just an app that exports a `TextToSpeechService` plus a few helper
activities. The manifest wires them up (already correct upstream — we did not touch it):

```
service .TtsService            → action android.intent.action.TTS_SERVICE   (the engine)
activity .CheckVoiceData       → action android.speech.tts.engine.CHECK_TTS_DATA
activity .GetSampleText        → action android.speech.tts.engine.GET_SAMPLE_TEXT
activity .MainActivity         → settingsActivity (declared in res/xml/tts_engine.xml)
```

Native libs live in `app/src/main/jniLibs/arm64-v8a/`:
`libonnxruntime.so` + `libsherpa-onnx-jni.so` (Portal is arm64-v8a — the other ABIs are `.gitkeep`
placeholders and can stay empty).

### The key design decision: models live on external storage, not in the APK

`app/src/main/assets/` is **empty** (only `.gitkeep`). Models are **not** bundled into the APK. The
engine reads them from the app's external files dir at runtime:

```
/sdcard/Android/data/com.k2fsa.sherpa.onnx.tts.engine/files/<model-directory>/
```

i.e. `context.getExternalFilesDir(null)`. This is what makes multi-GB multilingual model packs
practical — you `adb push` model folders onto the device and the engine picks them up with **no
rebuild**. The native model is loaded with `OfflineTts(assetManager = null, config = ...)` so it
reads from a filesystem path rather than from APK assets.

## What we changed vs. upstream

All edits are in
`portal-tts-src/android/SherpaOnnxTtsEngine/app/src/main/java/com/k2fsa/sherpa/onnx/tts/engine/`.

### 1. `VoiceCatalog.kt` (new) — the model/voice registry

This is the heart of the multi-language support. A declarative list of models, each describing the
files on disk and the voices it exposes:

```kotlin
data class TtsModelSpec(
    val id, val displayName,
    val directoryName,            // folder under getExternalFilesDir(null)
    val modelName,                // e.g. "model.onnx" or "ro_RO-mihai-medium.onnx"
    val voicesFile = "",          // kokoro: "voices.bin"
    val dataDirName = "",         // "espeak-ng-data" (g2p phonemization)
    val lexiconFiles = [],        // e.g. lexicon-us-en.txt, lexicon-zh.txt
    val ruleFstFiles = [],        // text normalization, e.g. phone-zh.fst, number-zh.fst
    val numThreads = null,
    val voiceSpecs: List<TtsVoiceSpec>,
)

data class TtsVoiceSpec(name, displayName, locale: Locale, sid: Int, quality, modelId)
```

- `isInstalled(context)` — checks the model file + `tokens.txt` (+ `voices.bin` / `espeak-ng-data`
  when required) exist on disk. **Only installed models are reported to Android.** Drop in a model
  folder → its language lights up; remove it → it disappears. No code change.
- `buildConfig(context)` — turns a spec into the sherpa `OfflineTtsConfig` (absolute paths into the
  external dir).
- `installedVoices()` / `findInstalledVoice()` / `findInstalledModel()` — lookups used by the service.
- `toAndroidVoice()` — maps a `TtsVoiceSpec` to an `android.speech.tts.Voice` so the OS voice picker
  shows every voice with the right `Locale` and quality.

Currently registered:
- **Kokoro multi-lang v1.0** (`kokoro-multi-lang-v1_0`) — 28 English voices (`af_*`/`am_*` US,
  `bf_*`/`bm_*` GB). The model itself is multilingual (espeak-ng-data + per-language lexicons + zh
  rule FSTs are wired), so additional languages are a matter of adding `TtsVoiceSpec` entries.
- **Piper Romanian** (`vits-piper-ro_RO-mihai-medium`) — one `ro_RO` voice; the template for adding
  any per-language Piper voice.

### 2. `TtsEngine.kt` — multi-model loader with hot-swap

Upstream `TtsEngine` was a giant block of commented "uncomment exactly one example" model config.
We replaced it with catalog-driven loading:

- `createTts(context): Boolean` — loads the user's saved voice (or the first installed one); returns
  `false` (instead of crashing) when **no** models are installed.
- `loadVoice(context, voiceName): Boolean` (`@Synchronized`) — resolves the voice → its model,
  **releases the previous `OfflineTts` and loads the new one** if the model changed, sets speaker id /
  language, persists the choice. This is what lets a single engine serve many languages from many
  separate model files.
- `resolveVoice(context, requestedName, language)` — pick order: explicit requested name → currently
  active voice (if its language matches) → first installed voice whose ISO-3 language matches the
  request → first installed voice. So a `speak()` in German finds a German voice if one is installed,
  else degrades gracefully.
- `isLanguageAvailable()` / `defaultLanguage()` — answer purely from installed voices (ISO-3 codes).

### 3. `TtsService.kt` — the `TextToSpeechService` overrides

- `onGetVoices()` → `VoiceCatalog.installedVoices().map { toAndroidVoice() }`.
- `onIsLanguageAvailable()` / `onLoadLanguage()` → ISO-3 match against installed voices.
- `onGetDefaultVoiceNameFor(lang,…)` → first installed voice for that language, else first overall.
- `onSynthesizeText()` → resolves the voice (honouring an explicit voice request), hot-swaps the
  model, maps Android's speech rate (100 = normal) to engine speed (1.0 = normal, clamped
  0.1–5.0), streams PCM16 samples back via the callback.
- **Custom param `sherpa_voice_name`** — an external app can force a specific voice regardless of
  locale by putting it in the synthesis request `Bundle`.

### 4. `CheckVoiceData.kt` — report the real language list

Upstream returned a single hard-coded `lang`. We now return the **distinct ISO-3 languages of all
installed voices** (falling back to `["eng"]`), so Android's "languages available" check reflects
what's actually on the device.

### 5. `PreferencesHelper.kt` — persist the selected voice

Added `voice_name` (alongside the existing `speed` and `speaker_id`) so the chosen voice survives
restarts and the engine boots straight into it.

## Build & install

```bash
# Build the engine APK (uses Android Studio's bundled JBR on this machine)
cd portal-tts-src/android/SherpaOnnxTtsEngine
JAVA_HOME="/c/Program Files/Android/Android Studio/jbr" \
ANDROID_HOME="/c/Users/GodricTM/AppData/Local/Android/Sdk" \
  ./gradlew.bat :app:assembleRelease
# → app/build/outputs/apk/release/app-release.apk

# Install on the Portal
adb install -r -d app/build/outputs/apk/release/app-release.apk
```

### Installing a voice / language (no rebuild)

1. Get a sherpa-onnx TTS model from the
   [tts-models releases](https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models) and unpack it
   (each archive is a folder of `model.onnx`, `tokens.txt`, often `espeak-ng-data/`, `voices.bin`,
   lexicons, rule FSTs).
2. Push the folder into the engine's external files dir, using the **exact `directoryName`** from the
   matching `TtsModelSpec`:
   ```bash
   adb push kokoro-multi-lang-v1_0 \
     /sdcard/Android/data/com.k2fsa.sherpa.onnx.tts.engine/files/
   ```
3. To add a language the catalog doesn't know yet: add a `TtsModelSpec` (+ its `TtsVoiceSpec`s) to
   `VoiceCatalog.models` and rebuild the APK once. After that, the model folder push is all that's
   needed.
4. Select it on-device: **Settings → System → Languages & input → Text-to-speech output → pick the
   Sherpa engine**, then choose a voice. (Open the app's `MainActivity` to preview / set speed; it
   shows "No TTS models installed. Put model folders under …" when the dir is empty.)

> Models live under `Android/data/<pkg>/files`, so uninstalling the engine deletes them. Keep a copy
> of the model folders on the host.

## How other apps use it (Portal Overlays)

Apps don't link against sherpa — they use the **platform** `TextToSpeech` API, which routes to the
selected engine. Portal Overlays' `Speaker.kt` does exactly this and prefers our engine explicitly:

```kotlin
val pkg = "com.k2fsa.sherpa.onnx.tts.engine"
tts = if (isInstalled(pkg)) TextToSpeech(ctx, listener, pkg)   // force the Portal engine
      else TextToSpeech(ctx, listener)                          // fall back to any engine
// init is async → queue the first utterance until onInit(SUCCESS); speak() is a no-op if none exists
```

This is the graceful-degradation rule for Portals: **always stay useful without speech.** If no
engine is installed, `TextToSpeech` init fails and the app must still work visually (the breaking-news
popup flashes + chimes silently). See `PortalOverlays/app/src/main/java/com/portal/overlays/Speaker.kt`.

## Gotchas

- **arm64 only matters.** Ship the `arm64-v8a` jniLibs; Portal is arm64. Empty other-ABI folders are fine.
- **`OfflineTts(assetManager = null, …)`** — must be null so the model loads from the filesystem path,
  not APK assets. Passing the real `AssetManager` makes it look inside the (empty) APK assets.
- **External-files dir, not `/sdcard/<random>`** — the engine only scans `getExternalFilesDir(null)`.
  Push to the exact `Android/data/<pkg>/files/<directoryName>` path or `isInstalled()` returns false.
- **espeak-ng-data is the g2p backbone** for Kokoro/Piper/VITS — without it, those models can't
  phonemize and won't speak. Make sure the model folder includes it when the spec lists `dataDirName`.
- **No in-app downloader.** Voice install is a manual `adb push` (or a host script). The app UI only
  lists what's already on disk.

Related: [[portal-overlays-app]], [[portal-dev-kit]], [[immortal-build-env]].
