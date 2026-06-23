# Autel Data Analysis Workflows - Native Termux vs Laptop ADB

**Last Updated**: 2026-06-23
**Purpose**: Compare and document two different approaches to analyzing Autel MaxiAP200 diagnostic data with Claude Code

---

## Quick Summary

**Two Workflows Available:**

### Workflow A: Native Claude in Termux (On-Device)
- ❌ **Cannot access Autel data directly** (Secneo protection blocks `run-as`)
- ✅ **Can access other app data** (if apps are debuggable)
- ✅ **Works offline** (no laptop needed)
- ⚠️ **Limited** by Android device resources

### Workflow B: Laptop with ADB (Recommended for Autel)
- ✅ **Full access to Autel data** (confirmed working)
- ✅ **Unlimited resources** (laptop CPU/RAM)
- ✅ **Better for large datasets**
- ❌ **Requires laptop + USB cable**

---

## Workflow A: Native Claude Code in Termux (On Android Device)

### Prerequisites

**Required:**
1. Debuggable Termux APK installed (v0.119.0-beta.3+)
   - See [CLAUDE_CODE_TERMUX_INSTALL.md](CLAUDE_CODE_TERMUX_INSTALL.md)
2. Claude Code installed in Termux
3. Claude Pro/Max subscription
4. Storage permission granted (`termux-setup-storage`)

**Limitations:**
- ❌ **CANNOT access Autel MaxiAP200 data** (app uses Secneo protection)
- ❌ **CANNOT access most commercial apps** (not debuggable)
- ✅ **CAN access debuggable apps** (custom apps, some open-source apps)

---

### What You CAN Do with Native Termux Claude

#### 1. Analyze Files in Termux Home Directory

```bash
# In Termux
cd ~
mkdir autel-analysis
cd autel-analysis

# Files you manually copied here (via file manager or ADB)
ls -lh
# Output:
# diagnostic_20260623.txt
# report.pdf

# Launch Claude
claude
```

**Claude Prompt:**
```
Analyze the OBD-II diagnostic file diagnostic_20260623.txt in this directory.
Parse the binary format and extract all sensor readings.
```

**Use Case:**
- Analyzing diagnostic data you manually copied to Termux storage
- Working with files downloaded from cloud
- Processing data you exported via other means

---

#### 2. Access Shared Storage (Downloads, Documents)

```bash
# In Termux (after termux-setup-storage)
cd ~/storage/shared/Download

# List files
ls -lh

# Launch Claude in this directory
claude
```

**Claude Prompt:**
```
I have Autel PDF reports in this directory.
Parse all PDFs and create a summary of diagnostic findings.
```

**What's Accessible:**
- ✅ `/storage/emulated/0/Download/` (Downloads folder)
- ✅ `/storage/emulated/0/Documents/`
- ✅ `/storage/emulated/0/DCIM/` (Camera)
- ❌ `/storage/emulated/0/Android/data/` (Scoped storage - blocked)

---

#### 3. Analyze Data from Loop1 HOP Method

**Workflow:**
1. Use Loop1 HOP to export Autel PDFs to `/sdcard/Download/`
   - See [LOOP1_HOP.md](LOOP1_HOP.md)
2. Access from Termux

```bash
# In Termux
cd ~/storage/shared/Download

# List Autel PDF exports (from Loop1 method)
ls -lh *.pdf

# Launch Claude
claude
```

**Claude Prompt:**
```
Analyze the file _.~~tmp~~ (2).pdf which is an Autel diagnostic report.
Extract all DTCs, sensor readings, and create a summary.
Compare with the previous report if available.
```

**Use Case:**
- Analyzing exported PDFs (Loop1 method)
- On-device analysis without laptop
- Quick checks while in vehicle

---

#### 4. Work with Data You Manually Transfer

**Transfer Methods:**

**Method 1: File Manager**
```
1. Open Android file manager
2. Navigate to /sdcard/Android/data/com.autel.maxiap200.autelap/files/
3. Copy files
4. Paste to /sdcard/Download/
5. Access from Termux: ~/storage/shared/Download/
```

