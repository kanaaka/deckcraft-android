# Changelog

## v0.1.2 — 2026-10-13 (Ctrl shortcuts)

| | |
|---|---|
| APK | `deckcraft-0.5.0-arm64-v8a-20261013b.apk` (29.1 MB) — supersedes the v0.1.1 asset, same signing key |
| sha256 | `c1d211868ab74de4f93b009a026b86f477f61f37c14a745c5752ee0e81a1b009` |
| Built from | upstream `storytold/deckcraft` @ `be14248577b92059eb2b8c0c9460cceffbc1f01d` + this repository at the commit that released it |
| Package | `ai.storyteller.deckcraft` — versionName `0.5.0`, versionCode `16778496`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

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

## v0.1.1 — 2026-10-13 (scrolling, modifiers, fonts, picker wording)

| | |
|---|---|
| APK | `deckcraft-0.5.0-arm64-v8a-20261013.apk` (29.1 MB) — supersedes the v0.1.0 asset, same signing key |
| sha256 | `16c4d66cd7955fbebe977bc3d37cd814279086eb672155fb2d965023b071d091` |
| Built from | upstream `storytold/deckcraft` @ `be14248577b92059eb2b8c0c9460cceffbc1f01d` + this repository at the commit that released it |
| Package | `ai.storyteller.deckcraft` — versionName `0.5.0`, versionCode `16778496`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- **Two-finger scrolling.** A two-finger drag now scrolls the deck (synthesised `MouseWheel` events
  from the touch centroid, with the lift event that starts the kinetic coast); one finger still selects
  and a pinch still zooms. Ctrl+wheel zooms.
- **A *Fonts* folder** at `Android/media/<package>/Fonts` (with `.fonts` and `.local/share/fonts`
  symlinked to it), so a font copied in over a USB cable is picked up.
- **Picker wording corrected.** An empty folder in shared storage no longer tells a user who has
  already granted *All files access* to grant it, and the USB tip now matches the folder
  (`Android/data` is unreachable over a cable, `Android/media` is not).

### Notes
The build was made from exactly the source in this repository (upstream pin + the patch + the port
crate) and signed with the same local test key as v0.1.0, so it installs over it.


## v0.1.0 — first published port (DeckCraft 0.5.0)

| | |
|---|---|
| APK | `deckcraft-0.5.0-arm64-v8a-v3.apk` (29.1 MB) |
| sha256 | `5a944e8c77784f45b7bf89bbf71c557719ec83ce052d1379499b13f128231f6d` |
| Built from | upstream `storytold/deckcraft` @ `be14248577b92059eb2b8c0c9460cceffbc1f01d` + `patches/0001-deckcraft-android-port.patch` + `apps/deckcraft-android/` |
| Package | `ai.storyteller.deckcraft` — versionName `0.5.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |
| Signature | throwaway local test key, not an official key |

### In this release
- Android shell crate: NativeActivity entry point, `Services` implementation, file logger, preferences
  written while the app runs (atomic), modifier-key synthesis, Activity not recreated on configuration
  change.
- **Save As and Export do not hold the frame** (`pick_save_async` + `save_dialogs`): a saved
  presentation learns its path, so the title bar follows, *Save* stops asking, closing stops prompting,
  and the file enters the recent list.
- **Ctrl / Shift / Alt work** — the platform reports no modifiers, so they are tracked from key events
  and re-injected.
- In-app picker for Open and Save As, with cached listings, "Show all files", hidden-file count and
  self-explaining locked roots.
- Launcher icon, storage permissions (images, video and audio), edge-to-edge theme variant.


### Known issues
See *Known gaps* in the [README](README.md) and [`docs/PORT-NOTES.md`](docs/PORT-NOTES.md).
