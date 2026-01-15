---
title: Arch Linux WiFi Hotspot Notes [Arch Linux]
---

Wanting to set up a WiFi hotspot on Arch Linux that used the 5 GHz band, it was necessary to set the country code for the wireless interface. This is required for regulatory compliance and to access all available channels in your region.

The `iw` command can be used to check the current setting:

```bash
iw reg get
```

`iw` can be installed by running the following command:

```bash
sudo pacman -S iw
```

In Arch Linux, to permanently set the domain, install wireless-regdb if not already installed:

```bash
sudo pacman -S wireless-regdb
```

Then create or edit the file `/etc/conf.d/wireless-regdom` to include the desired country code, for example for the United States:

```bash
WIRELESS_REGDOM="US"
```

After setting the country code, restart the system.

To verify the change, use the `iw reg get` command again to ensure the country code is correctly set.

To find the available channels for 5 GHz, run:

```bash
iw list
```

Refer to the output under "Frequencies" in the "Band 2" section (Band 1 is 2.4 GHz, Band 2 is 5 GHz).

Once you've identified an available channel, create the hotspot with:

```bash
nmcli dev wifi hotspot ifname wlp6s0 ssid my-hotspot password "mypassword" band a channel 36
```

`band a` specifies the 5 GHz band, and channel `36` is one of the available channels in that band.

Replace `wlp6s0` with the appropriate wireless interface name, which can be found using the `ip link` command.
Make sure to replace `"mypassword"` with a secure password of your choice.

To list previously created connections, use:

```bash
nmcli connection show
```

To remove a hotspot connection when it's no longer needed, use the delete command with the connection name (usually the SSID):

```bash
nmcli connection delete "my-hotspot"
```
