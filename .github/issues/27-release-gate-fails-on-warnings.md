# Release build gate fails on concurrency *warnings*, not errors

**Priority:** P0 — blocks every PR / push that touches Swift
**Area:** `.github/workflows/release-build-gate.yml`

---

## Problem

The "Release build gate" job builds the app in Release and then greps the build
log to hard-fail on Swift actor-isolation diagnostics:

```bash
grep -nE "actor-isolated|is 'async' but is not marked with 'await'|outside of its actor context|main actor-isolated property" /tmp/release-build.log
```

The project compiles in **Swift 5 mode** with
`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, so actor-isolation mismatches are
emitted as **warnings** ("…this is an error in the Swift 6 language mode") — they
do **not** break the build. The Release `build` succeeds (`** BUILD SUCCEEDED **`)
and the Mac App Store `archive` in the same run also succeeds.

Because the grep matches the warning *text* (not the `error:`/`warning:`
severity), the gate fails as a **false positive**. There are dozens of these
warnings spread across ~15 files, so the gate fails on essentially every PR even
though nothing is broken.

Representative log lines that trip it today:

```text
DemoConfig.swift: main actor-isolated property 'defaultMinutes' can not be referenced from a nonisolated autoclosure
ClaudeService.swift: main actor-isolated static method 'topicsPrompt(...)' cannot be called from outside of the actor
ElevenLabsService.swift: main actor-isolated property 'isInstaDemo' can not be referenced from a nonisolated context
```

## Proposed fix

Scope the grep to **error-severity** lines so the gate still catches the
Release-only regressions it was created for (issue #02) without false-positiving
on the warning sea:

```bash
grep -nE "error:.*(actor-isolated|is 'async' but is not marked with 'await'|outside of its actor context)" /tmp/release-build.log
```

`xcodebuild` already exits non-zero on real errors (the build step uses
`set -euo pipefail`), so this grep remains a backstop for masked exit codes.

## Acceptance criteria

- [ ] The gate passes when the Release build succeeds with only concurrency *warnings*.
- [ ] The gate still fails when a real `error:`-severity concurrency diagnostic appears.
- [ ] Comment in the workflow explains the Swift 5 warning-vs-error distinction.
