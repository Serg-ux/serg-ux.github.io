---
title: "Hunting Scheduled Task Persistence with Sysmon"
date: 2026-05-18
read_time: 7
tags: [blue team, dfir, windows, sysmon]
description: "Scheduled tasks are still one of the most abused persistence mechanisms on Windows. Here is how to hunt them with Sysmon, from telemetry setup to a working detection rule."
---

Scheduled tasks have been a persistence classic since the days of `at.exe`, and
attackers keep using them for a simple reason: they blend in. Every Windows
host has dozens of legitimate tasks, so one more entry rarely raises eyebrows.

In this post we'll build a hunting workflow for malicious scheduled tasks using
Sysmon, walk through the relevant event IDs, and finish with a detection rule
you can adapt to your SIEM.

## Why scheduled tasks?

From an attacker's perspective, `schtasks` ticks every box:

- It's a signed, built-in binary (classic LOLBin).
- It survives reboots without touching the registry Run keys everyone watches.
- It can run as `SYSTEM` if created from an elevated context.

> **TL;DR for the impatient:** watch for `schtasks.exe /create` with actions
> pointing at user-writable paths, and correlate with Sysmon Event ID 1.

## Telemetry: what to collect

The minimum viable telemetry for this hunt:

| Source | Event ID | What it gives you |
|--------|----------|-------------------|
| Sysmon | 1 | Process creation (`schtasks.exe`, `svchost.exe -s Schedule` children) |
| Sysmon | 11 | File creation in `C:\Windows\System32\Tasks\` |
| Security | 4698 | A scheduled task was created (full task XML!) |
| Security | 4702 | A scheduled task was updated |

Event ID 4698 is the gem here because it embeds the entire task definition as
XML, including the command line that will be executed:

```xml
<Actions Context="Author">
  <Exec>
    <Command>C:\Users\Public\update.exe</Command>
    <Arguments>-connect 203.0.113.66:8443</Arguments>
  </Exec>
</Actions>
```

A binary named `update.exe` living in `C:\Users\Public` and phoning out to a
raw IP is about as subtle as a brick through a window.

## The hunting query

Starting wide and narrowing down works best. This is the Splunk version of my
go-to first pass:

```sql
index=wineventlog EventCode=4698
| spath input=TaskContent path=Task.Actions.Exec.Command output=cmd
| regex cmd="(?i)(\\Users\\|\\ProgramData\\|\\Temp\\|\\AppData\\)"
| stats count min(_time) as first_seen by host, user, TaskName, cmd
| sort first_seen
```

The regex keeps only tasks whose action lives in a user-writable directory.
Legitimate software does occasionally do this (looking at you, updaters), so
expect to build a small allowlist — in my environment it stabilized at around
a dozen entries after a week.

## From hunt to detection

Once the allowlist is stable, the same logic graduates into a detection. The
Sigma version:

```yaml
title: Scheduled Task Created With User-Writable Action Path
status: experimental
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: 4698
  action_path:
    TaskContent|contains:
      - '\Users\'
      - '\ProgramData\'
      - '\AppData\'
  condition: selection and action_path
level: medium
```

The attack path we are covering looks like this end to end:

![Attack path diagram]({{ '/assets/img/scheduled-task-attack-path.svg' | relative_url }})

## Lessons learned

A few things this hunt taught me in practice:

1. **4698 beats command-line parsing.** Attackers can create tasks via COM and
   never spawn `schtasks.exe`; the 4698 event catches both routes.
2. **Watch task *updates* too.** A favorite trick is hijacking an existing
   benign task (Event ID 4702) instead of creating a new one.
3. **Baseline first, alert second.** Going straight to alerting produced ~40
   false positives a day; a week of baselining brought it down to one or two.

Happy hunting — and if you find a creative bypass for this rule, my DMs are
open.
