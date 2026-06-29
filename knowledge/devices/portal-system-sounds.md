# Portal system sounds & ringtones map

Stock audio assets shipped in `/system/media/audio` on Portal. **217 files**, all **Ogg Vorbis
(`.ogg`)**, all owned by `root` and **world-readable** (`-rw-r--r--`) — so any sideloaded app can open
and play them straight off the path. They're the standard AOSP sound library.

- **Source:** pulled from a Portal+ (`metavr adb shell "ls -lR /system/media/audio"`), build dated 2025‑10‑14.
- **Path base:** `/system/media/audio/<category>/<Name>.ogg`
- Size is the honest proxy for length: a few KB = short alert/tick; 100 KB+ = a full musical loop.

## How you can (and can't) use them

- ✅ **Play them inside your app** (overlay banners, nav clicks, alerts) — point `MediaPlayer`/`SoundPool`
  at the absolute path, or copy the bytes into your app on first run. No runtime permission needed:
  `/system/media/audio` is on the readable system partition, not scoped external storage.
- ❌ **Register them as Portal's system ringtone / notification tone** — `/system` is verified + read-only
  and the appliance SystemUI exposes no ringtone picker, so `RingtoneManager`/`MediaStore` writes won't
  stick. Play them yourself instead. (Same read-only-`/system` constraint as the OTA `update_engine` keys.)
- ⚠️ **Redistribution:** these are Google/AOSP's assets, not yours. For the public GitHub/Reddit release,
  prefer **playing from the on-device path at runtime** over bundling copies into the APK — avoids shipping
  someone else's audio. (See [[no-fake-data]] sibling principle: don't ship what isn't yours to ship.)
- Note the **ALLCAPS duplicates** (`ANDROMEDA.ogg` vs `Andromeda.ogg`, `PERSEUS` vs `Perseus`, etc.) —
  AOSP's alternate-mix variants. Both files genuinely exist (ext4 is case-sensitive); pick by ear/size.

---

## ui/ — short interaction sounds (17) · best for app feedback

Path: `/system/media/audio/ui/`

| File | KB | Likely use |
|---|---|---|
| Effect_Tick.ogg | 5.0 | generic tick / selection |
| KeypressStandard.ogg | 5.7 | key tap |
| KeypressSpacebar.ogg | 5.8 | space |
| KeypressDelete.ogg | 5.7 | backspace |
| KeypressReturn.ogg | 6.1 | enter |
| KeypressInvalid.ogg | 9.6 | rejected input / error |
| Lock.ogg | 8.1 | lock |
| Unlock.ogg | 7.7 | unlock |
| Trusted.ogg | 5.6 | trusted-state confirm |
| Dock.ogg | 6.2 | dock |
| Undock.ogg | 8.0 | undock |
| LowBattery.ogg | 12.1 | low-battery warning |
| WirelessChargingStarted.ogg | 12.0 | charge start |
| camera_click.ogg | 5.8 | shutter |
| camera_focus.ogg | 9.2 | focus lock |
| VideoRecord.ogg | 6.3 | record start |
| VideoStop.ogg | 6.4 | record stop |

## notifications/ — short cues (75) · best for banners/toasts/ntfy

Path: `/system/media/audio/notifications/`

Short alert blips: `Adara, Aldebaran, Altair, Alya, Antares, Antimony, Arcturus, Argon, Beat_Box_Android,
Bellatrix, Beryllium, Betelgeuse, CaffeineSnake, Canopus, Capella, Castor, CetiAlpha, Cobalt, Cricket,
DearDeer, Deneb, Doink, DontPanic, Drip, Electra, F1_MissedCall, F1_New_MMS, F1_New_SMS, Fluorine,
Fomalhaut, Gallium, Heaven, Helium, Highwire, Hojus, Iridium, Krypton, KzurbSonar, Lalande, Merope, Mira,
OnTheHunt, Palladium, Plastic_Pipe, Polaris, Pollux, Procyon, Proxima, Radon, Rubidium, Selenium, Shaula,
Sirrah, SpaceSeed, Spica, Strontium, Syrma, TaDa, Talitha, Tejat, Thallium, Tinkerbell, Upsilon, Vega,
Voila, Xenon, Zirconium, arcturus, moonbeam, pixiedust, pizzicato, regulus, sirius, tweeters, vega`

Handy picks: `Drip` (6.7 KB, subtle), `Doink` (8.9 KB, playful), `TaDa` (43.5 KB, success fanfare),
`DontPanic` (17 KB), `Tinkerbell`/`pixiedust`/`moonbeam` (gentle), `F1_New_SMS` (message ping).

