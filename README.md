# Autel with Claude - Vehicle Diagnostics Analysis Using AI

## Overview

**Formerly**: `termux-adb-tty-security`
**Now**: Complete guide for analyzing Autel MaxiAP200 diagnostic data using Claude Code running natively on Android via Termux.

This repository documents:
1. **Installing Claude Code in Termux** - Native Android installation without root
2. **Accessing Autel Diagnostic Data** - Direct filesystem access to OBD-II recordings
3. **ADB/Termux Security** - Understanding `run-as` PTY requirements and terminal injection vulnerabilities

## Repository Contents

- **[CLAUDE_CODE_TERMUX_INSTALL.md](CLAUDE_CODE_TERMUX_INSTALL.md)** - Complete Claude Code installation guide for Termux
- **[README.md](README.md)** (this file) - Quick start, Autel data locations, and ADB security documentation
- **[autel-conversation.md](autel-conversation.md)** - Detailed conversation log documenting the discovery process

---

## 📚 Table of Contents

1. [Quick Start - Using Claude Code with Autel Data](#-quick-start---using-claude-code-with-autel-data)
2. [Autel MaxiAP200 Data Storage Locations](#autel-maxiap200-data-storage-locations)
3. [ADB/Termux Security Background](#the-problem)
4. [TIOCSTI Terminal Injection Vulnerability](#why-this-happens-tiocsti-terminal-injection-vulnerability)
5. [Methods to Access Termux via ADB](#methods-to-access-termux-via-adb)

---

## ADB/Termux Security Background

This section documents:
1. Why `run-as com.termux` fails with "package not debuggable" when using standard Termux APKs
2. The security reasons behind `run-as` requiring proper PTY allocation in terminal sessions
3. Why TIOCSTI terminal injection attacks require isolation even after fixing debuggability

## The Problem

### Primary Issue: Termux Not Debuggable

**Using standard Termux APKs (Play Store, F-Droid):**
```bash
adb shell 'run-as com.termux /data/data/com.termux/files/usr/bin/bash'
# Result: "run-as: package not debuggable: com.termux"
```

**Root cause**: Play Store and F-Droid Termux APKs are NOT marked as debuggable in their AndroidManifest.xml.

**Solution**: Install debuggable Termux build from GitHub releases.

### Secondary Issue: PTY Requirement (When Using Debuggable APK)

Even with a debuggable Termux APK:

```bash
# This FAILS in detached tmux:
tmux new-session -d -s termux 'adb shell'
tmux send-keys -t termux 'run-as com.termux /data/data/com.termux/files/usr/bin/bash' C-m
# Result: Security check fails due to no proper PTY

# This WORKS in visible terminal:
gnome-terminal -- tmux new-session -s termux 'adb shell'
tmux send-keys -t termux 'run-as com.termux /data/data/com.termux/files/usr/bin/bash' C-m
# Result: Successfully enters Termux bash
```

## Why This Happens: TIOCSTI Terminal Injection Vulnerability

### The TIOCSTI Attack Vector

The TTY requirement in `run-as` is a **security measure to prevent terminal injection attacks**.

When privilege-switching utilities (like `run-as`, `su`, or `sudo`) change the UID of a process, the terminal remains accessible to the new (possibly less privileged) user. An attacker can exploit this through the `TIOCSTI ioctl()` system call.

**Attack Example:**

```c
// Non-privileged code can inject commands into the privileged terminal
int tty_fd = open("/dev/tty", O_RDWR);
char *malicious_cmd = "rm -rf /\n";
for (int i = 0; malicious_cmd[i]; i++) {
    ioctl(tty_fd, TIOCSTI, &malicious_cmd[i]);
}
```

This injects fake keyboard input into the terminal buffer. After the non-privileged shell exits, the privileged shell reads this "input" and executes it.

### How Proper PTY Prevents This

When a proper pseudo-terminal is allocated (like in a visible terminal window):

1. **Terminal Isolation**: The new process gets its own isolated PTY
2. **Separate /dev/tty**: The non-privileged process cannot access the original terminal
3. **No Injection Path**: TIOCSTI cannot affect the privileged terminal

**In Detached Sessions:**
- No proper PTY allocated
- Terminal isolation incomplete
- `run-as` **refuses to execute** as security protection

**In Visible Terminal:**
- Full PTY allocated with proper terminal device
- Complete terminal isolation
- `run-as` allows execution safely

## Download Debuggable Termux APK

**CRITICAL:** You must use the debuggable GitHub build of Termux for `run-as` to work. Play Store and F-Droid versions are NOT debuggable.

### Latest Release (v0.119.0-beta.3)

Download the appropriate version for your device:

**Universal (works on all devices, 114MB):**
```
https://github.com/termux/termux-app/releases/download/v0.119.0-beta.3/termux-app_v0.119.0-beta.3+apt-android-7-github-debug_universal.apk
```

**ARM64 (most modern devices, smaller):**
```
https://github.com/termux/termux-app/releases/download/v0.119.0-beta.3/termux-app_v0.119.0-beta.3+apt-android-7-github-debug_arm64-v8a.apk
```

**ARMv7 (older 32-bit devices):**
```
https://github.com/termux/termux-app/releases/download/v0.119.0-beta.3/termux-app_v0.119.0-beta.3+apt-android-7-github-debug_armeabi-v7a.apk
```

**Installation:**
```bash
# Download and install via ADB
adb install termux-app_v0.119.0-beta.3+apt-android-7-github-debug_arm64-v8a.apk
```

**Check Latest Releases:**
https://github.com/termux/termux-app/releases

---

## Methods to Access Termux via ADB

### Method 1: Visible tmux Session (WORKS)

```bash
# Open visible terminal with tmux session
gnome-terminal -- tmux new-session -s adb-termux 'adb shell'

# Send commands to the session
tmux send-keys -t adb-termux 'run-as com.termux /data/data/com.termux/files/usr/bin/bash' C-m

# Now you can interact with Termux
```

### Method 2: Direct Interactive Session (WORKS)

```bash
# Start ADB shell interactively
adb shell

# Then run inside the shell:
run-as com.termux /data/data/com.termux/files/usr/bin/bash
```

### Method 3: Root Access (WORKS, but requires root)

```bash
adb shell "su -c '/data/data/com.termux/files/usr/bin/bash -c \"your-command\"'"
```

### Method 4: Input Text Automation (LIMITED)

```bash
# Open Termux app
adb shell am start -n com.termux/com.termux.app.TermuxActivity

# Type command
adb shell input text 'ls -la'

# Press enter
adb shell input keyevent 66
```

**Limitation**: Output only appears on device screen, not in your terminal.

### Method 5: Tasker + Termux:Tasker (COMPLEX)

Requires installing Tasker (paid app) and Termux:Tasker plugin. Commands can be triggered via intents but output must be redirected to files.

## What Doesn't Work

### ❌ Detached tmux/screen Sessions

```bash
# FAILS: No proper PTY
tmux new-session -d -s termux 'adb shell'
tmux send-keys -t termux 'run-as com.termux ...' C-m
# Error: "run-as: package not debuggable"
```

### ❌ Direct File Path Access

```bash
# FAILS: Permission denied
adb shell '/data/data/com.termux/files/usr/bin/bash'
adb shell 'ls /data/data/com.termux/'
```

### ❌ run-as with Package Not Debuggable

```bash
# FAILS: Termux is not marked debuggable
adb shell 'run-as com.termux'
# (without proper PTY or if app isn't debuggable)
```

## Security Best Practices

1. **Always use proper terminal allocation** when running privilege-switching commands
2. **Avoid detached automation** that bypasses PTY security checks
3. **Use `sudo`'s `use_pty` option** on Linux systems to enforce terminal isolation
4. **Be aware of TIOCSTI** when writing privilege-escalation or user-switching code

## Technical Details

### TIOCSTI ioctl() Call

- **System Call**: `ioctl(fd, TIOCSTI, &char)`
- **Purpose**: Simulate terminal input (originally for terminal testing)
- **Security Risk**: Can inject commands into privileged sessions
- **Mitigation**: Proper PTY allocation and terminal isolation

### PTY vs TTY

- **TTY**: Physical or virtual terminal device
- **PTY**: Pseudo-terminal (software-emulated terminal)
- **Security Difference**: PTY can enforce proper isolation between sessions

## References

- [su/sudo from root to another user allows TTY hijacking and arbitrary code execution](https://ruderich.org/simon/notes/su-sudo-from-root-tty-hijacking)
- [Bypassing the "run-as" debuggability check on Android via newline injection](https://rtx.meta.security/exploitation/2024/03/04/Android-run-as-forgery.html)
- [runuser should defend against TIOCSTI terminal injection](https://github.com/util-linux/util-linux/issues/760)
- [Please consider enabling option `use_pty` by default (security)](https://github.com/sudo-project/sudo/issues/258)
- [Android Debug Bridge (adb) Documentation](https://developer.android.com/tools/adb)

## Contributing

If you've discovered additional methods or security considerations related to Android `run-as` and PTY requirements, please open an issue or submit a pull request.

## License

MIT License - See LICENSE file for details

---

## 🚀 Quick Start - Using Claude Code with Autel Data

For complete installation instructions, see **[CLAUDE_CODE_TERMUX_INSTALL.md](CLAUDE_CODE_TERMUX_INSTALL.md)**

**TL;DR**:
```bash
# Install Claude Code natively in Termux
curl -fsSL https://raw.githubusercontent.com/ferrumclaudepilgrim/claude-code-android/main/install.sh | bash

# Grant storage access
termux-setup-storage

# Navigate to Autel diagnostic data
cd ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/

# Launch Claude Code
claude
```

---

## Autel MaxiAP200 Data Storage Locations

**Package**: `com.autel.maxiap200.autelap`

### Recording/Data Logging Files

**Location**: `/sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/`

**Structure**:
- Each recording session creates a directory: `DATA_YYYYMMDDHHMMSS/`
- Contains binary `.txt` files with OBD-II data stream
- Also creates `.zip` archives of the sessions

**Example**:
```
/sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/
├── DATA_202605311841021/
│   └── 20260531183943_842133572.txt  (binary OBD-II data)
├── DATA_202605311841021.zip
├── DATA_202605311842293/
│   └── [timestamp].txt
└── DATA_202605311842293.zip
```

### PDF Reports

**Location**: `/sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/Data/PDF/`

### Accessing from Termux

```bash
# In Termux, after granting storage permission:
termux-setup-storage

# Navigate to Autel data:
cd ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/

# List recording sessions:
ls -lh

# Access latest recording:
cd $(ls -t | head -1)

# The data files are in binary format - will need parsing
```

**Note**: The recording files contain binary-encoded OBD-II diagnostic data. A parser will be needed to decode the vehicle data streams for use with Claude Code in Termux.

---

**Last Updated**: 2026-06-23
**Discovered During**: Attempts to automate Termux access via ADB without root and locate Autel AP200 diagnostic data
