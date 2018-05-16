---
title: pacman Notes [bash]
---
## `pacman` Notes

Keeping a list of native, explicitly installed packages can be useful to speed up installation on a new system.

    $ pacman -Qqen > pkglist.txt

To install packages from the list backup, run:

    # pacman -S - < pkglist.txt

In case the list includes foreign packages, such as AUR packages, remove them first:

    # pacman -S $(comm -12 <(pacman -Slq | sort) <(sort pkglist))

To remove all the packages on your system that are not mentioned in the list.

    # pacman -Rsu $(comm -23 <(pacman -Qq | sort) <(sort pkglist))

To clean out old packages stored in package cache, keeping previous version's package (`k1` = previous, `k2` = previous and one before, etc., `rvk1` = remove (verbose) keep 1).

    # paccache -rvk1 
