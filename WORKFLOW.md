# Autel + Claude Workflow - Quick Reference

**Purpose**: Step-by-step workflow for analyzing vehicle diagnostic data using Claude Code on Android

---

## Prerequisites Checklist

- [ ] Debuggable Termux APK installed (v0.119.0-beta.3+)
- [ ] Claude Code installed natively via wrapper
- [ ] Storage permission granted (`termux-setup-storage`)
- [ ] Claude Pro or Max subscription (for Claude Code authentication)

**If not complete**: See [CLAUDE_CODE_TERMUX_INSTALL.md](CLAUDE_CODE_TERMUX_INSTALL.md)

---

## Daily Workflow

### 1. Capture Diagnostic Data with Autel

**On Autel MaxiAP200:**
1. Connect OBD-II adapter to vehicle
2. Open Autel MaxiAP200 app
3. Select vehicle make/model
4. Navigate to "Data Stream" or "Live Data"
5. Start recording
6. Drive/test vehicle as needed
7. Stop recording
8. Data saves automatically

**Location**: `/sdcard/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/`

---

### 2. Launch Termux

Open Termux app on Android device.

---

### 3. Navigate to Latest Recording

```bash
# Go to Autel data directory
cd ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/

# List recordings (newest first)
ls -lt

# Enter latest recording directory
cd $(ls -t | grep DATA_ | head -1)

# Verify files exist
ls -lh
```

**Expected output:**
```
-rw-rw---- 1 root sdcard_rw 2.1M Jun 23 14:32 20260623143210_842133572.txt
```

---

### 4. Launch Claude Code

```bash
claude
```

**First time only**: You'll be prompted to authenticate via browser. Follow the OAuth flow and sign in with your Claude Pro/Max account.

---

### 5. Analyze Data with Claude

**Example prompts:**

```
Analyze the OBD-II data in this recording and summarize the vehicle's performance
```

```
Parse the binary .txt file and extract all sensor readings. Create a CSV with timestamps.
```

```
Identify any diagnostic trouble codes (DTCs) present in this data
```

```
Plot engine RPM vs vehicle speed over time
```

```
What was the average fuel trim during this recording?
```

```
Compare this recording to DATA_202605311841021 and show what changed
```

---

### 6. Export Results

Claude Code can create files directly in the Termux filesystem:

```
Create a Python script that parses all Autel recordings and generates a report
```

```
Write the analysis to report.md
```

```
Export all sensor data to autel_analysis.csv
```

---

### 7. Transfer Files Off Device (Optional)

**Via ADB:**
```bash
# From your computer
adb pull /sdcard/Download/report.md ./
```

**Via Termux:**
```bash
# Copy to Downloads folder (accessible via Android file manager)
cp report.md ~/storage/downloads/
```

**Via cloud sync:**
```bash
# If you have Termux packages installed
pkg install rclone
rclone copy report.md gdrive:/autel_reports/
```

---

## Advanced Workflows

### Batch Processing Multiple Recordings

```bash
# Navigate to parent directory
cd ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/

# Launch Claude
claude
```

**Prompt:**
```
Analyze all DATA_* directories in this folder and create a comparison report showing:
- Recording duration
- Vehicles tested
- Any DTCs found
- Performance trends over time
```

---

### Creating Custom Parsers

**Prompt:**
```
Create a Python script that:
1. Reads the binary .txt files from Autel recordings
2. Decodes the OBD-II data format
3. Extracts all PIDs (Parameter IDs)
4. Outputs JSON with timestamped sensor readings
5. Includes error handling for corrupt data

Make it reusable for future recordings.
```

Save as `autel_parser.py` and use for future sessions.

---

### Real-Time Monitoring (Experimental)

**If Autel app is recording in background:**

```bash
# Watch for new data files
watch -n 5 'ls -lht ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_$(date +%Y%m%d)*/​* | head'
```

**Prompt to Claude:**
```
Monitor this directory for new .txt files and parse them as they appear.
Alert me if any sensor value exceeds normal ranges.
```

---

## Troubleshooting

### Claude Code won't start

**Check installation:**
```bash
which claude
claude --version
```

**If missing**: Re-run installation
```bash
curl -fsSL https://raw.githubusercontent.com/ferrumclaudepilgrim/claude-code-android/main/migrate.sh -o migrate.sh
bash migrate.sh
```

---

### Can't access Autel data directory

**Check storage permission:**
```bash
ls ~/storage/shared/
```

**If empty**: Grant permission
```bash
termux-setup-storage
# Accept permission prompt
```

---

### Autel recording files are empty or corrupt

**Verify recording completed:**
- Open Autel app
- Check "Saved Recordings" or "History"
- Ensure recording finished (not interrupted)

**Check file size:**
```bash
du -sh DATA_*/
```

Zero-byte files = recording failed to save.

---

### Claude gives generic responses instead of analyzing data

**Ensure you're in the correct directory:**
```bash
pwd
# Should output: /data/data/com.termux/files/home/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data/DATA_XXXXXX
```

**Be specific in prompts:**
```
Read the file 20260623143210_842133572.txt in this directory and parse its binary format
```

Not:
```
Tell me about OBD-II data
```

---

## Tips for Better Results

### 1. Provide Context

```
This is a 2019 Honda Civic 1.5L Turbo. Analyze the data for turbo boost pressure anomalies.
```

### 2. Ask for Structured Output

```
Create a table showing:
- Timestamp
- Engine RPM
- Vehicle Speed
- Throttle Position
- MAF Sensor
```

### 3. Request Code for Reuse

```
Write a Python class that I can import in future sessions to parse Autel recordings
```

### 4. Compare Multiple Sessions

```
Compare this recording to the one in ../DATA_202605311841021/ and identify differences
```

### 5. Export Everything

```
Save all your analysis, code, and charts to a folder called autel_session_YYYYMMDD/
```

---

## File Organization

Recommended structure for Termux:

```
~/autel/
├── parsers/
│   ├── autel_parser.py
│   ├── pid_decoder.py
│   └── dtc_lookup.py
├── reports/
│   ├── 2026-06-23-analysis.md
│   ├── 2026-06-22-comparison.csv
│   └── charts/
│       ├── rpm_vs_speed.png
│       └── fuel_trim_trend.png
└── recordings/  (symlink to Autel data)
```

**Create symlink:**
```bash
mkdir -p ~/autel
ln -s ~/storage/shared/Android/data/com.autel.maxiap200.autelap/files/MaxiApScan/DataLogging/data ~/autel/recordings
```

---

## Next Steps

1. **Understand Autel Data Format**
   - Work with Claude to reverse-engineer the binary .txt format
   - Document PID mappings
   - Create comprehensive parser

2. **Build Analysis Library**
   - Python modules for common diagnostics
   - DTCs database
   - Sensor range validation

3. **Automate Workflows**
   - Bash scripts for batch processing
   - Scheduled analysis jobs
   - Auto-upload to cloud storage

4. **Share Your Findings**
   - Contribute parsers to this repository
   - Document new Autel data formats discovered
   - Help others analyze their diagnostic data

---

**Last Updated**: 2026-06-23
**Repository**: https://github.com/electronics101clt/Autel-with-Claude
