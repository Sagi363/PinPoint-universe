# PinPoint Walk-Mode POC — State of the Build (2026-06-21)

End-of-session snapshot of the walk-mode POC inside the **Forma Mobile** Android app. Covers feature scope, architecture, what works, what's broken, and what to do next.

Worktree: `/Users/sagihatzabi/Development/Android/build-mobile-pinpoint-walk-mode-poc/`
Branch: `shaggi/pinpoint-walk-mode-poc`
Target device: Pixel 9 Pro Fold (`47031FDKD002DS`), Android 17, build variant `assembleAltDebug` (package `com.plangrid.android.devgrid`).

---

## 1 — Feature scope

**Goal:** rapid placement of issue pushpins on a 2D floor plan, driven by the user "walking" through a space:

1. User opens a sheet, taps a **walk-mode icon** in the markup sidebar.
2. AlloyToast banner: "Select your start position."
3. User taps a point on the sheet → SAW (System Alert Window) **mini-map** opens at the bottom-left, centered on the tap. Blue user-location dot at the anchor. Compass-driven V-shaped direction cone fans out below the dot. Camera V2 launches immediately, mini-map stays floating above it.
4. User points the phone at the issue, taps **capture** → mini-map renders a **red temp marker** at 1m in front of the user in the cone direction.
5. Quick Create screen opens with the **sheet name pre-filled and DISABLED** in the placement field.
6. User fills title + tap **Review** → issue is created AND a real pushpin Placement is attached at the marker coords. IssuesActivity finishes, returning to the sheet viewer; camera re-opens for the next placement.

---

## 2 — Architecture (sheet-px as canonical anchor)

### Coordinate-space contract (LOCKED — do not regress)

```
sheet-px (file space, qozix-pan/zoom-invariant)
   │
   │  M_sheet_to_bitmap  (Matrix.setPolyToPoly from 3 SheetCoordinateMapper.sheetToView samples)
   ▼
bitmap-px (snapshot pixels, captured ONCE at overlay open)
   │
   │  M_bitmap_to_overlay = translate(pan) ∘ scale(zoom)  (rebuilt per frame in WalkModePanView.onDraw)
   ▼
overlay-px (SAW window pixels)
```

**Rules:**
- R1: every coordinate variable carries its space in the name (`sheetPx` / `bitmapPx` / `scaledContentPx` / `overlayPx`).
- R2: ONE sheet→bitmap conversion lives in `WalkModeOverlayController.buildSheetToBitmapMatrix(mapper)`. Do not duplicate.
- R3: ONE bitmap→overlay conversion lives in `WalkModePanView.bitmapToOverlayMatrix`. Layers receive it as data, never recompute.
- R4: NEVER read live qozix scroll/scale at render time — the snapshotted matrix is the truth.
- R5: new spaces require a documented boundary helper, not inline math.

### File layout (`android/app/src/main/java/com/plangrid/android/sheets/walkmode/`)

| File | Role |
|---|---|
| `WalkModeOverlayController.kt` | Owns SAW window lifecycle, snapshot, layer stack, sensor source, marker state. |
| `WalkModePanView.kt` | Custom View hosted in SAW. Owns per-frame matrix composition. |
| `WalkModeOverlayLayer.kt` | Interface — `draw(canvas, viewportOverlayPx, sheetToOverlayMatrix)`. |
| `WalkModeUserLocationLayer.kt` | 3-disc blue dot at the anchor sheet-px. |
| `WalkModeUserDirectionLayer.kt` | V cone (Path + RadialGradient), EMA-smoothed compass rotation. |
| `WalkModeTempMarkerLayer.kt` | Red ring + dot at the projected pushpin position. |
| `WalkModeTempMarker.kt` | Data class — `sheetPx`, `anchorSheetPx`, `headingDeg`, `distanceMeters`. |
| `WalkModePushpinPlacer.kt` | Pure math — `computeMarkerSheetPx(...)`, `resolvePxPerMeter(...)`. |
| `WalkModePositionSelector.kt` | Shared AlloyToast banner helper. |
| `HeadingSource.kt` | Interface (`StateFlow<Float?>`) — future-extensible. |
| `CompassHeadingSource.kt` | `TYPE_ROTATION_VECTOR` + `SENSOR_DELAY_GAME` impl. |

