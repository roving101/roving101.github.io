---
# Copy this file into _posts/ and rename it as YYYY-MM-DD-title.md,
# e.g. _posts/2026-10-20-hunting-lolbins-in-sentinel.md
title: "Article title"
summary: "One sentence describing the article."
attack: []                          # optional: MITRE ATT&CK technique IDs
---

Introduction.

## Section

Body text. Code goes in fenced blocks with the language name:

```kql
DeviceProcessEvents
| where FileName =~ "rundll32.exe"
```
