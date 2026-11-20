# Concept: stream quality & camera effects for video chats

An unofficial feature concept for Telegram iOS, implemented in the client's own code.
Branch: `concept/stream-quality`. All changes are a diff on top of upstream `master`.

## What it adds

**1. One entry point for video.** The Video button in a video chat gets a small chevron.
A tap opens a menu: *Turn On Camera* or *Share Screen*. Screen sharing no longer hides in the ⋯ menu.

**2. Share Screen with quality settings.** A sheet before the system broadcast picker:

| Setting | Options | Premium |
| --- | --- | --- |
| Resolution | 720p, 1080p, 1440p | 1440p |
| Frame Rate | 15, 30, 60 FPS | 60 FPS |
| Device Audio | on / off | — |

The settings are applied **inside the broadcast extension**, on every ReplayKit frame:

- resolution caps the short side of the frame (a portrait iPhone screen is 1179 px wide, so 1080p sends 1080 px), scaled with one `VTPixelTransferSession` and a small buffer pool to stay within the extension's 50 MB memory limit;
- frame rate is a presentation-time gate, so a still screen costs nothing and a busy one is capped;
- device audio off drops `audioApp` buffers.

The app and the extension already share the app-group container, so the settings travel as a small JSON file next to the existing `broadcast-coordination` files. No server or call-engine changes.

**3. Camera with video effects.** A sheet with a live preview:

- Background: None, Blur, Studio, Custom (your own photo, Premium);
- Appearance: Lighting, Touch Up;
- *Turn On Video*.

Effects run on the device: Vision person segmentation (only when a background effect is on) → Core Image (lighting, skin softening, background blend) → a custom `OngoingCallVideoCapturer` (`isCustom: true`, the same external-buffer path screen sharing uses). The call sees an ordinary video stream, so participants on any client see the result. Switching cameras keeps working.

## Files

| File | What |
| --- | --- |
| `submodules/TelegramVoip/Sources/ScreencastQualitySettings.swift` | Settings model, storage, target-size math, frame-rate gate, frame processor |
| `Telegram/BroadcastUpload/BroadcastUploadExtension.swift` | Applies the settings to video and audio buffers |
| `submodules/TelegramCallsUI/Sources/VideoChatShareScreenSettingsScreen.swift` | Share Screen sheet (glass sheet, list rows, context menus, Premium gating) |
| `submodules/TelegramCallsUI/Sources/VideoChatCameraSettingsScreen.swift` | Camera sheet: preview, background options, appearance chips |
| `submodules/TelegramCallsUI/Sources/VideoChatCameraEffectsCapturer.swift` | Capture session + effects pipeline feeding the call |
| `submodules/TelegramCallsUI/Sources/VideoChatScreen.swift` | Video button menu, chevron badge, opening the sheets |
| `submodules/TelegramCallsUI/Tests/VideoChatStreamQualityTests.swift` | Unit tests for the quality and Premium rules |
| `Telegram/Telegram-iOS/en.lproj/Localizable.strings` | New strings |

The UI is built only from components the app already uses (`ResizableSheetComponent`, `ListSectionComponent`, `ListActionItemComponent`, `GlassBarButtonComponent`, `ButtonComponent`, `ContextController`), so it matches the current Liquid Glass design and dark call theme.

## Build and test

CI builds the app and runs the tests on GitHub's `xcode-27` runner: see `.github/workflows/concept-stream-quality.yml`.

Locally, with the Xcode version from `versions.json`:

```
python3 build-system/Make/Make.py --cacheDir="$HOME/telegram-bazel-cache" test \
    --target //submodules/TelegramCallsUI:TelegramCallsUITests \
    --configurationPath build-system/appstore-configuration.json \
    --codesigningInformationPath build-system/fake-codesigning
```

## Notes

- The iOS Simulator has no camera and no ReplayKit broadcasts. There the camera sheet runs the effects on the selfie placeholder the app already uses for simulated video; screen sharing needs a device.
- Premium values are clamped when the settings are loaded, so an account that loses Premium can't keep 1440p/60 FPS.
- Unofficial concept, not affiliated with Telegram.
