# Patching Autel MaxiAP200 APK as Debuggable

**Last Updated**: 2026-06-23
**Purpose**: Enable direct Termux access to Autel diagnostic data by making the app debuggable

---

## Why Patch the APK?

Android 15's scoped storage prevents Termux from accessing `/sdcard/Android/data/com.autel.maxiap200.autelap/` even with storage permission granted.

**Solution**: Make Autel debuggable, then use `run-as` from Termux to access its data directly.

---

## What Was Changed

### Original APK
- **Package**: `com.autel.maxiap200.autelap`
- **Version**: 1.64 (versionCode 103)
- **Debuggable**: ❌ NO

### Patched APK
- **Package**: `com.autel.maxiap200.autelap` (unchanged)
- **Version**: 1.64 (versionCode 103) (unchanged)
- **Debuggable**: ✅ YES

### AndroidManifest.xml Change

**Before**:
```xml
<application android:allowBackup="false" ... >
```

**After**:
```xml
<application android:allowBackup="false" android:debuggable="true" ... >
```

That's it. One attribute added.

---

## Installation

### Install Patched APK

```bash
adb install autel-maxiap200-debuggable-aligned.apk
```

### Verify Debuggable Status

```bash
adb shell 'run-as com.autel.maxiap200.autelap whoami'
```

**Expected output**: `u0_a489` (or similar UID)

---

## Accessing Autel Data from Termux

```bash
# Access via run-as
adb shell 'run-as com.autel.maxiap200.autelap ls /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/'
```

---

**Status**: ✅ Patch Complete
**APK**: `autel-maxiap200-debuggable-aligned.apk` (106 MB)
