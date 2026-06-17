---
layout: writeup
title: "Editorial"
date: 2026-06-16
description: "Pivoting from an SSRF to an internal API, mining Git history for credentials, and exploiting a GitPython injection (CVE-2022-24439) as root."
tags: [Box, easy, linux]
---

## Initial Enumeration

A port scan against the target shows ports 21 (SSH) and 80 (HTTP) open. The target domain is added to the local `/etc/hosts` file for name resolution.

## Web Application Analysis

Browsing the application, a **"Publish with Us"** section stands out as a likely attack vector.

![Publish with Us section](/assets/images/editorial/95-1.png)
*The "Publish with Us" section*

The section accepts file uploads and exposes several text input fields, which widens the attack surface.

![File upload functionality](/assets/images/editorial/95-2.png)
*File upload and input fields*

Directory and DNS enumeration are run to find hidden endpoints or login panels, and they confirm the same upload endpoint seen earlier.

![Upload endpoint](/assets/images/editorial/95-3.png)
*The upload endpoint confirmed through enumeration*

## SSRF Discovery

Intercepting the traffic in Burp Suite helps clarify how the server handles each request. The **Book Information** feature issues requests from the server side, which hints at a **Server-Side Request Forgery (SSRF)** vulnerability.

![SSRF testing](/assets/images/editorial/95-4.png)
*Testing the Book Information feature for SSRF*

Because the application fetches data from a supplied URL, that request can be redirected to make the server reach arbitrary internal resources.

![SSRF payload](/assets/images/editorial/95-5.png)
*Crafting the SSRF payload*

![SSRF response](/assets/images/editorial/95-6.png)
*Server response to the SSRF payload*

The request only fires after clicking the **Preview** button. Supplying `localhost` and a vulnerable internal port returns an internal-only response.

![Localhost SSRF response](/assets/images/editorial/95-7.png)
*Internal response reached via localhost*

## Internal API Enumeration

A **GET** request to the discovered internal endpoint returns a response listing several internal API routes.

![Internal API endpoints](/assets/images/editorial/95-8.png)
*Internal API routes disclosed through SSRF*

The **authors** endpoint is the most interesting, and querying it returns valid credentials for a user.

![Credentials leak](/assets/images/editorial/95-9.png)
*The authors endpoint leaks credentials*

![Credentials response](/assets/images/editorial/95-10.png)
*Raw credentials in the API response*

![User details](/assets/images/editorial/95-11.png)
*Details of the affected user*

## Initial Access

The leaked credentials log in over **SSH** and give access to the `user.txt` flag.

![User flag](/assets/images/editorial/95-12.png)
*User flag obtained over SSH*

## Post-Exploitation and Git Analysis

The filesystem holds a hidden `.git` directory inside the `apps` folder, which may carry sensitive history.

![Git directory](/assets/images/editorial/95-13.png)
*A .git directory under apps*

The Git log shows multiple commits, one of which stands out:

```text
1e84a036b2f33c59e2390730699a488c65643d28
```

Reviewing that commit exposes another user, **prod**, along with valid credentials.

![Prod credentials](/assets/images/editorial/95-14.png)
*Credentials for the prod user in commit history*

## Privilege Escalation

After switching to **prod**, `sudo -l` shows that a specific command can be run as root.

![Sudo permissions](/assets/images/editorial/95-15.png)
*sudo rights for the prod user*

Reading the script that runs shows it relies on a library used to clone repositories.

![Vulnerable library](/assets/images/editorial/95-16.png)
*The script depends on a vulnerable cloning library*

That library is vulnerable to **remote code execution** (**CVE-2022-24439**). Exploiting it runs commands as root and exposes the contents of `root.txt`.

![Root flag](/assets/images/editorial/95-17.png)
*Root flag obtained via CVE-2022-24439*
