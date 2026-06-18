# PinPoint Knowledge Base

Source of truth for research, design decisions, ADRs, and task plans for the PinPoint hackweek project.

## Layout

```
knowledge-base/
└── <app>/
    └── <feature>/
        ├── research-*.md     # exploratory research / feasibility
        ├── adr-*.md          # architecture decisions
        ├── design-*.md       # design docs / specs
        └── tasks-*.md        # implementation task breakdowns
```

Non-markdown artifacts (HTML prototypes, images, PDFs) live alongside the markdown docs in the same `<app>/<feature>/` folder.

## Current Artifacts

- [build-mobile/pinpoint/research-indoor-positioning.md](build-mobile/pinpoint/research-indoor-positioning.md) — Indoor positioning feasibility research (PDR, VIO, calibration)
- [build-mobile/pinpoint/walkthrough-prototype.html](build-mobile/pinpoint/walkthrough-prototype.html) — ACC Walkthrough v0.25 — interactive HTML prototype of the Punch Walk auto-pin flow
