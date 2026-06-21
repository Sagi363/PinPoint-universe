# PDR Research Template

Use this template to document PDR research findings in the universe knowledge base.

## Research Title

*One-line summary of the research topic (e.g., "Accelerometer sampling rates vs. step detection accuracy")*

## Problem Statement

*What was the open question or challenge?*
- Context: where this matters (construction site demo, specific device, etc.)
- Constraints: battery, accuracy target, timeline
- Goal: what you aimed to understand

## Approach

*How did you investigate this?*

### Methodology
- Data source: simulation, real hardware, reference papers
- Test conditions: device type, environment, number of trials
- Measurement: what metrics you tracked

### Key References
- Papers, libraries, or online resources consulted
- Code references (links to implementations tested)

## Results

*What did you find?*

### Key Findings
- Quantitative results: numbers, trade-offs, thresholds
- Qualitative observations: device quirks, environmental factors
- Confidence level: high (tested), medium (simulated), low (theoretical)

### Trade-offs Identified
- Accuracy vs. battery drain
- Complexity vs. reliability
- Real-world constraints vs. ideal conditions

## Failure Modes & Edge Cases

*What can go wrong? When does this approach break?*
- Device-specific issues (Pixel 7 vs. Samsung, old vs. new)
- Environmental factors (steel beams, concrete, uneven terrain)
- User behavior (fast vs. slow gait, spinning, standing still)
- Boundary conditions: time limits, sensor saturation, etc.

## Recommendations for Geo

*How should Geo guide implementation based on these findings?*
- When to use this approach
- When to avoid it
- Integration with other techniques (map-matching, calibration, etc.)
- Validation steps before deployment

## Notes for Future Research

*What's still unknown? What should the next investigation focus on?*
- Open questions
- Next validation steps
- Seasonal/time-dependent factors
- Hardware updates to monitor

---

**Research Phase**: [Design / Implementation / Validation / Post-Demo Analysis]
**Date**: [YYYY-MM-DD]
**Confidence**: [High / Medium / Low]
**Device(s) Tested**: [Pixel 7, Samsung S21, etc.]
**Environment**: [Lab / Construction site mockup / Real site]
