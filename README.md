# Hack The Box — Write-ups

Personal collection of write-ups for retired [Hack The Box](https://www.hackthebox.com/) machines. Each entry walks through enumeration, exploitation, and privilege escalation, and includes the supporting files (scans, exploits, binaries) used along the way.

> ⚠️  These notes only cover **retired** machines, in line with HTB's policy on publishing solutions.

## 📊 Stats

- **Total machines:** 30
- **By difficulty:** 🟢 Easy: 18 · 🟡 Medium: 10 · 🔴 Hard: 1 · ⚫ Insane: 1
- **By OS:** 🐧 Linux: 16 · 🪟 Windows: 14

## 📚 Index

### 🟢 Easy

| Machine | OS | Topics |
|---|---|---|
| [Arctic](Arctic/README.md) | 🪟 Windows | Adobe ColdFusion 8, Remote Command Execution (RCE), JuicyPotato, SeImpersonatePrivilege |
| [Bastion](Bastion/README.md) | 🪟 Windows | Mounting SMB Share, Mounting VHD File, Remote Share, guestmount, Dump SYSTEM SAM, Dump,... |
| [Beep](Beep/README.md) | 🐧 Linux | Apache, PHP, Local File Inclusion, SMTP, Python |
| [Bounty](Bounty/README.md) | 🪟 Windows | Microsoft IIS 7.5, File Extension Enumeration, File Upload Exploitation, JuicyPotato, S... |
| [Forest](Forest/README.md) | 🪟 Windows | Active Directory, RPC Enumeration, ASREProast Attack, DCSync Attack |
| [Grandpa](Grandpa/README.md) | 🪟 Windows | WebDAV Exploit, Microsoft IIS 6.0, WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow... |
| [Granny](Granny/README.md) | 🪟 Windows | WebDAV Exploit, Microsoft IIS 6.0, WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow... |
| [Horizontall](Horizontall/README.md) | 🐧 Linux | Web, Virtual Hosting, nginx 1.14.0, Javascript, Subdomain Enum, Strapi, strapiVersion 3... |
| [Irked](Irked/README.md) | 🐧 Linux | UnrealIRCd (CVE-2010-2075), Binary Exploitation, ltrace |
| [Jerry](Jerry/README.md) | 🪟 Windows | Apache Tomcat, Default Credentials, Brute Force, Hydra HTTP Get Method, Malicious War F... |
| [Love](Love/README.md) | 🪟 Windows | SSRF, Server Side Request Forgery, File Scanner Free, Voting System, AlwaysInstallEleva... |
| [Nodeblog](Nodeblog/README.md) | 🐧 Linux | NoSQL Injection, XXE, XML External Entity (XML Injection), Sudo Privileges, Clear-Text ... |
| [Photobomb](Photobomb/README.md) | 🐧 Linux | Linux Path Environment Variable Hijacking, Command OS Injection |
| [Precious](Precious/README.md) | 🐧 Linux | pdfkit v0.8.6, YAML Malicious File, YML Privilege Escalation, YAML Vulnerability |
| [SAU](SAU/README.md) | 🐧 Linux | Server Side Request Forgery, SSRF, Command Injection, request-baskets v1.2.1, Maltrail ... |
| [Sauna](Sauna/README.md) | 🪟 Windows | Active Directory, User Enumeration, Kerbrute, Kerberos User Validation, ASREProast, Pas... |
| [Sense](Sense/README.md) | 🐧 Linux | Pfsense 2.1.3, Web File Extension Enumeration, FreeBSD 8.3 |
| [Valentine](Valentine/README.md) | 🐧 Linux | OpenSSH 5.8 (CVE-2018-15473), SSH User Enumeration (CVE-2018-15473), Heartbleed Attack,... |

### 🟡 Medium

| Machine | OS | Topics |
|---|---|---|
| [Bolt](Bolt/README.md) | 🐧 Linux | AdminLTE, Roundcube Webmail 1.4.6, Password Cracking, SSTI, Server Side Template Inject... |
| [Chatterbox](Chatterbox/README.md) | 🪟 Windows | Buffer Over Flow, BOF, AChat Chat, Password Reuse, Credential Dump from winlogon regist... |
| [Nineveh](Nineveh/README.md) | 🐧 Linux | Apache 2.4.18, phpLiteAdmin 1.9, LFI, Chkrootkit PrivEsc, Brute Force Web Login |
| [Node](Node/README.md) | 🐧 Linux | Javascript, Apache Hadoop, Stored credentials MongoDB Enumeration Mongo Task Injection,... |
| [Poison](Poison/README.md) | 🐧 Linux | LFI, Password reuse, VNCviewer, VNC Pass file, VNC Authentication over passwd file |
| [RedCross](RedCross/README.md) | 🐧 Linux | XSS, Haraka 2.8.8, Cookie Hijacking, Weak Permissions, PostgreSQL, GID Privilege Escala... |
| [Secnotes](Secnotes/README.md) | 🪟 Windows | CSRF, IIS File Upload to abuse PHP Shell, WSL Discovery, Plain-Text Creds, wmiexec |
| [Silo](Silo/README.md) | 🪟 Windows | Oracle TNS listener 11.2.0.2.0, Oracle Database Attacking Tool, odat.py |
| [SolidState](SolidState/README.md) | 🐧 Linux | Apache James 2.3.2, JAMES pop3d 2.3.2, JAMES smtpd 2.3.2, RCE, Default Credentials, Cro... |
| [TartarSauce](TartarSauce/README.md) | 🐧 Linux | WordPress Plugin Enumeration Manual, Gwolle Guestbook 1.5.3, Backup Custom Script to re... |

### 🔴 Hard

| Machine | OS | Topics |
|---|---|---|
| [Conceal](Conceal/README.md) | 🪟 Windows | UDP Scan SNMP, ISAKMP, VPN IPSec (Using Strongswan (IPSEC/VPN) [ipsec.secret/ipsec.conf... |

### ⚫ Insane

| Machine | OS | Topics |
|---|---|---|
| [BankRobber](BankRobber/README.md) | 🪟 Windows | XSS Cookie Hijacking, Cross-Site Request Forgery (CSRF), Malicious Javascript Cookie Hi... |

## 🗂️ Repository Structure

Each machine has its own top-level folder, with three sub-folders for the different kinds of artifacts produced during the box:

```
htb-writeups/
├── README.md           ← this index
├── <MachineName>/
│   ├── README.md       ← the write-up
│   ├── content/        ← screenshots and inline references
│   ├── nmap/           ← raw nmap output
│   └── exploit/        ← exploit scripts, binaries, payloads
└── ...
```

**`content/`** holds anything embedded in the write-up itself — screenshots, dashboards, request/response captures.

**`nmap/`** is for raw scanner output (`AllPorts`, `FullScan`, `HTTPNmapScan`, etc.) so others can compare or replay your enumeration.

**`exploit/`** is where the exploit scripts, compiled binaries, payloads, and any third-party tools used during the box live.

## ⚠️ Disclaimer

These notes are shared for **educational purposes only** as a personal study reference. Do not use any of the techniques, commands, or exploits documented here against systems you do not own or have explicit written permission to test. Please respect [HTB's rules](https://help.hackthebox.com/en/articles/5188824-hack-the-box-rules) and do not share solutions for active machines.

## 📬 Found an issue?

If a command is broken, an exploit no longer works, or you spot a typo — feel free to open an issue or send a pull request.
