# Patching Autel MaxiAP200 APK for Termux Data Access

## Current Autel App Status

**Package**: `com.autel.maxiap200.autelap`
**Version**: 1.64 (versionCode 103)
**Debuggable**: ❌ NO (defaults to false)
**FileProvider**: ✅ EXISTS but exported="false"

### Key Finding - file_paths.xml

```xml
<paths>
    <external-path name="external_storage_root" path="." />
</paths>
```

Autel already shares the entire external storage root through FileProvider, but it's not directly accessible.

---

## Patching Options

### Option 1: Make Debuggable (RECOMMENDED)

**Edit**: `autel-decompiled/AndroidManifest.xml`

**Change**:
```xml
<application android:allowBackup="false" ... >
```

**To**:
```xml
<application android:allowBackup="false" android:debuggable="true" ... >
```

**Result**:
- `run-as com.autel.maxiap200.autelap` will work from ADB
- Direct access to `/data/data/com.autel.maxiap200.autelap/` from Termux
- Can read DataLogging files directly without copying

---

### Option 2: Shared User ID with Termux

**Add to `<manifest>` tag**:
```xml
android:sharedUserId="com.termux"
```

**Result**: Autel and Termux share the same Linux UID, full access to each other's data

---

### Option 3: Export FileProvider (RISKY)

**Change**:
```xml
<provider android:exported="false" ... >
```

**To**:
```xml
<provider android:exported="true" ... >
```

**Result**: Any app can request Autel files via content:// URIs (security risk)

---

## Recommended Approach: Option 1 (Debuggable)

### Workflow

1. **Decompile**:
   ```bash
   apktool d autel-maxiap200.apk -o autel-decompiled
   ```

2. **Patch**:
   Add `android:debuggable="true"` to AndroidManifest.xml

3. **Recompile**:
   ```bash
   apktool b autel-decompiled -o autel-maxiap200-debuggable.apk
   ```

4. **Sign**:
   ```bash
   jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
     -keystore ~/debug.keystore -storepass android -keypass android \
     autel-maxiap200-debuggable.apk androiddebugkey
   ```

5. **Install**:
   ```bash
   adb uninstall com.autel.maxiap200.autelap
   adb install autel-maxiap200-debuggable.apk
   ```

6. **Test**:
   ```bash
   run-as com.autel.maxiap200.autelap sh -c 'ls /sdcard/Android/data/com.autel.maxiap200.autelap/files/'
   ```

---

## Access Methods

### From Termux with Debuggable Autel

```bash
# List DataLogging recordings
run-as com.autel.maxiap200.autelap sh -c 'ls -lh /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/'

# Copy latest recording to Termux home
run-as com.autel.maxiap200.autelap sh -c 'cat /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_*/​*.txt' > ~/latest_recording.txt

# Access PDF reports
run-as com.autel.maxiap200.autelap sh -c 'ls -lh /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/Data/PDF/'
```

---

## Benefits

✅ Direct filesystem access to Autel data from Termux
✅ No root required
✅ No copying files to shared storage
✅ Works with Android scoped storage restrictions
✅ Can read data in real-time as Autel records

---

**See [APK_PATCHING.md](APK_PATCHING.md) for complete guide with security considerations and troubleshooting.**