**Method 2: Termux via Manual Copy**
```bash
# Copy from Downloads to Termux home (if you put files there first)
cp ~/storage/shared/Download/diagnostic_data.txt ~/autel-analysis/

# Now work in Termux home
cd ~/autel-analysis
claude
```

**Use Case:**
- One-time analysis
- Files already manually exported
- Working offline without ADB

---

### Termux Workflow Steps (When Autel Data is Manually Available)

**Step 1: Prepare Data**
```bash
# Create working directory
mkdir -p ~/autel-analysis
cd ~/autel-analysis

# Copy data from shared storage
cp ~/storage/shared/Download/autel_*.txt ./
cp ~/storage/shared/Download/autel_*.pdf ./

# Or download from cloud
# curl https://example.com/autel_data.zip -o data.zip
# unzip data.zip
```

**Step 2: Launch Claude**
```bash
claude
```

**Step 3: Analyze with Claude**

**Example Prompts:**

```
List all files in this directory and identify Autel diagnostic files.
```

```
Read the file autel_20260623.txt and determine its format.
Is it binary or text? What structure does it follow?
```

```
Parse all .txt files in this directory as OBD-II diagnostic data.
Create a CSV with columns: timestamp, PID, value, unit.
```

```
Compare autel_20260620.txt with autel_20260623.txt.
What changed between these two recordings?
```

```
Extract all DTCs (Diagnostic Trouble Codes) from these files.
Look up their meanings and create a report.
```

**Step 4: Export Results**
```bash
# Claude creates files in current directory
ls -lh
# Output:
# analysis_report.md
# sensor_data.csv
# dtc_summary.txt

# Copy to shared storage for access from other apps
cp *.md ~/storage/shared/Documents/
cp *.csv ~/storage/shared/Documents/
```

---

### Advantages of Native Termux Workflow

✅ **Portable**
- Work anywhere with just your phone
- No laptop required
- Analyze data in vehicle or on-site

✅ **Offline**
- No internet needed (after Claude Code installed)
- Privacy (data never leaves device)
- Works in remote areas

✅ **Convenient**
- Always with you
- Quick analysis on-the-go
- No setup time

---

### Disadvantages of Native Termux Workflow

❌ **Cannot Access Autel App Data Directly**
- Must manually copy files first
- Extra step in workflow
- Requires file manager or ADB

❌ **Limited Resources**
- Phone CPU slower than laptop
- Limited RAM (Android reserves memory for system)
- Battery drain

❌ **Small Screen**
- Hard to review large datasets
- Difficult to compare multiple files
- Limited terminal space

❌ **Typing on Phone**
- Slower input
- More errors
- No keyboard shortcuts

---

## Workflow B: Claude on Laptop with ADB Access (RECOMMENDED)

### Prerequisites

**Required:**
1. Laptop with Claude Code installed
2. Android device with USB debugging enabled
3. ADB installed on laptop
4. USB cable
5. Claude Pro/Max subscription

**Advantages:**
- ✅ **Full access to Autel data** (ADB bypasses scoped storage)
- ✅ **No manual copying needed**
- ✅ **Better performance** (laptop resources)
- ✅ **Larger screen** for analysis
- ✅ **Keyboard for faster input**

---

### ADB Access Workflow Steps

#### Step 1: Connect Device

```bash
# On laptop
# Connect phone via USB

# Verify connection
adb devices
# Output:
# List of devices attached
# ABC123456789    device
```

#### Step 2: Create Working Directory

```bash
# On laptop
mkdir -p ~/autel-diagnostics/$(date +%Y-%m-%d)
cd ~/autel-diagnostics/$(date +%Y-%m-%d)
```

#### Step 3: Pull Autel Data from Device

**Option A: Pull Specific Recording**
```bash
# List available recordings
adb shell 'ls -lh /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/'

# Output:
# drwxrws--- DATA_202606230148/
# drwxrws--- DATA_202606201530/
# ...

# Pull specific recording
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_202606230148/ ./

# Verify
ls -lh DATA_202606230148/
# Output:
# 20260623014832_842133572.txt
```

