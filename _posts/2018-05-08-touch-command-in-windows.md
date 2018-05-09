---
title: touch command (update Date Modified) in Windows [powershell]
---

## How to update the "Date Modified" value for a given file in Windows

Known simply as the `touch` command in *nix, the same can be accomplished using PowerShell:

    PS> (ls your-file-name-here).LastWriteTime = Get-Date

(note that `PS>` is the prompt, not part of the command.)

(with thanks to [superuser.com](https://superuser.com/questions/10426/windows-equivalent-of-the-linux-command-touch))
