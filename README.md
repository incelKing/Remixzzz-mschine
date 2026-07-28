# Remix Machine — Native Android Rewrite (Phase 0: Voice Note Mode)

## Getting a real, installable APK without building it yourself

I can't compile this — no Android SDK, no Gradle, no network access at all
in the environment I write code in. But this project now includes
`.github/workflows/build-apk.yml`, which lets **GitHub's servers** build
it for you, for free, with nothing installed on your end except a
browser. Steps:

1. **Create a free GitHub account** at github.com if you don't have one.
2. **Create a new repository** (the "+" in the top right → "New
   repository"). Any name, public or private, doesn't matter. Don't
   initialize it with a README (you're uploading one).
3. **Upload this whole project folder.** On the new repo's page, click
   "uploading an existing file," then drag the *entire*
   `RemixMachineAndroid` folder (all of it — `app/`, `.github/`,
   `build.gradle.kts`, everything) onto the upload area. Modern GitHub
   preserves the folder structure when you drag a folder in, not just
   loose files. Commit the upload.
4. **Wait for the build.** Uploading the `.github/workflows/build-apk.yml`
   file automatically triggers it — click the **Actions** tab at the top
   of your repo, and you'll see a "Build APK" run in progress (a yellow
   dot). It takes a few minutes.
5. **Download the APK.** Once it finishes (green check), click into that
   run, scroll to **Artifacts** at the bottom, and download
   `remix-machine-debug-apk` — it's a zip containing `app-debug.apk`
   inside.
6. **Install it on your phone.** Transfer the APK to your phone (email it
   to yourself, save it via Drive, whatever's easiest), open it with a
   file manager, and tap install. Android will ask to allow installing
   from this source the first time — allow it. This is a **debug build**
   (signed with a debug key) — completely fine for using it yourself,
   just not something you'd submit to the Play Store as-is.

If step 3's drag-and-drop upload is finicky on your phone's browser, the
alternative is doing it from a computer, or via `git` if you have it
available (Termux on Android can do this too: `git init`, `git remote add
origin <your-repo-url>`, `git add .`, `git commit`, `git push`).

Re-run anytime by pushing an updated project (or hitting "Run workflow"
manually from the Actions tab) — you don't need to repeat the account/repo
setup, just steps 3-6 again.

---

## What this is, honestly

This is the **first real milestone** of the native Kotlin rewrite, not a
complete port of the web app. Full parity (Tracks, Pattern grid, Song
arrangement, Mixer, CURSED, vocal/beat detection, project save/load) is a
multi-month project even leaning hard on existing libraries — see the
research doc for the phased plan. Rather than half-build all of that at
once, this delivers **one complete, working vertical slice**: record a
voice note, mangle it hard, hear it, save/share it. Real code, nothing
faked.

This matches **Phase 0** of the roadmap: get a Compose skeleton running,
get audio in, get audio out, get a file exported — all in pure Kotlin, no
C++/NDK/Oboe yet. Once this feels solid on your device, Phase 1 is the
step-sequencer/pattern grid (loading MP3s, not just mic input), and
later phases move the real-time engine to Oboe when interactive latency
demands it.

## What's actually implemented

- **Record**: tap Record, up to 10 seconds (hard cap, auto-stops), or stop
  early. Raw 16-bit mono PCM via `AudioRecord` — no compression, so the
  mangle step works on clean samples.
- **Mangle** (`MangleProcessor.kt`):
  1. **Gap removal** — scans in 20ms windows, keeps only windows above a
     loudness threshold, splices the survivors together with a 5ms
     fade at each cut so there's no click. This is "get rid of the empty
     gaps": pauses are cut out entirely, not filled with something else.
  2. **Hard mangle**, applied to the now-gapless audio: bitcrush
     (bit-depth reduction), sample-and-hold (sample-rate reduction),
     random buffer stutter (rapid micro-repeats), ring modulation
     (multiply by a sine carrier), then peak-normalize. Same concepts as
     the web app's CURSED mode, ported to plain Kotlin math over a
     `ShortArray` — no real-time constraints here, so it's just a
     straight loop.
  3. Intensity slider defaults to **80%**, not a neutral 50% — the ask was
     "mangle it hard," so hard is the default, not something you have to
     dial up yourself.
- **Play**: static-mode `AudioTrack`, correct for a short one-shot clip.
- **Save/Share**: writes a real WAV file (proper 44-byte header) to the
  app's own external files directory (no storage permission needed on any
  supported API level), then hands it to Android's share sheet via
  `FileProvider` so you can save it wherever you want.

## What's NOT implemented (yet)

Everything else from the web app: multi-track loading, BPM/beat
detection, the pattern grid, song sequencing, the full FX chain (OTT
compression, echo, pump, nightcore), vocal/beat auto-arrangement, project
save/load. `MainActivity` currently hosts *only* the Voice Note screen —
see the comment there for where real navigation gets added.

## Important honesty note

I don't have Android Studio, a Kotlin compiler, or a device/emulator in
the environment I built this in — I can't compile or run this myself, so
treat it as carefully-written but **not yet build-verified**. I checked
every API call against current, standard Android/Kotlin usage, but the
very first thing to do is open it in Android Studio and get it to build.
If something doesn't compile, it's most likely a small API-shape mismatch
(see below) rather than a structural problem.

## Opening and building

1. **Android Studio**: use a current stable release (Ladybug/Meerkat-era
   or newer). File → Open → select the `RemixMachineAndroid` folder.
2. **Gradle wrapper jar**: this project includes
   `gradle/wrapper/gradle-wrapper.properties` (which Gradle version to
   use) but not the wrapper `.jar` itself (it's a binary file I can't
   produce from this environment). Android Studio handles this
   gracefully in one of two ways on first open:
   - It may prompt you to regenerate the wrapper automatically — accept it.
   - Or go to **File → Sync Project with Gradle Files**, and if that
     doesn't self-heal, run `gradle wrapper` from a terminal in the
     project root if you have Gradle installed, or let Android Studio's
     "New Project" wizard generate one in a throwaway project and copy
     `gradle/wrapper/gradle-wrapper.jar` + `gradlew`/`gradlew.bat` over.
3. **First sync**: Android Studio will very likely prompt you to upgrade
   the Android Gradle Plugin / Kotlin / Compose BOM versions pinned in
   the Gradle files. **Accept those upgrades** — they were pinned to
   what was current when this was generated, and newer is expected to be
   correct and safe.
4. **Run on a real device** if you can — the emulator's virtual
   microphone is unreliable for actually testing the record path.
5. Grant the microphone permission when prompted on first Record tap.

## If something doesn't compile

Most likely culprits, in rough order of likelihood:
- **Compose API shape drift** — `LinearProgressIndicator`'s `progress`
  parameter changed from a plain `Float` to a `() -> Float` lambda around
  Compose 1.6/Material3 1.2; this project uses the lambda form. If your
  resolved BOM is older, Android Studio's error will point you straight
  at the fix (either form, whichever your version wants).
- **Version catalog vs. direct version strings** — this project uses
  plain version strings in `build.gradle.kts` for simplicity rather than
  a `libs.versions.toml` catalog. Functionally equivalent; feel free to
  migrate to a catalog later if you prefer that style.
- **Package/namespace mismatches** — everything is under
  `com.kokjot.remixmachine`; if you rename the package, Android Studio's
  refactor tool handles all the references at once, don't hand-edit them.

## Design notes for whoever's reading this later (you)

- `VoiceRecorder`, `PcmPlayer`, and `WavExporter` are deliberately generic
  (mono 16-bit PCM in, mono 16-bit PCM out) so they're reusable once
  Phase 1 adds real MP3 track loading — the mangle/gap-removal logic
  doesn't care whether the samples came from a mic or a decoded file.
- `MangleProcessor.process()` is a pure function (`ShortArray` in,
  `ShortArray` out, no side effects) specifically so it's easy to unit
  test later and easy to reuse for the eventual CURSED-mode port in the
  real sequencer engine.
- The 30-second output cap in `MangleProcessor` exists because the
  stutter effect can grow the material significantly at high intensity —
  it's a safety net, not something you should expect to hit often from a
  10-second source.
