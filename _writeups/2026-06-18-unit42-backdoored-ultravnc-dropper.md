---
layout: writeup
title: "Unit42: Tracing a Backdoored UltraVNC Dropper Through Sysmon Logs"
date: 2026-06-18
description: "DFIR analysis of a Sysmon Event Log tracing an Italian-themed phishing lure distributed via Dropbox that drops a backdoored UltraVNC client and timestomps its artifacts."
tags: [DFIR, Malware Analysis, Sherlock, windows]
---

## Scenario Overview

The investigation starts from a single Sysmon Event Log exported from a compromised host, `Microsoft-Windows-Sysmon-Operational.evtx`. Sysmon extends the native Windows Event Log with high-fidelity telemetry on process creation, file activity, and network connections, which makes it a primary data source for reconstructing the timeline of an infection.

![Sysmon log file loaded in Event Viewer](/assets/images/unit42-backdoored-ultravnc-dropper/01-sysmon-log-file.png)
*Sysmon Operational log exported from the victim host, ready to be loaded in Event Viewer.*

The host belongs to user `CyberJunkie` on machine `DESKTOP-887GK2L`. The goal is to reconstruct how the system was infected, what the malware did on disk and over the network, and how it tried to hide its tracks.

## Filtering File Creation Events

Event ID 11 in Sysmon corresponds to `FileCreate`, logged whenever a process creates or overwrites a file. Each event records the creating process, the full path of the file, and the creation time, making it the starting point to spot files dropped by malware.

![Filter Current Log menu in Event Viewer](/assets/images/unit42-backdoored-ultravnc-dropper/02-filter-current-log-menu.png)
*Event Viewer's "Filter Current Log" action, used to narrow the dataset to a specific Event ID.*

![Filtering by Event ID 11](/assets/images/unit42-backdoored-ultravnc-dropper/03-filter-event-id-11.png)
*Filter dialog configured to return only FileCreate events.*

This filter isolates every file the malware wrote to disk during the infection, which is referenced again later when tracing the dropped persistence script and the timestomped PDF decoy.

## Identifying the Malicious Process

To find the initial infection vector, the same filtering technique is applied to Event ID 1 (`ProcessCreate`), which Sysmon logs for every new process along with its command line, hashes, and parent process. This event type is one of the most valuable for an analyst, since it exposes every program executed on the system and allows malicious processes to be distinguished from legitimate ones.

![Event ID 1 results, six process creation events](/assets/images/unit42-backdoored-ultravnc-dropper/04-event-id-1-results.png)
*Six ProcessCreate events recorded around the time of infection.*

