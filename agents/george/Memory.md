# George's Memory

## Project facts
- Path: `/Users/mishelkvit/Projects/Autodesk/quickplacement` (single-module `:app`).
- Package: `com.mishelk.quickplacement` (NOT `com.autodesk.*`).
- minSdk 34, targetSdk 36, compileSdk **bumped to 37** (core-ktx 1.19.0 required it).
- AIDL lives at `app/src/main/aidl/com/mishelk/quickplacement/service/`.

## 5 locked deltas
1. **Strict ordering**: `initTracking()` throws `IllegalStateException("setScale() must be called before initTracking()")` if scale unset. Enforced in `PositionServiceBinder`.
2. **Trust pxPerMeter**: no `isScaleEstimated` flag, no 100f fallback. Host commits to a real value.
3. **setScale carries north**: new signature `setScale(float pxPerMeter, float pdfNorthRadians)`. Walk-derived bias wins; `pdfNorthRadians` is a sanity warning only (log if `|biasFromWalk − pdfNorthRadians| > π/4`).
4. **No AIDL min-distance** on `updatePosition`. Algorithm guard: `stepCount < 3` → fallback stride 0.75 m.
5. **Confidence floor 0.1** (no "lost" state).

## Architecture
- Sensors: `TYPE_STEP_DETECTOR` + `TYPE_ROTATION_VECTOR` only. Magnetometer fused into rotation vector — never registered directly.
- Sensor callbacks push to `Channel<SensorReading>(UNLIMITED)`; worker coroutine on `Dispatchers.Default` consumes. Service-scoped `SupervisorJob`, cancelled in `onDestroy()`.
- 20 Hz streaming loop (50 ms `delay`) emits heartbeat even with no step. `RemoteCallbackList<IPositionListener>` for dispatch.
- Confidence: 1.0 on each anchor, decays 1 % per metre walked, floored at 0.1.
- Position math (PDF coords, +y down, heading 0 = toward PDF-top): `dx = stride·pxPerMeter·sin(h)`, `dy = −stride·pxPerMeter·cos(h)`.

## Build gotchas
- `kotlin-parcelize` plugin would not apply cleanly under AGP 9.2.1 + Kotlin 2.2.10 (`Unresolved reference 'parcelize'`). **Worked around by hand-writing `Parcelable` for `PositionUpdate`**; no plugin needed. Revisit if more parcelables get added.
- `compileSdk = 36` rejected by core-ktx 1.19.0; raised to 37.
- Coroutines are not in `libs.versions.toml`; added inline (`org.jetbrains.kotlinx:kotlinx-coroutines-{core,android}:1.9.0`).

## Milestones completed
M1 (AIDL+gradle), M2 (service+manifest+perm UI), M3 (sensors+state), M4 (calibration math+dead reckoning), M5 (20 Hz stream+confidence). **M6 skipped — needs real device.**
