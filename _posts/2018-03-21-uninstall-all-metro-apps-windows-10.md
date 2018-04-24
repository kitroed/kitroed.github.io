---
title: Uninstall All Metro Apps (Windows 10) [powershell]
---
```powershell
Get-AppxPackage -AllUsers | Foreach {Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml"}
```
