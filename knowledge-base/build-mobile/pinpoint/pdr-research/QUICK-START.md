# PDR Implementation Quick-Start

**For the impatient**: Start here. This is a 2-minute guide.

---

## The Goal
Android foreground service streaming real-time position + heading (PDR) with 95%+ accuracy on construction sites.

## The Reality
- **Without any tricks**: 80-85% accuracy (raw IMU drift dominates)
- **With ZUPT (zero-velocity detection)**: 90-92% accuracy (fast, practical)
- **With Kalman filter + map constraints**: 95%+ accuracy (complex, Phase 2)

## The Recommendation
**Build the Moderate approach** (sensor streaming + ZUPT). Ship in 12 hours.

---

## What You're Building

```
Your PDRForegroundService:
  ✓ Samples accelerometer + gyroscope @ 200 Hz (5 ms resolution)
  ✓ Detects when user stops (variance threshold)
  ✓ Resets gyroscope drift during pauses
  ✓ Streams position (x, y) + heading updates to clients
  ✓ Runs continuously with low-priority notification
  
Your client app:
  ✓ Receives PDRUpdate stream
  ✓ Can optionally refine with Kalman filter
  ✓ Plots trajectory on blueprint

Build time: 12 hours (includes testing)
Accuracy: 90-92% on construction sites
Battery: 5-8% per hour (acceptable for navigation)
```

---

## The 5-Step Implementation Path

### Step 1: Sensor Acquisition (2 hours)
Grab accelerometer, gyroscope, magnetometer at 200 Hz. Queue in ring buffer.
- File: `SensorAcquisitionThread.kt`
- Key: Non-blocking, avoid GC in sensor callback

### Step 2: Processing Pipeline (2 hours)
Extract gravity, compute heading, track variance.
- File: `SensorPipeline.kt`
- Key: Low-pass filter gravity; integrate gyro for heading

### Step 3: Zero-Velocity Detection (2 hours)
When sensors go quiet, freeze position + recalibrate gyro bias.
- File: `ZeroVelocityDetector.kt`
- Key: Variance < 0.1 (accel) and < 0.01 (gyro) = stationary

### Step 4: Dead Reckoning (2 hours)
Detect steps, update position (x, y) with heading.
- File: `StepDetector.kt` + `PDRProcessor.kt`
- Key: Step length ~0.7m; integrate direction from gyro

### Step 5: Foreground Service + Binding (2 hours)
Wrap it in a service, expose Flow to clients, add notification.
- File: `PDRForegroundService.kt` + `PDRBinder.kt`
- Key: START_STICKY, startForeground(), bind for low-latency streaming

---

## Critical Thresholds (Copy These)

```kotlin
// Sensor sampling (200 Hz = best balance of accuracy & battery)
ACCEL_PERIOD_US = 5_000
GYRO_PERIOD_US = 5_000
MAG_PERIOD_US = 20_000

// Zero-velocity detection (0.5 sec window)
ACCEL_VAR_THRESHOLD = 0.1f    // m²/s⁴
GYRO_VAR_THRESHOLD = 0.01f    // rad²/s²
WINDOW_SIZE = 100             // samples @ 200 Hz = 0.5 sec

// Dead reckoning
STEP_LENGTH = 0.7f            // meters (assumes 1.7m user height)
STEP_INTERVAL_MIN = 0.4f      // seconds (avoid false steps)
ACCEL_MAGNITUDE_THRESHOLD = 1.5f  // m/s² (step detection)

// Magnetometer reliability (Earth's field is 25-65 µT)
MAG_MIN = 20f
MAG_MAX = 70f
```

---

## The One Debug Trick That Works

If your PDR is drifting fast (heading error > 5° after 30 seconds):

1. **Check ZUPT is triggering**:
   ```kotlin
   if (zeroVelocityDetected) Log.d("PDR", "ZUPT active!")
   ```
   If this never logs during pauses → increase ACCEL/GYRO thresholds by 2x

2. **Check gyro bias estimation**:
   ```kotlin
   Log.d("PDR", "Gyro bias: ${gyroBias[2]}°/sec")
   ```
   Should be ±1°/sec. If > 5°/sec → device has bad gyro or mounting error

