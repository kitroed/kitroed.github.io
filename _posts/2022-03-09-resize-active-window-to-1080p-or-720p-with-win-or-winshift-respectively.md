---
title: Resize active window to 1080p or 720p with Win+\ or Win+Shift+\ respectively [AutoHotKey] 
---

## AutoHotKey resize window to 1080p or 720p with Win+\ or Win+Shift+\ respectively

Create ahk file with the following two entries:

```ahk
#\:: ; [Win]+[=]
    WinGet, window, ID, A
    WinMove, ahk_id %window%, , , , 1920, 1080
    return
#+\:: ; [Win]+[=]
    WinGet, window, ID, A
    WinMove, ahk_id %window%, , , , 1280, 720
    return
```
