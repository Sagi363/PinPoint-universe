# Geo — PDR Navigation Specialist Memory

## Project Context

**Pinpoint**: Pedestrian Dead Reckoning (PDR) navigation for construction sites where GPS and cellular signal are unavailable. Core use case: indoor floor plan navigation with real-time positioning.

**Hackathon Scope**: 12-hour sprint to deliver functional PDR demo with 90-92% accuracy on construction site test data.

---

## Architecture Decision: Moderate Approach (Gyro + ZUPT, No Magnetometer)

### Decision Summary
- **Sensor fusion**: Madgwick filter (gyro + accelerometer)
- **Drift correction**: ZUPT (Zero-Velocity Updates) — detect stationary periods every 5-10 seconds
- **Constraints**: Map boundaries as hard constraints (walls stop movement)
- **Deferred**: Magnetometer (heading accuracy sufficient without it; revisit if drift >5° between ZUPTs)

### Why This Works

#### ZUPT (Zero-Velocity Updates) Mechanics
Gyroscopes drift 1-3% per minute due to bias accumulation. ZUPT corrects this by:
1. **Detection**: Monitor accelerometer variance during standing periods
   - Threshold: < 0.05 m/s² for stationary classification
   - Window: 50ms buffers at 200 Hz sampling
2. **Reset**: When stationary detected, set gyro-integrated heading error to zero
3. **Frequency**: Construction site workers naturally pause 5-10 seconds between movements → frequent resets
4. **Benefit**: 3-8x accuracy improvement with minimal UX friction

#### Why No Magnetometer (Yet)
- **Failure modes on construction sites**: Magnetometers fail near steel beams, rebar, concrete walls, and machinery
- **Corrupted readings worse than no reading**: Bad compass data confuses Kalman filter, increases heading error
- **ZUPT handles drift adequately**: 2-3° per corridor walk, reset to ~0.5° at next ZUPT
- **Fallback plan**: If testing shows >5° heading drift between ZUPTs, add 20-line outlier-detection hybrid approach (30 min implementation)

#### Trade-offs Accepted
- Heading accuracy degrades slightly between ZUPTs (2-3° drift acceptable for construction site scale)
- No external references (requires proximity to walls/known features for map constraints)
- Step length must be calibrated per user (construction workers have variable gaits)

### Expected Performance
| Metric | Value | Notes |
|--------|-------|-------|
| Position accuracy | 90-92% | On construction sites; 95%+ near map anchors |
| Heading drift | ~2-3° per corridor | Reset to ~0.5° by next ZUPT |
| Dev timeline | 12 hours | Functional + testable |
| Sensor sampling | 200 Hz accel, 50 Hz gyro | 50ms window buffers |
| Position update rate | 30 Hz | API stream (position, heading, confidence) |

---

## Algorithm Overview

### Phase 1: Sensor Loop
- **Accelerometer**: 200 Hz, collect in 50ms windows
- **Gyro**: 50 Hz, collect in 50ms windows
- **Buffers**: Maintain rolling windows for variance/peak detection

### Phase 2: ZUPT Detection
```
for each 50ms window:
  - compute accel variance across axes
  - if variance < 0.05 m/s²:
    - mark as STATIONARY
    - reset gyro drift accumulator to zero
    - signal map constraint check
```

### Phase 3: Orientation (Madgwick Filter)
- **Moving mode**: Trust accelerometer for gravity correction, gyro for precision
- **Stationary mode (ZUPT)**: Trust accelerometer heavily, suppress gyro drift
- **Output**: Quaternion → Euler angles (pitch, roll, yaw/heading)

### Phase 4: Step Detection
- **Method**: Peak detection in vertical acceleration (Z-axis)
- **Peaks per 50ms window**: Count footsteps
- **Integration**: stride_length × count = distance traveled

### Phase 5: Position Integration
- **Dead reckoning**: 
  ```
  dx = stride_length × sin(heading)
  dy = stride_length × cos(heading)
  ```
- **Map constraints**: Clip position to walkable regions (walls = hard boundaries)
- **Confidence**: Higher near ZUPTs, lower between steps

### Phase 6: API Stream
- **Rate**: 30 Hz
- **Payload**: (x, y, heading, confidence)
- **Consumer**: UI floor plan overlay

---

## Implementation Checklist — Phase 1

- [ ] **Sensor Fusion Module**
  - [ ] Madgwick or complementary filter implementation
  - [ ] Quaternion-to-Euler conversion
  - [ ] Accelerometer/gyro calibration (static offset estimation)

- [ ] **ZUPT Detector**
  - [ ] Variance computation (50ms window)
  - [ ] Threshold tuning (start: 0.05 m/s², adjust based on tests)
  - [ ] Drift reset logic

- [ ] **Step Detection**
  - [ ] Peak detection in Z-axis acceleration
  - [ ] Peak frequency → step cadence
  - [ ] Stride length parameterization (default: 0.7m, tunable)

- [ ] **Position Integration**
  - [ ] dx/dy accumulation from headings + steps
  - [ ] Map boundary constraints (polygon clipping or grid-based)
  - [ ] Confidence scoring

- [ ] **Foreground Service API**
  - [ ] Stream (position, heading, confidence) at 30 Hz
  - [ ] Permission handling (location, foreground service)
  - [ ] Graceful degradation if sensors unavailable

- [ ] **Testing**
  - [ ] Synthetic test walks (corridor, corners, loops)
  - [ ] Real construction site test (if available)
  - [ ] Accuracy validation against ground truth (tape measure, known waypoints)
  - [ ] Heading drift measurement between ZUPTs

---

## Knowledge Base References

All research and templates stored in `/knowledge-base/pinpoint/pdr-research/`:

1. **architecture-overview.md** — System design, sensor fusion philosophy, API contract
2. **implementation-guide.md** — 880 lines of Kotlin templates (Madgwick, ZUPT, peak detection, position integration)
3. **zupt-algorithm.md** — Deep-dive into ZUPT mechanics, threshold tuning, construction site gotchas
4. **quick-start-guide.md** — 30-min skeleton to wire sensors and stream first position

### Key Snippets to Reuse
- Madgwick filter matrix math (don't reimplement)
- ZUPT variance threshold (start 0.05, log results for tuning)
- Map constraint clipping (convex hull or grid-based, depends on floor plan data structure)

---

## Deferred Decisions

### Magnetometer Hybrid Approach (If Needed)
If testing reveals >5° heading drift between ZUPTs:
1. Add outlier detection: compare compass reading to gyro-integrated heading
2. Accept compass only if diff < 2°; otherwise trust gyro
3. Implementation: ~20 lines in filter update loop
4. Timeline: 30 min to test and tune

### Advanced Features (Post-Hackathon)
- Particle filter for multi-hypothesis tracking (handles loop closures)
- Sensor fusion with WiFi fingerprinting (if site has stable APs)
- User model learning (stride length, sensor bias per person)
- IMU-only heading (eliminate magnetometer dependency entirely)

---

## Session Summary

**Date**: 2026-06-21  
**Decision**: Gyro + ZUPT, no magnetometer (moderate approach, 12 hours, 90-92% accuracy)  
**Rationale**: ZUPT handles construction site realities (magnetic interference, frequent pauses); magnetometer deferred pending drift testing.  
**Next**: Begin Phase 1 implementation with sensor fusion and ZUPT detector.
