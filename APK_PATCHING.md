# APK Patching - Making Apps Debuggable for Termux Data Access

**Purpose**: Enable direct filesystem access to Autel MaxiAP200 app data from Termux by patching the APK to be debuggable.

---

## Overview

Android's scoped storage restricts Termux from accessing `/sdcard/Android/data/` directories of other apps. By patching the Autel APK to be debuggable, we can use `run-as` to access its private data directory directly.

---

## Current Autel App Status

**Package**: `com.autel.maxiap200.autelap`
**Version**: 1.64 (versionCode 103)
**Debuggable**: ❌ NO (defaults to false)
**FileProvider**: ✅ EXISTS but exported="false"

### Key Finding - file_paths.xml

The Autel app already has a FileProvider configured:

```xml
<paths>
    <external-path name="external_storage_root" path="." />
</paths>
```

This shares the entire external storage root through FileProvider, but it's not directly accessible because the provider is not exported.

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

**Access Method**:
```bash
# From Termux (with debuggable Autel installed)
run-as com.autel.maxiap200.autelap sh -c 'cat /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_*/​*.txt' > ~/autel_data.txt
```

---

### Option 2: Shared User ID with Termux

**Edit**: `autel-decompiled/AndroidManifest.xml`

**Add to `<manifest>` tag**:
```xml
android:sharedUserId="com.termux"
```

**Result**: Autel and Termux share the same Linux UID, full access to each other's data

**WARNING**: This requires both apps to be signed with the same certificate, which means:
1. You'd need to rebuild and re-sign BOTH Termux and Autel
2. Replacing system Termux would break existing Termux data
3. Not recommended unless you're building both from source

---

### Option 3: Export FileProvider (RISKY)

**Edit**: `autel-decompiled/AndroidManifest.xml`

**Change**:
```xml
<provider android:exported="false" ... >
```

**To**:
```xml
<provider android:exported="true" ... >
```

**Result**: Any app can request Autel files via `content://` URIs

**Security Risk**: This exposes potentially sensitive diagnostic data to all apps on the device. Not recommended unless you fully understand the implications.

---

## Patching Workflow

### Prerequisites

```bash
# Install required tools
sudo apt install -y aapt apktool openjdk-17-jdk

# Or on Termux
pkg install aapt apktool openjdk-17
```

### Step 1: Pull the APK from Device

```bash
# Get APK path
adb shell 'pm path com.autel.maxiap200.autelap'

# Output example:
# package:/data/app/~~HASH~~/com.autel.maxiap200.autelap-HASH/base.apk

# Pull the APK
adb pull /data/app/~~HASH~~/com.autel.maxiap200.autelap-HASH/base.apk ~/Autel-with-Claude/autel-maxiap200.apk
```

### Step 2: Decompile the APK

```bash
cd ~/Autel-with-Claude
apktool d autel-maxiap200.apk -o autel-decompiled
```

### Step 3: Check Current Manifest

```bash
cd autel-decompiled
grep -A 20 "<application" AndroidManifest.xml | head -25
```

Look for `android:debuggable="true"` - if missing, the app is NOT debuggable.

### Step 4: Patch the Manifest

**Edit**: `autel-decompiled/AndroidManifest.xml`

Find the `<application` tag (around line 20-30) and add `android:debuggable="true"`:

**Before**:
```xml
<application
    android:allowBackup="false"
    android:appComponentFactory="com.secneo.apkwrapper.AP"
    android:extractNativeLibs="false"
    android:icon="@mipmap/icon_applogo"
    android:label="@string/app_name"
    ...>
```

**After**:
```xml
<application
    android:allowBackup="false"
    android:debuggable="true"
    android:appComponentFactory="com.secneo.apkwrapper.AP"
    android:extractNativeLibs="false"
    android:icon="@mipmap/icon_applogo"
    android:label="@string/app_name"
    ...>
```

### Step 5: Recompile the APK

```bash
cd ~/Autel-with-Claude
apktool b autel-decompiled -o autel-maxiap200-debuggable.apk
```

### Step 6: Sign the APK

Android requires all APKs to be signed. We'll use a debug keystore:

```bash
# Generate debug keystore (if you don't have one)
keytool -genkey -v -keystore ~/debug.keystore -alias androiddebugkey \
  -keyalg RSA -keysize 2048 -validity 10000 -storepass android -keypass android \
  -dname "CN=Android Debug,O=Android,C=US"

# Sign the patched APK
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore ~/debug.keystore -storepass android -keypass android \
  autel-maxiap200-debuggable.apk androiddebugkey

# Verify signature
jarsigner -verify -verbose -certs autel-maxiap200-debuggable.apk
```

**Optional**: Zipalign for optimization:
```bash
zipalign -v 4 autel-maxiap200-debuggable.apk autel-maxiap200-debuggable-aligned.apk
```

### Step 7: Install the Patched APK

```bash
# Uninstall original Autel app first (IMPORTANT: This deletes app data!)
adb uninstall com.autel.maxiap200.autelap

# Install patched version
adb install autel-maxiap200-debuggable.apk
```

**WARNING**: Uninstalling the original app will delete all Autel app settings, saved recordings, and calibration data. Back up important data first using:
```bash
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/ ~/autel-backup/
```

### Step 8: Verify Debuggable Status

