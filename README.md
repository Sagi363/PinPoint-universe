# PinPoint Universe

Hackweek 2026 universe for the **PinPoint** project — elevating the Punch Walk feature in Forma Mobile (Autodesk Build / ACC) so that issue pushpins auto-place on the 2D floor plan based on phone sensors and AR (Android-only POC).

## Scope

- Platform: **Android only** (POC)
- App: `build-mobile` monorepo (`android/` module)
- Feature: Punch Walk → auto-pin placement
- Core tech: ARCore (VIO) + PDR fallback + manual "tap to calibrate"

## Structure

```
agents/         — agent personas (empty for now)
skills/         — reusable capabilities (empty for now)
workflows/      — multi-agent orchestration scripts (empty for now)
rules/          — universe-wide rules (empty for now)
knowledge-base/ — research, ADRs, design docs
  build-mobile/
    pinpoint/
      research-indoor-positioning.md
```

## Knowledge Base Convention

All artifacts (research, design docs, ADRs, task plans) go to:

```
knowledge-base/<app>/<feature>/<artifact>.md
```

For PinPoint that means:

```
knowledge-base/build-mobile/pinpoint/<artifact>.md
```

## Status

Initial bootstrap. Research phase complete (see `research-indoor-positioning.md`). Agents and workflows to be added as the hackweek scope sharpens.