### Z-order (bottom → top in `panView.addLayer(...)`)

1. Tile bitmap (drawn in `onDraw`)
2. Annotation overlay via `drawFiltered` (ISSUE + PHOTO excluded)
3. `WalkModeUserDirectionLayer` (cone)
4. `WalkModeTempMarkerLayer` (red marker)
5. `WalkModeUserLocationLayer` (blue dot — covers cone apex visually)
6. Rounded-rect stroke frame

### Cross-Activity signal

`WalkModeBridge` (`android/domain/src/main/java/com/plangrid/android/domain/walkmode/WalkModeBridge.kt`) — `@Volatile var awaitingCameraReopen: Boolean = false`. Placed in `:android:domain` because both `:android:app` (sheet/PDF hosts) and `:android:issues` (Quick Create) need it without a circular dep.

### Camera + Quick Create integration

- Camera V2 launched via `CameraActivity.getLaunchIntent(context, outputDir, projectUid, CameraFeatureMode.QuickCreate.SingleCapture)`. SingleCapture auto-finishes after one capture (`CameraFragmentV2.kt:582-584`).
- Result observed via `registerForActivityResult(ActivityResultContracts.StartActivityForResult())` registered at field-init time on the host fragment.
- On `RESULT_OK`: `controller.onCaptureRequested()` snapshots heading + projects 1m ahead → marker stored.
- Then host launches `IssuesActivity` with `Intent` carrying `LAUNCH_FLOW = IssuesConstants.WALK_MODE_QUICK_CREATE` + placement extras (`hasWalkModePlacement`, `placementSheetUid`, `placementSheetName`, `placementX`, `placementY`).
- `IssuesActivity.onCreate` (`android/app/src/main/java/com/plangrid/android/issues/IssuesActivity.kt:120-126`) reads the flow and calls `navController.navigate(R.id.navigate_quick_create, Bundle(intent.extras).apply { putBoolean(IS_ISSUE_ACTIVITY_CONTAINED, true) })`.
- Quick Create nav-args were extended in `android/issues/src/main/res/navigation/navigation_issues.xml:540-602` with the 5 walk-mode args.
- `QuickCreateFragment` reads `walkModePlacement: WalkModePlacement?` when `navArgs.hasWalkModePlacement`. UI locks the placement field via `RenderData.isPlacementLocked` + `isPlacementEnabled = false`.
- Mobius Effect `LinkWalkModePushpin` is emitted after `Action.Internal.IssueCreated` (Updater at `QuickCreateUpdater.kt:418-429`). Processor handler at `QuickCreateProcessor.kt:452-475` calls `v2AnnotationRepository.linkWalkModePushpin(issueUid, sheetUid, sheetPxX, sheetPxY)`.
- Public wrapper at `V2AnnotationRepository.kt:706-747` looks up sheet dimens via `sheetRepository.getSheetByUid(...).pugSize`, builds a `PgbsAnnotation` via `MarkupCreator.createViaTap(PointerEvent(Vector(x, y)), PgbsType.ISSUE_STAMP, ...)`, delegates to the existing private `linkIssueAnnotation(..., ExistingIssuePayload(issueUid, "", null, false))`.

### Sensor source

