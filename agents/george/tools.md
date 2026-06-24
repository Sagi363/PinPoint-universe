# George's Tools

## Runtime Configuration

```yaml
runtime:
  preferred:
    tool: claude
    model: opus
max-turns: 40
allowed: [Read, Edit, Write, Bash, Grep]
```

### Rationale

- **claude/opus**: Service implementation requires deliberate code production with disciplined tool usage. Opus calls tools earlier (less internal monologue) and produces more careful code on multi-file AIDL + service + algorithm work. Slower per token, fewer wasted tokens overall.
- **max-turns: 40**: AIDL + service wiring touches many files in sequence; a single ask can span manifest, build.gradle, AIDL, Parcelable, service, sensor manager, algorithm, listener registration, and tests.
- **Read, Edit, Write, Bash, Grep**: Edit existing files (manifest, `build.gradle.kts`), Write new ones (AIDL, Kotlin classes, the service), Bash for `./gradlew assembleDebug` and `adb logcat`, Grep for sensor/permission patterns.

## Project Paths

| What | Path |
|---|---|
| Project root | `/Users/mishelkvit/Projects/Autodesk/quickplacement` |
| Module root | `/Users/mishelkvit/Projects/Autodesk/quickplacement/app` |
| Kotlin source | `/Users/mishelkvit/Projects/Autodesk/quickplacement/app/src/main/java` |
| AIDL source | `/Users/mishelkvit/Projects/Autodesk/quickplacement/app/src/main/aidl` (create if missing) |
| Manifest | `/Users/mishelkvit/Projects/Autodesk/quickplacement/app/src/main/AndroidManifest.xml` |
| Module build | `/Users/mishelkvit/Projects/Autodesk/quickplacement/app/build.gradle.kts` |
| Root build | `/Users/mishelkvit/Projects/Autodesk/quickplacement/build.gradle.kts` |
| Build command | `cd /Users/mishelkvit/Projects/Autodesk/quickplacement && ./gradlew app:assembleDebug` |
| Install command | `cd /Users/mishelkvit/Projects/Autodesk/quickplacement && ./gradlew app:installDebug` |

## Reference Files in This Agent

- `agents/george/service-contract.md` — the AIDL contract derived from the QuickPlacement hackathon flow (initTracking, updatePosition, position stream, state machine, open questions).

## Skills (Domain Knowledge)

```yaml
skills:
  - android/permissions-model
  - android/services-and-lifecycle
  - android/sensor-framework
  - android/foreground-services
  - android/aidl-and-binder
  - android/version-compatibility
  - pdr/step-detection
  - pdr/heading-estimation
  - pdr/dead-reckoning
  - pdr/calibration-from-anchors
```
