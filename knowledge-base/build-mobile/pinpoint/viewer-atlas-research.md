# Forma 2D Viewer — Atlas Knowledge Base

> Code-grounded reference for the Atlas agent. Every claim cites `file:line`. Generated 2026-06-18 by Rick's research dispatch.

---

## 0. Glossary

- **AV2** — *AV2Foundation* (a.k.a. **AV2F**). An external KMP/JVM library (Gradle coordinate `com.plangrid:AV2Foundation`, version `52.0.0-dev.1863` per `gradle/libs.versions.toml:154,424`) that owns the annotation/markup state machine, scene graph, and tool model used inside this repo. NOT "Autodesk Viewer v2". The "annotations" Android module is the thin host that adapts AV2F into the app; all annotations (including 2D issue pins) live as `com.plangrid.av2foundation.annotations.PgbsAnnotation` and are rendered by walking an AV2F `SceneGroup`.
- **AV2F** — Short for AV2Foundation; the term used in source comments (`V2AnnotationView.kt:211`, `V2AnnotationModule.kt:128`).
- **PGBS** — The PlanGrid Brush/Shape vector format inside AV2F (`PgbsLine`, `PgbsRect`, `PgbsEllipse`, `PgbsPolyline`, `PgbsPolygonElement`, `PgbsPathElement`, `PgbsTextElement`). All markups are made of these primitives plus a transform.
- **Elephant** — The modern native Forma viewer (`com.plangrid.android:elephant`, version `2026.06.11-11.32-main-6707e4e` per `gradle/libs.versions.toml:155,425`). Ships an `ElephantViewer` Compose composable that renders both 2D and 3D viewables (PDF, F2D, HLOD/BVH, etc.). Uses native rendering, not WebView. Co-developed with the Autodesk Elephant ESM (`autodesk-elephant-esm = 2026.01.30-15.06-main-259317f`, libs.versions.toml:73).
- **LMV** — *Large Model Viewer*. The legacy Autodesk WebView-based viewer (`com.autodesk.lmv.bridge.LmvWebFragment`). Used only via `android/lmv/`; superseded by Elephant for the modern app. References: `LmvFragment.kt:30` imports `LmvWebFragment`.
- **LLMV** — `android/llmv/` is a stale skeleton module (empty `src/`; only build outputs remain). Do NOT treat it as a viewer.
- **PGFoundation (PGF)** — Kotlin Multiplatform shared layer under `pgf/`. Hosts sync, repositories, SQLDelight schema. Modern 2D viewer surfaces (`pgf/feature/issues/issues-internal/...` for issue pins; `pgf/feature/annotations/...` for feature pins).
- **Punch Walk / "Place Me"** — Two-finger navigation features tied to **Elephant first-person mode** and the **LMV "drop me"** workflow. On the modern path, Punch Walk lives entirely inside the Elephant viewer (see `Preferences.kt:123-127`, `ElephantViewModelFactory.kt:49,65`). Legacy LMV had `LmvDropMeListener`/`LmvFirstPersonListener` (`LmvFragment.kt:69,71`).
- **Sheet** — A page/derivative of a document (PDF, F2D). Modern sheet entity = a *Viewable* of a *Document* surfaced through `com.plangrid.elephant.uistate.interfaces.Viewable` + `ViewableInfo`. Legacy PG sheet = `com.plangrid.android.domain.sheets.model.SheetViewable` driving the `SheetView2` + tile-pyramid pipeline.
- **Issue Pin** — A `PgbsAnnotation` whose `boundEntityId` is a `GGIssueUid` and whose persistence path goes through `IssuePinRepository.create2DIssuePin` / `create3DIssuePin` (`IssuePinRepositoryImpl.kt:118,80`).
- **Surface** — Logical canvas the markup lives on. Identified by `SurfaceUid`. For 2D Elephant viewables the surface UID is derived from `documentUid + viewableGuid` (`ElephantAnnotationsOverlayPresenter.kt:224-228,295`).
- **PUG file** — Legacy PlanGrid tiled-PDF artifact (`SheetView2.java:307` `obtainRenderer(File pugFile)`). Source of pixels for the legacy `SheetView2` tile renderer.
- **HLOD/BVH** — Hierarchical LOD bounding volumes binary for 3D derivatives (`ElephantFragment.kt:140`, `ModelFileType.HLOD_BVH -> BinaryType.VIEWABLE_HLOD_BVH`). Loaded on demand by the Elephant viewer.

---

## 1. The Big Picture (1-page summary)

- The "modern Forma 2D viewer" is **NOT** in this repo as a renderer. The actual pixel rendering for 2D viewables (and 3D) lives in the external `com.plangrid.android:elephant` artifact, hosted via the `ElephantViewer` Compose composable invoked at `android/elephant/src/main/java/com/plangrid/android/elephant/ui/ElephantFragment.kt:377-389`.
- Two **separate** 2D viewer code paths coexist in the binary:
  1. **Modern (Elephant) path** — used by GG Issues / modern Build / Forma. Entry: `ElephantMarkupsExtensionFragment` (extends `ElephantFragment`). Renders pixels via `ElephantViewer` (native, in-process). Drives a thin `SheetView2` instance in "elephantDriverMode(true)" purely for **gesture + viewport state** — the tile canvas is hidden (`SheetView2.java:141-156`).
  2. **Legacy (PUG + tiles) path** — used by older PlanGrid sheets. Entry: `AcsSheetFragment` + `AcsSheetPresenter` + `MarkupsFileViewerFragment`. `SheetView2 extends PGTileView extends com.qozix.tileview.TileView` (`SheetView2.java:60`, `PGTileView.java:33`). Tiles streamed from a PUG file via `TileBitmapRendererFactory`.
- **Markups (annotations + issue pins)** are unified across both paths via **AV2Foundation**. `V2AnnotationView` is a native Android `View` (custom `onDraw`) that paints the AV2F `SceneGroup` tree on top of whichever viewer is active. Tap coordinates are mapped to **unit coordinates** (`x / surfaceWidth`) before reaching the AV2F state machine.
- **PinPoint relevance**: Auto-placing an issue pin on a 2D sheet means producing a `PgbsAnnotation` (with a `Transform`) plus a `DraftingContext` + `SurfaceUid` and calling `IssuePinRepository.create2DIssuePin` (`IssuePinRepositoryImpl.kt:118`). The position is normalized by `surfaceWidth` (`FoundationExt.kt:9-13`). Coordinates that flow into PGFoundation are floats in **AV2F unit space**, not pixels.

---

## 2. A — Entry points & navigation

### Modern viewer entry (used by GG Issues + Forma + PinPoint targets)

- Compose host: `ElephantViewer(...)` invoked at `android/elephant/src/main/java/com/plangrid/android/elephant/ui/ElephantFragment.kt:377-389`:

```kotlin
// elephant/src/main/java/com/plangrid/android/elephant/ui/ElephantFragment.kt:377
ElephantViewer(
    elephantStateMachine = elephantViewModel.elephantStateMachine,
    drawerStateMachine = elephantViewModel.drawerStateMachine,
    toolbarStateMachine = elephantViewModel.toolbarStateMachine,
    renderContext = elephantViewModel.withInitializedViewStateOrError { renderContext },
    providedStrings = StringsProvider(requireContext()),
    drawerProvider = DrawerProvider(...),
    closeViewer = ::closeViewer,
    extension = elephantExtension
)
```

- For 2D viewables, the extension `ElephantMarkupsExtensionFragment` (extends `ElephantFragment`) attaches markup overlays. It is registered via `@FragmentKey(ElephantMarkupsExtensionFragment::class)` and `@ContributesMultibinding(ActivityGraph::class, ElephantExtension::class)` (`ElephantMarkupsExtensionFragment.kt:158-161`).
- Nav args: `ElephantFragmentArgs(documentUid, revision, bubblePath, dbPath, fileTitle, texturePath, viewableGuid, boundEntityId, preSelectedElement, lineageUrn)` — see `android/elephant/build/generated/source/navigation-args/debug/com/plangrid/android/elephant/ui/ElephantFragmentArgs.kt:16`.

### Legacy PG sheet entry

- Navigation graph: `android/app/src/main/res/navigation/navigation_acs_sheets.xml:11` → fragment `com.plangrid.android.sheets.ACSSheetsFragment` (collection) → opens `AcsSheetFragment` for an individual sheet.
- `AcsSheetFragment` is registered with `@FragmentKey(AcsSheetFragment::class)` (`AcsSheetFragment.kt`); it holds `SheetView2`, an `ElephantView`, a `V2AnnotationView`, plus all the mobius VMs (see `AcsSheetFragment.kt:447-450`).
- Legacy PDF document viewer: `com.plangrid.android.pdf.MarkupsFileViewerFragment` (registered in `navigation_markups_file_viewer.xml:11`), `MarkupsPdfActivity` (`AndroidManifest.xml:378`).

### "Current sheet" state

