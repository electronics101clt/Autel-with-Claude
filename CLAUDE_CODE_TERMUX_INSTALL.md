# Installing Claude Code in Termux - Complete Guide

**Last Updated**: 2026-06-23
**Purpose**: Document the complete process for installing Claude Code natively in Termux on Android

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Why Native Installation Matters](#why-native-installation-matters)
3. [Common Installation Failures](#common-installation-failures)
4. [The Working Solution](#the-working-solution)
5. [Migration from Older Installs](#migration-from-older-installs)
6. [Accessing Autel Data](#accessing-autel-data)
7. [Removing Unnecessary Ubuntu Proot](#removing-unnecessary-ubuntu-proot)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### 1. Install Debuggable Termux APK

**CRITICAL**: You must use the debuggable GitHub build of Termux. Play Store and F-Droid versions are NOT debuggable.

**Download Latest Release (v0.119.0-beta.3):**

**ARM64 (most modern devices, recommended):**
```
https://github.com/termux/termux-app/releases/download/v0.119.0-beta.3/termux-app_v0.119.0-beta.3+apt-android-7-github-debug_arm64-v8a.apk
```

**Universal (works on all devices, 114MB):**
```
https://github.com/termux/termux-app/releases/download/v0.119.0-beta.3/termux-app_v0.119.0-beta.3+apt-android-7-github-debug_universal.apk
```

**Installation:**
```bash
# Via ADB
adb install termux-app_v0.119.0-beta.3+apt-android-7-github-debug_arm64-v8a.apk

# Or download directly on device and install via file manager
```

**Check for newer releases**: https://github.com/termux/termux-app/releases

### 2. Update Termux Packages

Open Termux and run:
```bash
pkg update && pkg upgrade -y
```

---

## Why Native Installation Matters

Claude Code needs to run **natively in Termux** (not in a proot-distro Ubuntu container) to access Android filesystem paths where Autel diagnostic data is stored.

### Autel Data Location
```
/sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/
```

### Termux Access Path (after storage permission granted)
```
~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/
```

**Native Termux** can access this path directly.
**Proot-distro Ubuntu** cannot (the Android filesystem isn't mounted inside the container).

---

## Common Installation Failures

### ❌ Method 1: Direct npm Install (FAILS)

```bash
pkg install nodejs git -y
npm install -g @anthropic-ai/claude-code
```

**Error:**
```
CANNOT LINK EXECUTABLE "node": cannot locate symbol "_ZTVNSt6__ndk119basic_ostringstreamIcNS_11char_traitsIcEENS_9allocatorIcEEEE" referenced by "/usr/bin/node"...
```

**Why it fails**: Claude Code v2.1.113+ uses a native glibc binary. Android uses Bionic libc, not glibc. Symbol mismatch causes the binary to fail.

### ❌ Method 2: Proot-distro Ubuntu (WORKS, but WRONG approach)

```bash
pkg install proot-distro -y
proot-distro install ubuntu
proot-distro login ubuntu

# Inside Ubuntu:
apt update && apt install nodejs npm -y
npm install -g @anthropic-ai/claude-code
```

**Why it's wrong**:
- ✅ Claude Code runs successfully inside Ubuntu proot
- ❌ Cannot access Android `/sdcard` filesystem
- ❌ Cannot reach Autel diagnostic data
- ❌ Wastes ~1.5GB storage
- ❌ Slower than native Termux

### ❌ Method 3: Manual Binary Copy (FAILS)

Even if you manually download the linux-arm64 binary and copy it to the right location, it fails with the same glibc/Bionic incompatibility.

---

## The Working Solution

### ferrumclaudepilgrim/claude-code-android Wrapper

**GitHub**: https://github.com/ferrumclaudepilgrim/claude-code-android

This project patches the official linux-arm64 binary using Termux's `glibc-runner` package to make it run on Android's Bionic C library.

**Features:**
- ✅ Runs Claude Code natively in Termux
- ✅ Auto-updates and re-patches when Claude Code updates
- ✅ Preserves sessions, login, and settings
- ✅ Full access to Android filesystem (with storage permission)

### Fresh Install

**In native Termux (not proot):**

```bash
curl -fsSL https://raw.githubusercontent.com/ferrumclaudepilgrim/claude-code-android/main/install.sh | bash
```

**Installation time**: 5-10 minutes (~233MB binary + glibc-runner packages)

**What it does:**
1. Checks for existing npm installations and warns about conflicts
2. Downloads the latest official Claude Code binary
3. Installs Termux glibc-runner compatibility layer
4. Patches the binary to run on Bionic
5. Creates wrapper script at `/data/data/com.termux/files/usr/bin/claude`
6. Sets up auto-update mechanism

---

## Migration from Older Installs

If you previously installed Claude Code via npm or an older version of the wrapper, use the migration script instead of the installer.

### Check for Existing Install

```bash
which claude
# If it returns a path, you have an existing install
```

### Run Migration Script

```bash
curl -fsSL https://raw.githubusercontent.com/ferrumclaudepilgrim/claude-code-android/main/migrate.sh -o migrate.sh
bash migrate.sh
```

**What migration does:**
1. Backs up your existing `.claude` directory (sessions, login, settings)
2. Removes conflicting npm global installations
3. Installs the new wrapper-based version
4. Restores your backed-up configuration
5. Preserves all your work and authentication

**Migration output example:**
```
[info] Backing up existing Claude Code data...
[info] Removing npm global install...
[info] Installing Claude Code Android wrapper...
[info] Downloading glibc-runner...
[info] Patching binary...
[info] Restoring configuration...
[success] Migration complete!
```

### Verify Installation

```bash
claude --version
# Should output: Claude Code v2.1.186 (or later)
```

---

## Accessing Autel Data

### 1. Grant Storage Permission

**In Termux**, run:
```bash
termux-setup-storage
```

This will prompt Android to grant storage access. **Accept the permission.**

### 2. Navigate to Autel Data

```bash
cd ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/
```

### 3. List Recording Sessions

```bash
ls -lh
```

**Output example:**
```
drwxrwx--- 2 root sdcard_rw 4.0K May 31 18:41 DATA_202605311841021
-rw-rw---- 1 root sdcard_rw 2.1M May 31 18:42 DATA_202605311841021.zip
drwxrwx--- 2 root sdcard_rw 4.0K May 31 18:42 DATA_202605311842293
-rw-rw---- 1 root sdcard_rw 3.4M May 31 18:43 DATA_202605311842293.zip
```

### 4. Access Latest Recording

```bash
cd $(ls -t | grep DATA_ | head -1)
ls
```

### 5. Launch Claude Code in Autel Data Directory

```bash
claude
```

**Now Claude Code has direct access to your Autel diagnostic recordings!**

---

## Removing Unnecessary Ubuntu Proot

If you previously installed proot-distro Ubuntu as a workaround, you can safely remove it now that Claude Code runs natively.

### Remove Ubuntu Filesystem

```bash
proot-distro remove ubuntu
```

### Remove proot-distro Entirely (optional)

```bash
pkg remove proot-distro -y
```

**Storage saved**: ~1.5GB

---

## Troubleshooting

### Issue: "package not debuggable: com.termux"

**Solution**: You're using the wrong Termux APK. Download the debuggable version from GitHub (see [Prerequisites](#prerequisites)).

### Issue: "Permission denied" accessing Autel data

**Solution**: Run `termux-setup-storage` and grant permission when prompted.

### Issue: Claude Code crashes on startup

**Solution**:
1. Check if you have both npm and wrapper installs conflicting:
   ```bash
   npm list -g @anthropic-ai/claude-code
   which claude
   ```
2. If both exist, remove npm version:
   ```bash
   npm uninstall -g @anthropic-ai/claude-code
   ```
3. Reinstall using migration script

### Issue: "MODULE_NOT_FOUND" errors

**Solution**: This usually means npm installed Claude Code but the native binary didn't download. Use the wrapper instead (it handles this automatically).

### Issue: Binary installed but still shows "native binary not installed"

**Cause**: The official installer's postinstall script doesn't work on Android.

**Solution**: Use the wrapper installer/migration script — it bypasses npm postinstall and handles binary setup manually.

---

## Technical Background

### Why Official Install Fails

1. **glibc vs Bionic**: Linux uses GNU C Library (glibc), Android uses Bionic C Library
2. **Symbol Incompatibility**: Binaries compiled for glibc reference symbols that don't exist in Bionic
3. **No Official Android Binary**: As of v2.1.186, Anthropic hasn't released a native Android build
4. **npm Postinstall Failures**: The official `install.cjs` doesn't detect Android correctly in Termux proot

### How the Wrapper Works

1. **glibc-runner**: Termux package that provides glibc compatibility layer
2. **Binary Patching**: Modifies ELF headers and linkage to use glibc-runner
3. **Wrapper Script**: Shell script that sets up environment and executes patched binary
4. **Auto-Update**: Monitors for Claude Code updates and re-patches automatically

### Platform Detection in install.cjs

The official installer detects platform like this:

```javascript
const PLATFORMS = {
  'darwin-x64': '@anthropic-ai/claude-code-darwin-x64',
  'darwin-arm64': '@anthropic-ai/claude-code-darwin-arm64',
  'linux-x64': '@anthropic-ai/claude-code-linux-x64',
  'linux-arm64': '@anthropic-ai/claude-code-linux-arm64',
  'win32-x64': '@anthropic-ai/claude-code-win32-x64'
}
```

**Notice**: `android-arm64` is not in the list. Even though Node.js reports `process.platform === 'linux'` in Termux, the binary still fails due to glibc dependency.

---

## References

- **Claude Code Android Wrapper**: https://github.com/ferrumclaudepilgrim/claude-code-android
- **Official Claude Code**: https://www.npmjs.com/package/@anthropic-ai/claude-code
- **Termux GitHub Releases**: https://github.com/termux/termux-app/releases
- **Termux glibc-runner**: https://github.com/termux/glibc-packages
- **Android Bionic C Library**: https://android.googlesource.com/platform/bionic/

---

## Contributing

If you encounter issues or discover improvements to this installation process, please:

1. Open an issue in this repository
2. Contribute fixes to the upstream wrapper project: https://github.com/ferrumclaudepilgrim/claude-code-android
3. Report glibc compatibility issues to Termux: https://github.com/termux/termux-packages

---

## License

This documentation: MIT License

Referenced projects maintain their own licenses:
- Claude Code: Anthropic license
- claude-code-android wrapper: MIT License
- Termux: GPLv3

---

**Last Verified**: 2026-06-23
**Claude Code Version**: v2.1.186
**Termux Version**: v0.119.0-beta.3