- `Sensor.TYPE_ROTATION_VECTOR` + `SENSOR_DELAY_GAME` (~50 Hz). OS sensor-fusion handles accel+gyro+magnetometer; we never see raw magnetometer.
- Remap to display rotation via `SensorManager.remapCoordinateSystem(...)` on every event.
- Smoothing: circular EMA with τ = 80 ms inside `WalkModeUserDirectionLayer.setRawHeading`. Wrap-aware delta in `[-180, 180]`. Snap on first sample.
- Cone rotation: `canvas.rotate(headingDeg, cx, cy)` around the anchor inside `layer.draw()`. NOT in the matrix chain — keeps the coordinate contract pure.
- Magnetic north only (no calibration to true north or sheet north). Documented POC limitation.

### Marker projection math (1m in front of user)

```kotlin
val r = distanceMeters * pxPerMeter
val rad = Math.toRadians(headingDeg.toDouble())
markerSheetPx.x = anchorSheetPx.x + (r * sin(rad)).toFloat()
markerSheetPx.y = anchorSheetPx.y - (r * cos(rad)).toFloat()   // sheet y grows DOWN; heading 0° = north = -Y
```

Verified for 0° (north → up), 90° (east → right), 180° (south → down), 270° (west → left). Matches the cone orientation (cone path starts at canvas angle -90° = -Y).

`pxPerMeter` = `100f` constant fallback (`WalkModePushpinPlacer.FALLBACK_SHEET_PX_PER_METER`). TODO: read `V2AnnotationRepository.getSurfaceCalibration(surfaceUid)` for calibrated sheets.

---

## 3 — What works end-to-end (verified on Pixel 9 Pro Fold)

- ✅ Walk-mode icon armed/disarmed state in the shared `AnnotationToolbar`.
- ✅ "Select your start position" banner + tap-to-confirm flow.
- ✅ SAW mini-map opens at bottom-left, centered on tap, with rounded corners + stroke frame.
- ✅ Auto-refresh: new markers added on the live sheet repaint in the mini-map (R4 OnDrawListener pattern).
- ✅ ISSUE + PHOTO pins filtered out of the mini-map snapshot.
- ✅ Blue user-location dot at anchor (3-disc composite, fixed pixel sizes).
- ✅ Compass-driven V cone with EMA smoothing, gradient apex-blue → transparent, drawn below the dot.
- ✅ Cone outer radius bumped to `41dp` (was `24dp`) per user preference.
- ✅ Camera V2 single-shot mode auto-finishes; SAW survives Activity transition (`TYPE_APPLICATION_OVERLAY` is window-manager-scoped, not Activity-scoped).
- ✅ Quick Create opens with pre-filled DISABLED placement field showing the sheet name.
- ✅ Marker only renders when the camera capture button is tapped (was previously rendering at camera launch — fixed 2026-06-21).
- ✅ Build + install pipeline: `./gradlew :android:app:assembleAltDebug` ≈ 1 min, `adb install -r -d` clean.

---

## 4 — What's still broken

### Review tap → app crash (UNRESOLVED at end of session)

**Symptom:** Tapping the "Review" button in walk-mode-driven Quick Create crashes IssuesActivity.

**Stack (truncated):**
```
java.lang.NullPointerException
  at IssueDetailsV2FragmentKt.getIssueReferencesSectionConfig(IssueDetailsV2Fragment.kt:185)
  at IssueDetailsV2Fragment.onCreateView$lambda$0$0$0$0(IssueDetailsV2Fragment.kt:124)
  ...
```

Line 185: `backStackEntry = projectLevelNavController.currentBackStackEntry!!` — null because IssuesActivity was launched direct-into Quick Create without the normal master-detail bootstrap that populates the back-stack.

**Root path (confirmed via Timber stack-trace diagnostic):**
1. Review → `Updater.IssueCreated` → `Updater.ReviewComplete` → emits `Event.NavigateToIssueDetails(uid)`.
2. `QuickCreateScreenHost` collector → calls `eventHandler.onNavigateToIssueDetails(uid)`.
3. **My walk-mode early return at `QuickCreateFragment.kt:234` SHOULD fire but doesn't.**
4. Execution reaches `QuickCreateFragment.kt:281`: `masterDetailViewModel.openDetails(uid)` (tablet auto-select branch).
5. This sets `pendingDetailIssueUid` on the activity-scoped `MasterDetailViewModel`.
6. `IssuesMasterDetailFragment.onViewCreated`'s collector at `IssuesMasterDetailFragment.kt:259-264` fires `openDetails(uid)`.
7. `openDetails` instantiates `IssueDetailsV2Fragment` in the split-view detail pane (tablet/foldable).
8. Fragment renders, NPE at line 185.

