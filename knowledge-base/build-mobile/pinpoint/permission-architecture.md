# PDR Permission Architecture

## Problem Statement

The PDR service APK requires runtime permissions (`BODY_SENSORS`, `ACTIVITY_RECOGNITION`) to access IMU sensors on Android 6+, but:

1. **Services can't request permissions directly** — only Activities can trigger permission dialogs
2. **Separate APKs require separate permission scopes** — the test app's permission grants don't automatically cover the service APK
3. **Sensor registration fails silently** without permission, making debugging difficult

## Solution: Transparent Permission Activity

### Architecture

- **PermissionActivity.kt**: A transparent (no-UI) ComponentActivity in the service APK that handles runtime permission requests
- Chains requests: `BODY_SENSORS` → `ACTIVITY_RECOGNITION`
- Handles already-granted permissions (skips request dialog if already granted)
- Logs grant/deny outcomes clearly for debugging
- Declared in service manifest with action filter `com.autodesk.pdr.REQUEST_PERMISSIONS`

### Permission Flow

1. **Test App starts**:
   - `MainActivity.onCreate()` requests `BODY_SENSORS` and `ACTIVITY_RECOGNITION`
   - This grants permissions to the test app APK

2. **Service starts** (via AIDL binding):
   - `PDRService.onCreate()` checks permission status
   - If permissions missing, logs: `"Launch PermissionActivity (com.autodesk.pdr.REQUEST_PERMISSIONS) to request them"`
   - Logs all available sensors and their permission dependencies

3. **Permission Activity flow** (if needed):
   - Test app or service can launch via: `startActivity(Intent("com.autodesk.pdr.REQUEST_PERMISSIONS"))`
   - `PermissionActivity.onCreate()` chains: check BODY_SENSORS → request if missing → check ACTIVITY_RECOGNITION → request if missing
   - After both permission dialogs, activity auto-closes
   - All outcomes logged to logcat for easy debugging

### Files Modified

- **Service APK**:
  - `PermissionActivity.kt` — NEW: transparent permission activity
  - `PDRService.kt` — UPDATED: enhanced logging of permission status + sensor availability
  - `AndroidManifest.xml` — UPDATED: PermissionActivity declaration + permission declarations

- **Test App**:
  - `MainActivity.kt` — EXISTING: already requests permissions (unchanged)
  - `AndroidManifest.xml` — UPDATED: added BODY_SENSORS and ACTIVITY_RECOGNITION declarations

### Key Insights

1. **Permission Logging is Crucial**:
   - Service now logs: `"Permissions — BODY_SENSORS: GRANTED, ACTIVITY_RECOGNITION: DENIED"`
   - Lists all PDR-relevant sensors with availability and permission status

2. **Graceful Degradation**:
   - Service doesn't crash if permissions denied
   - Reports exactly which sensors are unavailable due to missing permissions
   - Test app can still run; just won't get step detection or specific IMU readings

3. **Two Request Paths**:
   - **Test App Request** (happens at test app startup): Covers test app's own sensor access
   - **Service Permission Activity** (optional, only if test app request fails or service needs independent grant): Covers service APK's sensor access

### Testing Checklist

- [ ] Install both APKs on real device (minSdk 34)
- [ ] Grant all permissions when test app starts
- [ ] Check logcat for `PDRService` tag — verify: `"Permissions — BODY_SENSORS: GRANTED, ACTIVITY_RECOGNITION: GRANTED"`
- [ ] Check logcat for `"Available sensors:"` — verify all IMU sensors listed as present
- [ ] Start tracking — verify PDR updates flowing with correct positions
- [ ] Deny permissions on test app — verify logcat shows DENIED status
- [ ] Launch PermissionActivity to grant them again
- [ ] Verify sensor registration succeeds after permission grant

### Logcat Patterns to Watch

**Success**:
```
PDRService: Permissions — BODY_SENSORS: GRANTED, ACTIVITY_RECOGNITION: GRANTED
PDRService: Available sensors:
  Accelerometer: present [BODY_SENSORS granted] — ... (vendor: ...)
  Gyroscope: present [BODY_SENSORS granted] — ...
  Step Counter: present [ACTIVITY_RECOGNITION granted] — ...
```

**Permission Missing**:
```
PDRService: Permissions — BODY_SENSORS: DENIED, ACTIVITY_RECOGNITION: DENIED
PDRService: One or more sensor permissions missing. Launch PermissionActivity (com.autodesk.pdr.REQUEST_PERMISSIONS) to request them.
```

**Sensor Registration Failure** (Android system log):
```
SensorService: com.mishelk.ad_pdr_service Tried enabling a sensor (step_counter Non-wakeup) without holding android.permission.ACTIVITY_RECOGNITION
```

---

## Related ADRs

- **PDR Algorithm**: `/knowledge-base/build-mobile/pinpoint/pdr-algorithm.md` — sensor fusion, ZUPT detection, heading estimation
- **Test App Integration**: `/knowledge-base/build-mobile/pinpoint/test-app-architecture.md` — how test app binds to service

## Changes Summary

- **2026-06-21**: Implemented transparent PermissionActivity, enhanced PDRService logging, updated manifests for dual-APK permission architecture
