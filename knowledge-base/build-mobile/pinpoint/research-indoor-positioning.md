# PinPoint: Indoor Positioning Research

> Hackathon research for elevating Punch Walk — auto-placing issue pushpins on 2D floor plans using phone sensors and/or AR.

## TL;DR

- **Pure PDR is not good enough for pushpin placement past one room.** Smartphone PDR drifts ~1–5% of distance traveled, so a 50 m walk on a steel-framed floor lands the pin 2–5 m off — wrong bay of a structural grid. ([Jiménez et al.](https://www.academia.edu/25496802/A_comparison_of_Pedestrian_Dead_Reckoning_algorithms_using_a_low_cost_MEMS_IMU), [Harle 2013](https://www.semanticscholar.org/paper/A-Survey-of-Indoor-Inertial-Positioning-Systems-for-Harle/f873082d86155bfaf4552e5b93aff18f5158cf9d))
- **The magnetometer is unusable on a bare-frame construction site.** Indoor surveys measure peak field strengths ~2.78× ambient with heading errors of tens of degrees near rebar; [Phidgets advises a 5 m clearance from any rebar/transformer](https://www.phidgets.com/docs/Magnetometer_Guide) that simply does not exist on site. Use `TYPE_GAME_ROTATION_VECTOR` (Android) and `xArbitraryZVertical` (iOS) so you never fuse magnetometer data.
- **VIO (ARKit/ARCore) is dramatically better than PDR for translation** — ARKit benchmarked at ~0.02 m/s relative drift and 0.19 m final-point error on a 50–100 m indoor course ([MDPI Sensors 2022](https://www.mdpi.com/1424-8220/22/24/9873)) — but it requires the camera to see textured features and the phone to be raised. Workers carrying the phone at the hip get `INSUFFICIENT_FEATURES` within seconds. ([Apple insufficientFeatures](https://developer.apple.com/documentation/arkit/arcamera/trackingstate/reason/insufficientfeatures), [ARCore TrackingFailureReason](https://developers.google.com/ar/reference/java/com/google/ar/core/TrackingFailureReason))
- **Always-on AR drains 30–50%/hr battery and thermally throttles within 15–30 min.** Worse, IMU calibration drifts with chassis temperature, so the longer VIO runs, the worse it gets. ([6D.ai](https://medium.com/6d-ai/how-is-arcore-better-than-arkit-5223e6b3e79d), [ARCore performance guide](https://developers.google.com/ar/develop/performance))
- **Day-1 calibration on a fresh construction site has exactly two viable options:** printed fiducial markers (ArUco/AprilTag at 1–5 cm and 1–5° accuracy at 1 m) and hybrid VIO-with-one-confirmation-tap (zero infrastructure, accuracy is exact at the tap point and ~0.02 m/s drift from there). Everything else (UWB, BLE AoA, WiFi RTT, VPS, magnetic fingerprinting) requires pre-deployment or pre-scanning that the site does not have. ([Apple ARImageAnchor](https://developer.apple.com/documentation/arkit/arimageanchor), [ARCore Augmented Images](https://developers.google.com/ar/develop/augmented-images))
- **Magnetic fingerprinting is dead-on-arrival for construction.** It needs a stable field map; a site rebuilds its field signature weekly as steel and partitions move. ([Res-T-LSTM 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12736867/))
- **WiFi RTT (FTM) is also dead.** Only ~3% of installed APs support 802.11mc and iOS has no public API; a construction site usually has no permanent APs at all. ([Wikipedia 802.11mc](https://en.wikipedia.org/wiki/IEEE_802.11mc), [MIT FTM AP list](https://people.csail.mit.edu/bkph/FTMRTT_aps))
- **LiDAR-equipped iPhones (Pro 12+ / iPad Pro 2020+) are a real lever.** Without LiDAR, ARKit can't track blank concrete walls; with LiDAR it gets instant world tracking from depth and ARKit-RoomPlan can produce parametric room polygons at 1–2 inch accuracy. ([Macoclock LiDAR](https://medium.com/macoclock/arkit-911-scene-reconstruction-with-a-lidar-scanner-57ff0a8b247e), [Apple RoomPlan](https://developer.apple.com/augmented-reality/roomplan/))

**What I would build in 5 days:** A hybrid pipeline that runs PDR continuously off `CMPedometer` events + `TYPE_GAME_ROTATION_VECTOR`-style fused gyro/accel (no magnetometer). When the worker raises the phone to view the floor plan, kick on ARKit/ARCore in the background to capture VIO frames; snap PDR position to VIO and look for two recalibration triggers: (a) a printed AprilTag taped to a column at a known plan coordinate, or (b) a single user tap on the floor plan at a known landmark, which back-solves position + yaw against the VIO trajectory. Persist `ARWorldMap` / Cloud Anchors per site so repeat visits relocalize. Restrict the pilot to LiDAR-equipped iPhones + ARCore-certified depth-capable Androids. Auto-pin issues to the current best-estimate location with a confidence radius rendered on the plan; user can drag to correct.

---

## Q1: Position & Direction from Phone Sensors

### Pedestrian Dead Reckoning (PDR)

PDR estimates a pedestrian's trajectory by detecting individual steps, estimating each step's length, and projecting it along the current heading. Three step-detection families dominate on phones: peak detection on the magnitude of acceleration, zero-crossing of band-pass-filtered acceleration, and a finite-state machine over the stance/swing/heel-strike/toe-off phases. The [Ho/Truong/Jeong work in Sensors (MDPI 2016)](https://pmc.ncbi.nlm.nih.gov/articles/PMC5038701/) applies an FFT-based smoother to the accelerometer signal before step-detection rules to suppress false positives at higher walking speeds, where simple peak detectors break down.

Step-length models trade complexity for personalization. The [Weinberg model](https://pmc.ncbi.nlm.nih.gov/articles/PMC5038701/) uses the fourth root of (a_max − a_min) per stride and a per-user constant; Kim uses the cube root of the mean of |a|; Scarlett uses the fourth root of (a_max − a_min)/N samples. **None of these is great for non-cooperative users.** A 2024 ACM ICAIPR study reported average absolute distance errors of **8.03 m (Weinberg), 7.04 m (Kim), and 0.64 m for a learned adaptive model** over the same walk ([ACM ICAIPR 2024](https://dl.acm.org/doi/10.1145/3703935.3704046)) — i.e., classical step-length models alone produce multi-meter errors over a typical floor traversal. A hybrid step-frequency/acceleration-variance/fixed-step-length model walked 42.6 m with 98.47% distance accuracy (~0.65 m error) per [ISPRS 2022](https://isprs-archives.copernicus.org/articles/XLVIII-3-W1-2022/19/2022/2022/isprs-archives-XLVIII-3-W1-2022-19-2022.pdf).

Heading is the other half of PDR and is the dominant error source. As [Jiménez et al.](https://www.academia.edu/25496802/A_comparison_of_Pedestrian_Dead_Reckoning_algorithms_using_a_low_cost_MEMS_IMU) notes, "the main source of positioning errors is the absolute orientation estimation," and small per-step heading errors compound: a 2° heading bias over 50 m of forward travel produces ~1.7 m cross-track error; 10° produces ~8.7 m.

### Sensor Fusion & Heading

Four fusion algorithms dominate IMU/MARG orientation estimation on phones: complementary filter, Kalman / Extended Kalman Filter (EKF), Madgwick, and Mahony. A [comparative evaluation in PMC 8069451](https://pmc.ncbi.nlm.nih.gov/articles/PMC8069451/) and the [NDSU SPIE 2018 study](https://web.cs.ndsu.nodak.edu/~siludwig/Publish/papers/SPIE20181.pdf) report that Madgwick and Mahony achieve orientation error within ~0.4° of an EKF (overall ~3.4° RMSE) at a fraction of the compute cost. Madgwick gives slightly better heading than Mahony; Mahony is faster.

What ships on phones is closer to an EKF. Apple does not publish its fusion algorithm, but `CMDeviceMotion` outputs a calibrated attitude derived from an internal Kalman-style filter running on the motion coprocessor. Android's `TYPE_ROTATION_VECTOR` and `TYPE_GAME_ROTATION_VECTOR` are vendor-implemented, typically Kalman or extended-complementary filters running in the sensor hub. **For a hackathon, do not roll your own fusion** — use the OS output; only fall back to Madgwick over raw gyro/accel if you need to suppress magnetometer contributions in a steel environment.

### Indoor Magnetometer Challenges

This is the showstopper for unaided PDR on bare-frame construction sites. Earth's field is ~25–65 µT. Indoor surveys reported in [arxiv 2510.10979 (AMO-HEAD, 2025)](https://arxiv.org/pdf/2510.10979) measured peak indoor magnitudes of **127.34 µT — about 2.78× the ambient mean** — with raw-magnetometer heading errors "reaching several tens of degrees at peak disturbance points." The [Phidgets magnetometer guide](https://www.phidgets.com/docs/Magnetometer_Guide) recommends staying at least **5 m away from any reinforced concrete structure, transformer, or metal fence** before calibration — a distance that does not exist on a construction site.

The two static distortion modes are hard-iron (a fixed offset added by ferromagnetic material moving with the phone — rare in a handheld) and soft-iron (a directional distortion of the ambient field by nearby ferrous material — exactly what rebar grids produce). The standard figure-8 calibration described by [Analog Devices](https://ez.analog.com/mems/w/documents/4493/hard-soft-iron-correction-for-magnetometer-measurements) only corrects local, time-invariant distortions; it cannot fix the spatially varying field a worker walks through. Per the [IEEE study on height and magnetic fields](https://ieeexplore.ieee.org/document/9354181/), the magnetic field also varies significantly in the vertical dimension at "over half" of sampled locations across three buildings — the planar assumption in magnetic-fingerprint products fails on multi-level frames and as floors get poured.

**Practical recommendation:** favor `TYPE_GAME_ROTATION_VECTOR` (gyro + accel only, no magnetometer) on Android and `xArbitraryZVertical` initialization on iOS. Accept gyro drift over magnetic bias.

### Drift Realism

Drift in PDR is typically expressed as a **percentage of distance traveled**, not meters per minute, because errors accumulate per step.

- **Top-tier foot-mounted IMU + ZUPT + EKF**, 100 m closed loop: RMSE 0.23 m, max <1 m. Foot-mounted only — not applicable to a phone in a hand or pocket. ([PMC 4732172](https://pmc.ncbi.nlm.nih.gov/articles/PMC4732172/), [Foxlin lineage](https://link.springer.com/chapter/10.1007/978-3-642-54900-7_10))
- **Handheld/pocket smartphone PDR, low-cost MEMS:** stride length errors ~1%; total positioning errors **<5% of total distance traveled**. Translated: ~0.5 m after 10 m, ~2.5 m after 50 m, ~5 m after 100 m — and that's *outdoors* with a clean compass. ([Jiménez et al.](https://www.academia.edu/25496802/A_comparison_of_Pedestrian_Dead_Reckoning_algorithms_using_a_low_cost_MEMS_IMU))
- **Indoor smartphone PDR without WiFi/landmarks:** ~1.29 m average error in a large conference hall; degrades roughly linearly with distance. ([PMC 5191115](https://pmc.ncbi.nlm.nih.gov/articles/PMC5191115/))
- **Android Game Rotation Vector gyro drift:** "10 to 20 degrees over 10 minutes" raw, "5 to 10 degrees" with additional filtering. At 1 deg/min sustained drift, after 5 min × 1 m/s walking, the cross-track error from yaw drift alone reaches ~13 m. ([RoNIN, arxiv 1905.12853](https://arxiv.org/pdf/1905.12853))

The [Harle 2013 IEEE Comm Surveys & Tutorials](https://www.semanticscholar.org/paper/A-Survey-of-Indoor-Inertial-Positioning-Systems-for-Harle/f873082d86155bfaf4552e5b93aff18f5158cf9d) verdict: "PDR techniques alone can offer good short- to medium-term tracking … but regular absolute position fixes from partner systems will be needed to ensure long-term operation."

### Platform APIs (iOS + Android)

**iOS — CoreMotion:**
- [`CMDeviceMotion`](https://developer.apple.com/documentation/coremotion/cmdevicemotion) exposes `attitude` (quaternion/matrix/Euler), `rotationRate` (gyro minus bias), `userAcceleration` (linear acceleration with gravity removed), `gravity`, `magneticField` (with calibration accuracy enum), and on supported devices a `heading` in degrees from magnetic north.
- [`CMAttitudeReferenceFrame`](https://developer.apple.com/documentation/coremotion/cmattitudereferenceframe) is critical. Per [NSHipster](https://nshipster.com/cmdevicemotion/), the four frames are `xArbitraryZVertical`, `xArbitraryCorrectedZVertical`, `xMagneticNorthZVertical`, `xTrueNorthZVertical`. The "arbitrary" frames lock the X axis to the device's heading at session start — exactly what you want indoors. Magnetic-north frames will silently fall back to arbitrary-X if calibration accuracy stays at `low`.
- [`CMPedometer`](https://developer.apple.com/documentation/coremotion/cmpedometer) provides step events via `startEventUpdates` and cumulative data via `startUpdates(from:)`. **Do not consume `CMPedometerData.distance`** — a Stanford study reported a **+43% ± 42% overestimate**, even though step counts were within −7.2% ± 13.8%. ([MobiHealthNews](https://www.mobihealthnews.com/news/study-iphones-step-tracker-solid-dont-rely-its-distance-measurement-features))

**Android — SensorManager** ([Android sensor types](https://source.android.com/docs/core/interaction/sensors/sensor-types)):
- `TYPE_ROTATION_VECTOR` — fused accel + gyro + magnetometer. **Will be wrong indoors with steel.**
- `TYPE_GAME_ROTATION_VECTOR` — accel + gyro only; relative yaw; drifts but immune to rebar. **The right choice on a construction site.**
- `TYPE_GEOMAGNETIC_ROTATION_VECTOR` — accel + magnetometer only, no gyro; useless under steel.
- `TYPE_STEP_DETECTOR` — per-step events with <2 s latency.
- `TYPE_STEP_COUNTER` — cumulative since boot; high accuracy, up to 10 s latency.
- `TYPE_LINEAR_ACCELERATION` — gravity removed; input to step-length estimators.

### Available SDKs & Libraries

None is a drop-in for a bare-frame construction site, but for context:

- **[IndoorAtlas](https://www.indooratlas.com/solutions/indooratlas-positioning-sdk/)** — geomagnetic fingerprinting + PDR; **requires MapCreator fingerprinting before positioning works.** Tiered pricing without public dollar amounts on [the pricing page](https://www.indooratlas.com/pricing/). Fingerprints expire as steel moves — non-starter on active sites.
- **[Oriient](https://www.oriient.me/)** — sub-1 m claim using only Earth's magnetic field, no hardware. Targets malls/airports/retail; same staleness problem.
- **Pointr** — enterprise BLE/WiFi/IMU hybrid; assumes deployed beacons.
- **[Mapbox indoor mapping](https://docs.mapbox.com/ios/maps/guides/indoor/)** — visualization only, marked experimental; defers positioning to partners.
- **[Apple Indoor Maps Program](https://register.apple.com/indoor)** — large-venue program (airports/malls/campuses) requiring a stable 2.4 GHz beaconing WiFi network ([WWDC19 Session 245](https://developer.apple.com/videos/play/wwdc2019/245/), [FAQ](https://register.apple.com/resources/indoor/program/faq)). Accuracy degrades in convention halls and warehouses — i.e., the construction-site profile.
- **[Situm](https://situm.com/en/)** — hybrid BLE + WiFi + sensors; quick SDK integration but assumes infrastructure.

The pattern is consistent: every commercial product assumes installed RF infrastructure or a stable magnetic environment. Construction sites violate both.

### Realistic Accuracy

- **Single 5–10 m room, doorway-to-corner walk:** smartphone PDR with `TYPE_GAME_ROTATION_VECTOR` heading initialized at the doorway gives **~0.5–1.5 m position error**. Acceptable for room-level pushpin placement; ambiguous between left wall and right wall of a 3 m room.
- **50 m floor traversal, no fixes, bare-steel-framed level:** **2–5 m positioning error** from step-length + gyro drift, plus 1–3 m extra if magnetometer is fused in. Pushpin lands in the right structural bay but maybe wrong stud line.
- **100 m+ walk or 5+ min continuous tracking:** drift exceeds 5–10 m; the pin can land in the wrong room. This is exactly the regime where Harle says "absolute fixes are mandatory."

---

## Q2: Does AR Help Anchor Better?

### Visual-Inertial Odometry Overview

VIO fuses two complementary sensor streams: high-rate IMU readings (100–1000 Hz) and lower-rate camera frames (~30–60 Hz). The IMU gives instantaneous motion but drifts violently; the camera tracks pixel-level features across frames to anchor the IMU and recover absolute scale. ARKit and ARCore both implement tightly-coupled VIO: Apple's docs say the system [recognizes notable features in the scene image, tracks differences in those features across video frames, and compares that with motion sensing data to produce a high-precision pose](https://developer.apple.com/documentation/arkit/understanding-world-tracking). The whole reason VIO is attractive for indoor walking is that the camera *bounds* the IMU error. Pure IMU-only PDR [typically yields 1.0–2.5 m localization error with drift accumulation](https://www.tandfonline.com/doi/full/10.1080/10095020.2024.2338225) — VIO bounds that.

### ARKit vs ARCore on Construction Sites

**ARKit (iOS).** Tracking quality is reported per frame via `ARCamera.trackingState` — `.notAvailable`, `.limited(Reason)`, or `.normal`. The `Reason` enum is the diagnostic signal: [`.initializing`, `.excessiveMotion`, `.insufficientFeatures`, and `.relocalizing`](https://developer.apple.com/documentation/arkit/arcamera/trackingstate/limited). `.insufficientFeatures` fires when the camera sees a blank wall or low-texture surface; `.relocalizing` fires after backgrounding while ARKit tries to match the current view to the previous world map. Subscribe via `ARSessionObserver.session(_:cameraDidChangeTrackingState:)` and use `ARWorldTrackingConfiguration`.

A rigorous head-to-head benchmark with OptiTrack ground truth reported [ARKit relative pose drift ~0.02 m/s and indoor final-drift error of 0.19 m on a U-corridor course, vs 3.98 m for ARCore, 1.49 m for Intel T265, 4.76 m for ZED 2](https://www.mdpi.com/1424-8220/22/24/9873). The same study notes ARCore had low endpoint error but trajectory shape didn't match the floor plan — meaning a "good final pose" can hide mid-trajectory distortion that would mis-place pushpins. A separate 19.1 m corridor test reported [ARCore drift 0.5 m (2.6%) vs ARKit 1.5 m, with ARCore rotational error 3.4° vs ARKit 8.6° after 3.5 turns](https://zhongyu-wang.medium.com/drifting-error-comparision-of-arcore-and-arkit-visual-odometry-16cb68e02b8a). Pure local-tracking error stabilizes around 2 m over extended indoor sessions ([arXiv 2308.05394](https://arxiv.org/pdf/2308.05394)).

**LiDAR (iPhone Pro / iPad Pro)** is the single biggest hardware lever. [Without LiDAR, ARKit needs motion, light, and texture; with LiDAR it can track blank walls, gets instant world tracking from depth, and raycasts hit a wider range of surfaces](https://medium.com/macoclock/arkit-911-scene-reconstruction-with-a-lidar-scanner-57ff0a8b247e). On bare concrete and drywall, LiDAR is the difference between working and not.

**ARCore (Android).** `TrackingState` is `TRACKING`, `PAUSED`, or `STOPPED`; on `PAUSED`, `Camera.getTrackingFailureReason()` returns one of [`NONE`, `BAD_STATE`, `INSUFFICIENT_LIGHT`, `EXCESSIVE_MOTION`, `INSUFFICIENT_FEATURES`, `CAMERA_UNAVAILABLE`](https://developers.google.com/ar/reference/java/com/google/ar/core/TrackingFailureReason). Fragmentation is real: ARCore certification involves [Google validating per-device camera, sensors, design, and CPU because Apple controls hardware-software while Google must ensure ARCore works across hugely varying camera quality](https://developers.google.com/ar/devices). [The Depth API is on >88% of active certified devices](https://developers.google.com/ar/devices). For a fleet-wide construction app, restrict to flagship Pixel + recent Galaxy S/Z + OnePlus flagships from the Depth-capable subset.

### Drift Comparison: VIO vs PDR

| Tracking layer | Typical indoor drift | Source |
|---|---|---|
| Foot-mounted IMU + ZUPT (not viable on phone) | 0.23 m RMSE / 100 m | [PMC 4732172](https://pmc.ncbi.nlm.nih.gov/articles/PMC4732172/) |
| Handheld smartphone PDR (outdoor, clean compass) | ~5% of distance | [Jiménez et al.](https://www.academia.edu/25496802/A_comparison_of_Pedestrian_Dead_Reckoning_algorithms_using_a_low_cost_MEMS_IMU) |
| Smartphone PDR (indoor, no fixes) | ~1.29 m / large hall | [PMC 5191115](https://pmc.ncbi.nlm.nih.gov/articles/PMC5191115/) |
| ARKit VIO indoor | ~0.02 m/s; 0.19 m final / 100 m | [MDPI 2022](https://www.mdpi.com/1424-8220/22/24/9873) |
| ARCore VIO indoor | 0.5–4 m endpoint depending on course | [MDPI 2022](https://www.mdpi.com/1424-8220/22/24/9873), [Zhongyu Wang](https://zhongyu-wang.medium.com/drifting-error-comparision-of-arcore-and-arkit-visual-odometry-16cb68e02b8a) |
| ORB-SLAM + PDR fusion | 62% mean-error reduction vs either | [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0263224124015276) |

VIO is dramatically better than PDR **when the camera works.**

### Limitations & UX Trade-offs

Construction interiors are adversarial:

- **Texture floor.** [8thWall](https://www.8thwall.com/docs/studio/troubleshooting/world-tracking-issues/) notes smooth low-contrast surfaces fail regardless of color. Quantified: reliable tracking needs [light estimate ≥0.3, Laplacian variance ≥20 per camera image, and point-cloud density ≥50 points/m³](https://arxiv.org/pdf/2109.14757). Fresh concrete, drywall, and painted gypsum routinely fail the Laplacian threshold.
- **Dynamic scenes.** [When dynamic regions dominate the view, both VIO accuracy and reliability are greatly reduced](https://arxiv.org/html/2411.19289). Workers, forklifts, scissor lifts trip it up.
- **Sensor faults.** [IMU faults dominate VIO failures by orders of magnitude over camera faults](https://arxiv.org/html/2602.14413).
- **Holding posture is the killer.** A phone in a hip holster or pocket sees fabric. ARKit/ARCore go to `INSUFFICIENT_FEATURES` within seconds, and the pose stream collapses to drift-bound IMU until the user raises the phone — and then ARKit must relocalize, which is exactly when pushpin placement would be wrong.
- **Battery and thermals.** AR keeps camera, IMU, and GPU running continuously; [camera in the background is a primary cause of significant battery drain and heating](https://arxiv.org/pdf/1805.03060). [Once the device warms up, IMU calibration drifts with temperature and tracking degrades](https://medium.com/6d-ai/how-is-arcore-better-than-arkit-5223e6b3e79d). Google's own [performance guide](https://developers.google.com/ar/develop/performance) warns about "VIO frequency low" CPU starvation. RoomPlan caps capture at 5 minutes for [overheating and battery reasons](https://itechcraft.com/blog/your-101-guide-to-using-apple-roomplan-api-for-your-next-app/). Expect 30–50% battery/hr in always-on AR; thermal throttling within 15–30 min on a warm site.

There is **no documented passive/background AR mode** on either platform. iOS suspends ARSession on background; Android needs an active camera and another app holding the camera produces `CAMERA_UNAVAILABLE`.

### Cloud Anchors / Scene Understanding

- **ARCore Cloud Anchors / Persistent Cloud Anchors.** [Pre-1.20 expired in 24h; the Persistent Cloud Anchors API now supports 1–365 days, with 365d requiring keyless OAuth auth](https://developers.google.com/ar/develop/cloud-anchors). Indoor use is officially supported — Google positions them for room-scale persistent posters/notes. Subject to quotas; meaningful network/CPU cost.
- **ARCore Geospatial API + Streetscape Geometry.** [Typical position accuracy <5 m, often ~1 m; rotation <5°](https://developers.google.com/ar/develop/geospatial). Mostly outdoor (Street View imagery). Indoors only works if camera sees recognizable outdoor structures through windows. Fully enclosed interiors fall back to plain VIO. Not a primary indoor localizer.
- **ARKit ARWorldMap.** Local-file persistence. [Save with `getCurrentWorldMap`, reload by setting `configuration.initialWorldMap`; ARKit attempts probabilistic relocalization](https://www.appcoda.com/arkit-persistence/). [Practitioners report needing a Core Location or QR/beacon hint to pick the right map before relocalization starts](https://medium.com/@hwqmtmd/how-i-built-persistent-ar-zones-with-arworldmap-and-core-location-07448463786f). ARKit Location Anchors (ARGeoAnchor) are GPS-based and outdoor-only.
- **ARKit RoomPlan** (iOS 16+, LiDAR-only). Outputs a parametric room as USD/USDA/USDZ with walls, windows, doors, openings, plus furniture. [Session capped at 5 minutes; room cap ~9 m × 9 m; lighting ≥50 lux required](https://developer.apple.com/augmented-reality/roomplan/). Third-party products cite [1–2 inch accuracy even in cluttered spaces](https://www.arcsite.com/blog/arcsite-unveils-lidar-powered-room-scanning). Usable as a scanning workflow but symmetric rooms produce 4-way yaw ambiguity.
- **ARCore Scene Semantics** is [explicitly outdoor-only with 11 outdoor classes (sky/building/tree/road/sidewalk/vehicle/person/etc.)](https://developers.google.com/ar/develop/scene-semantics). The Depth API works indoors and outdoors and powers per-pixel occlusion.

### Recommendation

**Pure VIO is the wrong primary layer for Punch Walk.** Workers carry phones at the hip; the camera will be against fabric most of the walking time; ARKit/ARCore will sit in `.limited(.insufficientFeatures)`; the pose collapses to drift-bound IMU; battery and thermal cost of always-on camera + VIO will be unacceptable across an 8-hour shift.

**Pure PDR is also wrong** — standalone smartphone PDR drifts 1–5 m over 50 m and is heading-blind on steel-framed sites.

**The defensible architecture is PDR-as-baseline with VIO recalibration on raise-to-look, plus persistent visual anchors:**

1. PDR runs continuously, low-power, while the phone is holstered.
2. When the user raises the phone (detect via proximity sensor or motion classifier), ARKit/ARCore starts a session. Within 1–3 s VIO converges; snap PDR position to the VIO pose and use the visual frame to look for a known anchor (QR/AprilTag sticker on a column, Cloud Anchor, or saved ARWorldMap node) to set an absolute reset.
3. Print a sparse grid of fiducial stickers — one per room or stairwell — and host them as ARCore Persistent Cloud Anchors (365-day keyless) or ARWorldMap snapshots tied to room IDs.
4. Restrict the iOS pilot to LiDAR devices; restrict Android to the Depth-capable ARCore-certified flagship subset.
5. Do not attempt always-on VIO.

[ORB-SLAM + PDR fusion has been published with 62% mean-error reduction vs either alone](https://www.sciencedirect.com/science/article/abs/pii/S0263224124015276) — this is the well-trodden path.

---

## Q3: Initial Position & Heading Calibration (Low-Friction)

### Fiducial Markers (QR / AprilTag / ArUco)

**How it works.** A printed planar marker with known physical dimensions is detected in the camera; relative camera pose is recovered via SolvePnP from corner correspondences. [ARKit's `ARImageAnchor`](https://developer.apple.com/documentation/arkit/arimageanchor) and [ARCore's `AugmentedImage`](https://developers.google.com/ar/develop/augmented-images) wrap this and then hand off to VIO so the world coordinate persists after the marker leaves the frame.

**Accuracy with numbers.**
- 2025 ArUco/SolvePnP study: mean translation error **58.5 cm and rotation 6.6° for raw SolvePnP**, dropping to **1.18 cm / 3.11° with learned regression refinement**. ([NCBI ArUco study](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12196723/))
- [AprilTag analysis](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6960891/): distance error grows linearly with distance; yaw is the dominant orientation error.
- 2025 hand-eye calibration study: AprilTag translation within **12 cm (8% of GT distance) and 6°** vs OptiTrack before refinement. ([arXiv 2507.23045](https://arxiv.org/pdf/2507.23045))
- ARCore practical heuristic: marker must fill **≥25% of the frame** to detect; physical width required for stable pose; tracking improves significantly for markers ≥75 cm. ([ARCore Augmented Images guide](https://developers.google.com/ar/develop/java/augmented-images/guide))

For a **10 cm marker at 1 m**, expect 2–5 cm position and 2–5° heading. **20 cm at 1 m** → 1–3 cm / 1–3°. At 3 m yaw error blows up.

**Setup cost / Day-1.** Lowest of any method. Print on letter paper, measure, tape to a column. Marker ID encodes the floor-plan coordinate (e.g. ID 47 = "column J3, X=12.40, Y=8.10").

**Pitfalls.** ARCore image tracking shows [edge-hop instability ~4-in-10 trials under tilt/rotation/scale](https://www.researchgate.net/figure/Results-of-ARCore-tracking-in-four-scenarios-a-view-changing-b-camera-rotation-c_fig2_325034028). Oblique angles destroy yaw. Printed paper warps on a wet site — laminate.

### NFC Tags

**How it works.** Passive NFC tag at a doorway encoded with NDEF URL `app://checkin?loc=DOOR_J3`. Tap-on-entry pins position. **No heading.**

**Accuracy.** Position ~30 cm (tap point); heading must come from subsequent IMU.

**Cost.** ~$0.20/tag in bulk. ([NFC Tagify](https://nfctagify.com/blogs/news/nfc-in-ios-android))

**iOS / Android.** [Background NDEF reads on iPhone XS+ (iOS 13+)](https://gototags.com/articles/how-to-use-nfc-tags-with-an-iphone-ios); older iPhones need an open app. Android background reads on essentially all NFC phones.

**Pitfalls.** No heading. iOS NDEF-only constraint limits anti-spoofing. Can't install in pre-drywall framing.

### BLE Beacons

**How it works.** Trilateration from RSSI (iBeacon / Eddystone / AltBeacon), or true direction-finding via Bluetooth 5.1 AoA/AoD with multi-antenna anchors.

**Accuracy.** RSSI typical **2.2–4 m** with 12–36 deployed beacons ([NCBI BLE RSSI](https://pmc.ncbi.nlm.nih.gov/articles/PMC8347277/), [Integra Sources](https://www.integrasources.com/blog/bluetooth-indoor-positioning-systems/)). BLE 5.1 AoA: **0.23 m static / 0.30 m moving**; sub-meter when fused with RSSI ([u-blox](https://www.u-blox.com/en/technologies/bluetooth-indoor-positioning), [BlueIOT](https://www.blueiot.com/blog/a-deeper-look-at-bluetooth-direction-finding.html)).

**Setup / Day-1.** Significant — mount, power, survey dozens of beacons; AoA needs multi-antenna anchors. **Calibration drifts as the site changes** — every new partition shifts RSSI propagation. Non-starter on Day 1.

### UWB

**How it works.** Sub-nanosecond Time-of-Flight pulses at 3.1–10.6 GHz to fixed anchors; AoA on multi-antenna chipsets ([Navigine](https://navigine.com/blog/uwb-technology-features-examples-of-application/)).

**Accuracy.** **10–30 cm typical**, single-digit cm in optimized setups ([Mapsted](https://mapsted.com/en-it/blog/uwb-positioning-explained), [Newly](https://newly.app/sensors/uwb-mobile-apps)).

**Hardware.** iOS: iPhone 11+ (U1) / iPhone 15+ (U2); framework [Nearby Interaction](https://developer.apple.com/nearby-interaction/). Android: Pixel 6 Pro+, Galaxy S21 Ultra+, S22/S23/S24 Ultra; API `UwbManager` since Android 12. Anchors $50–200 each; ≥3 for 2D, ≥4 for 3D.

**Cross-platform reality.** Both stacks pair over BLE to exchange session keys; cross-vendor interop depends on FiRa 2.0 or CCC Digital Key R3 alignment ([Newly](https://newly.app/sensors/uwb-mobile-apps)). Apple's "Nearby Interaction with UWB Interoperability Specification" formally opens third-party interop.

**Day-1.** No — needs pre-deployed surveyed anchors. NLOS through steel/concrete causes multi-decimeter bias ([NCBI NLOS](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9657962/)).

### WiFi RTT (FTM) & Fingerprinting

**How it works.** [IEEE 802.11mc FTM round-trip-time](https://en.wikipedia.org/wiki/IEEE_802.11mc) to FTM-capable APs; trilateration.

**Accuracy.** Sub-meter typical (~30 cm at 50th percentile, <1 m at 95th) under good conditions ([Android WifiRttManager](https://developer.android.com/develop/connectivity/wifi/wifi-rtt)).

**Platform.** Android API 28+ via `WifiRttManager`; **802.11az NTB** ranging added in **Android 15 (API 35)** ([AOSP Wi-Fi RTT](https://source.android.com/docs/core/connect/wifi-rtt)). **iOS has no public Wi-Fi RTT API as of 2026.**

**AP reality — the project killer.** A 2019 Boston survey found only **~3% of installed APs advertised 802.11mc support**, and adoption has not improved ([MIT FTM AP list](https://people.csail.mit.edu/bkph/FTMRTT_aps)). Known supporters: Google Wi-Fi / Nest Wi-Fi (5 GHz only), Aruba 505/515/535/555/635/655. **A construction site has no permanent APs at all.** Dead on arrival.

### Magnetic Fingerprinting

**How it works.** Steel-framed buildings distort Earth's field in spatially unique, temporally stable patterns. Phone magnetometer reads local field; matched against a pre-built map to recover position ([GPS World on IndoorAtlas](https://www.gpsworld.com/indooratlas-announces-geomagnetic-indoor-positioning-service/)).

**Accuracy.** 1–5 m typical, ~2 m optimal, 3–5 m in Blue-Dot deployments. IndoorAtlas quotes ~1 hour to map 25,000 sq ft for 6-foot accuracy.

**Can it work on a construction site? No.** This is the most-asked, most-misunderstood question. Magnetic fingerprinting requires a **temporally stable** field map. Recent research is unambiguous: *"when indoor infrastructure such as internal walls and elevator substructures are modified or added via remodel, changing the physical layout… the fingerprint map becomes invalid"* ([Res-T-LSTM 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12736867/)). A construction site is **infrastructure-in-motion by definition**: rebar today, structural steel tomorrow, drywall next week. Every event shifts the field map. Also, you'd need to map first — and by then the site has changed. **Verdict: not viable for construction.**

### Visual Localization / VPS

**How it works.** Pre-scan environment to build a 3D feature map in the cloud; runtime camera matches to the map for global 6-DoF pose ([Ghost Howls VPS overview](https://skarredghost.com/2024/12/09/visual-positioning-systems/)).

**Options.**
- **Niantic Lightship VPS** — centimeter-level claim, ~30,000 public locations, primarily outdoor ([Lightship Summit](https://nianticlabs.com/news/lightshipsummit), [Lightship docs](https://lightship.dev/docs/ardk/features/lightship_vps/)).
- **Immersal** — **5–15 cm precision**, indoor-capable, B2B custom maps ([Immersal](https://immersal.com/)). Deployed at Mall of Tripla (85,000 m²) and Nokia Arena.
- **ARCore Geospatial / ARKit ARGeoAnchor** — [<1 m typical, <5° rotation in VPS-covered areas](https://developers.google.com/ar/develop/geospatial); Street View coverage only.
- Hexagon HxDR / Trimble reality-capture stacks target finished/scanned environments.

**Day-1 for construction.** No. Every VPS requires pre-scan, and construction sites are feature-poor (blank drywall) which defeats vision matching. IP/security concerns also push back on industrial-site maps in third-party clouds. Immersal is the only practical indoor option but still needs scanning before localization.

### CV Floor-Plan Alignment (ARKit Room Plan, ARCore Scene Semantics)

**How it works.** Camera frame → ML detects geometric features (door corners, window frames, wall corners) → match to 2D floor plan → solve phone pose in plan coords.

**Research evidence.** Active area, not productized:
- **PALMS (UCSD 2024)** localizes a smartphone against public floor plans via particle filter with Certainly Empty Space + principal-orientation matching from a single observation. ([arXiv 2410.15694](https://arxiv.org/html/2410.15694))
- **PALMS+ (2025)** extends with a depth foundation model. ([arXiv 2511.09724](https://arxiv.org/pdf/2511.09724))
- [Robot localization via room-layout edge extraction CNN](https://arxiv.org/pdf/1903.01804).
- [Survey of vision-based IPS](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7249029/) — sub-meter to tens-of-cm accuracy with ML.

**Day-1 viability.** Poor — requires walls, doors, windows, corners to exist. Fresh slab with rebar only: nothing to match. Once partitions go up, plausible — but the floor plan is also in flux. Closest production analog is [Trimble FieldLink MR](https://fieldtech.trimble.com/en/products/mixed-reality/trimble-xr10-with-hololens-2) using BIM overlays + physical control points, not pure feature matching.

**ARKit RoomPlan / ARCore Scene Semantics specifically:** RoomPlan produces parametric polygons (walls/doors/windows) at 1–2 inch accuracy, LiDAR-only ([ArcSite](https://www.arcsite.com/blog/arcsite-unveils-lidar-powered-room-scanning), [Apple research](https://machinelearning.apple.com/research/roomplan)); each capture is a per-room scanning workflow with 5-min cap, 9×9 m room limit, ≥50 lux lighting. Symmetric rooms produce 4-way yaw ambiguity. Scene Semantics is outdoor-only.

### Hybrid Bootstrap Strategies

**How it works.** User opens app anywhere. ARKit/ARCore starts a VIO session with origin at the start point. Worker walks. When they pass an unambiguous physical feature (corner column, doorway, elevator shaft), they **tap once on the floor plan** at that feature's position. The app:
1. Reads the VIO trajectory from session start to the tap.
2. Solves for the rigid transform (translation + yaw) that aligns the trajectory's endpoint with the tapped landmark's plan coordinate.
3. Resolves heading from the trajectory's direction of motion (worker was walking toward the landmark).

**Accuracy.** Exact at the landmark; ~0.02 m/s VIO drift from that point ([VIO benchmark](https://pmc.ncbi.nlm.nih.gov/articles/PMC9785098/)). Over a 60-second pre-tap walk, drift ~1–2 m. ARKit's [~100 m anchor rendering horizon](https://ethansaadia.medium.com/ar-location-anchors-in-arkit-4-9a7ca19652f5) caps useful range without re-anchoring.

**Day-1.** **High.** Zero pre-deployed infrastructure. Single user tap. Heading falls out of the geometry.

**Has anyone shipped this?** Closest analogs: Azure Spatial Anchors "Coarse Relocalization," and the [Trimble FieldLink MR / XR10](https://fieldtech.trimble.com/en/products/mixed-reality/trimble-xr10-with-hololens-2) landmark-tap workflow in construction. Academic precedent: Handrich et al. (2019) installing identifiable landmarks and back-solving ARCore pose.

**Pitfalls.** VIO drift compounds — needs a re-tap every ~30–60 m. Featureless drywall corridors degrade VIO. Resolves position + yaw but not pitch/roll (fine for 2D floor plans).

### Trade-off Matrix

| Method | Initial Position Accuracy | Initial Heading Accuracy | Setup Cost | Day-1 Viable | iOS | Android | Hardware |
|---|---|---|---|---|---|---|---|
| **QR / Fiducial (10–20 cm marker)** | 1–5 cm @ 1 m, 5–10 cm @ 3 m | 1–5° @ 1 m, 3–6° @ 3 m | Print + tape (<$1) | **Yes** if user prints | ARKit ARImageAnchor | ARCore AugmentedImage | Any AR-capable phone |
| **NFC tag (door check-in)** | ~30 cm (tap point) | None | $0.20/tag + install | After doorframes exist | iOS 13+ on iPhone XS+ bg | All NFC phones | NFC reader |
| **BLE RSSI iBeacon** | 2–4 m | None (IMU) | Dozens of beacons + survey | No | iBeacon native | AltBeacon/Eddystone | BLE beacons |
| **BLE 5.1 AoA** | 0.2–1 m | Indirect | Multi-antenna anchors $$$ + survey | No | BLE 5.1 stack | BLE 5.1 stack | Multi-antenna anchors |
| **UWB** | 10–30 cm | AoA on supported HW | Pre-deployed anchors $$$ | No | iPhone 11+ (U1/U2) | Pixel 6 Pro+, S21 Ultra+ | UWB anchors |
| **WiFi RTT (FTM)** | 30 cm – 1 m | None | None if APs exist; APs rare | **No** (no APs on Day 1) | None | Android 9+ | 802.11mc APs |
| **Magnetic fingerprinting** | 1–5 m | Compass (defeated by steel) | Hours of pre-mapping | **No** — site flux invalidates map | Magnetometer | Magnetometer | None extra |
| **VPS (Niantic / Immersal / ARCore Geo)** | 5 cm – 1 m | <5° | Cloud pre-scan | No | ARKit + SDK | ARCore + SDK | Phone camera |
| **CV floor-plan matching** | Sub-m (research) | Sub-degree (research) | None | Only after walls exist | iOS | Android | Phone camera |
| **RoomPlan / Scene Semantics** | Few cm scan; reg. depends on room | Yaw-ambiguous in symmetric rooms | Per-room ~30 s | Limited; needs walls | iPhone Pro w/ LiDAR | ARCore Depth | LiDAR (iOS) |
| **Hybrid VIO + landmark tap** | Exact at landmark; ~0.02 m/s drift away | Recovered from geometry | None | **Yes** | ARKit | ARCore | Any AR phone |

---

## Recommended Hackathon Architecture

A buildable-in-5-days architecture that respects the construction-site reality:

**Tracking layer:** PDR baseline + opportunistic VIO + visual-anchor recalibration. Never always-on AR.

**Calibration (Day 1):** *Two parallel paths*, both supported in the same app build:

1. **Default / preferred path — Hybrid VIO + landmark tap.** When the user opens the app on a floor for the first time, app instructs: "Walk to any column, gridline, or doorway shown on the plan, then tap that feature on the plan." ARKit/ARCore captures the trajectory; the rigid-transform solve fixes position and heading. Zero pre-deployment.
2. **Fallback / optional path — Printed fiducial markers.** If the GC has access to the plan and a printer, generate AprilTag PNGs with IDs encoding plan coordinates (column J3 → ID 47 → X=12.40, Y=8.10). One letter-size laminated AprilTag taped to each column or doorframe gives 1–5 cm / 1–5° fixes on raise-to-look.

**Position fusion runtime:**

- **iOS:** `CMPedometer.startEventUpdates` for step events; `CMDeviceMotion` with reference frame `xArbitraryZVertical` (never magnetic) for orientation; `ARWorldTrackingConfiguration` started on raise-to-look. On `.normal` tracking state, snap PDR position to ARKit anchor pose. Look for `ARImageAnchor` (AprilTag) hits to do absolute resets. Persist with `ARWorldMap.getCurrentWorldMap` per floor.
- **Android:** `TYPE_STEP_DETECTOR` for step events; `TYPE_GAME_ROTATION_VECTOR` for orientation (no magnetometer); ARCore `Session` started on raise-to-look. On `TrackingState.TRACKING`, snap PDR position to ARCore camera pose. Look for `AugmentedImage` hits for absolute resets. Persist with ARCore Persistent Cloud Anchors (keyless OAuth, 365-day).

**Pin placement UX:**

- Auto-place pin at current best-estimate position when user taps "create issue."
- Render a **confidence radius** on the plan derived from time-since-last-fix and tracking state. (User sees clearly when accuracy is degrading.)
- Allow drag-to-correct; every correction is a new manual fix that the system folds into its trajectory back-solve.
- Prompt re-tap every ~30 m of dead reckoning, or whenever AR tracking drops to `.limited` / `PAUSED`.

**Device policy:**

- iOS pilot: LiDAR-equipped only (iPhone Pro 12+, iPad Pro 2020+). LiDAR is the difference between working and not on blank concrete.
- Android pilot: ARCore-certified Depth-capable subset (recent Pixel + Galaxy S/Z + OnePlus flagships).
- Document explicitly that low-end devices are out of scope for the pilot.

**Stretch goals (if time permits):**

- ARKit RoomPlan capture of a finished room to register the scanned polygon against the 2D floor plan automatically (skip the manual tap when the room shape is unambiguous).
- ARCore Geospatial fallback for outdoor staging / rooftop work where VPS works.
- Magnetometer-disturbance-aware fusion: detect when the field deviates >2× ambient and dynamically down-weight magnetometer contributions (relevant only if you ever opt into magnetic fusion).
- ML-based step-length personalization (the [adaptive ICAIPR model](https://dl.acm.org/doi/10.1145/3703935.3704046) showed 0.64 m vs 8 m Weinberg error).

---

## Open Questions / Risks

- **Raise-to-look detection latency.** ARKit/ARCore session init plus VIO convergence is 1–3 s. If the user creates the issue *before* AR catches up, the pin is from PDR alone. Need to validate that issue creation in the wild gives the system enough time.
- **VIO on bare concrete corridors.** [Laplacian variance ≥20 threshold](https://arxiv.org/pdf/2109.14757) likely fails on freshly poured floors. Pilot must measure tracking-state distribution across real walks.
- **Thermal throttling during a long Punch Walk.** Need to measure how the IMU calibration drifts after 30–60 min on a warm site, and whether the [warm-IMU drift feedback loop](https://medium.com/6d-ai/how-is-arcore-better-than-arkit-5223e6b3e79d) makes mid-walk fixes worse than start-of-walk fixes.
- **Marker placement on Day 1 vs Day N.** Printed markers need GC buy-in to install before workers arrive. Hybrid VIO+tap doesn't, but UX research is needed on whether workers will actually tap a landmark on the plan.
- **Cross-platform pose fusion** if a worker walks past a Cloud Anchor created by an iPhone teammate yesterday: ARWorldMap is iOS-only; Persistent Cloud Anchors are cross-platform. Standardize on Cloud Anchors.
- **Multi-floor handling.** Stairwells confuse both PDR and VIO — VIO loses tracking on stair turns, PDR step-length models don't account for vertical translation. May need barometer-based floor detection ([CMAltimeter on iOS](https://developer.apple.com/documentation/coremotion/cmaltimeter), `TYPE_PRESSURE` on Android).
- **Battery target.** Validate <15% drain per hour of active inspection use; if VIO recalibration on raise-to-look blows this, restrict to fewer triggers.
- **Magnetometer always lies on site.** Test this in a real bare-frame building before locking the architecture. If the building is sufficiently dis-shielded (e.g., wood-frame residential), magnetometer fusion might actually help — but assume worst case.
- **Privacy / camera-on-site policy.** Some GCs prohibit cameras. AR layer must be opt-in per site; PDR + NFC fallback must still work.

---

## Sources

1. [Apple Developer — CMDeviceMotion](https://developer.apple.com/documentation/coremotion/cmdevicemotion) — Fused-motion API exposing attitude/rotationRate/userAcceleration/gravity/magneticField.
2. [Apple Developer — CMAttitudeReferenceFrame](https://developer.apple.com/documentation/coremotion/cmattitudereferenceframe) — Four reference frames; arbitrary-X initialization avoids magnetometer.
3. [Apple Developer — CMPedometer](https://developer.apple.com/documentation/coremotion/cmpedometer) — Step events and pedometer data; do not trust `distance`.
4. [Apple Developer — ARImageAnchor](https://developer.apple.com/documentation/arkit/arimageanchor) — ARKit anchor for known 2D markers.
5. [Apple Developer — Understanding World Tracking](https://developer.apple.com/documentation/arkit/understanding-world-tracking) — How ARKit's VIO works.
6. [Apple Developer — ARCamera.TrackingState.limited](https://developer.apple.com/documentation/arkit/arcamera/trackingstate/limited) — Limited tracking reasons enum.
7. [Apple Developer — insufficientFeatures](https://developer.apple.com/documentation/arkit/arcamera/trackingstate/reason/insufficientfeatures) — Failure mode when scene lacks features.
8. [Apple Developer — RoomPlan](https://developer.apple.com/augmented-reality/roomplan/) — Parametric room scanning, USD output, 5-min cap.
9. [Apple Machine Learning Research — RoomPlan](https://machinelearning.apple.com/research/roomplan) — Parametric room model architecture.
10. [Apple Developer — Nearby Interaction](https://developer.apple.com/nearby-interaction/) — iOS UWB framework.
11. [Apple — Indoor Maps Program registration](https://register.apple.com/indoor) and [program FAQ](https://register.apple.com/resources/indoor/program/faq) — Large-venue program requirements.
12. [Apple WWDC19 Session 245 — Introducing the Indoor Maps Program](https://developer.apple.com/videos/play/wwdc2019/245/) — Venue eligibility and accuracy caveats.
13. [NSHipster — CMDeviceMotion](https://nshipster.com/cmdevicemotion/) — Practical notes on attitude and calibration accuracy.
14. [MobiHealthNews — Stanford study on iPhone step tracker](https://www.mobihealthnews.com/news/study-iphones-step-tracker-solid-dont-rely-its-distance-measurement-features) — Steps ~-7%, distance +43%.
15. [Android Open Source Project — Sensor types](https://source.android.com/docs/core/interaction/sensors/sensor-types) — Rotation/Game/Geomagnetic/Step sensor definitions.
16. [Android Developers — Wi-Fi RTT Ranging](https://developer.android.com/develop/connectivity/wifi/wifi-rtt) — `WifiRttManager`, 802.11az NTB in Android 15.
17. [AOSP — Wi-Fi RTT (802.11mc/802.11az)](https://source.android.com/docs/core/connect/wifi-rtt) — HAL requirements and KPIs.
18. [ARCore — Augmented Images developer guide](https://developers.google.com/ar/develop/augmented-images) — Marker size, frame-fill requirements.
19. [ARCore — AugmentedImage Java API](https://developers.google.com/ar/reference/java/com/google/ar/core/AugmentedImage) — Android marker primitive.
20. [ARCore — Java AugmentedImages guide](https://developers.google.com/ar/develop/java/augmented-images/guide) — Tracking state vs method.
21. [ARCore — TrackingFailureReason](https://developers.google.com/ar/reference/java/com/google/ar/core/TrackingFailureReason) — `INSUFFICIENT_FEATURES`/`CAMERA_UNAVAILABLE`/etc.
22. [ARCore — Camera class](https://developers.google.com/ar/reference/java/com/google/ar/core/Camera) — Pose validity semantics.
23. [ARCore — Supported Devices](https://developers.google.com/ar/devices) — Certification and Depth-capable subset.
24. [ARCore — Cloud Anchors](https://developers.google.com/ar/develop/cloud-anchors) — Persistent Cloud Anchors 1–365 days; indoor room-scale use.
25. [ARCore — Geospatial API](https://developers.google.com/ar/develop/geospatial) — VPS accuracy <5 m, often ~1 m; <5° rotation.
26. [ARCore — Geospatial Depth (Java)](https://developers.google.com/ar/develop/java/depth/geospatial-depth) — Outdoor-only, ~65 m range.
27. [ARCore — Scene Semantics](https://developers.google.com/ar/develop/scene-semantics) — Outdoor-only, 11 classes.
28. [ARCore — Performance Considerations](https://developers.google.com/ar/develop/performance) — Battery/thermal and "VIO frequency low" diagnostics.
29. [arcore-android-sdk Issue #1553 — `CAMERA_UNAVAILABLE` on Samsung A23](https://github.com/google-ar/arcore-android-sdk/issues/1553) — Real-world fragmentation failure.
30. [MDPI Sensors 2022 — Benchmark of Four Proprietary VIO Systems](https://www.mdpi.com/1424-8220/22/24/9873) — ARKit drift 0.19 m vs ARCore 3.98 m indoor.
31. [arXiv 2207.06780 — Empirical Evaluation of Four Proprietary VIO Systems](https://arxiv.org/pdf/2207.06780) — Preprint of above with OptiTrack methodology.
32. [Zhongyu Wang — Drifting Error Comparison ARCore vs ARKit](https://zhongyu-wang.medium.com/drifting-error-comparision-of-arcore-and-arkit-visual-odometry-16cb68e02b8a) — 19.1 m corridor drift numbers.
33. [arXiv 2308.05394 — Robust Localization with VIO Constraints](https://arxiv.org/pdf/2308.05394) — Pure local ARKit drift stabilizes ~2 m extended sessions.
34. [arXiv 2109.14757 — Hologram Stability in Markerless Smartphone AR](https://arxiv.org/pdf/2109.14757) — Light ≥0.3, Laplacian ≥20, point cloud ≥50/m³ thresholds.
35. [arXiv 2411.19289 — ADUGS-VINS: VIO in Highly Dynamic Environments](https://arxiv.org/html/2411.19289) — VIO degrades when dynamic regions dominate.
36. [arXiv 2602.14413 — Sensor Vulnerabilities in Industrial XR Tracking](https://arxiv.org/html/2602.14413) — IMU faults dominate VIO failures.
37. [arXiv 1805.03060 — CloudAR Framework](https://arxiv.org/pdf/1805.03060) — Battery cost of continuous camera/AR.
38. [Medium / 6D.ai — How is ARCore better than ARKit](https://medium.com/6d-ai/how-is-arcore-better-than-arkit-5223e6b3e79d) — Thermal IMU drift feedback loop affecting VIO.
39. [Macoclock / Medium — ARKit Scene Reconstruction with LiDAR](https://medium.com/macoclock/arkit-911-scene-reconstruction-with-a-lidar-scanner-57ff0a8b247e) — LiDAR tracking on blank walls.
40. [AppCoda — ARKit Persistence](https://www.appcoda.com/arkit-persistence/) — `ARWorldMap` save/load and relocalization.
41. [Medium / hwqmtmd — Persistent AR Zones with ARWorldMap + Core Location](https://medium.com/@hwqmtmd/how-i-built-persistent-ar-zones-with-arworldmap-and-core-location-07448463786f) — Practical indoor relocalization patterns.
42. [Ethan Saadia — AR Location Anchors in ARKit 4](https://ethansaadia.medium.com/ar-location-anchors-in-arkit-4-9a7ca19652f5) — ~100 m anchor horizon, ARGeoAnchor.
43. [iTechCraft — Apple RoomPlan How-To Guide](https://itechcraft.com/blog/your-101-guide-to-using-apple-roomplan-api-for-your-next-app/) — Capture caps and thermal rationale.
44. [ArcSite — LiDAR-Powered Room Scanning](https://www.arcsite.com/blog/arcsite-unveils-lidar-powered-room-scanning) — 1–2 inch production accuracy from RoomPlan.
45. [Trimble FieldLink MR with HoloLens 2](https://fieldtech.trimble.com/en/products/mixed-reality/trimble-xr10-with-hololens-2) — Closest production analog of landmark-tap alignment in construction.
46. [8th Wall — World Tracking Issues](https://www.8thwall.com/docs/studio/troubleshooting/world-tracking-issues/) — Surface texture/feature requirements for SLAM.
47. [Sensors (MDPI 2016) — Step Detection and Adaptive Step-Length](https://pmc.ncbi.nlm.nih.gov/articles/PMC5038701/) — FFT-smoothed step detection; Weinberg/Kim formulas.
48. [ACM ICAIPR 2024 — Adaptive Step Frequency Detection and Stride-Length](https://dl.acm.org/doi/10.1145/3703935.3704046) — Weinberg 8.03 m / Kim 7.04 m / learned 0.64 m error.
49. [ISPRS Archives 2022 — Improvement of PDR Algorithm](https://isprs-archives.copernicus.org/articles/XLVIII-3-W1-2022/19/2022/2022/isprs-archives-XLVIII-3-W1-2022-19-2022.pdf) — 42.6 m walk at 98.47% distance accuracy.
50. [Academia.edu — Jiménez et al., PDR Algorithm Comparison on MEMS IMU](https://www.academia.edu/25496802/A_comparison_of_Pedestrian_Dead_Reckoning_algorithms_using_a_low_cost_MEMS_IMU) — ~5% total positioning error.
51. [PMC 8069451 — IMU fusion comparison](https://pmc.ncbi.nlm.nih.gov/articles/PMC8069451/) — Madgwick/Mahony/Kalman comparison.
52. [NDSU SPIE 2018 — Basic / Madgwick / Mahony comparison](https://web.cs.ndsu.nodak.edu/~siludwig/Publish/papers/SPIE20181.pdf) — Heading RMSE.
53. [arXiv 2510.10979 — AMO-HEAD adaptive MARG heading](https://arxiv.org/pdf/2510.10979) — 127.34 µT indoor peak, 2.78× ambient; tens-of-degrees error.
54. [Phidgets — Magnetometer Guide](https://www.phidgets.com/docs/Magnetometer_Guide) — 5 m clearance from rebar/transformers.
55. [Analog Devices — Hard/Soft Iron Correction](https://ez.analog.com/mems/w/documents/4493/hard-soft-iron-correction-for-magnetometer-measurements) — Calibration math and figure-8 procedure.
56. [IEEE Xplore — Impact of Height on Indoor Magnetic Positioning](https://ieeexplore.ieee.org/document/9354181/) — Vertical magnetic variability.
57. [PMC 4732172 — Foot-mounted IMU navigation](https://pmc.ncbi.nlm.nih.gov/articles/PMC4732172/) — 0.23 m RMSE over 100 m loop.
58. [Springer — Indoor Pedestrian Navigation w/ Shoe-Mounted Inertial Sensors](https://link.springer.com/chapter/10.1007/978-3-642-54900-7_10) — Foxlin ZUPT reference.
59. [arXiv 1905.12853 — RoNIN](https://arxiv.org/pdf/1905.12853) — Android Game Rotation Vector drift 10–20°/10 min.
60. [Semantic Scholar — Harle 2013 Survey of Indoor Inertial Positioning](https://www.semanticscholar.org/paper/A-Survey-of-Indoor-Inertial-Positioning-Systems-for-Harle/f873082d86155bfaf4552e5b93aff18f5158cf9d) — Canonical PDR survey; "absolute fixes mandatory."
61. [PMC 5191115 — Landmarks + PDR Indoor Positioning](https://pmc.ncbi.nlm.nih.gov/articles/PMC5191115/) — 0.14 m office vs 1.29 m hall.
62. [PMC 9785098 — VIO Benchmark of Four Systems](https://pmc.ncbi.nlm.nih.gov/articles/PMC9785098/) — ARKit ~0.02 m/s relative drift.
63. [ScienceDirect — ORB-SLAM + PDR Fusion](https://www.sciencedirect.com/science/article/abs/pii/S0263224124015276) — 62% mean-error reduction vs either alone.
64. [T&F — Context-Assisted Personalized PDR](https://www.tandfonline.com/doi/full/10.1080/10095020.2024.2338225) — Standalone PDR error 1.6–2.5 m.
65. [IndoorAtlas Positioning SDK](https://www.indooratlas.com/solutions/indooratlas-positioning-sdk/) and [pricing](https://www.indooratlas.com/pricing/) — Magnetic fingerprinting; tiered plans.
66. [Oriient Indoor GPS](https://www.oriient.me/) — Magnetic-field-only sub-1 m claim.
67. [Situm](https://situm.com/en/) — Hybrid BLE/WiFi/sensor SDK.
68. [Mapbox iOS Indoor mapping guide](https://docs.mapbox.com/ios/maps/guides/indoor/) — Visualization-only, experimental.
69. [GPS World — IndoorAtlas Announces Geomagnetic IPS](https://www.gpsworld.com/indooratlas-announces-geomagnetic-indoor-positioning-service/) — Origin of steel-distortion fingerprinting.
70. [IndoorAtlas — Ensuring High-Quality Maps](https://www.indooratlas.com/blog/ensuring-high-quality-indooratlas-maps/) — Mapping workflow / accuracy quotes.
71. [Wikipedia — Magnetic Positioning](https://en.wikipedia.org/wiki/Magnetic_positioning) — Fundamentals.
72. [NCBI: Res-T-LSTM Magnetic Positioning](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12736867/) — Magnetic maps invalid when infrastructure changes.
73. [NCBI: BLE RSSI Indoor Positioning](https://pmc.ncbi.nlm.nih.gov/articles/PMC8347277/) — RSSI accuracy.
74. [Integra Sources — Bluetooth Indoor Positioning](https://www.integrasources.com/blog/bluetooth-indoor-positioning-systems/) — Beacon-count vs accuracy field data.
75. [u-blox — Bluetooth Indoor Positioning](https://www.u-blox.com/en/technologies/bluetooth-indoor-positioning) — Bluetooth 5.1 AoA mechanism.
76. [BlueIOT — Bluetooth Direction Finding](https://www.blueiot.com/blog/a-deeper-look-at-bluetooth-direction-finding.html) — AoA/AoD theory.
77. [Navigine — UWB Technology Guide](https://navigine.com/blog/uwb-technology-features-examples-of-application/) — UWB accuracy in real deployments.
78. [Newly — UWB in Mobile Apps](https://newly.app/sensors/uwb-mobile-apps) — iOS/Android UWB stacks, FiRa profiles.
79. [Mapsted — UWB Positioning Explained](https://mapsted.com/en-it/blog/uwb-positioning-explained) — UWB physics.
80. [NCBI — NLOS Mitigation for UWB IPS](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9657962/) — UWB NLOS error sources.
81. [Wikipedia — IEEE 802.11mc](https://en.wikipedia.org/wiki/IEEE_802.11mc) — Standard background and AP penetration.
82. [MIT — Which Wi-Fi APs Support 802.11mc](https://people.csail.mit.edu/bkph/FTMRTT_aps) — AP-by-AP FTM catalog.
83. [NCBI — Improving Fingerprint Positioning with FTM/RTT](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9824134/) — Real-world RTT accuracy.
84. [Niantic — Lightship VPS Summit](https://nianticlabs.com/news/lightshipsummit) — cm-level positioning, 30k locations.
85. [Niantic Lightship VPS Docs](https://lightship.dev/docs/ardk/features/lightship_vps/) — Implementation reference.
86. [Immersal](https://immersal.com/) — Commercial indoor VPS, 5–15 cm.
87. [Ghost Howls — VPS Overview](https://skarredghost.com/2024/12/09/visual-positioning-systems/) — Lightship vs Immersal vs Geospatial.
88. [arXiv 2410.15694 — PALMS](https://arxiv.org/html/2410.15694) — Single-shot floor-plan localization.
89. [arXiv 2511.09724 — PALMS+ Depth Foundation](https://arxiv.org/pdf/2511.09724) — Floor-plan localization with depth model.
90. [arXiv 1903.01804 — Robot Localization in Floor Plans via Edge Extraction](https://arxiv.org/pdf/1903.01804) — Room-layout-to-floor-plan CNN.
91. [NCBI — Comprehensive Survey of Vision-Based Indoor Localization](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7249029/) — Accuracy across vision methods.
92. [NCBI — Analysis and Improvements in AprilTag](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6960891/) — Yaw is dominant AprilTag error.
93. [NCBI — Regression-Based Docking with ArUco (2025)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12196723/) — 58.5 cm raw vs 1.18 cm refined.
94. [arXiv 2507.23045 — Robot-World Hand-Eye Calibration](https://arxiv.org/pdf/2507.23045) — AprilTag 12 cm / 6° vs OptiTrack.
95. [ResearchGate — ARCore Tracking Results in Four Scenarios](https://www.researchgate.net/figure/Results-of-ARCore-tracking-in-four-scenarios-a-view-changing-b-camera-rotation-c_fig2_325034028) — ARCore image-tracking instability under tilt.
96. [GoToTags — NFC Tags with iPhone (iOS)](https://gototags.com/articles/how-to-use-nfc-tags-with-an-iphone-ios) — iPhone XS+ background NFC, NDEF formatting.
97. [NFC Tagify — NFC in iOS & Android](https://nfctagify.com/blogs/news/nfc-in-ios-android) — Cross-platform NFC capability matrix.
