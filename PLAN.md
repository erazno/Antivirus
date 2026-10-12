# Personal Antivirus — Project Plan

> Living document. Update it as decisions change.

## 1. Goals

1. **Learn** how malware detection works, one layer at a time.
2. **Protect** a Windows 11 PC day to day, as an *extra* layer alongside Microsoft Defender.

## 2. Honest limits (what this will *not* do)

- **Not a Defender replacement.** Keep Defender turned on.
- **Reacts, doesn't prevent.** Without a kernel driver (C/C++, signed by Microsoft) we only see a process *after* it starts. We can kill it fast, but we can't stop it from starting.
- **Can be attacked.** Malware running as admin can kill or hide from a normal program. We add basic self-protection, but can't fully prevent this.
- **No real malware on the main PC. Ever.** Live samples are only handled in the isolated lab VM (see §10).

## 3. Key decisions

| Topic | Decision | Why |
|---|---|---|
| Platform | Windows 11 (64-bit) | Main machine |
| Language | Python 3.12+ (64-bit) | Already known; strong libraries. Rewrite specific hot spots in Rust later only if needed |
| GUI | PySide6 (Qt) | Real windows, system-tray icon, built-in notifications |
| Structure | Background service + separate GUI | Protection keeps running when the window is closed |
| Detection | Layers + scoring | No single check is reliable; combined evidence is |
| Response | Quarantine, kill process, block connection, alert | Project requirement |
| Default mode | Learning (alert-only) → Protect (automatic actions) | Avoid damage from false positives |
| Storage | SQLite (built into Python) | One file, no server, easy to query |

## 4. Architecture

Analogy: the **service** is a security guard who never sleeps; the **GUI** is the control room. The control room gives orders, the guard does the physical work.

```
+-----------------------------+                                +----------------------------------+
| GUI app (runs as you)       |   commands / status / alerts   | Service (runs as SYSTEM)         |
|  - Scan button + results    | <----------------------------> |  - Scanners: hash, YARA, static  |
|  - Protection on/off        |     local named pipe           |  - Monitors: files, processes,   |
|  - Alerts + tray icon       |     (locked down)              |    network, persistence, canary  |
|  - Quarantine manager       |                                |  - Responder: quarantine, kill,  |
|  - Settings / allowlist     |                                |    firewall block                |
+-----------------------------+                                |  - Threat-feed updater           |
                                                               +----------------+-----------------+
                                                                                |
                                                          SQLite DB + logs + quarantine folder
```

**Securing the GUI-to-service channel (important):** if any program can talk to the service, malware could simply send "turn protection off". So:
- Only the GUI (and admins) may connect to the pipe.
- Sensitive commands (turn protection off, restore from quarantine, add to allowlist) require Windows admin confirmation (UAC).

## 5. Planned folder layout

```
antivirus/
  core/         config, logging, database, allowlist
  detection/    hash scan, YARA scan, PE static analysis, scoring
  monitors/     files, processes, network, persistence, canary
  response/     quarantine, process killer, firewall
  intel/        threat-feed downloader/updater
  service/      Windows service entry point + IPC server
  gui/          PySide6 windows, tray icon, notifications
rules/          YARA rules
tests/          unit tests + harmless test samples (EICAR)
docs/           notes from each phase (what you learned)
```

## 6. Detection layers

| # | Layer | What it checks | Catches | Misses |
|---|---|---|---|---|
| 1 | Hash | SHA-256 of file vs known-bad list | Exact known malware, instantly | Any modified copy (1 byte = new hash) |
| 2 | YARA rules | Byte/text patterns | Malware families and variants | Brand-new or heavily obfuscated code |
| 3 | Static PE analysis | Inside .exe/.dll *without running it*: entropy, imports, signature | Packed or suspicious unknown files | Clean-looking files that act badly later |
| 4 | Process behavior | Parent/child, path, command line, baseline | Script droppers, abused Windows tools | Very short-lived processes (if polling) |
| 5 | Network | Listening ports, remote IPs, owning process | Backdoors, contact with known C2 servers | Traffic to new servers over normal ports |
| 6 | Persistence | Run keys, Startup folders, tasks, services | Malware setting itself up to survive reboot | Advanced tricks (e.g., WMI subscriptions) at first |
| 7 | Canary files | Decoy files being modified | Ransomware | — |