- For the **modern path** the current viewable state is held inside `ElephantViewModel` (`com.plangrid.elephant.ElephantViewModel`), wrapped with `viewModels<ElephantViewModel>()` via `ElephantViewModelFactory` (`android/components/src/main/java/com/plangrid/android/components/elephant/ElephantViewModelFactory.kt:16-50`). The state is observed as a `kotlinx.coroutines.flow.Flow<ElephantState>` (e.g. `elephantViewModel.elephantStateMachine.states.flowWithLifecycle(...)` in `ElephantMarkupsExtensionFragment.kt:362-369`).
- For the **legacy path** the state is in `SheetActivityViewModel` + `SheetViewerMobius` (Mobius-style ViewModel) — see `AcsSheetFragment.kt:431-432`:

```kotlin
// app/.../sheets/AcsSheetFragment.kt:431
private val sheetActivityViewModel by activityViewModels<SheetActivityViewModel> { sheetActivityViewModelFactory }
private val sheetViewerMobius by activityViewModels<SheetViewerMobius> { sheetViewerMobiusFactory }
```

### PG Tasks vs GG Issues split

- **PG Tasks** (legacy PlanGrid issues) use `IssueDetailFragment` and the legacy `SheetView2` + `V2AnnotationView` overlay path through `AcsSheetFragment`. No Compose viewer.
- **GG Issues** (modern) use `IssueTypesViewModel.createIssueAnnotation` / `createIssueAnnotationFromTemplate` (`android/issues/src/main/java/com/plangrid/android/issues/type/IssueTypesViewModel.kt:556,585`) and flow into `V2AnnotationRepository.createIssueAnnotation` → `IssuePinRepository.create2DIssuePin` (`V2AnnotationRepository.kt:560-590`).

---

## 3. B — Rendering pipeline

### Modern 2D rendering (the one PinPoint targets)

