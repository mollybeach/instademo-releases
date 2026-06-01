# Clean up Swift concurrency warnings (DemoAudioPlayer / ScreenRecorder / ThumbnailCache)

**Priority:** P1 — pre-work for Swift 6 language mode
**Area:** `Services/DemoAudioPlayer.swift`, `Services/ScreenRecorder.swift`, `DesignSystem/Theme.swift`

---

## Problem

Three spots emit actor-isolation / `Sendable` warnings in the Release build that
will become hard errors under Swift 6 language mode:

```text
DemoAudioPlayer.swift:64/68 — main actor-isolated 'audioPlayerDidFinishPlaying'/'…DecodeErrorDidOccur'
  cannot satisfy nonisolated requirement from 'AVAudioPlayerDelegate'
ScreenRecorder.swift:196   — main actor-isolated property 'startWallClock'
  referenced from a Sendable (Timer) closure
Theme.swift:202–243        — Task<NSImage?, Never> / NSImage are not Sendable;
  cannot cross the main-actor boundary in ThumbnailCache
```

These don't break the current build but add noise and block enabling Swift 6.

## Proposed fix

- **DemoAudioPlayer** — mark the `AVAudioPlayerDelegate` conformance
  `@preconcurrency` (delegate callbacks already arrive on the main thread), as the
  compiler itself suggests.
- **ScreenRecorder** — move the `startWallClock` read inside the existing
  `Task { @MainActor in … }` so no main-actor state is touched from the Timer's
  Sendable closure.
- **ThumbnailCache (Theme.swift)** — wrap the generated `NSImage` in a small
  `@unchecked Sendable` struct so the cached/in-flight `Task` is fully `Sendable`
  and the value can return to the main actor without warnings.

## Acceptance criteria

- [ ] Zero actor-isolation / Sendable warnings from these three files in the Release build log.
- [ ] No behavior change to audio playback, recording duration, or thumbnail generation.
- [ ] Changes are compatible with eventually turning on Swift 6 language mode.