### Examples of suspicious signals

**Static (files):**
- High entropy (contents look random) → likely packed or encrypted
- Imports `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread` → injecting code into other processes
- Unsigned, or invalid digital signature
- Double extension (`invoice.pdf.exe`), or extension doesn't match the real content

**Processes:**
- Office apps (`winword.exe`, `excel.exe`) launching `powershell.exe`, `cmd.exe`, `wscript.exe`, `mshta.exe`
- Running from `%TEMP%`, `%APPDATA%` or `Downloads`
- System-process name in the wrong place (`svchost.exe` outside `C:\Windows\System32`)
- PowerShell with `-EncodedCommand` / `-enc`, or a hidden window
- Not in the learned baseline

**Network:**
- New listening port not in the baseline
- Connection to a blocklisted IP or domain
- Programs that shouldn't network (e.g., `notepad.exe`) making connections
- Many connections to many different IPs quickly (scanning / worm behavior)

**Persistence:**
- New entries under `HKCU` / `HKLM` `\Software\Microsoft\Windows\CurrentVersion\Run` (and `RunOnce`)
- New files in Startup folders
- New scheduled tasks or services

## 7. Scoring

Like a referee: one small foul is a warning, several add up to a red card. Each signal adds points and the total decides the action. **Starting values: tune them with real data.**

| Signal | Points |
|---|---|
| Hash matches known malware | 100 |
| Canary file modified | 100 |
| Connection to blocklisted IP | 90 |
| High-severity YARA match | 80 |
| System-process name in wrong folder | 70 |
| Office app spawns shell / script host | 60 |
| Encoded PowerShell command | 40 |
| Injection-style imports | 30 |
| New Run key / startup item / task | 30 |
| New listening port (not in baseline) | 25 |
| Packed / high entropy | 20 |
| Unsigned exe in Temp / Downloads | 15 |
| Validly signed by trusted publisher | −30 |

| Total score | Action |
|---|---|
| 0–39 | Log only |
| 40–79 | Alert; the user decides |
| 80+ | Alert + automatic response (Protect mode only) |

## 8. Response actions

- **Quarantine:** move the file into a locked folder (SYSTEM/Admins only), scramble its contents so it can't run by accident, and record original path, hash, time and reason in SQLite. Restorable from the GUI. **Never auto-delete.**
- **Kill process:** terminate the process and its children. **Never** kill critical Windows processes (`System`, `smss.exe`, `csrss.exe`, `wininit.exe`, `winlogon.exe`, `services.exe`, `lsass.exe`). Killing these crashes Windows (blue screen).
- **Block connection:** add a Windows Firewall rule blocking the remote IP and/or the program. All our rules share a name prefix (e.g., `PAV-Block-`) so they can be listed and removed.
- **Alert:** tray notification + entry in the GUI event log showing *what* happened, *why* (score breakdown), *what was done*, and an **Undo** button where possible.

## 9. Safety rails

- **Allowlist:** Microsoft-signed files in `C:\Windows`, the critical-process list, and anything the user approves.
- **Signature checks:** verify Authenticode digital signatures before trusting a publisher.
- **Two modes:**
  - *Learning* (first ~7 days, or until switched): alerts only; builds the baseline of normal processes, ports and startup items.
  - *Protect:* automatic actions at score ≥ 80.
- **Self-protection basics:** service restarts automatically if killed; GUI shows a red warning if the service is down; program, config and quarantine folders writable by admins only.
- **Privacy:** VirusTotal lookups send **hashes only**, never files. Uploaded files can be downloaded by VirusTotal's paying customers, so never upload personal documents.

## 10. Test lab

**Current:** Ubuntu Server on the AI Lab VLAN.

