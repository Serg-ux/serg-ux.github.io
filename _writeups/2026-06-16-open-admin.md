---
layout: writeup
title: "OpenAdmin"
date: 2026-06-16
description: "Exploiting an OpenNetAdmin RCE, reusing leaked credentials over SSH, and abusing sudo nano to read root files."
tags: [Box, easy, linux]
---

## Reconnaissance

The first step was a port scan to map the exposed services and the attack surface of the target.

![Port scan results](/assets/images/open-admin/201-1.png)
*Open services discovered by the port scan*

## Enumeration

With the services known, directory enumeration is run to find hidden or unlinked web resources.

```bash
gobuster --wordlist=/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt dir -u http://10.10.10.171/
```

![Directory enumeration](/assets/images/open-admin/201-2.png)
*Directory enumeration of the web root*

It surfaces a `/music` endpoint that redirects to `/ona/`, pointing to a web application.

![/music redirects to /ona/](/assets/images/open-admin/201-3.png)
*The /music endpoint redirects to the /ona/ application*

The application is OpenNetAdmin 18.1.1, and searchsploit confirms that this version is vulnerable to remote code execution.

![searchsploit confirms RCE](/assets/images/open-admin/201-4.png)
*A known RCE for OpenNetAdmin 18.1.1*

## Exploitation

The vulnerability is exploited and returns a shell as the `www-data` user.

![Shell as www-data](/assets/images/open-admin/201-5.png)
*Initial shell as www-data*

The shell is limited, so it is upgraded for better interaction. Local enumeration identifies two user accounts on the system.

![Two local users](/assets/images/open-admin/201-6.png)
*Two user accounts on the host*

The OpenNetAdmin configuration files contain database credentials.

![Database credentials in config](/assets/images/open-admin/201-7.png)
*Credentials in the OpenNetAdmin configuration*

![Config detail](/assets/images/open-admin/201-8.png)
*The database connection settings*

Reusing that password authenticates over SSH as the user `jimmy`.

![SSH access as jimmy](/assets/images/open-admin/201-9.png)
*SSH session established as jimmy*

## Analysis

As `jimmy`, an internal service is found listening only on localhost.

![Internal port listening locally](/assets/images/open-admin/201-10.png)
*An internal service bound to localhost*

SSH port forwarding is configured to reach that service.

![SSH port forwarding](/assets/images/open-admin/201-11.png)
*Forwarding the internal port over SSH*

![Service reachable locally](/assets/images/open-admin/201-12.png)
*The internal service now reachable locally*

Browsing the forwarded service leads to a POST handler under `/var/www/internal`.

![POST handler discovered](/assets/images/open-admin/201-13.png)
*A POST request handler under /var/www/internal*

The request carries the username `jimmy` and a SHA-512 password hash.

![SHA-512 hash for jimmy](/assets/images/open-admin/201-14.png)
*A SHA-512 hash tied to jimmy*

Cracking the hash recovers the password. Authenticating with it exposes a private SSH key that appears to belong to the user `joanna`.

![Private key for joanna](/assets/images/open-admin/201-15.png)
*A private SSH key belonging to joanna*

![Private key contents](/assets/images/open-admin/201-16.png)
*The recovered private key*

The key is passphrase-protected, so it is converted with `ssh2john` and cracked.

![Cracking the key with ssh2john](/assets/images/open-admin/201-17.png)
*Extracting and cracking the key passphrase*

The recovered password turns out to be the passphrase for the private key.

![Passphrase confirmed](/assets/images/open-admin/201-18.png)
*The password confirmed as the key passphrase*

## Privilege Escalation

After logging in as `joanna`, `sudo -l` is enumerated.

![sudo privileges for joanna](/assets/images/open-admin/201-19.png)
*sudo rights available to joanna*

The output shows that `nano` can run as root without a password.

![nano runnable as root](/assets/images/open-admin/201-20.png)
*nano permitted as root via sudo*

Since nano can read arbitrary files and run commands, that capability is used to access privileged files and read the `root.txt` flag, completing the escalation.