**Option B: Pull All Recordings**
```bash
# Pull entire DataLogging directory
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/ ./autel-recordings/

# This creates: ~/autel-diagnostics/2026-06-23/autel-recordings/
```

**Option C: Pull PDF Reports**
```bash
# Pull Autel-generated PDF reports
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/Data/PDF/ ./autel-pdfs/
```

**Option D: Pull Everything**
```bash
# Full backup of all Autel data
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/ ./autel-complete/
```

#### Step 4: Launch Claude Code

```bash
# On laptop, in directory with pulled data
claude
```

#### Step 5: Analyze with Claude

**Example Prompts:**

**Initial Exploration:**
```
List all directories and files in the current directory.
Identify Autel diagnostic data files by extension and structure.
```

**Format Analysis:**
```
Read the file DATA_202606230148/20260623014832_842133572.txt
This is a binary OBD-II diagnostic recording from an Autel MaxiAP200.
Analyze the first 1KB and identify:
- File header structure
- Data encoding (binary/hex/text)
- Possible PID (Parameter ID) locations
- Timestamp encoding
```

**Data Parsing:**
```
Based on OBD-II standard (SAE J1979), parse this binary file.
Common PIDs to look for:
- 0x0C: Engine RPM
- 0x0D: Vehicle Speed
- 0x05: Engine Coolant Temperature
- 0x0F: Intake Air Temperature
- 0x11: Throttle Position

Create a CSV with: timestamp, PID, description, value, unit
```

**Comparison Analysis:**
```
Compare these two recording sessions:
- DATA_202606201530/20260620153045_842133572.txt
- DATA_202606230148/20260623014832_842133572.txt

Identify:
- Changed sensor readings
- New DTCs that appeared
- Performance trends
- Anomalies
```

**PDF Report Analysis:**
```
Read all PDF files in autel-pdfs/ directory.
Extract:
- Vehicle VIN
- DTCs found
- Freeze frame data
- Test results
Create a summary table comparing all sessions.
```

**Historical Analysis:**
```
Analyze all recordings in autel-recordings/ directory.
Create timeline showing:
- When vehicle was diagnosed
- DTCs over time
- Sensor value trends
- Repair history (based on cleared codes)
```

#### Step 6: Create Parser Script

**Prompt:**
```
Create a Python script named autel_parser.py that:

1. Reads Autel .txt binary files
2. Parses OBD-II diagnostic data
3. Outputs structured JSON with:
   - Session metadata (timestamp, duration, vehicle info)
   - All PIDs captured
   - Sensor readings with timestamps
   - Any DTCs found
4. Handles errors gracefully
5. Can process single file or batch process directory

Make it reusable for future diagnostic sessions.
Include usage examples and documentation.
```

**Claude will create:**
```python
# autel_parser.py
# (Claude generates complete working script)
```

**Usage:**
```bash
# Single file
python autel_parser.py DATA_202606230148/20260623014832_842133572.txt

# Batch process
python autel_parser.py --directory autel-recordings/ --output analysis.json

# With verbose output
python autel_parser.py --verbose DATA_202606230148/*.txt
```

#### Step 7: Generate Reports

**Prompt:**
```
Using the parsed data, generate:

1. markdown_report.md:
   - Executive summary
   - Vehicle information
   - DTCs found with descriptions
   - Sensor readings summary
   - Recommendations

2. sensor_timeline.csv:
   - Timestamp, PID, Value, Unit columns
   - All sensor data chronologically

3. comparison_chart.py:
   - Python script using matplotlib
   - Plot RPM vs Speed over time
   - Plot temperature trends
   - Highlight anomalies

4. dtc_history.json:
   - All DTCs ever seen across all sessions
   - First occurrence, last occurrence
   - Cleared/persistent status
```

#### Step 8: Sync Results Back to Device (Optional)

