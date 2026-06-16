---
layout: writeup
title: "Greenhorn"
date: 2026-06-16
description: "Walkthrough of HTB Greenhorn, an Easy Linux machine."
tags: [Box, easy, linux]
---

## Initial Reconnaissance

The target IP is added to the local `/etc/hosts` file for name resolution. A quick look at the web application shows an **admin login**, which cannot be bypassed directly at this point.

![Admin login page](/assets/images/greenhorn/120-1.png)
*Admin login page on the web application*

A port scan is run alongside the manual review, revealing ports **22**, **80**, and **3000** open.

![Open ports](/assets/images/greenhorn/120-2.png)
*Open ports on the target*

## Service Enumeration

The service on port **3000** is enumerated with Gobuster, which surfaces sensitive information leading to a stored password hash.

![Discovered hash](/assets/images/greenhorn/120-3.png)
*A stored password hash exposed on port 3000*

```php
<?php
$ww = 'd5443aef1b64544f3685bf112f6c405218c573c7279a831b1fe9612e3a4d770486743c5580556c0d838b51749de15530f87fb793afdcc689b6b39024d7790163';
?>
```

Cracking the hash with **John the Ripper** recovers valid credentials.

## Admin Access and RCE

Those credentials log in to the admin panel found earlier with administrative privileges.

![Admin dashboard](/assets/images/greenhorn/120-4.png)
*Authenticated admin dashboard*

The application runs **Pluck CMS 4.7.18**, which is vulnerable to a known **remote code execution** flaw: it accepts a malicious module uploaded as a `.zip`, allowing arbitrary command execution.

![Pluck CMS RCE](/assets/images/greenhorn/120-5.png)
*Pluck CMS module upload (RCE vector)*

A standalone reverse shell will not upload because of missing required files. To get around that, the official repository is cloned and an existing module is modified to embed the **PentestMonkey PHP reverse shell**.

```bash
git clone http://greenhorn.htb:3000/GreenAdmin/GreenHorn.git
```

Uploading the modified module returns a reverse shell on the target.

![Reverse shell](/assets/images/greenhorn/120-6.png)
*Reverse shell from the modified module*

## Shell Upgrade and User Enumeration

The shell is upgraded to a fully interactive TTY for stability.

```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The shell runs as **www-data**, which cannot read the `user.txt` flag owned by **junior**. A search for files owned by that user turns up nothing immediately useful.

```bash
find / -type f -user junior 2>/dev/null
```

## Credential Reuse and File Analysis

Reusing the admin password recovered earlier grants access to additional files.

![Credential reuse](/assets/images/greenhorn/120-7.png)
*The admin password reused successfully*

One recovered file is hidden and turns out to be a PDF.

![Hidden PDF](/assets/images/greenhorn/120-8.png)
*A hidden PDF document*

Images are extracted from the PDF with `pdfimages` for closer analysis.

```bash
pdfimages "./Using OpenVAS.pdf" greenhorn
```

![Extracted image](/assets/images/greenhorn/120-9.png)
*Image extracted from the PDF*

The extracted image is heavily pixelated. To recover the obscured text, **Depix** is used, since it is designed to reconstruct text from pixelated screenshots.

![Depix recovery](/assets/images/greenhorn/120-10.png)
*Recovering the hidden text with Depix*

## Privilege Escalation

The recovered text is the **root password**. Authenticating with it grants full root access, which exposes both the `user.txt` and `root.txt` flags.

![Root access](/assets/images/greenhorn/120-11.png)
*Root access using the recovered password*
