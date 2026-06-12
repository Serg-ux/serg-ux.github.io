---
layout: writeup
title: "HTB Wifinetic"
date: 2026-02-10
description: "Easy Linux box involving anonymous FTP access to an OpenWrt backup, plaintext wireless credentials, and a WPS PIN attack via Reaver for root access."
tags: [Box, easy, linux]
---

## Network Scan

A full port scan is run against the target to identify all open services.

```bash
nmap 10.10.11.247 -Pn -sV -sC -p- --open -oN scan.txt
```

|Flag|Purpose|
|---|---|
|`-Pn`|Disables host discovery — treats the host as always online|
|`-sV`|Detects service versions|
|`-sC`|Runs default NSE scripts to gather additional service information|
|`-p-`|Scans all 65535 ports|
|`--open`|Only displays open ports|
|`-oN scan.txt`|Saves output to a file|

The scan reveals two services running: **FTP** and **SSH**. Notably, the default `-sC` scripts include `ftp-anon`, which automatically checks whether anonymous FTP login is enabled and lists accessible files — confirming that the FTP service allows unauthenticated access.

## FTP Enumeration & Credential Discovery

### Connecting as Anonymous

The FTP service is accessed without credentials using the anonymous user.

```bash
ftp 10.10.11.247
```

Before downloading files, the local destination directory is set.

```bash
ftp> lcd /home/sergux/Desktop/htb/boxes/wifinetic/
```

### Reviewing the Files

After downloading and reviewing the available files, a few items stand out: a couple of email addresses, a phone number, and a backup archive — `backup-OpenWrt-2023-07-26.tar`.

### Extracting the Backup

The archive is decompressed to inspect its contents.

```bash
tar -xf backup-OpenWrt-2023-07-26.tar
```

The extracted files include a `passwd` file containing system user information, and a `config` folder. Inside the config folder, a file named `wireless` contains a **plaintext password**.

> OpenWrt backup archives store the router's full configuration, including wireless settings. Passwords in these files are often stored in plaintext, making exposed backups a critical information disclosure risk.

## SSH Access — User Flag

### Identifying the User

Cross-referencing the `passwd` file with the password found in the `wireless` config file, the `netadmin` user is identified as a likely match.

### Logging In

```bash
ssh netadmin@10.10.11.247
```

The connection succeeds. The **user flag** is found in the home directory.

## Privilege Escalation — WPS Attack via Reaver

### Initial Enumeration

`sudo -l` returns no useful misconfigurations. Attention shifts to the network configuration.

```bash
ifconfig
```

Several wireless interfaces are present on the machine. One of them — `mon0` — is in **monitor mode**, meaning it is already configured to passively capture wireless traffic without being associated to any network.

### Identifying wpa_supplicant

Listing active services reveals **wpa_supplicant** running on the system.

```bash
systemctl status
systemctl status wpa_supplicant.service
```

`wpa_supplicant` is the Linux daemon responsible for managing WPA/WPA2 wireless authentication. Its configuration file (`wpa_supplicant.conf`) typically contains network credentials, but direct access to it is denied due to file permissions.

### Gathering Wireless Interface Details

```bash
iwconfig
```

The `wlan1` interface is identified as an access point. The output also provides its **BSSID** (`02:00:00:00:00:00`) — the MAC address that uniquely identifies the wireless access point and is required to target it directly.

### Finding Reaver

```bash
apt list --installed
```

**Reaver** is found installed on the system. Reaver is a tool designed to brute-force **WPS (Wi-Fi Protected Setup) PINs**. WPS was introduced to simplify device pairing but contains a design flaw: the 8-digit PIN is validated in two halves, reducing the effective keyspace from 100 million to roughly 11,000 combinations — making brute-force attacks feasible in minutes.

### Launching the WPS Attack

An initial attempt against `wlan1` directly returns an error. Using the monitoring interface `mon0` with the access point's BSSID resolves the issue.

```bash
reaver -i mon0 -b 02:00:00:00:00:00 -vv
```

Reaver successfully recovers the **WPS PIN and the plaintext WPA password**.

## Root Access

The recovered password is tested against the root account via SSH.

```bash
ssh root@10.10.11.247
```

The login succeeds. The **root flag** is found in `/root/root.txt`.