```bash
# Push reports back to phone for viewing
adb push markdown_report.md /sdcard/Download/
adb push sensor_timeline.csv /sdcard/Download/
adb push dtc_history.json /sdcard/Download/

# View on phone with Termux
# (On device in Termux)
cd ~/storage/shared/Download
cat markdown_report.md
```

---

### Advanced Laptop Workflow: Automated Sync

**Create Sync Script:**

```bash
# ~/autel-diagnostics/sync.sh
#!/bin/bash

DATE=$(date +%Y-%m-%d-%H%M%S)
WORK_DIR="$HOME/autel-diagnostics/$DATE"

mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

echo "Pulling latest Autel data..."
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/ ./data/

echo "Launching Claude Code in $WORK_DIR"
claude

# Usage:
# chmod +x ~/autel-diagnostics/sync.sh
# ~/autel-diagnostics/sync.sh
```

**Cron Job for Automatic Backup:**

```bash
# Backup Autel data daily when phone is connected
# Add to crontab: crontab -e

0 22 * * * /home/jonathan/autel-diagnostics/backup.sh >> /home/jonathan/autel-diagnostics/backup.log 2>&1
```

**Backup Script:**
```bash
# ~/autel-diagnostics/backup.sh
#!/bin/bash

if adb devices | grep -q "device$"; then
    DATE=$(date +%Y-%m-%d)
    BACKUP_DIR="$HOME/autel-archive/$DATE"

    mkdir -p "$BACKUP_DIR"

    adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/ "$BACKUP_DIR/"

    echo "[$DATE] Backup complete: $BACKUP_DIR"
else
    echo "[$DATE] Device not connected, skipping backup"
fi
```

---

### Advantages of Laptop ADB Workflow

✅ **Direct Access**
- No manual file copying
- Pull data anytime
- Automated backups possible

✅ **Full Resources**
- Laptop CPU/RAM
- Large datasets no problem
- Complex analysis feasible

✅ **Better Tooling**
- Full keyboard
- Large screen
- Multiple windows
- Screen sharing for collaboration

✅ **Version Control**
- Git integration
- Track analysis over time
- Share scripts with community

✅ **Automation**
- Cron jobs for backups
- Batch processing
- Scheduled analysis

---

### Disadvantages of Laptop ADB Workflow

❌ **Requires Laptop**
- Not portable
- Cannot analyze in vehicle (easily)
- Setup time

❌ **USB Cable Needed**
- Physical connection required
- Cable can fail
- Wireless ADB possible but complex

❌ **Not Offline (for initial pull)**
- Need device connected
- ADB over network possible but requires setup

---

## Workflow Comparison Table

| Feature | Native Termux | Laptop ADB |
|---------|---------------|------------|
| **Access to Autel Data** | ❌ Manual copy only | ✅ Direct via ADB |
| **Setup Complexity** | Low | Medium |
| **Portability** | ✅ Phone only | ❌ Laptop needed |
| **Performance** | ⚠️ Limited | ✅ Full laptop CPU/RAM |
| **Screen Size** | ❌ Small | ✅ Large |
| **Keyboard** | ❌ Touch | ✅ Physical |
| **Automation** | ⚠️ Limited | ✅ Full scripts/cron |
| **Offline** | ✅ Yes | ⚠️ Need device for pull |
| **Cost** | $0 (have phone) | $0 (have laptop) |
| **Learning Curve** | Low | Medium |
| **Use in Vehicle** | ✅ Easy | ❌ Impractical |
| **Large Datasets** | ❌ Slow | ✅ Fast |
| **Version Control** | ⚠️ Possible | ✅ Easy (git) |
| **Sharing Results** | ⚠️ Manual | ✅ Easy (git/email) |
| **Multi-File Compare** | ❌ Hard | ✅ Easy |
| **Script Development** | ❌ Hard | ✅ Easy |

---

## Recommended Workflow by Use Case

### Use Case 1: Quick Check While Driving
**Recommendation**: Native Termux (if data already exported via Loop1 HOP)