**The puzzle:** the diagnostic stack proves line 281 is reached. My check at line 234 (`if (navArgs.hasWalkModePlacement) { ... return@QuickCreateEventHandler }`) should prevent that. So `navArgs.hasWalkModePlacement` evaluates **false** at the moment the lambda runs — despite IssuesActivity passing `putExtra("hasWalkModePlacement", true)` through the Intent into `Bundle(intent.extras)` and the nav graph declaring the arg.

**Next-session debug:** add `Timber.i("hasWalkModePlacement=${navArgs.hasWalkModePlacement} placementSheetUid=${navArgs.placementSheetUid}")` at the very top of `onNavigateToIssueDetails` (BEFORE the gate check), reproduce, inspect the logged values. If false, the bundle round-trip through `setGraph + navigate` is dropping the key. If true, something else is calling `openDetails` and we missed it.

### Failed approaches (do not retry without new evidence)

1. **`setStartDestination(R.id.fragment_quick_create)` on `navigation_project`** — throws `IllegalArgumentException: navigation destination ... is not a direct child of this NavGraph` because Quick Create is nested inside the `navigation_issues` sub-graph, not directly under `navigation_project`.
2. **Replacing `!!` on line 185 of `IssueDetailsV2Fragment` with a graceful fallback** — hides the symptom (NPE) but the fragment still renders in a half-broken state. Prevention must happen at the source: don't instantiate `IssueDetailsV2Fragment` for walk-mode flow.

### Lower-priority gaps

- **100 px/m calibration stub.** Real sheet calibration via `V2AnnotationRepository.getSurfaceCalibration(surfaceUid)` is a follow-up; current marker distance is sheet-scale-agnostic.
- **Magnetic north only.** No declination correction (true north) or sheet-orientation calibration (sheet-up may not be north in real building floor plans).
- **PDF host pushpin attach.** PDF host's `surfaceUid` is the doc-version uid, not a true sheet uid; the linkWalkModePushpin call would silently no-op on a wrong surface uid. Only the Acs sheet host gets the full attach flow.
- **Camera permission gate.** Not added upfront; relies on Camera V2's own runtime CAMERA permission flow. The SAW overlay may obscure the permission dialog on first-ever launch.
- **`startActivity` not `startActivityForResult` for IssuesActivity launch.** We use a process-scoped `WalkModeBridge` singleton instead. Acceptable for POC, but production should switch to a launcher so cancellation semantics are explicit.

---

## 5 — Architectural pitfalls (capture these or step on them again)

### Tablet master-detail auto-select trap

`IssuesMasterDetailFragment.onViewCreated` at `android/issues/src/main/java/com/plangrid/android/issues/masterdetail/IssuesMasterDetailFragment.kt:259-264` installs a permanent collector:

```kotlin
viewModel.pendingDetailIssueUid
    .filterNotNull()
    .collect { issueUid ->
        openDetails(issueUid)
        viewModel.consumePendingDetail()
    }
```

Any code that calls `MasterDetailViewModel.openDetails(uid)` on the activity-scoped VM triggers auto-rendering of `IssueDetailsV2Fragment` in the split-view detail pane. Known call sites:
- `QuickCreateFragment.kt:218` — `onNavigateToIssueTab` tablet auto-select.
- `QuickCreateFragment.kt:262` — `onNavigateToIssueDetails` `fromProjectHome` branch.
- `QuickCreateFragment.kt:281` — `onNavigateToIssueDetails` non-fromProjectHome tablet auto-select.
- `IssuesListFragment.kt:274` — `onIssueClicked` from the issues list.

