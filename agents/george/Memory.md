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

## M7 — Demo / Test activity (2026-06-22)
- New package: `com.mishelk.quickplacement.demo` (kept off `ui/` so production UI stays clean).
- 6 files: `TestActivity`, `TestViewModel`, `TestUiState`, `TestMode`, `PdfViewer`, `SampleAssets`.
- Test mode FSM: `IDLE → AWAITING_INIT_TAP → AWAITING_CALIBRATION_TAP → TRACKING_REFINE_ON_TAP`. PDF taps only consumed in the three AWAITING_* states.
- Hardcoded demo defaults in `SampleAssets`: `TEST_PX_PER_METER = 100f`, `TEST_PDF_NORTH_RADIANS = 0f`. Asset name `test_floorplan.pdf` in `app/src/main/assets/` (placeholder grid renders if missing — `PdfRenderer` returns null then `drawPlaceholderGrid` paints an 800x1000 sheet with 1 m cells).
- Listener stub forwards `onPositionUpdate` / `onConfidenceUpdate` to ViewModel on `Dispatchers.Main` via a `Handler(Looper.getMainLooper())`. ViewModel maintains a 20-element trail.
- Sheet-px ↔ view-px conversion in `PdfViewer.computeFit`: uniform `min(viewW/sheetW, viewH/sheetH)` scale + centred offsets. Taps outside the sheet rectangle are clamped (ignored).
- TestActivity added as a second LAUNCHER (label "QuickPlacement Test"). Two icons on the home screen; simpler than wiring an Intent inside MainActivity Compose.
- `SensorSampler.start()` now permission-gated: `checkSelfPermission(ACTIVITY_RECOGNITION)` before registering step detector, logs `Log.e` if missing — service never crashes.
- Added `androidx.lifecycle:lifecycle-viewmodel-{ktx,compose}` (version pinned to `lifecycleRuntimeKtx` 2.11.0) — needed for `by viewModels()` delegate.

## M7 gotchas
- `IPositionService.aidl` does NOT yet have `getApiVersion()` (despite the contract spec mentioning one). TestActivity skips the version check; flagged as open question. Adding it would need both AIDL + binder edits.
- `androidx.compose.foundation.layout.weight` is **internal** — `weight` is a RowScope/ColumnScope extension; never import it directly. (Burned 1 build cycle.)
- `PdfRenderer` needs a seekable `ParcelFileDescriptor` → must copy the asset to `cacheDir` first. minSdk 34 accepts the API without ceremony.

