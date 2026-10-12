# Port notes — DeckCraft on Android

Upstream: [`storytold/deckcraft`](https://github.com/storytold/deckcraft) @
`be14248577b92059eb2b8c0c9460cceffbc1f01d` (v0.5.0)
Published APK: `deckcraft-0.5.0-arm64-v8a-v3.apk`

The port is `apps/deckcraft-android/` (`src/lib.rs` 509 lines, `src/file_picker.rs` 403,
`src/modifiers.rs` 92, `src/prefs_watch.rs` 134) plus **3 upstream files** — the same save-dialog seam
GridCraft needed (`save_dialogs.rs`, new), because the same `pick_save` shape cannot work on a host
that draws its own picker between frames.

## 1. The shell crate

`deckcraft_ui_egui::Services` is a struct of `Option<Box<dyn Fn…>>` fields deriving `Default`; the
desktop binary and the browser shell fill it their own way, and Android is the third. Load-bearing in
every port of this set:

- **`[lib] name = "deckcraft"` + `crate-type = ["cdylib"]`** → `lib/arm64-v8a/libdeckcraft.so`, with the
  injected `android.app.lib_name` pointing at that artifact name (`-` → `_`). A mismatch installs and
  then dies on launch.
- **`options.android_app = Some(app)`** — `eframe::run_native` errors without it.
- **A file logger in the app's own data directory** — `logcat` needs adb, and a sideloaded build has
  none.
- **Preferences under `internal_data_path()`**: `ui.json` (recent documents) and `prefs.json`, written
  while the app runs (§3).

## 2. A saved presentation did not learn which file it is (`pick_save_async` + `save_dialogs.rs`)

`Services::pick_save` is the blocking shape — panel shown, path returned in the same call — which is
exactly what cannot happen on Android, where the picker is drawn with egui between frames. Falling
back to the web `download` path is not equivalent: the app is never told where the bytes went, so the
presentation keeps `path = None` — the title bar stays "Untitled", *Save* asks for a location again,
and closing prompts to save what was just saved.

The upstream patch adds the asynchronous seam, `Services::pick_save_async(&str) ->
Receiver<Option<String>>` (an empty answer is a cancellation), and a new
`crates/ui-egui/src/save_dialogs.rs` that holds the pending save and runs what asked on the frame the
path arrives; `poll_save()` is driven from the frame loop. Save As goes through it, and so does
**Export**:

- **Export** used to run inside the same call as the dialog. It now asks and exports on the answer
  frame, merging the PDF options into the parameters as before. The presentation is untouched either
  way — an export is a copy and must not become the deck's own file.
- The recent list gets the saved path at the top (`retain` + `insert(0)`), which is where the desktop
  puts it, instead of leaving the file out of Recent entirely.

## 3. Preferences are written while the app runs

The desktop writes its preferences on its way out; Android gives no such moment, so a swipe from
Recents took every preference with it — including the recent-documents list, which is why saved decks
kept vanishing from the Start screen. `prefs_watch` writes `ui.json` and `prefs.json` from the frame
loop: change-detected, at most every two seconds, atomically (temp file + rename, so a kill mid-write
cannot leave truncated JSON where the preferences were), one writer per file.

## 4. Ctrl, Shift and Alt did nothing — `src/modifiers.rs`

winit's Android backend emits only six window events and never `ModifiersChanged`, and `egui-winit`
updates modifiers from that event and nowhere else; egui carries modifiers as an *event*, so
`ctx.input(|i| i.modifiers)` is the all-false default for the entire run — Ctrl+S, Ctrl+Z, Ctrl+D and
Shift-click are dead. The key events do arrive, so the shell tracks the modifiers in `raw_input_hook`
and re-injects them the way egui expects: folded into the held state, stamped onto every `Key` and
`PointerButton` event, plus one `ModifiersChanged` per frame. State is carried **between** frames,
because the press and the click usually land in different ones.

## 5. The Activity is not recreated (`config_changes`)

`[package.metadata.android.application.activity] config_changes = "orientation|keyboardHidden|…"` so a
rotation, the keyboard appearing, a density or font-scale change, or a split-screen resize does not
recreate the Activity. A recreated Activity destroys the native window while the render thread lives
on — a blank surface — and that is the symptom the porter reported when the app was put into split
screen.

## 6. File dialogs, permissions, icon, theme

The picker is drawn with egui on DeckCraft's own seam (`src/file_picker.rs`), with the rules the whole
set shares: sized from `ctx.content_rect()`, enlarged padding for fingers, a repaint requested while
open, listings and readability probes cached per visit rather than per frame, locked roots left
tappable so they name the permission and the Settings path they need, the filter shown as the union of
every extension the app asked for with **"Show all files"** and a count of the hidden ones, and a save
answer of `dir.join(name)` so the app's atomic write finds its parent.

Permissions: `MANAGE_EXTERNAL_STORAGE` (granted by hand under Settings ▸ Special access; Google Play
restricts it to file-manager-class apps, which does not apply to a sideload), `READ_MEDIA_IMAGES`,
`READ_MEDIA_VIDEO` and `READ_MEDIA_AUDIO` for Android 13+ — a presentation app imports all three — and
the legacy storage permissions capped with `max_sdk_version`. `res/values/styles.xml` carries the
set's two-variant edge-to-edge theme (`targetSdkVersion 35` puts the app under a transparent status
bar on Android 15; winit 0.30 has no safe-area API) and `res/mipmap-*` the launcher icon.

## 7. No audio

`Services::audio_out` stays `None`: `cpal` has no Android backend here. Playback of an embedded audio
track is therefore silent on the tablet — the code path reports what it cannot do rather than failing
at runtime.

## Upstream files this port changes (3)

| File | Change | Why |
|---|---|---|
| `crates/ui-egui/src/save_dialogs.rs` | **new** — pending save + what to run when the path arrives | a host that draws its own picker cannot return a path in the same call (§2) |
| `crates/ui-egui/src/lib.rs` | `Services::pick_save_async`, `pending_save`, `ask_save`, `poll_save`; Save As asks asynchronously; recent list updated on save | §2 |
| `crates/ui-egui/src/dialogs.rs` | Export asks asynchronously and exports on the answer frame | §2 |

## Not verified here

The APK in Releases was not re-run on a device while publishing this repository. The fixes came from
the porter's hand-testing on an arm64 tablet — the bugs they fix were reported there — and there is no
automated device test here, so "verified" in the README means the source-side checks listed there plus
that testing.
