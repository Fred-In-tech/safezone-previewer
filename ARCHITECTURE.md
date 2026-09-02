# SafeZone — Architecture

A single Flutter binary with no backend. The interesting parts are the boundaries: where geometry comes from, who owns the video decoder, and what happens when every external dependency is missing at once.

*Interactive map: [architecture map](https://fred-in-tech.github.io/safezone-previewer/map/)*

## Module layout

```
lib/
├── main.dart                   entrypoint: orientation lock + DI
├── app/                        root widget, theme, design tokens, brand mark
├── core/bootstrap.dart         AppServices: immediate() then warmUp()
└── features/
    ├── safezone/               geometry: model, presets, repository, remote source
    ├── overlay/presentation/   the rendering engine (pure maths + painters)
    ├── media/                  one decoder, strict ownership, transport
    ├── capture/                record inside the guides
    ├── projects/               local library, thumbnails, home screen
    ├── settings/               preferences, theming
    ├── export/                 capture → permission → gallery
    ├── ads/                    cadence gate, manager, presenters, banner
    └── preview/                editor state, providers, screen, controls
```

Feature-first, and dependencies point inward: `overlay` knows about geometry but nothing about projects; `safezone` knows nothing about widgets at all. A shared `PlatformBadge` that had been written inside the projects UI was moved into `safezone/presentation` next to the brand it renders, precisely because the capture flow was importing the projects screen just to draw a coloured tile — a dependency pointing the wrong way between two unrelated features.

## The safe-zone pipeline

```
SafeZonePresets (compiled)  ─┐
                             ├─→ SafeZoneRepository ─→ OverlayGeometry ─→ painters ─→ canvas
Firebase Remote Config ─────┘        (merge + validate)      (pure)
```

**`SafeZoneConfig`** quotes every figure in pixels against a 1080×1920 master canvas — top header, bottom caption, right column, left margin, action icon count, aspect ratio — and scales onto any viewport. **`OverlayGeometry`** derives every rectangle the overlay needs. Neither touches Flutter's widget layer, so the maths is directly unit-testable and pixel-probed in widget tests.

**`SafeZoneRepository`** is where the trust boundary sits. Bundled presets are the floor. Remote overrides are applied key by key onto a preset, and the *candidate* is checked before it is accepted: a result leaving less than 5% usable canvas, or an aspect ratio outside 0.1–10.0, is thrown away in favour of the bundled value. That upper/lower aspect bound is not arbitrary — a mistyped value reaching `AspectRatio` asserts on non-finite input and lays out degenerately without the assert.

An override for an id the app has never seen, carrying complete geometry, **adds** a platform. A new feed can be supported without shipping a build.

### Parsing the one input a human types

Every other input to this app is produced by code. The `safe_zones` parameter is typed into a console by a person, usually in a hurry, usually to correct geometry that has already started misleading creators. A trailing comma, unquoted keys, an array where an object belongs, or prose pasted into the field all resolve to "no overrides" rather than an exception on a background fetch — each pinned by a test.

Making that testable required extracting the parse from `fetchOverrides`, where it had been welded to a live `FirebaseRemoteConfig` handle. It was the least-tested code handling the least trustworthy input.

## Media ownership

A leaked `VideoPlayerController` holds native decoder and texture memory, so ownership is centralised in `MediaSession` and abstracted behind a `PlayableHandle` interface — which is what makes the leak paths testable without a platform decoder in the loop:

- Loading a new file always releases the previous handle, *first*.
- A handle whose initialisation throws is released, not stranded.
- A load superseded by a newer load releases its own handle and does not overwrite the newer result.
- A load completing after `dispose()` releases immediately.

The session is declared in Riverpod as depending on the current project, so it is hosted in the editor's scope rather than the root container. Leaving the editor disposes the decoder instead of leaving a native texture alive behind the project list.

`pause()` is deliberately distinct from `dispose()`: on backgrounding the user is coming back to this frame, so the decoder is kept but must stop consuming battery and audio focus.

## Lifecycle

The app originally had no lifecycle handling, which is easy to miss because nothing looks wrong on screen. Three things were wrong:

- The **camera** is taken away by the OS on a call or an app switch; holding the controller across that leaves a dead session and a frozen preview. It is released on the way out and rebuilt on return — and a take in progress is *finished and returned* rather than abandoned, because the user recorded those seconds.
- A **video** kept looping behind whatever the user switched to, holding a decoder for a frame nobody was watching.
- The **wakelock** kept the display awake in the background, which is the worst version of that bug.

## Export path

The export renders exactly what the preview shows: the same `RepaintBoundary` subtree, rasterised at a scale that reaches a true 1080×1920 frame. The transport bar and the ad banner sit *outside* that boundary by construction, so neither can be burned in.

`ExportService` never throws. Permission denial, permanent denial, empty bytes and save failure are all `ExportResult` values the UI renders directly — an export failing is a normal outcome the user needs told about, not a crash. The gallery saver sits behind a one-method interface, which is why replacing the originally specified saver package (unbuildable against AGP 8: no namespace declared, no release since 2023) with `gal` changed a single adapter file and nothing above it.

Thumbnails reuse the same machinery at low resolution, rendered *clean* without guides — the card is showing the creator's frame, not the tool's markings. That avoided pulling in a native video-thumbnail plugin for a 15KB image.

## Storage and privacy

There is no server, no account and no upload. The project library is a plain local list, written defensively so a single corrupt entry cannot cost someone their library. Recordings live in `app_documents/captures/` — moved there out of the cache the camera writes to, which the OS is free to purge, and unlike a picked clip a recording exists nowhere else. Deletion is gated on ownership: only files inside the app's own captures folder qualify, so a clip picked from the photo library is left strictly alone.

`ios/Runner/PrivacyInfo.xcprivacy` declares the three required-reason APIs the app touches (user defaults for preferences, file timestamps for takes and thumbnails, disk space) and the data AdMob collects; the app's own collection is empty and `NSPrivacyTracking` is false. It was verified present inside the built `Runner.app`, not just in the source tree — a file in the folder is not in the bundle until it is in the target's resources.

## Monetisation as policy, not plumbing

`AdMoment` has exactly two values — `appLaunch` and `beforeExport` — and that enum *is* the policy: if a moment is not listed there, no full-screen ad can fire for it. A test asserts the enum has not grown. `InterstitialGate` is pure, with an injected clock and no SDK handles, so the entire cadence (including a 90-second lock spanning all full-screen formats) is verifiable without an ad network.

Two failure rules, both covered: the cooldown starts only on a real presentation, so a failed ad costs the user nothing; and with ads disabled `showRewarded()` grants the reward outright rather than stranding the user behind an outage. Switching platforms, toggling guides, scrubbing and zooming are what the app is *for*, and none of them can trigger a full-screen ad.

Ad unit ids come from per-platform config templates (`.example` files in the repo, real ones gitignored). With no configuration the app runs on Google's official test units, so a fresh clone cannot generate invalid traffic against a live account, and a half-configured override is rejected wholesale rather than mixing live and test units.

## Testing and CI

55 test files, roughly 8.6k lines of test against 9k lines of source. The suite is run twice in CI — once normally, once with `--test-randomize-ordering-seed=random`, which found two order-dependent tests.

The layers that carry the most tests are the ones where being wrong is expensive rather than the ones that are easy to test: decoder ownership, which file the app may delete, remote-config parsing, the two-moment ad policy, and WCAG contrast across both themes. A sweep for declaration-only API found six public symbols that existed solely so a test could read them; deleting them forced the better version of those tests — asserting the consequence (no inventory means no ad and no blocked user) rather than an internal ready flag. A test that reads private state passes when the state is right and the behaviour is wrong.

CI runs format, analyser with `--fatal-infos`, both test passes, coverage, and a secret scan that fails on any tracked keystore, signing config, service file or ad configuration, and on any AdMob publisher id other than Google's public test one. Only then do the Android (obfuscated bundle plus debug symbols) and iOS (unsigned release) build jobs run.
