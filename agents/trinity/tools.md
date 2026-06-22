# Trinity's Tools

## Runtime Configuration

```yaml
model: sonnet
max-turns: 30
allowed: [Read, Edit, Bash, Grep]
```

### Rationale

- **sonnet**: Android implementation requires practical engineering judgment and knowledge of the framework. Sonnet balances depth and speed for iterative debugging.
- **max-turns: 30**: Android issues often require multiple read-diagnose-fix cycles (manifest check, code update, test, repeat).
- **Read/Edit/Bash/Grep**: Core tools for Android development—reading manifests, editing code, running builds/tests, grepping logs.

## Reference Assets

**AndroidManifest.xml** files in both projects:
- Service APK: `/Users/mishelkvit/Projects/Autodesk/ad-pdr-service/app/src/main/AndroidManifest.xml`
- Test app: `/Users/mishelkvit/Projects/Autodesk/pdr_test_app/app/src/main/AndroidManifest.xml`

**Sensor logs** from the issue:
- Look for `SensorService` and `SensorManager` logs
- Missing permission patterns: `Tried enabling a sensor without holding`

## Skills (Domain Knowledge)

```yaml
skills:
  - android/permissions-model
  - android/services-and-lifecycle
  - android/sensor-framework
  - android/foreground-services
  - android/inter-process-communication
  - android/version-compatibility
```
