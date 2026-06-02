# 👻 Ghost-RAT v1.0

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
- [Architecture](#architecture)
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
- **Persistence** → Registry Run key + Scheduled Task
- **Multi-victim** → Unique victim IDs allow targeting individual or all victims

---

## 🏗️ Architecture

```
┌──────────────────────┐     Telegram Bot API      ┌──────────────────────┐
│   VICTIM (Windows)   │ ◄──────────────────────► │   ATTACKER (Linux)   │
│                      │    Private Telegram Group  │                      │
│  Ghost-RAT.exe       │                           │  c2_server.py        │
│  - Polls for cmds    │    ┌──────────────────┐   │  - Manages victims   │
│  - Executes & sends  │───►│  Telegram Group   │◄──│  - Sends commands    │
│    results back      │    │  (Bot Messages)   │   │  - SQLite tracking   │
│  - Random jitter     │    └──────────────────┘   │                      │
└──────────────────────┘                           └──────────────────────┘
```

**Flow:**
1. Implant starts → generates unique victim ID → installs persistence
2. Implant sends check-in message to Telegram group
3. C2 server detects check-in → registers victim in SQLite database
4. Operator sends command in group: `GHOST-XXXXXXXX /sysinfo`
5. Implant polls, finds command, executes, sends result back to group
6. C2 server logs the response

---

## ✨ Features

### Core
| Feature | Description |
|---------|-------------|
| Unique Victim ID | SHA-256 hash of hostname + MAC + username → `GHOST-XXXXXXXX` |
| Persistence | Registry Run Key + Scheduled Task (survives reboots) |
| Jitter | 30-60 second randomized check-in intervals |
| Multi-victim | Target specific victims or broadcast to all |

### Commands
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
| `/browsers` | Chrome saved passwords |
| `/proclist` | List running processes |
| `/prockill <pid>` | Kill a process |
| `/shutdown` | Shutdown victim machine |
| `/restart` | Restart victim machine |
| `/selfdestruct` | Remove persistence + delete implant |

### C2 Server Commands
| Command | Description |
|---------|-------------|
| `/victims` | List all registered victims |
| `/help` | Show command reference |

---

## 📁 File Structure

```
ghost-rat/
├── implant.py              # Main Windows implant payload
├── c2_server.py            # Linux C2 server (Telegram bot)
├── config.py               # Configuration (bot token, group ID, settings)
├── requirements.txt        # Python dependencies
├── README.md               # This file
├── utils/
│   ├── __init__.py
│   ├── victim_id.py        # Unique victim ID generation & caching
│   └── persistence.py      # Registry + Scheduled Task persistence
├── commands/
│   ├── __init__.py
│   └── handler.py          # Command dispatcher (19 commands)
└── dist/
    └── Ghost-RAT.exe       # Compiled implant (after build)
```

---

## 🛠️ Setup Guide

### Prerequisites
- **Python 3.10+** on both machines
- A **Telegram account**
- **Windows VM** (victim — isolated, no internet access to real targets)
- **Linux machine** (attacker — or WSL)

### Step 1: Create a Telegram Bot

1. Open Telegram and message **@BotFather**
2. Send `/newbot` and follow the prompts
3. Copy the **Bot Token** (looks like `1234567890:ABCdefGHIjklMNOpqrSTUvwxYZ`)

### Step 2: Create a Private Telegram Group

1. Create a new group in Telegram
2. Add your bot to the group
3. Send any message in the group
4. Visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in a browser
5. Find the `"chat": {"id": -100XXXXXXXXXX}` — this is your **Group Chat ID**

### Step 3: Configure Ghost-RAT

Edit `config.py`:

```python
BOT_TOKEN = "1234567890:ABCdefGHIjklMNOpqrSTUvwxYZ"
GROUP_CHAT_ID = "-100XXXXXXXXXX"
```

### Step 4: Install Dependencies

```bash
# On BOTH machines (or just the build machine):
pip install -r requirements.txt
```

---

## 🔨 Building the Implant

### Compile to a single Windows .exe:

```bash
pyinstaller --onefile --noconsole --name Ghost-RAT --icon=NONE implant.py
```

**Flags explained:**
- `--onefile` → Single standalone .exe (no extra files needed)
- `--noconsole` → No command prompt window appears when running
- `--name Ghost-RAT` → Output filename
- `--icon=NONE` → No custom icon (or specify your own `.ico`)

The compiled `.exe` will be in the `dist/` folder:
```
dist/Ghost-RAT.exe
```

### Optional: Add a custom icon
```bash
pyinstaller --onefile --noconsole --name Ghost-RAT --icon=myicon.ico implant.py
```

### Optional: UPX compression (smaller .exe)
```bash
# Install UPX first, then:
pyinstaller --onefile --noconsole --name Ghost-RAT --upx-dir=/path/to/upx implant.py
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
    ║       Educational Red Team Tool v1.0      ║
    ╚═══════════════════════════════════════════╝

2024-01-01 12:00:00 [INFO] Database initialized: victims.db
2024-01-01 12:00:00 [INFO] ✅ C2 server is ONLINE. Waiting for victims...
```

### 2. Deploy the Implant (Victim — Windows VM)

Copy `dist/Ghost-RAT.exe` to the Windows VM and run it. The implant will:
1. Generate a unique victim ID
2. Install persistence
3. Send a check-in message to the Telegram group

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

---

## 🔬 Detection & Analysis (Blue Team)

This section helps SOC analysts and blue teamers understand what to look for:

### Network Indicators
- **Telegram API traffic**: HTTPS connections to `api.telegram.org`
- **Periodic beaconing**: Regular HTTP(S) requests every 30-60 seconds
- **File uploads**: Large POST requests to Telegram's file upload endpoint

### Host Indicators
- **Registry**: Check `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` for `WindowsSecurityHealth`
- **Scheduled Tasks**: Look for task named `WindowsSecurityHealth` in Task Scheduler
- **Processes**: Look for unsigned executables with network connections to `api.telegram.org`
- **File System**: Hidden `.ghost_id` file in `%APPDATA%`

### YARA Rule (Example)
```yara
rule Ghost_RAT {
    meta:
        description = "Detects Ghost-RAT implant"
        author = "SOC Team"
    strings:
        $s1 = "GHOST-RAT CHECK-IN" ascii
        $s2 = "ghost_id" ascii
        $s3 = "WindowsSecurityHealth" ascii
        $s4 = "api.telegram.org" ascii
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

---

## 🔧 Troubleshooting

### "BOT_TOKEN not configured!"
→ Edit `config.py` and set your Telegram bot token.

### "GROUP_CHAT_ID not configured!"
→ Follow Step 2 in Setup to find your group's chat ID.

### Implant doesn't check in
1. Verify the bot is in the group
2. Check internet connectivity on the Windows VM
3. Run `implant.py` directly (not as `.exe`) to see error messages
4. Ensure the bot has permission to read/send messages in the group

### PyInstaller build fails
```bash
# Try upgrading PyInstaller:
pip install --upgrade pyinstaller

# If missing modules, use hidden imports:
pyinstaller --onefile --noconsole --hidden-import=pynput.keyboard._win32 --hidden-import=pynput.mouse._win32 --name Ghost-RAT implant.py
```

### Commands not executing
- Make sure you're using the correct victim ID format: `GHOST-XXXXXXXX`
- The implant only processes messages from the configured group
- Check that the command format matches exactly (e.g., `/shell whoami`, not `shell whoami`)

---

## 📜 License

This project is provided for **educational purposes only**. No warranty is provided. Use responsibly and legally.

---

*Built for cybersecurity education and defensive research.*