**Plan:**
1. Install **KVM/QEMU + libvirt** on the Ubuntu server and create a **Windows 11 VM**.
   - Windows 11 requires TPM 2.0 and Secure Boot → use **swtpm** (virtual TPM) and **OVMF** (UEFI firmware).
   - ISO: **Windows 11 Enterprise 90-day evaluation** from the Microsoft Evaluation Center (free, can be reinstalled).
   - Resources: minimum 4 GB RAM (8 GB better), 2+ vCPUs, 64 GB+ disk.
   - First confirm the CPU supports hardware virtualization (Intel VT-x / AMD-V).
2. Take a **clean snapshot** after setup; revert to it after every test.
3. **Network isolation:**
   - A VLAN only isolates if firewall rules stop it from reaching the home LAN. Verify that VLAN → LAN traffic is blocked.
   - For live samples, put the VM on a libvirt *isolated* network with **no internet**. Optionally run **INetSim** on the Ubuntu host to fake internet services so malware "thinks" it is online.
   - Disable shared folders and shared clipboard between VM and host while handling samples.
4. **Early phases (1–4)** only use the harmless **EICAR test file**, so they're safe on the main PC or in **Windows Sandbox** (Windows 11 Pro/Enterprise only).
5. **Prefer simulated behavior over real malware:** harmless scripts that add a Run key, open a listening port or rewrite canary files. **Atomic Red Team** (by Red Canary) is a well-known library of such tests.
6. **If real samples are ever used:** get them from research sources (e.g., MalwareBazaar), keep them in password-protected zips (password `infected` is the convention), and only unzip inside the VM.

## 11. Roadmap

Each phase lists **Build** / **Learn** / **Done when**.

### Phase 0: Setup
- **Build:** Python virtual environment on Windows, repo skeleton, test runner, Windows 11 test VM + clean snapshot.
- **Learn:** virtual environments, VMs, snapshots.
- **Done when:** the VM reverts to a clean snapshot and the (empty) test suite runs on Windows.

### Phase 1: Hash scanner + quarantine + simple GUI
- **Build:** walk folders, SHA-256 each file, compare with a hash list in SQLite, quarantine + restore, PySide6 window with a "Scan folder" button and results list.
- **Learn:** hashing, walking the file system, why hashes alone are weak.
- **Done when:** scanning a folder containing EICAR flags and quarantines it, and Restore puts it back.

### Phase 2: YARA + static PE analysis + scoring
- **Build:** load YARA rules; PE checks (entropy, imports, signature, double extensions); scoring engine with the score breakdown shown in the GUI.
- **Learn:** PE file format, entropy, what imports reveal, false positives.
- **Done when:** test files get the expected scores and normal Windows programs score low.

### Phase 3: Service + IPC + tray + notifications
- **Build:** Windows service, locked-down named pipe, GUI talks to the service, tray icon, protection on/off toggle, notifications.
- **Learn:** Windows services, privilege levels, inter-process communication, UAC.
- **Done when:** closing the GUI keeps protection running, the toggle works, and a non-admin program cannot turn protection off.

### Phase 4: Real-time file monitoring
- **Build:** watch Downloads, Desktop, Temp and Startup folders; scan new/changed executables and scripts once they're fully written.
- **Learn:** file-system events, race conditions.
- **Done when:** dropping EICAR into Downloads triggers quarantine within seconds.

### Phase 5: Process monitoring + baseline + kill
- **Build:** process-start *events* (WMI, or Sysmon via the Windows Event Log; preferred over polling), parent/child, path and command-line rules, baseline in Learning mode, kill in Protect mode.
- **Learn:** process trees, abused Windows tools (LOLBins), polling vs events.
- **Done when:** a harmless test (a script launching PowerShell with an encoded command) is detected and killed.

### Phase 6: Network monitoring + firewall blocking
- **Build:** map connections/ports to processes, baseline listening ports, IP blocklist feeds, add/remove firewall rules.
- **Learn:** TCP/UDP, ports, command-and-control (C2), Windows Firewall.
- **Done when:** a connection to a lab IP placed on the blocklist (e.g., the Ubuntu server) is blocked and alerted, and Undo removes the rule.

