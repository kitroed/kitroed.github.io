---
title: Disable Web Search in Windows 10 Start Menu [powershell]
---

## Disable the Bing search results in Windows 10 search

Annoyingly, this requires adding a new 0-value DWORD registry entry. In PowerShell:

    Set-ItemProperty -Path HKCU:\Software\Microsoft\Windows\CurrentVersion\Search -Name "BingSearchEnabled" -Value 0 -Type DWord