## M8 — Field-test fix + compass accuracy (2026-06-22)
- **Heading refinement removed from `updatePosition()` Tracking branch** (PdrEngine.kt ~200-249). Field test showed each post-calibration "Refine Position" tap rotated the dot the wrong way. Root cause: code computed `biasFromWalk = atan2(dx, -dy) − latestAzimuthRad`, but `latestAzimuthRad` is the sensor reading AT THE TAP MOMENT, not the average heading during the walk segment. After a 180° turn the walk vector and post-tap azimuth point in different orientations; EMA-alpha 0.4 then poisoned the calibration (after 3 refines only ~6% of the original survived). Stride EMA is still refined — only `headingBiasRad` is now frozen at first-calibration value (preserved in `s.calibration.copy(strideMeters = ..., lastUpdateMs = ...)`).
- **Compass accuracy surfaced end-to-end**. Chain: `SensorEvent.accuracy` → `SensorSampler.onSensorChanged` writes to `AtomicInteger rotationAccuracy` + stamps `HeadingSample.accuracy` → `PdrEngine.latestCompassAccuracy` (@Volatile Int, default `SENSOR_STATUS_UNRELIABLE`) → every `PositionUpdate.compassAccuracy` (new field at END of Parcel layout — see Parcel layout block in PositionUpdate.kt) → TestActivity StatusBar renders `Compass: HIGH/MEDIUM/LOW/UNRELIABLE` with green/yellow/orange/red, plus italic "Wave phone in a figure-8 to calibrate compass" line when UNRELIABLE.
- **`SensorReading.HeadingSample` gained an `accuracy: Int` field** — middle parameter (azimuthRad, accuracy, timestampMs).
- **Parcelable additive-field discipline**: PositionUpdate.kt now has a "Parcel layout" doc block. New fields go at the END. Default value `SENSOR_STATUS_UNRELIABLE` keeps Kotlin call sites that omit the field source-compatible (used by the engine's `.copy()` calls during heartbeat).
- **`getApiVersion()` impact**: PositionUpdate is wire-compatible only as long as both ends use this build. Once `getApiVersion()` lands, **bump it** whenever this Parcel layout changes — flagged in PositionUpdate.kt header comment.
- **Explicit non-changes** (do NOT undo): sample rate stayed at `SENSOR_DELAY_GAME` ("highest accuracy" is reported by the sensor, not requested — faster sampling ≠ more accurate). No separate `TYPE_MAGNETIC_FIELD` registration: `TYPE_ROTATION_VECTOR` is already the fused accel+gyro+mag estimate. Position updates are NOT gated on accuracy — dot keeps moving even when UNRELIABLE, only the warning surfaces.
- **Heartbeat path** in `PdrEngine.emitStream()` now also refreshes `compassAccuracy` on each 50 ms tick (not just timestamp), so a recovering compass surfaces within one heartbeat instead of waiting for the next step.

## M9 — Step accuracy debounce (2026-06-22)
- Field-test 2 feedback: "we over-step by 1-2 steps sometimes". Hypothesis: tapping the PDF to call `initTracking()`/`updatePosition()` shakes the phone; `TYPE_STEP_DETECTOR` (hardware-fused) interprets the impact as phantom steps. Secondary: vendors occasionally double-fire on a single foot strike.
- Two debounce gates in `PdrEngine.consumeSensorReadings()`, both BEFORE `steps.increment()` so phantoms never reach the position integrator:
  1. **Tap-quiet zone** (`STEP_TAP_DEBOUNCE_MS = 500L`): `initTracking()`/`updatePosition()` set `stepQuietUntilMs = now + 500`. StepEvents with `timestampMs < stepQuietUntilMs` are dropped with `Log.d` "step debounced (tap quiet zone)".
  2. **Min inter-step interval** (`STEP_MIN_INTERVAL_MS = 250L`): drop any step whose delta vs `lastAcceptedStepMs` is <250 ms (anything faster than ~4 Hz cadence is not human). Reset `lastAcceptedStepMs = 0L` on `initTracking()`/`stopTracking()` so the first real step is always accepted.
- Diagnostic log on every ACCEPTED step (after both gates): `Log.d(TAG, "step #${steps.total} at t=${reading.timestampMs}ms")`. Lets us scrape logcat post-walk to inspect cadence.
- **Timestamp model deviation**: task spec referenced `event.timestampNanos` + `SystemClock.elapsedRealtimeNanos()`, but `SensorReading.StepEvent` only carries `timestampMs` (System.currentTimeMillis() — set in `SensorSampler.onSensorChanged`). Constants are therefore in MS, not NS. Unit choice is irrelevant to behaviour; recorded here for the next agent who reads the field-test brief and expects NS constants.
- **Explicit non-changes**: did NOT switch to accelerometer peak-picking, did NOT change sensor sampling rate (step detector is interrupt-driven anyway), did NOT touch stride-length EMA.
- Both `@Volatile` Long vars; written on binder thread, read on worker coroutine — no torn 64-bit reads on 32-bit ARM.

## M11 — Cross-APK enablement landed (2026-06-23)
- `PositionService` now `android:exported="true"`, intent action `com.mishelk.quickplacement.action.BIND_POSITION_SERVICE` declared. FQCN `.service.PositionService` is the cross-APK contract — marked with a "do not move/rename" comment above the manifest entry.
- Custom permission `com.mishelk.quickplacement.permission.BIND_POSITION_SERVICE`, `protectionLevel="normal"` (signature was rejected because build-mobile / quickplacement use different signing certs in this hackathon). Service `android:permission=` requires it; manifest also declares `<uses-permission>` for the same name so our own demo `TestActivity` can still bind. Strings live in `app/src/main/res/values/strings.xml` (label + description).
- `IPositionService.getApiVersion()` added as the FIRST AIDL method → returns literal `1` (constant `PositionServiceBinder.API_VERSION = 1`). Host MUST call this on connect; mismatch is fatal, not a downgrade. Generated AIDL stub confirms `TRANSACTION_getApiVersion = FIRST_CALL_TRANSACTION + 0`. Bump on any wire-format change (incl. `PositionUpdate` Parcel layout changes — see M8 note).
- `PositionServiceBinder` now takes `Context` (passed `applicationContext` from `PositionService.onCreate`) and does a defensive `ContextCompat.checkSelfPermission(ACTIVITY_RECOGNITION)` at the top of `initTracking()` BEFORE the existing `isScaleSet` check. Missing perm → throws `SecurityException` with message pointing the user to grant Physical Activity in the QuickPlacement APK. AIDL KDoc updated to document that Stub exceptions cross the binder.
- `TestActivity.onServiceConnected` now does the API-version handshake before `setScale()`; logs `Bound to PositionService API v1` on success, calls `viewModel.reportBindError(...)` and returns on mismatch (no further state mutation).
- Scope reminder for next agent: AcsSheet host only. PDF host integration is deferred (Atlas's concern). Pin-write path remains untouched. Host commits to `pdfNorthRadians = π/2` because the hackathon sheet has north on the RIGHT.
- `./gradlew app:assembleDebug` green. Not installed, not committed.

## M12 — Hackathon floor-plan asset cleaned (2026-06-21)
- `pdr_test_app` asset `app/src/main/assets/evacuation_map.pdf` stripped for PDR testing: removed all red evacuation overlays (routes, stair boxes, safety icons, "You Are Here" dot), top-right legend box, top-left "EVACUATION PLAN" title + underline, and central "You Are Here" label. Base architectural drawing retained (walls, room numbers, area boxes).
- Output is a high-res raster embedded in PDF (~379 KB). Same sheet geometry as the hackathon evacuation plan; north remains on the PDF-right edge → host `pdfNorthRadians = π/2` still applies.
- Do NOT reintroduce evacuation-route red markup into this asset — it confuses PDR overlay testing against a clean floor plan.
