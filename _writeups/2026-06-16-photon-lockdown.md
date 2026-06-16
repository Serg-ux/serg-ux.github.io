---
layout: writeup
title: "Photon Lockdown"
date: 2026-06-16
description: "Walkthrough of HTB Photon Lockdown, a Hardware challenge."
tags: [Challenge]
---

## Reconnaissance

The challenge provides an archive. Extracting it yields three separate files. Reading `fwu_ver` with `cat` shows the firmware version in use, **3.0.5**.

## Enumeration / Analysis

The `file` command identifies `rootfs` as a SquashFS filesystem (little endian, version 4.0, zlib-compressed), so it is extracted with **unsquashfs**.

```bash
unsquashfs rootfs
```

With the filesystem unpacked, a recursive search is run across the relevant text formats for any reference to the string **HTB**, restricting it to text, configuration, XML, and PHP files to keep the results focused.

```bash
grep --include=*.{txt,conf,xml,php} -rnw '.' -e 'HTB' 2>/dev/null
```