```bash
# Check if debuggable
adb shell 'dumpsys package com.autel.maxiap200.autelap | grep -i debuggable'

# Should output: debuggable=true
```

### Step 9: Test run-as Access

```bash
# From ADB
adb shell 'run-as com.autel.maxiap200.autelap sh -c "ls /data/data/com.autel.maxiap200.autelap/"'

# From Termux
run-as com.autel.maxiap200.autelap sh -c 'ls /sdcard/Android/data/com.autel.maxiap200.autelap/files/'
```

If this works, you now have direct access to Autel's data directories!

---

## Using run-as to Access Autel Data

Once the patched APK is installed, you can access Autel data from Termux:

### Access External Storage Data

```bash
# List DataLogging recordings
run-as com.autel.maxiap200.autelap sh -c 'ls -lh /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/'

# Copy latest recording to Termux home
run-as com.autel.maxiap200.autelap sh -c 'cat /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_*/​*.txt' > ~/latest_recording.txt

# List PDF reports
run-as com.autel.maxiap200.autelap sh -c 'ls -lh /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/Data/PDF/'
```

### Access Internal App Data

```bash
# Explore app's private data directory
run-as com.autel.maxiap200.autelap sh -c 'ls -lR /data/data/com.autel.maxiap200.autelap/'

# Check shared preferences (app settings)
run-as com.autel.maxiap200.autelap sh -c 'cat /data/data/com.autel.maxiap200.autelap/shared_prefs/*.xml'

# Access app databases
run-as com.autel.maxiap200.autelap sh -c 'ls -lh /data/data/com.autel.maxiap200.autelap/databases/'
```

### Create Symlink for Easy Access

```bash
# Create a symlink in Termux home pointing to Autel data
ln -s /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan ~/autel-data

# Now you can access it directly (through run-as):
run-as com.autel.maxiap200.autelap sh -c 'cat ~/autel-data/DataLogging/data/DATA_*/​*.txt'
```

---

## Troubleshooting

### "Package not debuggable" Error

**Symptom**:
```bash
run-as com.autel.maxiap200.autelap
# Error: run-as: package not debuggable: com.autel.maxiap200.autelap
```

**Fix**:
1. Verify patched APK is actually installed:
   ```bash
   adb shell 'dumpsys package com.autel.maxiap200.autelap | grep versionName'
   ```
2. Check if `android:debuggable="true"` is in the manifest:
   ```bash
   aapt dump badging autel-maxiap200-debuggable.apk | grep debuggable
   ```
3. Ensure you installed the patched APK (not the original)

### "Verification failed" During Installation

**Symptom**:
```
adb install autel-maxiap200-debuggable.apk
# Failure [INSTALL_PARSE_FAILED_NO_CERTIFICATES]
```

**Fix**: APK wasn't signed properly. Re-run the signing step:
```bash
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore ~/debug.keystore -storepass android -keypass android \
  autel-maxiap200-debuggable.apk androiddebugkey
```

### App Crashes After Patching

**Possible Causes**:
1. **APK obfuscation/protection**: Some apps detect modifications and refuse to run
2. **Signature verification**: App checks its own signature at runtime
3. **Native library issues**: Decompilation/recompilation corrupted native code

**Workaround**:
- Try disabling signature verification (advanced, requires Frida or Xposed)
- Use Option 3 (export FileProvider) instead of making debuggable
- Root device and access data directly without patching

### Original App Data Lost

**Symptom**: After reinstalling patched APK, all Autel settings and data are gone.

**Cause**: Uninstalling the original app deleted app data.

**Prevention**: Always backup before uninstalling:
```bash
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/ ~/autel-backup/
adb backup -f autel.ab com.autel.maxiap200.autelap
```

**Restore**:
```bash
adb restore autel.ab
adb push ~/autel-backup/ /sdcard/Android/data/com.autel.maxiap200.autelap/
```

---

## Security Considerations

### Making an App Debuggable

**Risks**:
- Any process with ADB access can read/write app's private data
- Passwords, API keys, session tokens in app data are exposed
- On rooted devices, any app can use `run-as`

**Mitigations**:
- Only patch apps on devices you fully control
- Don't patch banking/financial apps
- Keep ADB disabled when not in use
- Use this only for diagnostics and development

### Signing with Debug Key

**Risks**:
- Debug keystore is not secure (password is "android" by default)
- Anyone with the APK can extract and modify it
- Updates from Play Store won't install (signature mismatch)

**Mitigations**:
- Generate a unique keystore with strong password
- Don't distribute patched APKs to others
- Keep original APK for restoring official version

---

## Alternative: Root-Based Access

If you have root access on your device, you don't need to patch the APK:

```bash
# From Termux with root
su -c 'cp /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_*/​*.txt /sdcard/Download/'

# Or use root file manager apps
```

But root is not required if you use the patching method above.

---

## References

- [Android Debug Bridge (ADB) Documentation](https://developer.android.com/tools/adb)
- [APKTool Documentation](https://apktool.org/)
- [Android App Signing](https://developer.android.com/studio/publish/app-signing)
- [run-as command usage](https://developer.android.com/studio/command-line/adb#run-as)

---

**Last Updated**: 2026-06-23
**Status**: Documented, not yet implemented
**Next Step**: Patch and test Autel APK for debuggable access

