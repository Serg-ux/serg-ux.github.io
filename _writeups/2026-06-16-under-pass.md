---
layout: writeup
title: "UnderPass"
date: 2026-06-16
description: "Enumerating a daloradius service over SNMP and abusing a privileged command's attached debugger for root."
tags: [Box, easy, linux]
---

## Reconnaissance

The assessment began with a full TCP and UDP port scan to identify the exposed services.

```bash
nmap -sVC -p- --open -sS --min-rate 5000 -v -n --stats-every=5s -Pn -oN underpass_scan 10.10.11.48
```

The TCP scan shows SSH on port 22 and an Apache HTTP service on port 80. The UDP scan adds an SNMP service on port 161.

![SNMP on UDP 161](/assets/images/under-pass/181-1.png)
*UDP scan revealing SNMP on port 161*

## Enumeration

Because SNMP is reachable with the public community string, it is enumerated with snmpwalk.

```bash
snmpwalk -v2c -c public 10.10.11.48
```

The SNMP data indicates the host runs a daloradius service, so web enumeration is focused there.

```bash
gobuster --wordlist=/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt dir -u http://underpass.htb/daloradius/
```

Enumeration exposes the daloradius web interface, including the operator login panel.

[http://underpass.htb/daloradius/app/operators/login.php](http://underpass.htb/daloradius/app/operators/login.php)

![Enumeration output](/assets/images/under-pass/181-2.png)
*daloradius paths discovered during enumeration*

The application accepts default credentials.

![Default credentials accepted](/assets/images/under-pass/181-3.png)
*Logging in with default operator credentials*

After authenticating, an MD5 hash is found within the application.

![MD5 hash in the application](/assets/images/under-pass/181-4.png)
*An MD5 hash stored in daloradius*

![Hash detail](/assets/images/under-pass/181-5.png)
*The stored credential record*

![Cracked credential](/assets/images/under-pass/181-6.png)
*The cracked plaintext for the hash*

## Exploitation

The recovered credentials give access as a standard user inside daloradius.

![Standard-user access](/assets/images/under-pass/181-7.png)
*Authenticated as a standard daloradius user*

Exploring the functionality reveals a command that can be run with administrative privileges.

![Administrative command found](/assets/images/under-pass/181-8.png)
*A command available with elevated privileges*

![Command detail](/assets/images/under-pass/181-9.png)
*The privileged command in the interface*

The application can launch a binary with debugging enabled, and the program is started with the debugger attached to allow privileged interaction.

![Program started under the debugger](/assets/images/under-pass/181-10.png)
*Launching the binary with debugging enabled*

![Debugger session](/assets/images/under-pass/181-11.png)
*Interacting with the program through the debugger*

## Privilege Escalation

Abusing the debugger attached to the privileged program yields command execution as root, resulting in a fully privileged shell.

![Root shell obtained](/assets/images/under-pass/181-12.png)
*Root shell through the debugger*