## alarms/ — sustained/looping (23) · attention-grabbing

Path: `/system/media/audio/alarms/`

`Alarm_Beep_01, Alarm_Beep_02, Alarm_Beep_03, Alarm_Buzzer, Alarm_Classic, Alarm_Rooster_02, Argon,
Barium, Carbon, Cesium, Fermium, Hassium, Helium, Krypton, Neon, Neptunium, Nobelium, Osmium, Oxygen,
Platinum, Plutonium, Promethium, Scandium`

Picks: `Alarm_Beep_01/02/03` + `Alarm_Buzzer` are the short, hard-edged ones for genuine alerts;
the element-named ones (`Neon` 177 KB, `Nobelium` 177 KB) are longer melodic loops.

## ringtones/ — long musical loops (102) · incoming-call / big events

Path: `/system/media/audio/ringtones/`

`ANDROMEDA, Andromeda, Aquila, ArgoNavis, Atria, BOOTES, Backroad, BeatPlucker, BentleyDubs, Big_Easy,
BirdLoop, Bollywood, BussaMove, CANISMAJOR, CASSIOPEIA, Cairo, Calypso_Steel, CanisMajor, CaribbeanIce,
Carina, Centaurus, Champagne_Edition, Club_Cubano, CrayonRock, CrazyDream, CurveBall, Cygnus, DancinFool,
Ding, DonMessWivIt, Draco, DreamTheme, Eastern_Sky, Enter_the_Nexus, Eridani, EtherShake, FreeFlight,
FriendlyGhost, Funk_Yall, GameOverGuitar, Gimme_Mo_Town, Girtab, Glacial_Groove, Growl, HalfwayHome,
Hydra, InsertCoin, Kuma, LoopyLounge, LoveFlute, Lyra, Machina, MidEvilJaunt, MildlyAlarming, Nairobi,
Nassau, NewPlayer, No_Limits, Noises1, Noises2, Noises3, OrganDub, Orion, PERSEUS, Paradise_Island,
Pegasus, Perseus, Playa, Pyxis, Rasalas, Revelation, Rigel, Ring_Classic_02, Ring_Digital_02,
Ring_Synth_02, Ring_Synth_04, Road_Trip, RomancingTheTone, Safari, Savannah, Scarabaeus, Sceptrum,
Seville, Shes_All_That, SilkyWay, SitarVsSitar, Solarium, SpringyJalopy, Steppin_Out, Terminated,
Testudo, Themos, Third_Eye, Thunderfoot, TwirlAway, URSAMINOR, UrsaMinor, VeryAlarmed, Vespa, World,
Zeta, hydra`

Short enough to double as long notifications: `Ding` (15.5 KB), `InsertCoin` (15 KB, arcade),
`NewPlayer` (15.6 KB), `Pyxis` (16.6 KB), `Carina` (15.4 KB). Big set-pieces: `Sceptrum` (316 KB),
`Perseus` (236 KB), `ArgoNavis` (226 KB), `CrazyDream` (207 KB).

---

## Playing them — drop-in helper

`.ogg` plays fine on Portal's audio stack. Use `SoundPool` for the short ui/notification blips (low
latency, preloaded) and `MediaPlayer` for the long ringtones.

```kotlin
object SystemSounds {
    private const val BASE = "/system/media/audio"

    fun ui(name: String)            = "$BASE/ui/$name.ogg"
    fun notification(name: String)  = "$BASE/notifications/$name.ogg"
    fun alarm(name: String)         = "$BASE/alarms/$name.ogg"
    fun ringtone(name: String)      = "$BASE/ringtones/$name.ogg"

    // Short blips — preload once, fire instantly.
    private val pool = SoundPool.Builder()
        .setMaxStreams(4)
        .setAudioAttributes(
            AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_ASSISTANCE_SONIFICATION)
                .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
                .build()
        ).build()

    fun preload(path: String): Int = pool.load(path, 1)          // returns soundId
    fun play(soundId: Int) = pool.play(soundId, 1f, 1f, 1, 0, 1f)

    // One-shot for longer clips (ringtones/alarms).
    fun playOnce(path: String) = MediaPlayer().apply {
        setDataSource(path)
        setOnCompletionListener { it.release() }
        prepare(); start()
    }
}

// e.g.
val tick = SystemSounds.preload(SystemSounds.ui("Effect_Tick"))
SystemSounds.play(tick)
SystemSounds.playOnce(SystemSounds.notification("Drip"))
```

Verify a file is actually present on a given unit before relying on it (asset sets can vary by build):
```bash
metavr adb shell "ls /system/media/audio/notifications/Drip.ogg"
```
