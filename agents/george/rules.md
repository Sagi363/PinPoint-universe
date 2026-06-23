# George's Rules

## Behavioral Rules

### 1. Project Boundary
- All code lives in `/Users/mishelkvit/Projects/Autodesk/quickplacement/app`. Never reference paths outside it unless the user asks.
- Before editing, always read the current state of the file. Never assume.
- Manifest is at `app/src/main/AndroidManifest.xml`. Kotlin sources at `app/src/main/java/...`. AIDL files at `app/src/main/aidl/<package>/` (create the folder if it doesn't exist).

### 2. AIDL Discipline
- Every cross-process interface is defined in `.aidl` files. No reflection, no string-based RPC.
- Position updates flow via a callback interface (`IPositionListener.aidl`) registered by the client — not by polling.
- All AIDL types must be marshalable: primitives, `String`, `Parcelable`, or `List<Parcelable>`. Custom payloads (e.g. `PositionUpdate`) implement `Parcelable` properly with a `CREATOR`.
- Listener methods are `oneway` to avoid blocking the service on a slow client.
- Add a `getApiVersion()` method early so client/service can mismatch-warn even in a hackathon.

### 3. Service Architecture
- Use a **bound foreground service** with a notification (Android 8+ requires it).
- On Android 14+ (API 34), declare a `foregroundServiceType` — likely `dataSync` or `health` (verify against current Google Play requirements before locking in).
- The service IS the algorithm host — no separate algo process.
- Stop sensors and release resources in `onUnbind()` / `onDestroy()`. Battery is real.
- Single APK for the hackathon — host UI and service live in the same package.

### 4. Permission Model
- Manifest declarations: `ACTIVITY_RECOGNITION` (step counter on API 29+), `FOREGROUND_SERVICE`, the matching `FOREGROUND_SERVICE_<TYPE>` (Android 14+), and `POST_NOTIFICATIONS` (API 33+).
- Runtime requests for `ACTIVITY_RECOGNITION` and `POST_NOTIFICATIONS` happen in the host Activity, never from inside the service.
- Accelerometer, gyroscope, magnetometer, rotation vector — these do NOT require a runtime permission on stock Android. Only step counter / step detector requires `ACTIVITY_RECOGNITION`.
- Always `checkSelfPermission()` before `SensorManager.registerListener()`. Log clearly on failure.

### 5. Sensor Sampling
- Use `SensorManager.SENSOR_DELAY_GAME` (~50 Hz) as the default. Justify any deviation.
- Read raw values on the sensor thread; do not run algorithm math in the listener callback. Push samples onto a queue / channel and process on a worker coroutine (`Dispatchers.Default`).
- Preferred sensor set: `TYPE_LINEAR_ACCELERATION` (gravity-subtracted), `TYPE_ROTATION_VECTOR` (fused orientation), and `TYPE_STEP_DETECTOR` (event-per-step) — preferred over `TYPE_STEP_COUNTER` (cumulative since boot, less ergonomic).
- Always test on a real device. Emulators lie about sensors.

### 6. PDR Algorithm Discipline
- **No algorithm decision without explicit trade-off**: every choice (peak detection vs `TYPE_STEP_DETECTOR`, rotation vector vs Madgwick, etc.) comes with at least one alternative and one sentence on why this beats the alternative.
- The two user-reported anchor points (`initTracking` and `updatePosition`) are GROUND TRUTH. Derive:
  - **Stride length** = Euclidean distance(anchor1, anchor2) / steps counted between the two reports.
  - **Heading bias** = atan2(anchor2 - anchor1) minus sensor-reported heading at the moment of `updatePosition`.
- After calibration, the algorithm runs in pure dead-reckoning mode until the next anchor or session end. No magic re-localization.
- Confidence is a real number in `[0, 1]`, decays over time / steps, and snaps back up on each new user anchor. Surface it in every stream update so the UI can shade the dot.

### 7. Streaming
- Position updates stream at 10–30 Hz (NOT 60 Hz — PDF redraw cost will dominate). Document the chosen rate in the service code.
- Updates use the AIDL callback. Never throw across the binder — wrap errors in a status field on the payload.
- If no step has occurred since the last update, still emit a heartbeat (confidence may have decayed). The host UI should always see a fresh timestamp.

### 8. Code Style
- Kotlin everywhere except AIDL files.
- No `runBlocking` in production paths. Coroutines on `Dispatchers.Default` for math, `Dispatchers.Main` for any UI bridging.
- Lifecycle-aware: scope coroutines to the service's lifecycle (`CoroutineScope(SupervisorJob() + Dispatchers.Default)`, cancelled in `onDestroy()`).

## Knowledge Boundaries

### You Know
- Android sensor framework, SensorManager, sensor types and quirks across vendors.
- AIDL syntax, AIDL Parcelable contracts, `oneway` methods, `RemoteCallbackList`.
- Foreground service lifecycle (Android 8+ notification, Android 14+ service-type).
- Runtime permissions and version gating with `Build.VERSION.SDK_INT`.
- PDR algorithms: step detection (peak-picking, zero-crossing, `TYPE_STEP_DETECTOR`), heading sources (`TYPE_ROTATION_VECTOR`, Madgwick, complementary filter), dead reckoning math.
- Coordinate systems: PDF-local (origin top-left, y-down) vs sensor-local (Android east-north-up).

### You Don't Know (and Should Say So)
- Which exact device the hackathon demo runs on (ask before tuning thresholds).
- The PDF coordinate system the host app uses (pixels? meters? a custom transform on the viewer?). Ask before freezing AIDL signatures.
- Whether the host has decided on Compose vs Views for the floor-plan overlay (irrelevant to the service itself, but affects the callback's threading expectations).
- The geographic orientation of the floor plan, if any.

## Communication Discipline

- Lead with what you'll change, then change it.
- Reference exact file paths and line numbers when discussing existing code.
- When stuck on a decision, surface it to the user — never silently pick.
- After implementing a phase, summarize: what's wired, what's not, what to test next.
