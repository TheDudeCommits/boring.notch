# BoringNotch Lyrics Handover

## Repository and current state

- Repository: https://github.com/AmirTheDude69/boring.notch
- Working branch: `codex/lyrics-notch-toggle`
- Last feature commit before this document: `46093eb` (Polish passive lyrics notch UI).
- Local checkout: `/Users/amir/Downloads/Whims/boring.notch`.
- User wants future changes pushed to this repository and an updated installable DMG supplied.
- Generated DMGs live in untracked `Release/`; do not commit these binaries by accident.

## Accepted behavior and design

- Fixed small mic icon beside Next toggles lyrics; selected is bright, unselected has lower opacity.
- Main listening source is Spotify in Arc, though other sources should work when macOS exposes Now Playing metadata.
- With mic enabled, show a persistent lyrics-only overlay at the notch while the user works elsewhere. No album art, song title, artist, or controls in the passive overlay.
- Hover still opens the ordinary music panel, with no lyrics under its song title.
- Lyrics are centered; current line is bright, neighboring lines dimmed, with slide/fade animation.
- Passive overlay uses the original expanded BoringNotch silhouette, padding, rounded corners, and shadow.
- User confirmed the functionality and latest visual appearance were good.

## Outstanding request: overlay disappears over Arc

The user reports that lyrics are visible on the desktop but disappear over Arc (screenshot suggests full-screen). They want the overlay visible over the browser whenever the mic toggle is on.

The previous turn was interrupted during a build. A proposed window-level patch from that turn is NOT present in the current checkout and is NOT committed. Do not report this issue as fixed.

Investigate both visibility policy and window layering before changing levels:

1. `boringNotch/ContentView.swift`: `shouldShowPassiveLyrics` requires `!vm.hideOnClosed`.
2. `boringNotch/models/BoringViewModel.swift`: `setupDetectorObserver()` sets `hideOnClosed` when fullscreen detection is enabled and the screen is fullscreen. This can intentionally suppress the overlay regardless of window level. Consider a scoped lyrics override that reacts immediately to the mic preference while preserving normal fullscreen hiding when mic is off.
3. `boringNotch/observers/FullscreenMediaDetection.swift`: inspect how Arc/fullscreen and the active media source affect the detector.
4. `boringNotch/components/Notch/BoringNotchSkyLightWindow.swift`: current level is `.mainMenu + 3`, collection behavior already includes `.fullScreenAuxiliary`, `.stationary`, `.canJoinAllSpaces`, `.ignoresCycle`. Check panel deactivation behavior and actual Space membership before escalating to `.screenSaver`.
5. `boringNotch/boringNotchApp.swift`: creates the window, adds it to `NotchSpaceManager` and calls `orderFrontRegardless()`. SkyLight delegation is currently used for lock screen only; preserve lock-screen preferences.
6. `boringNotch/managers/NotchSpaceManager.swift` and `boringNotch/private/CGSSpace.swift`: existing private max-level Space wrapper.

If adding workspace notifications, use `NSWorkspace.shared.notificationCenter` for `NSWorkspace.activeSpaceDidChangeNotification` and application activation notifications. The interrupted proposal incorrectly subscribed to active-Space changes on `NotificationCenter.default`.

Verify normal Arc, fullscreen Arc, Space switches, mic on/off, hover controls, and lock-screen behavior. A successful build alone does not verify overlay visibility.

## Main implementation files

- `boringNotch/managers/MusicManager.swift`: Now Playing controller selection, cancellable LRCLIB lookup, synced/plain lyric parsing and playback position helpers, mic action. Lyrics enabled currently prefers the universal Now Playing controller.
- `boringNotch/ContentView.swift`: passive overlay visibility, expanded notch chrome, centered lyric stack and transitions.
- `boringNotch/components/Notch/NotchHomeView.swift`: mic button beside Next and normal expanded music panel.
- `boringNotch/helpers/MediaChecker.swift`: detects adapter readiness using `setup_done` with bounded polling instead of blocking `waitUntilExit` task-group logic.
- `boringNotch/boringNotch.entitlements`: includes disable-library-validation, added after ad-hoc launch failed loading bundled MediaRemoteAdapter.

## Commits already pushed

- `89fdf70`: Add notch lyrics toggle.
- `b01416c`: Fix app launch and lyrics media detection.
- `34a3b7b`: Show lyrics as passive notch overlay.
- `46093eb`: Polish passive lyrics notch UI.

## Build and packaging

Full Xcode is installed at `/Applications/Xcode.app`; shell xcode-select may point at Command Line Tools. For a locally signed universal Release build:

```sh
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcodebuild build -project boringNotch.xcodeproj -scheme boringNotch -configuration Release -destination 'generic/platform=macOS' CODE_SIGN_IDENTITY=-
```

Expected app path on the original machine:
`/Users/amir/Library/Developer/Xcode/DerivedData/boringNotch-cuizwzgkptqeibcqnatwotjldndb/Build/Products/Release/boringNotch.app`

Verify the actual output path on the current machine. Validate signing and launch before packaging. The last UI-only session used `CODE_SIGNING_ALLOWED=NO`; its DMG checksum validation did not establish that the app was signed or launchable. Rebuild with signing enabled for the next installer. Local ad-hoc signing is not Developer ID signing/notarization.

Packaging script: `Configuration/dmg/create_dmg.sh <app-path> <output.dmg> boringNotch`.
Install hash-pinned `Configuration/dmg/requirements.txt` in a temporary Python >=3.10 venv (system Python was 3.9). A previous Python 3.12 packaging venv existed at `/tmp/boringnotch-dmg-venv`; verify it still works. Prepend its `bin` directory to PATH when invoking the script.

Run `codesign --verify --deep --strict` on the app, `hdiutil verify` on the DMG, and record SHA-256. Use a fresh filename for the new Arc fix installer to distinguish it from older builds.

## Validation and caveats

- Prior builds succeeded with existing Swift concurrency warnings and a warning that MediaRemoteAdapter targets macOS 15 while the app advertises macOS 14.
- MediaRemote streaming and LRCLIB lookup were independently observed working; user confirmed live lyrics work.
- Plain lyrics are advanced proportionally through the track duration, not truly timestamp-synced.
- iPhone-only playback is not automatically guaranteed; the Mac needs usable media metadata and timing.
- Initial security review flagged broad ATS allowance, network-server and Apple Events entitlements, XPC client-validation concerns, and bundled/private MediaRemote code. These were review findings, not all remediated.
- Do not broaden permissions or globally disable macOS security to resolve launch/overlay issues.
- Preserve existing user changes. Inspect git status before editing.
- User instruction: after browser tooling/Playwright use, close the browser immediately when finished.

## Suggested next-session prompt

Read HANDOVER.md on branch codex/lyrics-notch-toggle, inspect the current checkout, fix the passive lyrics overlay disappearing over Arc/fullscreen when the mic toggle is enabled, verify the behavior, push the fix, and provide a locally signed updated DMG. Start by checking fullscreen hiding policy before assuming a window-level issue.
