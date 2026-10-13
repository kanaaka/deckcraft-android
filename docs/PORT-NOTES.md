# Port notes — DeckCraft on Android

Upstream: [`storytold/deckcraft`](https://github.com/storytold/deckcraft) @
`be14248577b92059eb2b8c0c9460cceffbc1f01d` (v0.5.0)
Published APK: `deckcraft-0.5.0-arm64-v8a-v3.apk`

The port is `apps/deckcraft-android/` (`src/lib.rs` 509 lines, `src/file_picker.rs` 403,
`src/modifiers.rs` 202, `src/prefs_watch.rs` 134) plus **3 upstream files** — the same save-dialog seam
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

## 2026-10-13 — Two-finger scrolling (and the wheel Android never sends)

**The complaint:** on the tablet a finger drag selected text instead of moving the page, and there was
no way to scroll at all.

**Why:** every app in this suite scrolls from `egui::Event::MouseWheel` — `WordCraft`'s `scroll.rs` is
explicit about the contract (pixel deltas move the page 1:1, `TouchPhase::End` on the last event is the
"fingers lifted" signal that starts the kinetic coast, wheel notches count as three lines). None of
that runs on Android, because winit's Android backend has no mouse and therefore emits no wheel
events at all (`src/platform_impl/android/mod.rs` sends six window events and a wheel is not one of
them). Touch arrives as `WindowEvent::Touch`, which egui-winit forwards as `egui::Event::Touch` *and*
turns into an emulated mouse pointer — so a finger drags a selection and the page never moves.

**The change — `src/touch_scroll.rs` (new, 231 lines, 3 tests), wired into the shell's
`raw_input_hook`:**

- one finger: untouched, so dragging still selects — the emulated pointer works as before;
- two or more fingers: the centroid's movement becomes `Event::MouseWheel` with
  `MouseWheelUnit::Point` and `TouchPhase::Move`;
- a final `TouchPhase::End` wheel event when the fingers lift, which is what makes the app coast to a
  stop instead of stopping dead;
- while a two-finger gesture is live the emulated **pointer button** events are dropped, or the first
  finger would drag a selection across the page at the same time as it scrolls. Pointer *position*
  events are kept, because the apps only scroll while the pointer is over the canvas;
- the touch events themselves stay in the batch: egui's pinch handling reads them, so a pinch still
  zooms and a two-finger *pan* moves nothing but the page.

**Ctrl+wheel zoom** works too, which is why `modifiers.rs`'s `apply()` now returns the state it tracked
(`egui::Modifiers`) instead of nothing: the bridge stamps it onto the wheel event it synthesises, and
the touch events carry it as well.

**Checked:** the three tests in `touch_scroll.rs` (one-finger pass-through, two-finger centroid →
wheel, the lift event) run on the host with `cargo test -p deckcraft-android`; the shell compiles for
`aarch64-linux-android`; and the string table of the `.so` inside the released APK carries the module.
On-device behaviour is the porter's tablet test, not an automated one.

## 2026-10-13 — `modifiers.rs` returns the tracked state

`apply(&mut self, raw)` now returns the `egui::Modifiers` it folded into the frame's events. The
synthesised two-finger wheel event has to carry modifiers itself (Ctrl+wheel is zoom in these apps,
plain wheel is scroll), and only the tracker knows them, so the state is handed back rather than
recomputed. Nothing else about the modifier handling changed.

## 2026-10-13 — The *Fonts* folder, in the shells that did not have one

PhotoCraft and VectorCraft already pointed their platform-font hook at a USB-visible
`Android/media/<package>/Fonts`; this shell did not, so there was no way to install a font on the
tablet at all.

The shell now creates `<App media>/Fonts`, sets `HOME` to the app's media directory, and symlinks
`.fonts` and `.local/share/fonts` to that one folder, so every path a font scan is likely to consult
lands in the same visible place. A `.ttf`/`.otf` dropped in over a cable appears in the app's font
list on the next scan. `Android/media` rather than `Android/data`, deliberately: Android 11+ hides
`Android/data` from MTP.

## 2026-10-13 — The picker no longer tells a *granted* user to grant access

The empty-folder message in shared storage used to end with "Grant *All files access*…", and it was
shown to everyone, including users who had already granted it. The trigger had been
`/storage/emulated/0/Android/data` failing to list — **measured on the emulator: that folder stays
unlistable to the app on API 35 even with the grant**, so the tell was wrong and the advice was
insulting to the one user who had done what it asked.

Now the note is conditional on the folder being shared storage, is worded as a possibility ("If your
files are here, the app may lack shared-storage access…"), and leaves the decision to the user,
because the grant cannot be probed from inside the app.

The USB tip became folder-dependent for the same reason — Android 11+ hides `Android/data` from MTP,
so a file copied there over a cable never arrives and the folder merely looks empty (which reads as a
picker bug), while `Android/media` *is* reachable over a cable. Saying the wrong one of those two is
worse than saying nothing.
## 2026-10-13 — Ctrl shortcuts: the tracker reads the physical key too

### Fixed
- **Ctrl+anything did nothing, while Shift+click and Shift+arrow worked.** winit's Android
  logical-key table names `Shift` and `Alt` but **not** `Control` — so a Ctrl press arrives with an
  unidentified *logical* key (egui-winit falls back to the physical one) while its physical key is
  correctly `ControlLeft`/`ControlRight`. `src/modifiers.rs` matched the logical key alone, so it
  never saw Ctrl go down: Ctrl+A, Ctrl+C, Ctrl+V, Ctrl+S, Ctrl+B and every other modifier shortcut
  was dead, and only the modifiers winit happens to name worked. The tracker now reads the physical
  key as well as the logical one, on both left and right, and keeps everything else: the per-event
  modifier copies (so a press and a click in one frame stay ordered) and `ModifiersChanged` as the
  authority (so a modifier held while the app goes to the background does not stick).
  The same tracker is in all four ports of this set; it is verified end-to-end in WordCraft
  (`Ctrl+A` selects the text, `Ctrl+B` emboldens it). This app's own shortcuts are the tablet check.

## What is deliberately *not* in the patch

`Cargo.lock` is excluded on purpose: it moves only because the port crate is a new workspace member,
and `cargo` regenerates it on the first build. The patch contains only changes that matter to a reader
— every tracked file the port touches, plus any new file that has to live inside an upstream directory.

## Not verified here

The APK attached to **v0.1.2** was built from exactly this tree on 2026-10-13 (upstream pin + the patch + the port crate), and the checks in the README's verification table were run against it. What that does *not* buy is a runtime claim: it was not launched on a device or emulator as part of publishing, so every behavioural statement below rests on the hand-testing the porter did on an arm64 tablet. The fixes came from
the porter's hand-testing on an arm64 tablet — the bugs they fix were reported there — and there is no
automated device test here, so "verified" in the README means the source-side checks listed there plus
that testing.
