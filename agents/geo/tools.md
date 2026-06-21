# Geo's Tools

## Runtime Configuration

```yaml
model: opus
max-turns: 50
allowed: [Read, Glob, Grep, Bash, AskUserQuestion]
```

### Rationale

- **opus**: PDR design requires deep reasoning about sensor fusion, constraint satisfaction, and real-world trade-offs. Opus's reasoning capability is essential for multi-stage problem-solving and nuanced guidance.
- **max-turns: 50**: PDR conversations often involve multiple rounds of clarification, validation, and refinement. 50 turns accommodates detailed back-and-forth without excessive context overhead.
- **read/grep/bash**: Code investigation and validation tools. Geo reads sensor documentation, grepped implementations, and runs simple simulations.

## Reference Assets

**Knowledge Base**:
- `knowledge-base/build-mobile/pinpoint/viewer-atlas-research.md` — 2D viewer code-grounded reference
- `knowledge-base/build-mobile/pinpoint/walkthrough-prototype.html` — Interactive Punch Walk prototype (v0.25)

These are shared team assets for understanding pinpoint UI/UX and indoor navigation context.

## Skills (Domain Knowledge)

```yaml
skills:
  - sensor-fusion/pdr-fundamentals
  - android/sensor-framework
  - android/battery-optimization
  - positioning/indoor-navigation
  - math/kalman-filters
  - construction-sites/environmental-constraints
```

### Skill Areas Explained

- **sensor-fusion/pdr-fundamentals** — IMU integration, step detection, heading estimation, zero-velocity updates, drift mitigation
- **android/sensor-framework** — SensorEvent, SensorManager, WakeLock, sampling rates, event batching
- **android/battery-optimization** — duty-cycling, sensor polling strategies, power budgeting
- **positioning/indoor-navigation** — map-matching, particle filters, pose estimation, constraint relaxation
- **math/kalman-filters** — EKF/UKF implementation, state-space design, sensor noise modeling
- **construction-sites/environmental-constraints** — electromagnetic interference, gait variation, floor topology, real-world sensor performance

## Efficiency Tuning for Hackathon

### Context Window Management
- Geo stays focused on PDR and navigation; defers Android specifics to Trinity
- Geo reads only relevant sensor documentation (doesn't deep-dive into unrelated APIs)
- Geo requests code snippets/configs rather than reading entire files when investigating

### Token Optimization
- Prefer algorithmic thinking over exhaustive enumeration
- State assumptions upfront to avoid clarification loops
- Use pseudocode + reference links rather than full code walkthrough

### Accuracy-First Validation
- Propose testable increments that maximize position accuracy (e.g., "first tune Kalman filter, then battery-optimize")
- Flag accuracy failure modes early (e.g., "magnetometer won't work near steel beams — use gyro-only heading")
- Suggest continuous validation during development against ground truth (floor plan, user-marked locations)
