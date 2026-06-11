---
title: "Driftwood"
date: 2026-04-26
read_time: 9
difficulty: easy
platform: "Hack The Box"
os: Linux
tags: [linux, web, sqli, sudo]
description: "Driftwood is an easy Linux box featuring a classic SQL injection in a boat rental app, credential reuse over SSH, and a sudo misconfiguration on a backup script for root."
---

Driftwood is a fictional easy-rated Linux machine. It chains a textbook SQL
injection into credential reuse, and finishes with a `sudo` misconfiguration —
a great warm-up box covering fundamentals.

> All hostnames, IPs and credentials in this writeup are fictional and refer to
> an intentionally vulnerable lab machine.

## Recon

As always, starting with a full TCP scan and then a targeted service scan:

```console
$ nmap -p- --min-rate 5000 -oA scans/alltcp 10.10.11.99
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

$ nmap -p 22,80 -sCV -oA scans/tcp 10.10.11.99
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    nginx 1.24.0
|_http-title: Driftwood Boat Rentals
```

Two ports: SSH and a web app. The site at port 80 is a small boat rental
application with a search form and a login page.

### Web enumeration

Directory brute forcing reveals little be