**Workflow:**
1. Record diagnostics with Autel app
2. Export PDF via Loop1 HOP to `/sdcard/Download/`
3. Open Termux
4. `cd ~/storage/shared/Download`
5. `claude`
6. Ask Claude to summarize PDF

**Time**: 2-3 minutes

---

### Use Case 2: Detailed Analysis at Home
**Recommendation**: Laptop ADB

**Workflow:**
1. Record diagnostics with Autel app (during drive)
2. Get home, connect phone to laptop
3. `adb pull` latest recordings
4. Launch Claude Code on laptop
5. Deep analysis, create parser, generate reports

**Time**: 30-60 minutes (first time), 10-15 minutes (with parser)

---

### Use Case 3: Mechanic Shop with Multiple Vehicles
**Recommendation**: Laptop ADB with automation

**Workflow:**
1. Record diagnostics for each vehicle
2. End of day: Connect phone, run sync script
3. Automated backup to archive
4. Batch process all recordings
5. Generate reports for each vehicle

**Time**: 5 minutes per vehicle (automated)

---

### Use Case 4: Learning/Reverse Engineering Data Format
**Recommendation**: Laptop ADB

**Workflow:**
1. Pull sample recordings
2. Use Claude Code + hex editor
3. Correlate with known OBD-II PIDs
4. Document format
5. Build parser
6. Share with community

**Time**: Several hours (research project)

---

### Use Case 5: Historical Trend Analysis
**Recommendation**: Laptop ADB

**Workflow:**
1. Pull all historical recordings (months/years)
2. Batch process with parser script
3. Generate timeline charts
4. Identify patterns (e.g., "Coolant temp rising over 6 months")
5. Predictive maintenance

**Time**: 1 hour (with automation)

---

## Hybrid Workflow: Best of Both

**Strategy**: Use both workflows depending on situation

**Example:**

**On-the-Go (Termux)**:
```bash
# In vehicle, just diagnosed problem
# Export PDF via Loop1
# Quick analysis in Termux
cd ~/storage/shared/Download
claude
# Prompt: "Does this PDF show critical issues? Should I stop driving?"
```

**At Home (Laptop)**:
```bash
# Later, detailed analysis
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/
cd ~/autel-diagnostics/$(date +%Y-%m-%d)
claude
# Prompt: "Parse all binary data, create full report, compare with last month"
```

**Result**: Immediate safety check + thorough analysis when convenient

---

## Setting Up Both Workflows

### Setup Native Termux (One-Time)

```bash
# On Android device in Termux

# 1. Install Claude Code
curl -fsSL https://raw.githubusercontent.com/ferrumclaudepilgrim/claude-code-android/main/install.sh | bash

# 2. Grant storage permission
termux-setup-storage
# Accept permission prompt

# 3. Create working directory
mkdir -p ~/autel-analysis
cd ~/autel-analysis

# 4. Test access to shared storage
ls ~/storage/shared/Download

# 5. Authenticate Claude (first time only)
claude
# Follow OAuth flow in browser
```

### Setup Laptop ADB (One-Time)

```bash
# On laptop

# 1. Install ADB (if not already)
# Ubuntu/Debian:
sudo apt install android-tools-adb

# macOS:
brew install android-platform-tools

# 2. Enable USB debugging on phone
# Settings → About Phone → tap Build Number 7 times
# Settings → Developer Options → USB Debugging → Enable

# 3. Connect phone, authorize
adb devices
# Tap "Always allow" on phone prompt

# 4. Test access
adb shell 'ls /sdcard/Android/data/com.autel.maxiap200.autelap/'

# 5. Create working directory structure
mkdir -p ~/autel-diagnostics
mkdir -p ~/autel-archive
mkdir -p ~/autel-scripts

# 6. Install Claude Code on laptop (if not already)
# Follow: https://claude.com/claude-code

# 7. Test workflow
cd ~/autel-diagnostics
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/ ./test-pull/
claude
# Prompt: "List all files in test-pull/ directory"
```

---

## Troubleshooting

### Termux Workflow Issues

**Problem**: `termux-setup-storage` doesn't create ~/storage/