- Pixels are produced **inside the Elephant library** (closed source from this repo's perspective — the artifact is `libs.plangrid.android.elephant`). The composable `ElephantViewer` is the call site (`ElephantFragment.kt:377`).
- For 2D viewables, `ElephantViewer` consumes derivatives — the analytics enum lists the file formats:

```kotlin
// android/elephant/src/main/java/com/plangrid/android/elephant/ui/AnalyticsReceiver.kt:372
ElephantAnalyticsViewableType.F2D -> DerivativeType.F2D
```

So 2D inside Elephant is **F2D** (Autodesk's 2D Forge derivative). 3D uses `HLOD_BVH` (`ElephantFragment.kt:140`).

- The **markup overlay** drawn on top of Elephant pixels is rendered in this repo by `V2AnnotationView extends android.view.View` (`android/annotations/src/main/java/com/plangrid/android/annotations/views/V2AnnotationView.kt:153-154`). It does its own `onDraw`:

```kotlin
// annotations/.../views/V2AnnotationView.kt:1676
@SuppressLint("CanvasSize")
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    if (surfaceDimens.first != 0f && surfaceDimens.second != 0f) {
        // NOTE: Do not use canvas.setMatrix in any draw routine, or it will break screenshot exporting.
        val initialState = canvas.save()
        sceneGroup?.renderNodes
            ?.reversed()
            ?.forEach { node -> drawNode(node, surfaceContext, canvas) }
        canvas.restoreToCount(initialState)
    }
}
```

- The scene graph is produced by AV2F: `sceneGroup = SceneBuilder.create(globalState = data.globalState)` (`V2AnnotationView.kt:831,868`).
- The viewer is **synced with the SheetView2 zoom/pan** via `renderContext.syncWithScrollView(scale, contentOffset, screenDensity)` (`ElephantAnnotationsOverlayPresenter.kt:380-387`).

### Legacy 2D rendering (tile pyramid)

- `SheetView2 extends PGTileView extends com.qozix.tileview.TileView` (`SheetView2.java:60`).
- `TILE_SIZE = 512` (`SheetView2.java:62`).
- Tile pixels come from a PUG file rendered through `TileBitmapRendererFactory.getTileBitmapRenderer(pugFile, bitmapLruCache, paddingColor)` (`SheetView2.java:314`).
- Backing tile-view code is the vendored qozix library in `android/tile-view/src/main/java/com/qozix/tileview/...` — `TileCanvasViewGroup`, `DetailLevelManager`, `BitmapProvider`, `RefCounted`, etc. (See `find` listing in `android/tile-view/`.)
- Low-res placeholder image is loaded via Picasso into `lowResImage: ImageView` (`SheetView2.java:64,100`).

### "Elephant driver mode"

When the modern viewer is open, `SheetView2.elephantDriverMode(true)` HIDES the tile canvas but keeps the `SheetView2` alive as a touch/viewport surrogate:

```java
// app/.../sheets/views/SheetView2.java:141
public void elephantDriverMode(Boolean isEnabled) {
    isVector = isEnabled;
    if (isEnabled) {
        setBackgroundColor(Color.TRANSPARENT);
        getTileCanvasViewGroup().setVisibility(View.INVISIBLE);
        getThumbnailView().setVisibility(View.INVISIBLE);
        setScaleLimits(getMinScale(), getMinScale()*vectorSheetMaxZoom);
    } else { ... }
    zoomPanStateSubject.onNext(currentZoomPanState());
}
```

Switched on by `ElephantAnnotationsOverlayPresenter.setupObservers` (`ElephantAnnotationsOverlayPresenter.kt:369`) and `AcsSheetPresenter.kt:119,127`.

### AV2 verification

- AV2 = **AV2Foundation**, not Autodesk Viewer V2. Evidence:
  - Gradle dep `plangrid-av2foundation = { module = "com.plangrid:AV2Foundation", version.ref = "plangridAV2Foundation" }` (`gradle/libs.versions.toml:424`).
  - 546 source references to package `com.plangrid.av2foundation.*` in `android/`. Examples: `V2AnnotationView.kt:101-131` imports `com.plangrid.av2foundation.annotations.PgbsAnnotation`, `...av2foundation.scene.SceneBuilder`, `...av2foundation.state.GlobalState`, etc.
  - Comments in code spell it out: `// AV2F's state managers (ClientApp / Tool / Markups / Palette / FeatureFlag)` (`V2AnnotationTestClasses.kt:151`, `V2AnnotationModule.kt:128`).
  - Feature flag names: `AV2_2D_ISSUE_PIN_CREATION`, `AV2_V2_USE_VIEW_VIEW_MODEL`, `AV2_DASHED_LINE_STROKES`, etc. (`V2AnnotationFeatureFlag.kt:30-81`).

---

## 4. C — Coordinate systems (CRITICAL)

There are **four** coordinate spaces relevant to a 2D auto-pin:

```
+----------+   sheetToView    +----------+   layout    +----------+
| sheet px |  ------------>   | view px  |  -------->  | window   |
+----------+   viewToSheet    +----------+             +----------+
     |  divide by surfaceWidth                              |  rawX/rawY
     v                                                      v
+----------+      mobius API uses                +-----------------+
|  AV2F    |  <----------------------------------|  MotionEvent    |
|  unit    |   FoundationVector(x, y).toUnit()   |  raw coordinates|
+----------+                                     +-----------------+
```

### sheet ↔ view (pixels)

`SheetCoordinateMapper` is the canonical pixel-space transform (`android/domain/src/main/java/com/plangrid/android/domain/helpers/SheetCoordinateMapper.kt:18-92`). It is constructed with `(sheetSize: Size, tileView: TileView)` and reads live state from `tileView.detailLevelManager.viewport` and `tileView.scale`.

Forward (sheet → view):

```kotlin
// domain/.../SheetCoordinateMapper.kt:38
fun sheetToView(sheetCoordinates: PointF, clampToView: Boolean): PointF {
    val viewport: Rect = tileView.detailLevelManager.viewport
    val scale: Float = tileView.scale
    var viewCoordinates = PointF(
        sheetCoordinates.x * scale + getSheetXOffset(viewport, scale) - viewport.left,
        sheetCoordinates.y * scale + getSheetYOffset(viewport, scale) - viewport.top
    )
    if (clampToView) viewCoordinates = clampToView(viewCoordinates, tileView)
    return viewCoordinates
}
```

Inverse (view → sheet) — **this is the function used for tap-to-pin**:

```kotlin
// domain/.../SheetCoordinateMapper.kt:51
fun viewToSheet(viewX: Float, viewY: Float): PointF {
    val viewport: Rect = tileView.detailLevelManager.viewport
    val scale: Float = tileView.scale
    return PointF(
        (viewX - getSheetXOffset(viewport, scale) + viewport.left) / scale,
        (viewY - getSheetYOffset(viewport, scale) + viewport.top) / scale
    )
}
```

`getSheetXOffset` / `getSheetYOffset` (`SheetCoordinateMapper.kt:26-36`) account for centered padding when the sheet is smaller than the viewport.

### window → sheet (used by gesture detector)

`SheetCoordinateMapper.toSheet(event: MotionEvent): PointF` uses `event.rawX`/`rawY` and the view's `getLocationOnScreen` to compute sheet coordinates (`SheetCoordinateMapper.kt:67-84`). Called from `ElephantAnnotationsOverlayPresenter.onTouchSheet` (`ElephantAnnotationsOverlayPresenter.kt:273-282`):

```kotlin
// app/.../annotations/ElephantAnnotationsOverlayPresenter.kt:273
private fun onTouchSheet(event: MotionEvent?) {
    if (event != null) {
        sheetCoordinateMapper?.let { sheetMapper ->
            val handled = content.v2AnnotationView.onTouchEvent(event, sheetMapper.toSheet(event))
            content.tileView.setTouchLocked(handled && event.action != MotionEvent.ACTION_UP)
            ...
        }
    }
}
```

### sheet px → AV2F unit space (the space the markup data model uses)

Once the tap is in **sheet pixel** space, `V2AnnotationView` normalizes to AV2F **unit space** by dividing by `surfaceWidth`:

```kotlin
// annotations/.../ext/FoundationExt.kt:9
fun FoundationVector.toUnit(surfaceWidth: Float): FoundationVector =
    FoundationVector(
        x = this.x / surfaceWidth,
        y = this.y / surfaceWidth
    )
```

Called inside the gesture handler:

```kotlin
// annotations/.../views/V2AnnotationView.kt:421
override fun onSingleTapUp(event: MotionEvent): Boolean {
    mobius.handleOnUp(touchedPoint.toUnit(surfaceWidth))
    mobius.handleOnTap(FoundationVector(event.x, event.y).toUnit(surfaceWidth))
    return true
}
```

> NOTE: y is divided by `surfaceWidth` (not `surfaceHeight`). This means the unit space has equal scale on both axes — y values exceed 1.0 for portrait-orientation sheets. PinPoint must use the same convention to match.

### Where matrices live

- AV2F's per-element transform: `com.plangrid.av2foundation.geometry.Matrix` and `Transform` (`V2AnnotationView.kt:108,148`). Each `PgbsAnnotation` carries a `transform` (`PgbsLine`, `PgbsRect`, etc. each carry `transform`). The render-time composition is in `V2AnnotationView.drawNode` (`V2AnnotationView.kt:1758`).
- Android matrix composition (unit → screen) in `drawNode`:

```kotlin
// annotations/.../views/V2AnnotationView.kt:1772
val transformMatrix = (parentTransform ?: FoundationMatrix.identity()).times(element.transform.matrix)

matrixValues[0] = transformMatrix.a
matrixValues[1] = transformMatrix.c
matrixValues[2] = transformMatrix.e
matrixValues[3] = transformMatrix.b
matrixValues[4] = transformMatrix.d
matrixValues[5] = transformMatrix.f
matrixValues[6] = 0f; matrixValues[7] = 0f; matrixValues[8] = 1f
reusableMatrix.setValues(matrixValues)
reusableMatrix.postScale(scale * surfaceWidth, scale * surfaceWidth)
```

That `postScale(scale * surfaceWidth, scale * surfaceWidth)` is the inverse of `toUnit`: unit-space → screen-space, with the **view scale** applied so the markup matches the underlying viewer zoom.

- The `scale` flowing in here is delivered by `annotationMobius.scaleUpdated(scale, pxToDp(scaledWidth))` from `tileView.newZoomPanStateObservable` (`ElephantAnnotationsOverlayPresenter.kt:362-367`).
- `surfaceDimens` (a `Pair<Float, Float>`) is set from the viewable's natural pixel size: `content.tileView.setSize(viewableSize.width.toFloat(), viewableSize.height.toFloat())` (`ElephantAnnotationsOverlayPresenter.kt:326`), and pushed into the mobius via `setSurfaceDimens(surfaceSize)` (`ElephantAnnotationsOverlayPresenter.kt:323-324`, `:398-399`).

### Inverse path for a programmatic placement

For PinPoint (programmatic placement), you do **not** need to go through `SheetCoordinateMapper`. You only need:

1. The target point in **sheet pixel** space (the document's natural raster dimensions).
2. Divide by `surfaceWidth` (the sheet's natural pixel width) to get unit-space `FoundationVector`.
3. Construct a `PgbsAnnotation` with an appropriate `transform` (translation = unit-space coords), then call `IssuePinRepository.create2DIssuePin(...)` — see Section 5.

### Surface calibration (real-world scale)

`ParcelableSurfaceCalibration` (`annotations/.../calibration/domain/ParcelableSurfaceCalibration.kt:10-26`) maps to AV2F's `SurfaceCalibration` and adds `scale`, `distance`, `displayUnits`, `distanceUnits`, etc. It is the user-defined real-world-to-sheet ratio (e.g. 1" = 10' on a printed plan). PinPoint needs this **only** if asked to place pins by physical distance, not by image-space coordinates. The mapper is `SurfaceCalibrationMapper.toSurfaceCalibration(...)` (`annotations/.../calibration/utils/SurfaceCalibrationMapper.kt:13,36`).

---

## 5. D — Pushpin / marker system

### Marker data model

Every markup — including 2D issue pins — is a `com.plangrid.av2foundation.annotations.PgbsAnnotation`. The annotation carries:

- `transform: Transform` — position + rotation + scale in unit space.
- `boundEntityId: String?` — for issue pins, this is the `GGIssueUid`.
- `meta` (a subtype: `CloudMeta`, `PolyMeta`, etc.).
- `published: Boolean`, `createdAt`, `updatedAt`, `createdById`, etc. (see analytics enumerations in `V2AnnotationsAnalytics.kt:72,212,289`).
- `boundEntityType: BoundEntityType?` — one of `ISSUE`, `RFI`, `PHOTO`, `CALIBRATION`, `LOCATION`, `ASSET`, `QUEST` (`ElephantMarkupsExtensionFragment.kt:300-308`).

The persisted issue-pin aggregate is `IssuePin` (`pgf/feature/issues/issues-internal/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/issues_internal/models/IssuePin.kt:8-13`):

```kotlin
// pgf/feature/issues/issues-internal/.../models/IssuePin.kt:8
public data class IssuePin(
    val issue: GGIssue,
    val placements: List<IssuePlacement>,
    val annotation: PgbsAnnotation,
    val draftingContext: DraftingContext,
)
```

The wire model for placement (`pgf/wire-internal/.../models/annotation/Placement.kt:14`):

```kotlin
public data class Placement(
    val lock: String,
    val published: Boolean,
    val transform: List<Float>,   // float matrix elements
    val z: Double?,
    @JsonNames("layer_uid") val layerUid: LayerUid,
    @JsonNames("surface_uid") val surfaceUid: SurfaceUid,
)
```

The `transform` is serialized as a list of **floats** (an affine 2×3 matrix flattened). `z` is a separate double for 3D height. Surface identity is `SurfaceUid` (per-sheet/per-viewable).

### Tap → create-pin flow (manual user flow)

1. `V2AnnotationView.onTouchEvent` receives a `MotionEvent` plus `mappedTouchPoint` (already in sheet-pixel space via `sheetCoordinateMapper.toSheet(event)`). (`V2AnnotationView.kt:429-503`).
2. On `ACTION_DOWN` it records `touchedPoint` and calls `mobius.handleOnDown(touchedPoint.toUnit(surfaceWidth))`.
3. The `GestureDetector`'s `onSingleTapUp` calls `mobius.handleOnTap(...)` with the same unit-space vector (`V2AnnotationView.kt:421-425`).
4. The Mobius/MVI loop in `MarkupsViewInteractor` (`AnnotationMobius` or `MarkupsViewModel`, depending on `AV2_V2_USE_VIEW_VIEW_MODEL` flag at `AcsSheetFragment.kt:422`) decides what to do — for issue creation flows, it eventually emits `AnnotationParentMobius.Event.CreateIssueAnnotation` and the host fragment opens the issue editor.
5. When the user picks a type/template, `V2AnnotationRepository.createIssueAnnotation(...)` is called (`V2AnnotationRepository.kt:560`), which branches on the `AV2_2D_ISSUE_PIN_CREATION` flag:

```kotlin
// annotations/.../V2AnnotationRepository.kt:571
if (v2AnnotationFeatureFlag.is2DIssuePinCreationEnabled) {
    issuePinRepository.create2DIssuePin(
        operationDescription = operationDescription,
        issue = issue,
        annotation = updatedAnnotation,
        draftingContext = createDraftingContext(surfaceUid, dimens, viewableInfo),
        issueUid = issue.uid
    )
} else {
    issueBoundAnnotationRepository.createIssueAnnotation(
        updatedAnnotation,
        createDraftingContext(surfaceUid, dimens, viewableInfo),
        issue,
        operationDescription
    )
}
```

6. `IssuePinRepositoryImpl.create2DIssuePin` (`pgf/feature/issues/issues-internal/.../impl/repositories/IssuePinRepositoryImpl.kt:118-171`) builds a `CreateIssuePinOperation`, calls `pushOperationConsumerProvider().add(operation)` — i.e. it goes onto the push queue, then up to Forge.

### Rendering markers

- Markers are drawn by `V2AnnotationView.onDraw` (`V2AnnotationView.kt:1676`) — same canvas, separate `View` overlay above the Elephant `ComposeView` / SheetView2.
- The view is bound to `FrameLayout` / `CoordinatorLayout` inside `ElephantMarkupsExtensionFragmentBinding` (`content.v2AnnotationView`, see `ElephantAnnotationsOverlayPresenter.kt:346`).

### Hit testing

- Hit testing is done **inside AV2F** (`SceneBuilder` produces hit-test groups too). The Android code only forwards the tap; AV2F state then changes selection. The resulting `selectedAnnotations` set is observed in `V2AnnotationView` (`V2AnnotationView.kt:814,851`) and the host fragment opens the issue editor for `BoundEntityType.ISSUE` (`ElephantMarkupsExtensionFragment.kt:300-310`).

### Drag-to-relocate

Drag is handled in `V2AnnotationView.onTouchEvent` (`V2AnnotationView.kt:484-488`):

```kotlin
MotionEvent.ACTION_MOVE -> {
    if (isADrag) {
        handled = mobius.handleOnDrag(FoundationVector(mappedEvent.x, mappedEvent.y).toUnit(surfaceWidth))
    }
}
```

Drag threshold uses `touchSlop * 1 / scale` (`V2AnnotationView.kt:468`) — scaled inversely so a drag of N screen pixels is the same regardless of zoom.

### Markup types supported (modern, AV2F)

From the `PgbsType` enum surface in `V2AnnotationView.kt:117-118` and `mapPgbsElement`:

- `PgbsLine`, `PgbsRect`, `PgbsEllipse`, `PgbsPolyline`, `PgbsPolygonElement`, `PgbsPathElement`, `PgbsTextElement` (`V2AnnotationView.kt:1793-1801`).
- Compound types via `CloudMeta` (revision cloud), `PolyMeta` (multi-segment polylines) (`V2AnnotationView.kt:103-106`).
- Photo stamp: `PgbsType.PHOTO_STAMP` (`V2AnnotationView.kt:605`).
- Bound entity flavors: `ISSUE`, `RFI`, `PHOTO`, `CALIBRATION`, `LOCATION`, `ASSET`, `QUEST`.

The legacy markup system on PG Tasks (`MarkupsFileViewerFragment`) uses the same AV2F path now (it also references `PlanGridProjectFeatureFlag.AV2_V2_USE_VIEW_VIEW_MODEL`, `MarkupsFileViewerFragment.kt:191`).

---

## 6. E — Sync & data layer

### Where Sheet entities live

- The **legacy sheet** (PlanGrid PNG-tiled) entity is `com.plangrid.android.domain.sheets.model.SheetViewable` (used in `SheetView2.java:67`).
- The **modern document → viewable** entities live in PGFoundation:
  - `pgf/.../viewables/repositories/ViewableRepository.kt`
  - Document model in `pgf/.../models/...` (referenced as `documentRepository.getFileByUid(documentUid)` in `ElephantFragment.kt:307`).
- Sheet sync (legacy) is in `pgf/legacy/src/commonMain/kotlin/com/plangrid/pgfoundation/sheets/sync/SheetSync.kt` and siblings (`SheetCollectionSync`, `SheetFavoriteSync`, `SheetHistorySync`, `SheetPermissionSync`, `SheetTextSync`, `SheetDisciplineSync`, `SheetCollectionFavoriteSync`).

### Has sheet sync migrated to responsive sync?

**Not yet, in full.** The `pgf/feature/sheets/...DefaultSyncConfig.kt` (`pgf/sync/src/commonMain/kotlin/com/plangrid/pgfoundation/sync/responsiveSync/DefaultSyncConfig.kt`) has **no Sheet entries** (grep returned 0 matches for "Sheet"/"sheet" in that file). The sheet legacy sync still lives in `pgf/legacy/.../sheets/sync/...`. The `ProjectSheetsSettingsSync` is also under `pgf/legacy/`.

### Offline behavior

- `SheetView2` reads from PUG files on local storage (`SheetView2.java:307,314`). Project-level pre-download is governed by `ProjectSyncBlobApplicationCoordinator` — Elephant pauses the sync coordinator while the viewer is in use (`ElephantFragment.kt:283-285`):

```kotlin
// elephant/.../ElephantFragment.kt:282
when (isInUse) {
    true -> projectSyncBlobApplicationCoordinator.pause()
    false -> projectSyncBlobApplicationCoordinator.resume()
}
```

- HLOD binaries are tracked via `BinaryResourceDownloader` (`ElephantFragment.kt:138-143`):

```kotlin
override fun requestTrackFile(path: String, size: Long, type: ModelFileType): Boolean {
    val binaryType = when (type) {
        ModelFileType.HLOD_BVH -> BinaryType.VIEWABLE_HLOD_BVH
    }
    return brd.trackExternalBinary(path, documentUid.uidString, revision, binaryType, size)
}
```

### Pin write path (modern, 2D)

End to end:

1. UI handler `V2AnnotationView` → `MarkupsViewInteractor.handleOnTap(...)`.
2. Mobius effect → `V2AnnotationRepository.createIssueAnnotation(...)` (`V2AnnotationRepository.kt:560-590`).
3. `IssuePinRepository.create2DIssuePin(...)` (`IssuePinRepositoryImpl.kt:118-171`).
4. `IssuePinRepositoryImpl.upsert(item: IssuePin)`:

```kotlin
// pgf/.../IssuePinRepositoryImpl.kt:57
override fun upsert(item: IssuePin) {
    issuePlacementRepository.upsert(item.toIssuePlacement())
    issueRepository.upsert(item.issue)
    val annotationCreatable = AnnotationCreatable(item.annotation, item.draftingContext).model
    annotationRepository.upsertIssuePin(annotationCreatable, item.draftingContext)
}
```

5. A `CreateIssuePinOperation` is queued on `PushOperationConsumer` (`IssuePinRepositoryImpl.kt:148-170`) and uploaded by the push pipeline. Wire model: `IssuePin` becomes `IssuePlacement.SheetCreate` / `SheetUpsert` (per build output: `pgf/wire-internal/build/.../IssuePlacement$SheetCreate.class`).

### Pin read / display path (modern, in viewer)

`PinDataSourceShim` (`android/elephant/.../ui/PinSupport.kt:105-161`) adapts PGF's `PGFPinDataSource` to Elephant's `PinDataSource` interface. The Elephant viewer calls `loadPins(viewableGuid, initialSelection)` and gets back a `PinLoadResult` containing `MarkupPin` instances with positions in `DVec3D` (x, y, z doubles) — see `PinSupport.kt:53-64`, `:109-119`.

### Issue creation analytics surface

`PGFIssuePinCreationAnalytics` → `ACCPinAnalytics.issuePinWasCreated` → `IssueAnalytics.createSuccess` (`PinSupport.kt:192-203`). NOTE the analytics labels 2D pins with `PlacementSurfaceType.CASE3D` regardless — that's pre-existing analytics behavior, not a bug Atlas should "fix" silently.

---

## 7. F — AV2 vs legacy viewer coexistence

### The two paths and what they target

| Path | Entry fragment | Renderer for sheet pixels | Markup overlay | Used by |
|------|----------------|--------------------------|----------------|---------|
| **Modern** | `ElephantMarkupsExtensionFragment` (extends `ElephantFragment`) | `ElephantViewer` Compose (native, from `com.plangrid.android:elephant`) | `V2AnnotationView` on top, AV2F-driven, native `onDraw` | GG Issues, Forma/Build, PinPoint targets |
| **Legacy ACS** | `AcsSheetFragment` | `SheetView2`/PGTileView/PUG tile pyramid (`SheetView2.java:60`) | `V2AnnotationView` on top, **same AV2F-driven overlay** | Older PlanGrid sheets that still go through ACS fragment |
| **Legacy PDF** | `MarkupsFileViewerFragment` | `PdfViewerFragment` (Android PDF renderer or external tooling) | `V2AnnotationView` overlay (same AV2F) | Legacy PDF document viewer entry |
| **3D LMV** | `LmvFragment` (`android/lmv/`) | WebView via `com.autodesk.lmv.bridge.LmvWebFragment` (LmvFragment.kt:30) | LMV's own JS-side markup tooling (no V2AnnotationView) | Legacy 3D viewer; supplanted by Elephant for new file types |

### Punch Walk routing

Punch Walk (place-me / first-person walking through a 3D model) on the modern path runs **inside Elephant** — see `Preferences.kt:123-127` (joystick / gravity / auto-level prefs), `ElephantViewModelFactory.kt:49,65` (`canShowJoystick`, `isEnableJoystickOnPlaceMeEnabled`), and the join with PlaceMe analytics in `ACCElephantAnalytics`.

Legacy LMV's drop-me/first-person path is in `android/lmv/...listener/LmvDropMeListener.kt`, `LmvFirstPersonListener.kt`, hooked up in `LmvFragment.kt:69,71,501`. That path is being replaced; LMV is the WebView-based legacy 3D viewer, while Elephant is the native successor for both 2D and 3D.

For PinPoint's 2D auto-placement work, **Punch Walk is not on the critical path** — it operates in 3D viewables.

### Migration status

- The legacy module `android/llmv/` is **empty** (`src/` directory has no Kotlin source — only build outputs in `build/`).
- `MarkupsFileViewerFragment` still exists and is registered in `navigation_markups_file_viewer.xml:11`. Its `createIssueAnnotation` (`MarkupsFileViewerFragment.kt:994`) still uses the same V2 flow.
- `AcsSheetFragment.kt:422` keeps a flag-gated split: `PlanGridProjectFeatureFlag.AV2_V2_USE_VIEW_VIEW_MODEL` picks between the new `MarkupsViewModel` and the old `AnnotationMobius`.
- The big rendering migration is the move from PUG/SheetView2 to Elephant's F2D rendering. This is gated by `BetaBannerViewModel.ViewerMode.ELEPHANT` (`AcsSheetPresenter.kt:116`).
- Recent commits (12 months) include `AV2-10807` (`FEATURE_PIN type awareness on Android, iOS, and PGF`), `AV2-10808` (`Introduce Feature Pin SDK in PGF`) — the responsive-sync onboarding for pins is in progress (`pgf/feature/annotations/.../featurepins/FeaturePinManager.kt`, `PinPlacementDraft.kt`, `FeaturePinStore.kt`).

---

## 8. G — Performance & lifecycle

### Memory

- `BitmapLruCache` in `android/app/src/main/java/com/plangrid/android/tile/BitmapLruCache.java` underlies the legacy tile renderer (`SheetView2.java:80`). `RefCounted` bitmaps (`SheetLoupeView2.kt:235`) decrement on teardown.
- `V2AnnotationView` uses `WeakHashMap` caches (`paintMap`, `fillMap`, `pathCache`, `cloudRenderNodesCache`) to bound memory (`V2AnnotationView.kt:1690-1705`):

```kotlin
val paintMap: WeakHashMap<SceneNode, Paint> = WeakHashMap()
val fillMap: WeakHashMap<SceneNode, Paint> = WeakHashMap()
val pathCache: PathCache = WeakHashMap(128)
val cloudRenderNodesCache: WeakHashMap<CloudMeta, List<SceneNode>> = WeakHashMap(256)
```

- The `reusableMatrix` + `matrixValues` arrays avoid per-element allocation (`V2AnnotationView.kt:1697-1698,1774-1786`).
- Recent commit `AV2-10189 - Improve Annotations memory consumption (#29437)` (visible in git archaeology) tightened these caches.

### Rotation / configuration changes

- `ElephantFragment` saves restoration tokens (`RESTORE_STATE_BUNDLE_KEY`, `ElephantFragment.kt:188`) and re-initializes the `elephantViewModel` only if `!elephantViewModel.isValid` (`ElephantFragment.kt:293`).
- `ElephantMarkupsExtensionFragment` saves the viewport (`KEY_SAVED_VIEWPORT`, `ElephantMarkupsExtensionFragment.kt:147`) and restores it inside `onResume` (`ElephantMarkupsExtensionFragment.kt:402-409`).
- The legacy `AcsSheetFragment` has a `BehaviorRelay<String> sheetUidRelay` that re-publishes on `setSheet` after rotation (`AcsSheetFragment.kt:511`).

### Backgrounding / sync coordination

- The Elephant viewer pauses the sync blob applier while in use (`ElephantFragment.kt:282-287`, also `AcsSheetFragment.kt:546-549`).
- `viewerInstanceCount = ViewerInstanceCount()` tracks open viewers across screens (`AcsSheetFragment.kt:391-393`, `ElephantFragment.kt:189`).
- A specific ANR fix exists (`25a7cfc9e02 LFNT-3277: Fix ElephantFragment ANR by moving sync pause/resume to...`).

### Low memory / OOM

- `OutOfMemoryError` is not explicitly trapped — bitmap allocations are bounded by `BitmapLruCache` size. No `onTrimMemory` hook is implemented in `ElephantFragment`/`AcsSheetFragment` based on the imports — Atlas should verify against current code before claiming a hook exists.
- `MPX-3951: Forma app crashing for specific user on multiple devices...` is a recent crash fix in the area.

### Battery / sensors

- `GyroscopeHandler.kt` lives in `android/lmv/src/main/java/com/plangrid/android/lmv/util/GyroscopeHandler.kt` — used by LMV first-person mode (gyro to steer the camera). Modern viewer (Elephant) does its own sensor handling internally.
- No continuous GPS while viewer is open (not part of the modern 2D viewer surface).

---

## 9. H — Known pain points (git archaeology, last 12 months)

Themes that recur in the commit log (search command results above):

### Markup / pin coordinate & rendering bugs

- `c374e6425d8 SCCOM-33242: Preserve sheet zoom/pan when returning from issue details` — viewport state-restoration bug.
- `f24527dae66 / c17bdc8479a / 017ba571d3d AV2-10712: Fix Photo Markup Zoom Performance (#29811 / #29454)` — the photo-stamp markup was slow at zoom; a fix and a revert in the trail.
- `ea3c557aa82 AV2-10889: Fix crash from edit menu positioning (#30035)` and `b2439344245 AV2-10889: Fix edit menu positioning on initial tap. (#29862)` — popover-positioning bugs in `V2AnnotationView`.
- `94678108318 Change method for counting polygon points (#29102)` — geometry counting bug.

### Sheet loading / lifecycle

- `7c863da6dd1 ACSD-53580: Fix Android Sheets portrait loading (#29538)`.
- `0b728f66a8b ACSD-53547: Restore lightweight sheets list loading (#29540)`.
- `9be4110a873 ACSD-53414 Fix pasted sheets search refresh (#29439)`.

### Crashes & ANRs

- `ad0868c29e1 Fix Int32 overflow crash in GGIssueAnalytics duration fields (#30234)` — most recent on `dev`.
- `b5558424914 MPX-3951: Forma app crashing for specific user on multiple devices...`.
- `25a7cfc9e02 LFNT-3277: Fix ElephantFragment ANR by moving sync pause/resume to...`.
- `062b7d80da2 AV2-10681: Fix crash with PDF fragment creation. (#29435)` and `a4e47ad69fb Fix crash with fragment creation.`.
- `2484616c9d8 Guard against externally setting STATE_SETTLING on annotation bot...` — bottom-sheet state crash guard.

### Pin / Issue placement plumbing (very active)

- `dfc503dad53 AV2-10807 - FEATURE_PIN type awareness on Android, iOS, and PGF (...`.
- `d8ad32ad308 AV2-10808 - Introduce Feature Pin SDK in PGF (#29959)`.
- `8e23a317ade AV2-10888: Android support for markup sheet version history (#29966)`.
- `5f3d1ca8a30 / ba0e7f3d8d0 [SCCOM-33871] Fix empty viewable_type by firing drawerView on Set...`.

### Sync / responsive sync nearby

- `1f03748c22d Migrate LoaderPrioritizer tests to sync-internal (#30215)`.
- `5fa4cbcba00 Extract pending-download stage into PendingEntityDownloader (#30223)`.
- `b7a01bc63b7 [SCMP-9335] Subsequent entries to a detail view should re-fetch the data (#30179)`.

These all hint that the **pin write path and viewable-load path** are the hot spots — both directly relevant to PinPoint auto-placement.

---

## 9.5 — Walk-mode coordinate-space & snapshot reference (Section I)

> Persisted 2026-06-23 from the walk-mode POC. Read this BEFORE touching any
> overlay layer, the SAW snapshot path, or any code that reads/writes
> `tileView.scale` programmatically. Every claim has a `file:line` citation —
> open the file before you doubt it.

### 9.5.1 — The five coordinate spaces

The Section 4 diagram listed four spaces (sheet-px → view-px → window, plus
AV2F-unit). The walk-mode POC made a fifth one explicit because forgetting it
is the failure mode for every overlay-view author.

```
sheet-px (file space)
  └─ × qozixScale (sheetView.scale) ───────────────► scaled-content-px
       (qozix lays out CHILDREN of ZoomPanLayout in this space —
        V2AnnotationView and WalkModeViewerOverlayView are children)
  └─ via SheetCoordinateMapper.sheetToView ────────► view-px (viewport)
       (subtracts viewport.left/top + adds centering offset; this is the
        space that the qozix PARENT canvas reaches the screen in after
        the parent applies its scroll/pan)
  └─ via View.getLocationOnScreen + rawX/rawY ─────► window-px
sheet-px ÷ surfaceWidth ─────────────────────────► AV2F unit-space
  (the on-wire annotation transform space; see Section 4)
```

Crucial rule of thumb (the one we kept relearning):

- Children of `SheetView2` (a qozix `ZoomPanLayout`) draw on a canvas whose
  CONTENT space is **scaled-content-px**. qozix's
  `ZoomPanLayout.onLayout(...)` lays each child out at
  `child.layout(mOffsetX, mOffsetY, mScaledWidth + mOffsetX, mScaledHeight + mOffsetY)`
  (`android/tile-view/src/main/java/com/qozix/tileview/widgets/ZoomPanLayout.java:130-134`),
  and the parent applies scroll/pan. So the child's local canvas already has
  scroll/pan baked in — the only remaining transform a child layer needs is
  the scale `sheet-px × qozixScale`. NO translation.
- Anything that lives OUTSIDE `SheetView2` (notably the SAW
  `WalkModePanView`, which is a top-level system-overlay window) is NOT in
  scaled-content-px. It has to build its own sheet-px → overlay-px matrix
  from scratch — see §9.5.4.

### 9.5.2 — `SheetCoordinateMapper.viewToSheet` is scale-invariant by design

The mapper is the canonical sheet ↔ view-px transform for the qozix host.
It reads live qozix state on every call, so any tap converted through it is
already correct at whatever zoom the user happens to be at.

```kotlin
// android/domain/src/main/java/com/plangrid/android/domain/helpers/SheetCoordinateMapper.kt:51-58
fun viewToSheet(viewX: Float, viewY: Float): PointF {
    val viewport: Rect = tileView.detailLevelManager.viewport
    val scale: Float = tileView.scale
    return PointF(
        (viewX - getSheetXOffset(viewport, scale) + viewport.left) / scale,
        (viewY - getSheetYOffset(viewport, scale) + viewport.top) / scale,
    )
}
```

`viewport` is the qozix scroll window —
`(scrollX, scrollY, scrollX+W, scrollY+H)` updated every layout/scroll pass
by `TileView.updateViewport()`:

```java
// android/tile-view/src/main/java/com/qozix/tileview/TileView.java:783-789
protected void updateViewport() {
    int left = getScrollX();
    int top = getScrollY();
    int right = left + getWidth();
    int bottom = top + getHeight();
    mDetailLevelManager.updateViewport( left, top, right, bottom );
}
```

The centering branches `getSheetXOffset` / `getSheetYOffset`
(`SheetCoordinateMapper.kt:26-36`) ONLY kick in when the scaled sheet is
*narrower or shorter* than the viewport (i.e. fit-zoom-or-out). At any
zoom-in level (the typical case) they collapse to `0f`. There is no
zoom-specific branch in the mapper — it is one affine, parameterised by
live state.

**Existence proof at the canonical tap site**: the calibration tap reads
`mapper.viewToSheet(e.x, e.y)` at `android/app/src/main/java/com/plangrid/android/sheets/AcsSheetFragment.kt:1617`,
and the same mapper is used at `:2482` to feed
`v2AnnotationView.onTouchEvent(event, sheetCoordinateMapper.toSheet(event))` —
pins land correctly at any pinch-zoom level. Any "tap-at-zoom-X-looks-wrong"
bug is NOT this mapper.

### 9.5.3 — qozix `setScale()` does NOT request layout

Load-bearing fact for the SAW snapshot dance. When something programmatically
mutates the scale (e.g. the snapshot path), qozix does NOT re-run `onLayout`:

```java
// android/tile-view/src/main/java/com/qozix/tileview/widgets/ZoomPanLayout.java:241-251
public void setScale( float scale ) {
    scale = getConstrainedDestinationScale( scale );
    if( mScale != scale ) {
        float previous = mScale;
        mScale = scale;
        updateScaledDimensions();
        constrainScrollToLimits();
        onScaleChanged( scale, previous );   // notifies internal sublayouts only
        invalidate();                        // schedule redraw — NO requestLayout()
    }
}
```

Consequences (every one of them bit us at least once during the POC):

1. `onLayout` (`ZoomPanLayout.java:122-138`) does NOT re-run. The child
   container keeps its previous layout rect: `(mOffsetX, mOffsetY,
   mScaledWidth + mOffsetX, mScaledHeight + mOffsetY)` is stale, with
   `mScaledWidth/Height` and `mOffsetX/Y` reflecting the PRE-mutate scale.
2. `ZoomPanListener` callbacks do NOT fire for programmatic `setScale`.
   Listener notification only happens through the animator's
   `broadcastProgrammaticZoom{Begin,Update,End}` path. So
   `V2AnnotationView` — which only learns about scale via Mobius's
   `Action.ScaleUpdated` driven by `SheetView2.zoomPanStateSubject`
   (which itself only emits from the ZoomPanListener) — does NOT see the
   new scale.
3. The `invalidate()` does schedule a redraw, and tile detail-level
   recompute is dispatched, BUT tile load + decode is async. A synchronous
   `sheetView.draw(canvas)` immediately after a `setScale(...)` is racing
   against the tile pipeline. (This is exactly why the earlier
   "always-zoom-out-on-entry" Approach C reverted — see Memory 2026-06-21.)

If you mutate `tileView.setScale(...)` programmatically and need any layer
that depends on Mobius scale state to follow, the host has to **republish
the scale** itself (e.g. the earlier `onHostScaleProgrammaticallyChanged`
hook that lived in the controller before A1+B1+C2+D2 — now removed because
the snapshot path no longer mutates the user's scale).

### 9.5.4 — SAW mini-window snapshot dance recipe (canonical)

The SAW lives in its own `TYPE_APPLICATION_OVERLAY` window, so it cannot
re-use the qozix child-canvas trick. It needs a synthetic `M_sheet_to_bitmap`
and an opaque, full-sheet bitmap regardless of where the user is currently
zoomed in the host. Recipe (post-fix, all citations into
`android/app/src/main/java/com/plangrid/android/sheets/walkmode/WalkModeOverlayController.kt`):

```kotlin
// captureTileLayerSnapshot — :1733-1912 (excerpted)
val widthBitmapPx = sheetView.width
val heightBitmapPx = sheetView.height
val tileViewForSnapshot = sheetView as? TileView
val baseW = tileViewForSnapshot?.baseWidth ?: 0
val baseH = tileViewForSnapshot?.baseHeight ?: 0
val liveScale = tileViewForSnapshot?.scale ?: 1f
val fitZoom = if (tileViewForSnapshot != null && baseW > 0 && baseH > 0) {
    minOf(widthBitmapPx.toFloat() / baseW.toFloat(),
          heightBitmapPx.toFloat() / baseH.toFloat())
} else 1f

val bitmap = Bitmap.createBitmap(widthBitmapPx, heightBitmapPx, ARGB_8888)
bitmap.eraseColor(Color.WHITE)                            // :1785

try {
    val canvas = Canvas(bitmap)
    annotationView?.visibility = View.INVISIBLE
    viewerOverlay?.visibility = View.INVISIBLE
    try {
        if (applyFitZoomForSnapshot && tileViewSafe != null) {
            tileViewSafe.setScale(fitZoom)                // :1813
            tileViewSafe.scrollTo(0, 0)                   // :1814
        }
        sheetView.draw(canvas)                            // :1816
    } finally {
        if (applyFitZoomForSnapshot && tileViewSafe != null) {
            tileViewSafe.setScale(liveScale)              // :1820 restore
            tileViewSafe.scrollTo(prevScrollX, prevScrollY)
        }
        annotationView.visibility = prevAnnotationVisibility
        viewerOverlay.visibility = prevViewerVisibility
    }
} catch (t: Throwable) { bitmap.recycle(); return }

// Synthetic matrix — pure scale, NO centering.                :1861-1871
val sheetToBitmapMatrix = Matrix().apply { setScale(fitZoom, fitZoom) }
```

Why each step is the way it is:

- **`eraseColor(Color.WHITE)`** (`:1785`) — ARGB_8888 default is
  `0x00000000` (transparent black). `setScale(fitZoom) + scrollTo(0,0)`
  draws the sheet TOP-LEFT in the bitmap with extent
  `(baseW*fitZoom, baseH*fitZoom)`, which is smaller than the bitmap on at
  least one axis. The unfilled right/bottom letterbox bands stay
  transparent if you don't pre-clear. Then `WalkModePanView`'s
  `bitmapPaint.isFilterBitmap = true`
  (`android/app/src/main/java/com/plangrid/android/sheets/walkmode/WalkModePanView.kt:153-154`)
  at `currentZoomOverlay = 2.5f` (`WalkModePanView.kt:129`) bilinear-smears
  the alpha boundary across ~2.5 overlay-px, producing the "blurry
  gradient" the user reported. Fill with opaque white = crisp paper +
  uniform color for the filter to interpolate against.
- **Temp `setScale(fitZoom) + scrollTo(0,0)`, then restore in `finally`**
  (`:1812-1822`). Because of §9.5.3 the child container's layout rect
  doesn't refresh — that's load-bearing for the matrix (next bullet).
  Restore lives in `finally` so any draw exception cannot leave the host
  in a bad scale.
- **No centering offset in the matrix** (`:1861-1871`). Pre-mutate state:
  user is interactively zoomed in (we only enter this branch when
  `|liveScale - fitZoom| > 0.001`, so `liveScale > fitZoom`). Pre-mutate
  `mScaledWidth ≥ width` ⇒ qozix had `mOffsetX = 0`
  (`ZoomPanLayout.java:127`). After `setScale(fitZoom)`, `onLayout` does
  NOT re-run (§9.5.3) — the child container stays at
  `(0, 0, oldScaledW, oldScaledH)`. During the snapshot draw,
  `TileCanvasViewGroup.onDraw` does `canvas.scale(fitZoom, fitZoom)` against
  that stale rect, so tiles land in `(0, 0, baseW*fitZoom, baseH*fitZoom)`
  in the bitmap. **No centering is produced by the draw.** Therefore the
  matrix MUST also be pure `setScale(fitZoom, fitZoom)` with no centering
  translate. Adding a `getSheetXOffset`-style centering term — which
  `SheetCoordinateMapper.sheetToView` does — would shift the overlay
  layers (user dot, cone, session pins) off the bitmap content. That was
  the X-misalignment bug.
- **`sheetView.draw(canvas)` is synchronous** (`:1816`). Tile detail level
  recompute is async, but most of the time the fit-zoom detail level is
  already cached and the bitmap renders sharply. If a future bug demands
  it, post the call or hook the tile-loaded callback — but DON'T do it
  speculatively (the Approach C revert burned us once already).

### 9.5.5 — Single-source-of-truth invariant for `sheetToOverlayMatrix`

The bitmap and the layers are TWO sets of pixels that have to agree on
where every sheet-px lands in the overlay window. The bitmap was BUILT
with one matrix (`sheetToBitmapMatrix`), and the layers (user dot, cone,
session pins, trail) DRAW through another (`sheetToOverlayMatrix`
composed inside `WalkModePanView.onDraw` from `sheetToBitmap` and
`bitmapToOverlay = translate(pan) * scale(zoom)`).

**Invariant**: the bitmap matrix and the layer matrix must be the same
matrix (or factored composition of it). If you ever fork them — e.g.
build the bitmap with one centering convention and feed the layers a
matrix with a different one — the layers and the content disagree on
offset by exactly the centering term. Symptom: dots-on-right,
sheet-content-on-left (the X-misalignment we shipped a fix for in
Memory 2026-06-19).

Concrete encoding in code:

- §9.5.4 builds `sheetToBitmapMatrix` exactly once
  (`WalkModeOverlayController.kt:1861-1871`).
- `WalkModePanView.setSnapshot(...)` stores that matrix verbatim and
  composes `sheetToOverlayMatrix = bitmapToOverlay.preConcat(sheetToBitmap)`
  per frame.
- Layers receive `sheetToOverlayMatrix` as data (not as canvas state) via
  `WalkModeOverlayLayer.draw(canvas, viewportOverlayPx, sheetToOverlayMatrix)`,
  map their sheet-px anchors with `Matrix.mapPoints`, and draw at
  FIXED overlay-px sizes — that's how the dot stays dot-sized while the
  bitmap behind it zooms.

### 9.5.6 — In-host overlay matrix rule (pure scale, no translation)

The SAW mini-window is one rendering surface. There is a SECOND one — the
in-host overlay drawn ON TOP of the live qozix sheet for the user to see
"where am I" without opening the mini-window. That's
`WalkModeViewerOverlayView`
(`android/app/src/main/java/com/plangrid/android/sheets/walkmode/WalkModeViewerOverlayView.kt`).

It is a CHILD of `SheetView2`. Its canvas is therefore in
**scaled-content-px** (§9.5.1) — qozix has already applied scroll/pan to
the parent canvas. The matrix it builds for its layers is:

```kotlin
// WalkModeViewerOverlayView.kt:143-147
val origin = mapper.sheetToView(originScratch.apply { set(0f, 0f) }, false)
val unitX  = mapper.sheetToView(unitXScratch.apply { set(1f, 0f) }, false)
val qozixScale = unitX.x - origin.x
sheetToViewMatrix.reset()
sheetToViewMatrix.setScale(qozixScale, qozixScale)   // pure scale; NO translate
```

This mirrors `V2AnnotationView.drawNode` (`V2AnnotationView.kt:1841-1842`):

```kotlin
reusableMatrix.setValues(matrixValues)
reusableMatrix.postScale(scale * surfaceWidth, scale * surfaceWidth)
```

— a pure scale composition, no translation. AV2F annotations and walk-mode
in-host layers share this convention because they share the qozix-child
canvas convention.

**Anti-pattern**: do NOT call `SheetCoordinateMapper.sheetToView` and use
the full output as the in-host overlay's anchor. The mapper subtracts
`viewport.left/top` and adds the centering offset — values that are about
to be re-applied by the qozix PARENT scroll/pan when the child canvas
hits the screen. You'd be subtracting the parent's transform from a
child's local coordinates → double-applied scroll/pan → drift. That's
exactly the bug the comment at `WalkModeViewerOverlayView.kt:43-49`
warns about. The clean trick used there is to sample the mapper at two
points and take the *delta* — the viewport/offset terms are constant
across the two samples and cancel, leaving just the scale.

### 9.5.7 — "Workaround smell" heuristic

When a workaround locks a free interaction ("always zoom out before
entering walk-mode," "block user pan during snapshot," "force the user to
fit-zoom first") to "fix coordinates," the real bug is almost never in
the coordinate-conversion code. It is in a snapshot/render path that read
stale state at a moment the coordinate code couldn't know about.

The walk-mode POC accepted "always zoom out on entry" early on. The actual
bug was the SAW snapshot: bitmap built from one matrix, layers fed
another; mapper math was already correct. The fix landed when the
snapshot matrix and the layer matrix were unified (§9.5.5) and the
snapshot was made independent of live qozix state (§9.5.4).

Smell list to watch for going forward:

- "Always zoom out before X" — usually a snapshot/render bug.
- "Block pan during X" — usually a render/event-consumption bug.
- "Lock the user to fit-zoom" — usually the matrix the renderer uses
  doesn't track the live mapper.
- "Subtract this magic constant to make the dot land right" — usually a
  single-source-of-truth violation.

### 9.5.8 — Calibration capture surface — touch consumption

Adjacent topic, lives in the same bug class as §9.5.7 because the
original "can't pinch during calibration" was misdiagnosed as a mapper
bug and the proposed fix was to lock zoom (an even bigger workaround).
Real fix: a gesture-scoped multi-touch latch on the capture surface so
single-pointer drags consume (block pan per UX choice C2) but a
2+-pointer pinch falls through to qozix's `ScaleGestureDetector`:

- qozix host:
  `android/app/src/main/java/com/plangrid/android/sheets/AcsSheetFragment.kt:1672-1739`
  (`captureView.setOnTouchListener { ... }` with a `gestureWentMultiTouch`
  latch and `ACTION_POINTER_DOWN` switch-to-forward-to-tile).
- PDF host:
  `android/app/src/main/java/com/plangrid/android/pdf/MarkupsFileViewerFragment.kt:1207-1256`
  (same FSM, but on `binding.fragmentTouchView.onInterceptTouchFn`).

Single-tap-up still fires via the standard `GestureDetector.SimpleOnGestureListener`
because the tap is detected from `ACTION_DOWN + ACTION_UP` and the latch
only flips on `ACTION_POINTER_DOWN`. Returning `true` unconditionally
from the capture surface is the bug — it eats the pinch before
`ScaleGestureDetector` ever sees the second pointer
(`ZoomPanLayout.java:529-534`).

### 9.5.9 — PDR pipeline / pushpin projection are scale-invariant

For completeness: the PDR step-integration and the temp-marker /
session-pin projection helpers
(`WalkModeOverlayController.kt:777` derives `sheetPxPerMeter` from
`|P2-P1| / 3m` once at calibration tap #2, then never re-reads the
mapper) operate in pure sheet-px. They do not consume live qozix scale
or viewport. Zoom regressions cannot leak into them.

### 9.5.10 — Bug class catalog (misdiagnoses we've seen)

Quick triage table. When a walk-mode bug presents, match it to the class
BEFORE proposing a fix.

| Symptom | Wrong diagnosis | Actual class | Where to look |
|---|---|---|---|
| Tap at zoom X lands offset | "mapper math wrong" | event-consumption / wrong-host check | `mapper.viewToSheet` is correct (§9.5.2); check whether the gesture even reached the right view |
| Dot on right, sheet on left | "qozix needs zoom out" | layer matrix ≠ bitmap matrix (§9.5.5) | `WalkModePanView.setSnapshot` matrix vs `sheetToBitmapMatrix` construction |
| Blurry gradient at top-right of SAW | "render is low-res" | ARGB_8888 letterbox + `isFilterBitmap` alpha-edge smear (§9.5.4) | `bitmap.eraseColor(Color.WHITE)` before draw |
| Snapshot has wrong content under dot at zoom 3× | "always zoom out before entry" | qozix programmatic `setScale` → stale layout → top-left draw + matrix that added centering (§9.5.3, §9.5.4) | `captureTileLayerSnapshot` matrix MUST be pure `setScale(fitZoom)` with NO centering translate |
| Existing pin shifts in sheet-coords when toolbar tapped | "annotation cache stale" | `ZoomPanListener` does NOT fire for programmatic `setScale` (§9.5.3) — V2AnnotationView's Mobius scale stays stale | If you mutate `tileView.setScale`, republish the scale to listeners yourself |
| Can't pinch during calibration | "mapper not scale-invariant" | capture surface returns `true` unconditionally and eats the gesture | §9.5.8 multi-touch latch |
| Pin lands at ~11M sheet-px after walk-mode auto-attach | "wire payload broken" | bypassed `.toUnit(surfaceWidth)` — bypassed the established touch→Mobius→AV2F unit-space conversion | `FoundationExt.kt:9-13` is the boundary; new write paths MUST go through it |

---

## 10. Open Questions / Things Atlas Should Verify In-Session

1. **Surface UID derivation for auto-placed pins.** `ElephantAnnotationsOverlayPresenter.setupSurfaceUid` builds the surface UID from `(documentUid, viewableGuid)` AFTER the user opens the viewer (`Action.CreateSurfaceUidFromDocumentUidAndGuid`, line 224-228). Auto-placement may need to produce or look up a surface UID **without** opening the viewer first — verify whether `IssuePinRepository.create2DIssuePin` can accept a freshly minted SurfaceUid, or whether a surface row must already exist locally.
2. **PgbsAnnotation construction without UI.** The repo's `createIssueAnnotation` is always invoked from the V2AnnotationView gesture path. The annotation is provided pre-built (presumably by AV2F's tool flow). Atlas must verify the minimal `PgbsAnnotation` shape (transform + meta) needed for a synthetic pin: is `PgbsType.PHOTO_STAMP` the right pin type, or is there a dedicated `ISSUE_PIN` PGBS type? Read `com.plangrid.av2foundation.pgbs.PgbsType` (artifact only; check decompiler or rely on usage in `V2AnnotationView.kt:1793-1801`).
3. **AV2_2D_ISSUE_PIN_CREATION flag rollout.** All `create2DIssuePin` paths are gated on `is2DIssuePinCreationEnabled` (`V2AnnotationRepository.kt:571,603`). Verify project-level enablement on PinPoint test projects.
4. **Unit space anisotropy.** y is divided by `surfaceWidth` (not `surfaceHeight`) — confirm this is intentional in AV2F by looking at the iOS counterpart or the AV2F docs. If it's a bug it has lived a long time; if it's intentional, PinPoint must follow exactly.
5. **Pre-existing pin overlap / deduplication.** If PinPoint places a pin near an existing one, AV2F's hit-test/disambiguation kicks in (`DisambiguateBottomSheetFragment.kt`). Verify whether auto-placement should respect this or always succeed silently.
6. **`DraftingContext` construction.** `createDraftingContext(surfaceUid, dimens, viewableInfo)` is invoked everywhere but not visible in the open code — confirm whether `AnnotationViewableInfo` is required or optional for auto-placed pins.
7. **OOM hook.** No explicit `onTrimMemory` was found — Atlas should grep before claiming bitmap pressure is handled by lifecycle.
8. **Modern viewer state persistence.** `pendingRestoreStateToken: String?` in `ElephantFragment.kt:203` — verify restoration semantics across process death (not just rotation).
9. **F2D vs PUG when both exist.** `BetaBannerViewModel.ViewerMode.ELEPHANT` (`AcsSheetPresenter.kt:116`) gates whether F2D rendering takes over from PUG. For PinPoint, confirm coordinates are identical between both — `setSize(viewableWidth, viewableHeight)` is shared by both rendering modes.

---

## 11. Index of Most-Important Files (open these first)

| # | Path | Why |
|---|------|-----|
| 1 | `android/elephant/src/main/java/com/plangrid/android/elephant/ui/ElephantFragment.kt` | Modern viewer host; binds the Elephant Compose viewer; documents lifecycle/sync interaction. |
| 2 | `android/app/src/main/java/com/plangrid/android/fragments/ElephantMarkupsExtensionFragment.kt` | The 2D modern viewer entry: hosts the `ElephantViewer` + the `V2AnnotationView` markup overlay. |
| 3 | `android/app/src/main/java/com/plangrid/android/annotations/ElephantAnnotationsOverlayPresenter.kt` | Wires up `SheetCoordinateMapper`, `V2AnnotationView`, surface UID, scale flow. The bridge layer. |
| 4 | `android/annotations/src/main/java/com/plangrid/android/annotations/views/V2AnnotationView.kt` | The markup `View` itself — `onDraw`, tap → mobius, unit-space conversion, hit-test plumbing. |
| 5 | `android/annotations/src/main/java/com/plangrid/android/annotations/V2AnnotationRepository.kt` | The Android-side repository that calls into PGF for issue-pin creation. |
| 6 | `pgf/feature/issues/issues-internal/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/issues_internal/impl/repositories/IssuePinRepositoryImpl.kt` | The PGF KMP repo that turns an `IssuePin` into a push operation. |
| 7 | `pgf/feature/issues/issues-internal/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/issues_internal/models/IssuePin.kt` | The cross-platform pin aggregate. |
| 8 | `pgf/wire-internal/src/commonMain/kotlin/com/plangrid/pgfoundation/wire_internal/models/annotation/Placement.kt` | The wire-level placement model with transform/surface/layer. |
| 9 | `android/domain/src/main/java/com/plangrid/android/domain/helpers/SheetCoordinateMapper.kt` | Pixel-space sheet/view/window coordinate transforms. |
| 10 | `android/annotations/src/main/java/com/plangrid/android/annotations/ext/FoundationExt.kt` | The `toUnit(surfaceWidth)` that bridges pixel → AV2F unit space. |
| 11 | `android/app/src/main/java/com/plangrid/android/sheets/views/SheetView2.java` | Legacy tile viewer; also drives gestures in elephant-driver-mode. |
| 12 | `android/views/src/main/java/com/plangrid/android/views/PGTileView.java` | Tile-view subclass with scale limits, fling, double-tap. |
| 13 | `android/tile-view/src/main/java/com/qozix/tileview/TileView.java` | Vendored qozix tile-view library (DetailLevel, BitmapProvider, TileCanvasViewGroup). |
| 14 | `android/elephant/src/main/java/com/plangrid/android/elephant/ui/PinSupport.kt` | `PinDataSourceShim`, `ACCPinAnalytics`, `ACCPinOperationDescription` — the seam between Elephant's `PinDataSource` and PGFoundation. |
| 15 | `android/annotations/src/main/java/com/plangrid/android/annotations/V2AnnotationFeatureFlag.kt` | Names every AV2 feature flag — the source of truth for `is2DIssuePinCreationEnabled` etc. |
| 16 | `android/app/src/main/java/com/plangrid/android/sheets/AcsSheetFragment.kt` | Legacy 2D ACS entry; useful comparison for the modern path. |
| 17 | `android/app/src/main/java/com/plangrid/android/sheets/views/AcsSheetPresenter.kt` | Drives `elephantDriverMode(true/false)` transitions. |
| 18 | `android/components/src/main/java/com/plangrid/android/components/elephant/ElephantViewModelFactory.kt` | Boots the Elephant `ElephantViewModel` (pushpinEnabled, joystick, viewpoints, units). |
| 19 | `pgf/feature/annotations/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/annotations/featurepins/PinPlacementDraft.kt` | The new "feature pin" abstraction for future generic pins. |
| 20 | `pgf/feature/annotations/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/annotations/featurepins/FeaturePinManager.kt` | The KMP-side manager for the new pin SDK. |
| 21 | `android/annotations/src/main/java/com/plangrid/android/annotations/calibration/domain/ParcelableSurfaceCalibration.kt` | The surface calibration model — scale, units, distance. |
| 22 | `android/annotations/src/main/java/com/plangrid/android/annotations/calibration/utils/SurfaceCalibrationMapper.kt` | Maps between Android calibration and AV2F `SurfaceCalibration`. |
| 23 | `android/app/src/main/res/navigation/navigation_acs_sheets.xml` | Sheet navigation entry. |
| 24 | `gradle/libs.versions.toml` | All external artifact versions (AV2Foundation, Elephant, ESM). |
| 25 | `android/lmv/src/main/java/com/plangrid/android/lmv/ui/LmvFragment.kt` | Legacy WebView LMV — what Elephant replaces. |
