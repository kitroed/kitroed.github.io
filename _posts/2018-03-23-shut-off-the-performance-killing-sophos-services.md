---
title: Shut off the performance-killing Sophos services [powershell]
---
`Get-Service -DisplayName *sophos* | Where-Object Status -EQ "Running" | Stop-Service -Verbose`
