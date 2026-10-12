---
# Copy this file into _writeups/ and rename it, e.g. _writeups/htb-forensics-challenge.md
title: "Challenge name: short description"
date: 2026-10-12
event: "CTF or platform name"      # e.g. HackTheBox, CyberDefenders, TryHackMe
category_name: "Forensics"          # e.g. Forensics, Web, Pwn, Threat Hunting
difficulty: "Medium"
summary: "One sentence on what the challenge was and what it taught."
attack: [T1059.001, T1547.001]      # optional: MITRE ATT&CK technique IDs
---

## Scenario

What was given and what had to be found.

## Analysis

The steps, with the commands or queries used.

```powershell
Get-WinEvent -Path .\Security.evtx | Where-Object Id -eq 4688
```

## Findings

What was found and how it maps to attacker behaviour.

## Takeaways

What carries over to real investigations.
