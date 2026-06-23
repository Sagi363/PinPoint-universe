# George's Service Contract

The AIDL contract this service implements. Derived from the QuickPlacement hackathon flow. Treat the signatures here as a starting point — refine in code, but keep the state machine and call ordering intact.

## End-to-End Flow

1. Host app: user taps their current location on the PDF → host calls `initTracking(x, y)`.
2. Host app: user walks ~3 meters in a chosen direction.
3. Host app: user taps their new location on the PDF → host calls `updatePosition(x, y)` (calibration anchor).
4. Service: derives stride length + heading bias from the two anchors, starts the PDR algorithm.
5. Service: streams `PositionUpdate(x, y, heading, confidence, timestamp)` via the registered listener at 10–30 Hz.
6. Host app: draws the user's latest position on the PDF.
7. Host app: when the user creates an issue, calls `getLastKnownPosition()` and pins the issue there.

## AIDL Interfaces (proposed shape — refine in code)

```aidl
// IPositionService.aidl
package com.autodesk.quickplacement.service;

import com.autodesk.quickplacement.service.IPositionListener;
import com.autodesk.quickplacement.service.PositionUpdate;

interface IPositionService {
    int getApiVersion();

    /** Record the user's starting anchor on the PDF. Resets any prior tracking. */
    void initTracking(in float x, in float y);

    /**
     * Record the calibration anchor after the user has walked ~3m.
     * Service uses this to derive stride length + heading bias, then starts streaming.
     */
    void updatePosition(in float x, in float y);

    /** Latest position estimate. Used by the host when an issue is created. */
    PositionUpdate getLastKnownPosition();

    void registerListener(in IPositionListener listener);
    void unregisterListener(in IPositionListener listener);

    /** Stops the PDR loop and releases sensors. Service stays alive (bound). */
    void stopTracking();
}
```

```aidl
// IPositionListener.aidl
package com.autodesk.quickplacement.service;

import com.autodesk.quickplacement.service.PositionUpdate;

oneway interface IPositionListener {
    void onPositionUpdate(in PositionUpdate update);
}
```

```aidl
// PositionUpdate.aidl
package com.autodesk.quickplacement.service;

parcelable PositionUpdate;
```

`PositionUpdate.kt` (Parcelable):

```kotlin
data class PositionUpdate(
    val x: Float,              // PDF-local X (units TBD — confirm with host app)
    val y: Float,              // PDF-local Y
    val headingRadians: Float, // 0 = PDF-up; clockwise positive (TBD with host)
    val confidence: Float,     // 0..1; decays over time, spikes on user anchor
    val timestampNanos: Long   // SystemClock.elapsedRealtimeNanos()
) : Parcelable
```

## State Machine

| State | Enter on | Streaming? | Exit on |
|---|---|---|---|
| `IDLE` | service start | no | `initTracking()` |
| `AWAITING_CALIBRATION` | `initTracking()` | no (counts steps in background) | `updatePosition()` |
| `TRACKING` | `updatePosition()` (after calibration math) | yes | `stopTracking()` or unbind |
| `STOPPED` | `stopTracking()` | no | `initTracking()` again |

In `AWAITING_CALIBRATION`, the service is already running step detection and recording sensor data so it can compute stride length the moment the second anchor arrives. It does NOT stream positions during this state.

## Open Questions for the Host App Author

1. **PDF coordinate system** — pixels, meters, or a custom transform applied by the PDF viewer? The service needs to know how `x, y` map to meters so dead reckoning produces matching units.
2. **North direction on the PDF** — does the floor plan have a known geographic orientation, or do we treat heading as purely PDF-local (i.e., we don't care about the magnetic compass, only the relative rotation since `updatePosition()`)?
3. **Update rate the UI can swallow** — 10 Hz / 20 Hz / 30 Hz? Affects PDF redraw cost more than service cost.
4. **Same APK assumption** — confirmed (single-module hackathon project)? If multi-APK is on the table later, AIDL gets a different lifecycle.

These must be answered before AIDL signatures get frozen.