Every new flow that touches Quick Create on tablet/foldable must gate these call sites or accept that `IssueDetailsV2Fragment` will render.

### `IssueDetailsV2Fragment.kt:185` NPE

`projectLevelNavController.currentBackStackEntry!!` — null when the fragment is instantiated before the project-level NavController has a populated back-stack. This is reached whenever IssuesActivity is launched directly into a deep destination (deep link, our walk-mode flow) without the normal master-detail → list → detail sequence.

### `SingleCapture` mode quirks

`CameraFeatureMode.QuickCreate.SingleCapture` auto-finishes after the first photo (`CameraFragmentV2.kt:582-584`). No need to handle a "done" button. Result intent has no extras; photos are pulled from `MultiCaptureFetcher.getOutputDirectory(fileSystem)`.

### Cross-graph navigation requires Intent-routing

`AcsSheetFragment.findNavController()` is scoped to the sheet's nav graph — does NOT contain `R.id.navigate_quick_create`. Calling it directly throws. Route via `IssuesActivity` Intent + `LAUNCH_FLOW = WALK_MODE_QUICK_CREATE`.

### Heading-snapshot timing matters

Capture the device heading at the moment the **camera's capture button** is tapped, not when the camera launches. The user may rotate the phone between launch and capture. Marker projection uses the heading at the moment of `RESULT_OK`.

---

## 6 — Key external references

- `CameraFeatureMode.QuickCreate.SingleCapture` — `android/core/src/main/java/com/plangrid/android/core/camera/CameraFeatureMode.kt:17`
- `CameraActivity.getLaunchIntent` — `android/app/src/main/java/com/plangrid/android/camera/activities/CameraActivity.kt:223-234`
- `V2AnnotationRepository.linkWalkModePushpin` (new public wrapper) — `android/annotations/src/main/java/com/plangrid/android/annotations/V2AnnotationRepository.kt:706-747`
- `MarkupCreator.createViaTap` for 2D ISSUE_STAMP — usage example at `android/annotations/src/test/java/com/plangrid/android/annotations/V2AnnotationRepositoryPublishByDefaultTest.kt:113-124`
- `Placement` data model — `pgf/feature/annotations/annotations-data/.../Placement.sq:12-33` (transform BLOB AS Matrix; translation.x/y = sheet-px)
- `IssuePlacement` link table — `pgf/feature/issues/issues-data/.../IssuePlacement.sq:10-17`
- `Calibration` table (unused this turn) — `Calibration.sq:6-26`
- `IssueDetailsV2Fragment` NPE site — `android/issues/src/main/java/com/plangrid/android/issues/details/IssueDetailsV2Fragment.kt:185`
- `IssuesMasterDetailFragment` auto-select collector — `android/issues/src/main/java/com/plangrid/android/issues/masterdetail/IssuesMasterDetailFragment.kt:259-264`

---

## 7 — Build / install / test commands

```bash
# Build
cd /Users/sagihatzabi/Development/Android/build-mobile-pinpoint-walk-mode-poc
./gradlew :android:app:assembleAltDebug

# Install on Pixel
adb -s 47031FDKD002DS install -r -d \
  android/app/build/outputs/apk/alt/debug/app-alt-debug.apk

# Watch walk-mode logs
adb -s 47031FDKD002DS logcat -d | grep -E "(walkmode|PinPoint POC|walkmode-temp|walkmode-diag)" -A 5

# Pull crash stack
adb -s 47031FDKD002DS logcat -d | grep "FATAL EXCEPTION" -A 30
```

Package id on alt variant: `com.plangrid.android.devgrid` (not `com.plangrid.android` — multiple times we've installed the right APK and looked at the wrong package).

Display id for inner screen on Pixel 9 Pro Fold: `4619827677550801152` (passed to `screencap -d` when the auto-default picks the outer cover screen).