### Phase 7: Persistence + canary files
- **Build:** snapshot and compare Run keys, Startup folders, scheduled tasks and services; place decoy files in Documents/Desktop named to sort first (e.g., `!000_canary.docx`, because ransomware often walks folders alphabetically); kill the likely writer.
- **Learn:** persistence techniques, ransomware behavior.
- **Done when:** a harmless script that adds a Run key is flagged, and a script that rewrites canaries is killed.
- **Known challenge:** a normal program can't easily see *which* process changed a file. To research: ETW file events. Fallback: the process with the highest recent disk writes.

### Phase 8: Polish + packaging
- **Build:** daily threat-feed updates, settings page, history/reports, installer (PyInstaller or Nuitka), optional code signing (reduces false alarms from other antivirus products).
- **Done when:** it installs on a clean VM, starts at boot and survives reboots.

### Stretch ideas
- ETW (Event Tracing for Windows) for richer real-time events
- Beaconing detection (connections at regular intervals = malware checking in with its C2)
- AMSI provider to scan PowerShell scripts after they're decoded (needs a C/C++/Rust DLL)
- Rewrite the scanner core in Rust
- Machine-learning classifier for PE files (e.g., trained on the EMBER dataset)

## 12. Planned dependencies

| Package | Purpose |
|---|---|
| `psutil` | Processes, network connections, kill, disk I/O stats |
| `yara-python` | Run YARA rules |
| `pefile` | Read .exe/.dll internals |
| `watchdog` | File-change notifications |
| `pywin32` | Windows service, Event Log, registry, firewall (COM), file permissions |
| `wmi` | Process-start events |
| `PySide6` | GUI, tray icon, notifications |
| `requests` | Download threat feeds |
| `pytest` | Tests |

Built into Python: `hashlib`, `sqlite3`, `logging`, `winreg`, `multiprocessing.connection` (named pipes).

## 13. Threat-intel sources

- **MalwareBazaar** (abuse.ch): known malware hashes
- **ThreatFox / URLhaus** (abuse.ch): bad IPs, domains, URLs
- **Feodo Tracker** (abuse.ch): botnet C2 IP blocklist
- **VirusTotal API** (optional): hash lookups; the free tier is rate-limited
- **Community YARA rules**: e.g., the Yara-Rules project, Neo23x0 signature-base (check each license)

abuse.ch services may require a free account / API key.

## 14. Open questions

- [ ] Main PC: Windows 11 Home or Pro? (Windows Sandbox and Hyper-V need Pro.)
- [ ] Ubuntu server specs (RAM, CPU, free disk): enough for a Windows 11 VM?
- [ ] Is the AI Lab VLAN blocked from reaching the home LAN by firewall rules?
- [ ] Length of Learning mode (default 7 days?)
- [ ] Extra folders to watch in real time beyond the defaults?

## 15. Glossary

- **Hash:** fixed-length fingerprint of a file's contents (we use SHA-256).
- **Signature (antivirus):** a pattern known to identify malware.
- **Digital signature:** a publisher's cryptographic stamp proving who made a file and that it hasn't changed.
- **YARA:** rule language for describing malware patterns.
- **PE (Portable Executable):** Windows' file format for .exe and .dll files.
- **Entropy:** how random data looks; packed/encrypted data is close to the maximum.
- **Packer:** a tool that compresses/encrypts an exe to hide what's inside.
- **IOC (Indicator of Compromise):** a hash, IP, domain or file path linked to an attack.
- **C2 (Command and Control):** the server malware "phones home" to for instructions.
- **LOLBins ("living off the land" binaries):** legitimate Windows tools (PowerShell, mshta, certutil) abused by attackers.
- **Baseline:** a record of what's normal on this machine.
- **False positive / false negative:** a clean thing flagged as bad / a bad thing missed.
- **Quarantine:** isolate a file so it can't run, without deleting it.
- **Persistence:** how malware survives a reboot.
- **ETW (Event Tracing for Windows):** Windows' built-in high-speed event stream.
- **Sysmon:** free Microsoft Sysinternals tool that logs detailed process/network events to the Windows Event Log.
- **AMSI (Antimalware Scan Interface):** lets antivirus scan scripts in memory, after they're decoded.