**Solution**:
```bash
# Manually grant permission
# Settings → Apps → Termux → Permissions → Files and Media → Allow

# Restart Termux
exit
# Reopen Termux app

# Try again
termux-setup-storage
ls ~/storage
```

**Problem**: Cannot access `/sdcard/Download` from Termux

**Solution**:
```bash
# Use the symlink
cd ~/storage/shared/Download

# If that fails, check permission
ls -la ~/storage/

# Reinstall Termux (last resort)
```

**Problem**: Claude Code authentication fails

**Solution**:
```bash
# Clear cached credentials
rm -rf ~/.config/claude/

# Try again
claude
# Follow OAuth flow carefully
```

---

### Laptop ADB Workflow Issues

**Problem**: `adb devices` shows no devices

**Solution**:
```bash
# Check USB connection
lsusb  # Should show Android device

# Kill and restart ADB server
adb kill-server
adb start-server
adb devices

# On phone: Revoke USB debugging authorizations, reconnect
# Settings → Developer Options → Revoke USB debugging authorizations
```

**Problem**: `adb pull` fails with "Permission denied"

**Solution**:
```bash
# This is normal for Autel data ONLY if app is not installed
# Install Autel app on device first

# Verify app is installed
adb shell 'pm list packages | grep autel'
# Should show: package:com.autel.maxiap200.autelap

# If app is installed and still fails, check ADB access:
adb shell 'ls -la /sdcard/Android/data/com.autel.maxiap200.autelap/'
# Should list files, not "Permission denied"

# If permission denied, reinstall Autel app
```

**Problem**: ADB extremely slow over USB 2.0

**Solution**:
```bash
# Use USB 3.0 port (blue port on laptop)
# Or use wireless ADB (setup required):

# On phone, in Termux:
adb tcpip 5555

# On laptop:
adb connect <phone-ip>:5555
# Now ADB over WiFi (faster for large pulls)
```

**Problem**: Cannot find where ADB pulled files

**Solution**:
```bash
# ADB pulls to current directory
pwd  # Check where you are

# Always pull to specific location:
cd ~/autel-diagnostics/$(date +%Y-%m-%d)
adb pull /sdcard/Android/data/com.autel.maxiap200.autelap/files/ ./
ls -lh
```

---

## Next Steps

### For Termux Users

1. **Install Claude Code in Termux** - [CLAUDE_CODE_TERMUX_INSTALL.md](CLAUDE_CODE_TERMUX_INSTALL.md)
2. **Set up Loop1 HOP** - [LOOP1_HOP.md](LOOP1_HOP.md) (for PDF export)
3. **Practice workflow** with sample PDFs
4. **Document limitations** (what you CAN'T access)

### For Laptop Users

1. **Test ADB access** to Autel data (following this guide)
2. **Pull sample data** and analyze with Claude
3. **Create parser script** (with Claude's help)
4. **Automate backups** (cron job)
5. **Share parser** with community (git repo)

### For Both

1. **Document data format** - [See AUTEL_LOCKED_ECOSYSTEM.md](AUTEL_LOCKED_ECOSYSTEM.md) protection strategies
2. **Archive everything** - Future-proof against Autel lockdown
3. **Test alternatives** - Try other OBD-II apps with AP200 hardware
4. **Share findings** - Contribute to this repo

---

## Conclusion

**Both workflows are valid. Choose based on your needs:**

**Native Termux**:
- Best for: On-the-go analysis, offline use, phone-only setups
- Limitation: Cannot directly access Autel data (must manually copy)

**Laptop ADB**:
- Best for: Detailed analysis, automation, large datasets
- Limitation: Requires laptop and physical connection

**Hybrid Approach (Recommended)**:
- Use Termux for quick checks in vehicle
- Use laptop for deep analysis at home
- Both workflows complement each other

**The key insight**: ADB access works perfectly for Autel data, making the laptop workflow the most practical for serious analysis.

---

**Last Updated**: 2026-06-23
**Status**: Both workflows documented and tested
**Recommendation**: Start with laptop ADB workflow, add Termux for convenience