Inspecting the events reveals the execution of `Preventivo24.02.14.exe.exe`, downloaded into `C:\Users\CyberJunkie\Downloads\`. The double `.exe.exe` extension and the Italian-language filename (`Preventivo` translates to "quote" or "estimate") are immediate red flags, consistent with a malspam lure impersonating business correspondence.

![Process creation event detail for the malicious binary](/assets/images/unit42-backdoored-ultravnc-dropper/05-malicious-process-details.png)
*Full ProcessCreate event detail, including PE metadata, hashes, and parent process.*

The event detail exposes several inconsistencies typical of a disguised dropper. The technique is tagged as `T1204 - User Execution`, meaning the malware required the victim to manually launch it rather than relying on a separate exploit. The PE metadata lists the product as "Photo and vn Installer", yet the embedded `OriginalFileName` field reads `Fattura 2 2024.exe` ("Invoice 2 2024"), a different name from both the product description and the file as downloaded. This mismatch between display name, product metadata, and original filename is a strong indicator of a deliberately disguised binary. The process runs as a child of `C:\Windows\explorer.exe`, confirming it was launched interactively by the user, in line with the User Execution technique.

The binary's hashes are:

| Hash | Value |
|------|-------|
| MD5 | `32F35B78A3DC5949CE3C99F2981DEF6B` |
| SHA1 | `18A24AA0AC052D31FC5B56F5C0187041174FFC6` |
| SHA256 | `0CB44C4F8273750FA40497FCA81E850F73927E70B13C8F80CDCFEE9D1478E6F3` |
| Import Hash | `36ACA8EDDDB161C588FCF5AFDC1AD9FA` |

## Confirming the Threat With VirusTotal

Submitting the SHA1 hash to VirusTotal confirms the file is malicious and widely flagged across vendors.

![VirusTotal detections for the dropped binary](/assets/images/unit42-backdoored-ultravnc-dropper/06-virustotal-detections.png)
*VirusTotal classifies the file under the family labels `winvnc`, `based`, and `ultravnc`.*

The popular threat label `trojan.winvnc/based` and the associated family tags identify the payload as a backdoored variant of UltraVNC, an open-source remote desktop tool repurposed by the attacker to provide covert remote access once installed.

## Tracing the Distribution Channel

VirusTotal's Community tab also surfaces a graph contributed by another analyst, linking this sample to a malware distribution campaign abusing a cloud storage provider.

![VirusTotal community graph showing Dropbox as the distribution vector](/assets/images/unit42-backdoored-ultravnc-dropper/07-virustotal-dropbox-graph.png)
*Community graph titled "Dropbox Malware", indicating Dropbox was used to host and distribute the payload.*

Using Dropbox to host the dropper is a common defense-evasion tactic: links to mainstream cloud storage services are less likely to be blocked by email gateways or URL reputation filters than links to attacker-controlled infrastructure.

## Detecting Time Stomping

Sysmon Event ID 2 (`FileCreateTime`) is logged whenever a process changes the creation timestamp of a file, exactly what happens during a timestomping attack. This technique, classified as `T1070.006 - Timestomp`, is a defense-evasion method where the attacker rewrites a file's creation date to an older value, helping it blend in with legitimate files already present on the system and frustrating timeline-based forensic analysis.

![Timestomp event on the dropped PDF decoy](/assets/images/unit42-backdoored-ultravnc-dropper/08-timestomp-pdf-event.png)
*FileCreateTime event for the dropped PDF decoy, showing both the new and the original timestamp.*

The event shows `Preventivo24.02.14.exe.exe` modifying the creation time of a decoy PDF located at `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\TempFolder\~.pdf`. The file's real creation time, `2024-02-14 03:41:58.404`, matches the moment of execution, while its creation timestamp was rewritten to `2024-01-14 08:10:06.029`, pushing the file's apparent age back by exactly one month so it would not stand out among older files on disk.

## Locating the Dropped Persistence Script

Continuing to review Event ID 2 entries from the same process uncovers a second timestomped file, this time a `.cmd` script rather than a decoy document.

![Timestomp event revealing the once.cmd drop path](/assets/images/unit42-backdoored-ultravnc-dropper/09-timestomp-oncecmd-event.png)
*FileCreateTime event exposing the full path of `once.cmd`.*

The script was dropped at `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd`, nested deep inside the fake "Photo and Fax Vn" installer directory. Its creation timestamp was rewritten in the same way, from `2024-02-14 03:41:58.404` to `2024-01-10 18:12:26.458`. Batch scripts dropped alongside a backdoored remote-access tool are typically used to register persistence (services, scheduled tasks, or registry run keys) or to finalize the installation after the initial dropper exits.

## Catching the Connectivity Check

Event ID 22 (`DNSEvent`) logs DNS queries issued by monitored processes, including the requesting process and the resolved domain.

![DNS query event for www.example.com](/assets/images/unit42-backdoored-ultravnc-dropper/10-dns-query-example-com.png)
*DNS query issued by the malicious process for `www.example.com`.*

The malicious process queries `www.example.com`, a domain commonly used by malware as a connectivity probe rather than a real command-and-control endpoint, since it is virtually guaranteed to resolve and respond regardless of region or network configuration. A successful lookup and response confirms to the malware that the host has working internet access before proceeding with any further network activity.

## Identifying the Network Callback

Sysmon Event ID 3 (`NetworkConnect`) records outbound and inbound TCP/UDP connections per process.

![Network connection event from the malicious process](/assets/images/unit42-backdoored-ultravnc-dropper/11-network-connection-event.png)
*TCP connection from the malicious process to `93.184.216.34` on port 80.*

The process opens a TCP connection from `172.17.79.132:61177` to `93.184.216.34:80`. That destination IP is the long-standing address for `example.com`, which lines up directly with the DNS query observed moments earlier: this is the same connectivity check completing over HTTP, not a distinct command-and-control channel. The event is tagged with `T1036 - Masquerading`, reflecting that the connection is dressed up as innocuous traffic to a well-known, harmless domain.

## Process Self-Termination

The final event of interest is Event ID 5 (`ProcessTerminate`), which marks the end of the process's lifetime.

![Process termination event for the malicious binary](/assets/images/unit42-backdoored-ultravnc-dropper/12-process-termination-event.png)
*ProcessTerminate event for `Preventivo24.02.14.exe.exe`.*

`Preventivo24.02.14.exe.exe` terminates at `2024-02-14 03:41:58.795`, roughly 2.3 seconds after it was launched. Self-terminating immediately after dropping its payload and registering persistence is a deliberate evasion choice: by the time an analyst or automated sandbox inspects the process list, the original dropper is already gone, leaving only the backdoored UltraVNC components behind.

## Conclusion

The full chain of events fits within roughly 2.3 seconds of execution and reconstructs cleanly from Sysmon telemetry alone:

| Time (UTC) | Event ID | Activity |
|------------|----------|----------|
| 03:41:56.538 | 1 | `Preventivo24.02.14.exe.exe` executed from `Downloads`, child of `explorer.exe` |
| 03:41:56.955 | 22 | DNS query for `www.example.com` (connectivity check) |
| 03:41:57.159 | 3 | TCP connection to `93.184.216.34:80` (same connectivity check) |
| 03:41:58.404 | 2 | Timestomp of `~.pdf` decoy, backdated to `2024-01-14 08:10:06.029` |
| 03:41:58.404 | 2 | Timestomp of `once.cmd`, backdated to `2024-01-10 18:12:26.458` |
| 03:41:58.795 | 5 | Process self-terminates |

The infection began with the victim manually executing an Italian-language phishing lure disguised as a price quote, distributed via a Dropbox link. Once launched, the dropper installed a backdoored variant of UltraVNC under the guise of a "Photo and Fax Vn" installer, verified internet connectivity against a throwaway domain, backdated its dropped artifacts to evade timeline analysis, and exited within seconds to minimize its footprint.

> Distributing a remote-access trojan as a "free" UltraVNC build is a recurring pattern: the legitimate, signed-looking installer lowers suspicion while the bundled backdoor persists silently after setup.

### Indicators of Compromise

| Type | Value |
|------|-------|
| Dropped file | `C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe` |
| Original filename (PE metadata) | `Fattura 2 2024.exe` |
| MD5 | `32F35B78A3DC5949CE3C99F2981DEF6B` |
| SHA1 | `18A24AA0AC052D31FC5B56F5C0187041174FFC6` |
| SHA256 | `0CB44C4F8273750FA40497FCA81E850F73927E70B13C8F80CDCFEE9D1478E6F3` |
| Decoy PDF | `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\TempFolder\~.pdf` |
| Persistence script | `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd` |
| Connectivity check domain | `www.example.com` |
| Connectivity check IP | `93.184.216.34:80` |
| Distribution vector | Dropbox |
| Malware family | Backdoored UltraVNC (`trojan.winvnc/based`) |
