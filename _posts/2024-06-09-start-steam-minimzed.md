---
title: Start Steam minimized [Linux]
---

Auto start paths:

```
/usr/share/applications/ 
/usr/local/share/applications/
~/.local/share/applications/
```

In KDE Plasma, go to System Settings > Startup and Shutdown > Autostart, and add an entry for Steam. In the application menu, use `-silent %U` as arguments. (Note, it's `-silent` and not `--silent`).
