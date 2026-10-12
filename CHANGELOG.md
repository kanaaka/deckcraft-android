# Changelog

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