3. **Check step length is reasonable**:
   ```kotlin
   Log.d("PDR", "Step: $stepLength m")
   ```
   Should be 0.6-0.8m. If > 1m → user height model is wrong

---

## Testing in 30 Minutes

```
Setup:
1. Mark a 100m circuit (tape measure + chalk)
2. Pause 2 seconds at each 10m mark (trigger ZUPT)
3. End at start point

Run:
1. Start PDR service
2. Walk circuit
3. Check final position vs. start

Success: < 2m error (4% of 100m)
Failure: > 5m error → ZUPT not working or step length wrong
```

---

## Files You Need (Copy from pdr-implementation-guide.md)

| File | Purpose | Lines |
|------|---------|-------|
| `SensorSnapshot.kt` | IMU data structure | ~50 |
| `ProcessedSensor.kt` | Calibrated sensor output | ~30 |
| `SensorAcquisitionThread.kt` | Sensor listener + buffering | ~100 |
| `SensorPipeline.kt` | Gravity extraction, heading | ~150 |
| `CalibrationManager.kt` | Gyro bias correction | ~80 |
| `ZeroVelocityDetector.kt` | ZUPT algorithm | ~60 |
| `StepDetector.kt` | Step detection | ~80 |
| `PDRProcessor.kt` | Position/heading integration | ~120 |
| `PDRUpdate.kt` | Output data class | ~30 |
| `PDRForegroundService.kt` | Service + notification + binding | ~180 |
| **Total** | | **~880 lines** |

---

## Build Configuration (gradle.kts)

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.1")
    implementation("androidx.core:core:1.10.0")
}

android {
    compileSdk = 34
    minSdk = 31  // Android 12+ (foreground service requirement)
}
```

---

## Permissions (AndroidManifest.xml)

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.BODY_SENSORS" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<service android:name=".pdr.PDRForegroundService"
         android:foregroundServiceType="location" />
```

---

## Common Mistakes (Don't Make These)

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Sample rate < 50 Hz | Step detection misses every other step | Use 200 Hz minimum |
| No ZUPT | Drift accumulates unbounded | Implement zero-velocity detection |
| Step length hardcoded 1.0m | 40% position error on short users | Use 0.7m default; detect outliers |
| Bind to main thread | ANR during sensor processing | Use Dispatchers.Default for pipeline |
| No notification | Crash on Android 12+ foreground service | Call startForeground() in onCreate |
| Magnetometer always trusted | 30° heading error near steel | Add reliability check (field magnitude) |

---

## What Doesn't Ship in Phase 1 (Defer to Phase 2)

- Kalman filter (nice-to-have; client can implement if needed)
- Map constraint integration (requires blueprint parser)
- Multi-floor support (needs barometer tuning)
- Machine learning step length (needs user training data)

---

## How to Know You Won. Succeeded

After 12 hours of coding:

✓ Service starts, notification appears  
✓ Sensor stream is flowing (check logcat)  
✓ ZUPT triggers during pauses (check log: "ZUPT active")  
✓ Walk 100m circuit, end within 2m of start  
✓ Heading stays within ±5° of actual during straight walks  
✓ Battery drain is 5-8% per hour (acceptable)  

If all 6 checkmarks pass, you've shipped Phase 1. Congratulations.

---

## Where to Get Help

- **Architecture questions?** → Read `pdr-foreground-service-architecture.md`
- **Implementation stuck?** → Copy from `pdr-implementation-guide.md`
- **ZUPT not working?** → Debug checklist in `pdr-zero-velocity-detection.md`
- **Accuracy still bad?** → Check Section 3 (construction site challenges)

---

## Next Steps (After you ship Phase 1)

1. **Validate on actual construction site** (2-3 walks, 30 min each)
2. **Collect ground truth data** (GPS + manual position checks)
3. **Tune ZUPT thresholds** per your target buildings
4. **Plan Phase 2: Kalman filter** (adds ±2% accuracy, 4 hours)
5. **Plan Phase 2: Map integration** (adds ±3% accuracy, 8 hours)

---

**TL;DR**: Copy the 10 files from `pdr-implementation-guide.md`, adapt sensor thresholds for your device, ship in 12 hours. You'll get 90-92% accuracy on construction sites. Next 8% comes from Kalman + maps (Phase 2).

Good luck. You've got this.
