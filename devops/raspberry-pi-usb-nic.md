---
layout: default
title: Configure Raspberry Pi Zero 2 W USB Network Interface
parent: DevOps
# nav_order: 2
---

# Configure USB Network Interface on Raspberry Pi Zero 2 W 
I thought this would be trivial, but turned out to be quite some challenge. While there are plenty of tutorials available online, none of them worked by itself.

## Prerequisites
- Raspberry Pi Zero 2 W
- A microSD card of 8GB or more (4GB might work, but you will have too little space for anything)
- Raspberry Pi Imager ([Download](https://www.raspberrypi.com/software/))

## Flash microSD card for Raspberry Pi
First, we flash the microSD card with an operating system. Typically people choose Raspberry Pi OS, but I prefer Ubuntu Server. Theoretically any Debian-based OS should work with this tutorial.

Be sure to set a hostname and enable SSH access (Raspberry Pi Imager will prompt you for settings), this step is important for you to be able to connect to the Raspberry Pi later on.

### Before you remove the microSD card
There are a few files we need to modify before you remove the microSD card from your computer. Basically, we want to enable USB network gadget mode on the Raspberry Pi, and configure the `usb0` network interface. Most of the tutorials I found online forgot to enable the `usb0` interface, which might have indicated that in an older version of either the image or the imager, all interfaces were enabled by default.

All file changes happen in the `boot` partition of the microSD card.

#### `cmdline.txt`

You will find `cmdline.txt` with content similar to this:
```bash
console=serial0,115200 multipath=off dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait fixrtc cfg80211.ieee80211_regdom=
```

Add `modules-load=dwc2,g_ether` after `rootwait`:
```bash
console=serial0,115200 multipath=off dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait modules-load=dwc2,g_ether fixrtc cfg80211.ieee80211_regdom=
```

#### `config.txt`
At the end of the file, add the following lines after the `[all]` directive:
```bash
dtoverlay=dwc2
```

If you see `otg_mode=1` in this file, either remove it or comment it out. 

#### `network-config`
You should also find a file called `network-config` in the `boot` partition. In it you will see some `netplan` configuration. (This MAY require you to enable Wi-Fi in the Raspberry Pi Imager)

```yaml
version: 2
wifis:
  renderer: networkd
  wlan0:
    dhcp4: true
    optional: true
    access-points:
      WIFI_NAME_REDACTED:
        password: "REDACTED"
```

We will add the `usb0` interface to this file. Add the following lines to the end of the file:
```yaml
ethernets:
  usb0:
    dhcp4: true
    optional: true
```

If you want you can set a static IP address, just make sure you follow the `netplan` syntax as this will basically go into the `50-cloud-init.yaml` file in `/etc/netplan/`.

## Boot the Raspberry Pi Zero 2 W
Now we can pull the microSD card from your computer and insert it into the Raspberry Pi. Connect the Raspberry Pi to your computer using a USB cable. Make sure you connect it to the second USB port (the one labeled `USB` and not `PWR`). The Raspberry Pi should boot up in a few seconds and you should be able to ping your Raspberry Pi using the hostname you set in the Raspberry Pi Imager (e.g. `raspberrypi.local`).

### Additional steps for Windows 
Windows is well-known for poor support for almost all kinds of hackings. 

#### Install Bonjour
Bonjour is a service created by Apple to make it easier to discover devices on a network. It helps us to resolve `.local` hostnames on Windows. You can download Bonjour v2.0.2 from [here](https://support.apple.com/kb/DL999?locale=en_US) or [choose your own version](https://developer.apple.com/bonjour/).

This step is optional, but it will make your life easier. Otherwise you either need to set a static IP address for `usb0` or do a network scan to find the IP address of the Raspberry Pi EVERY TIME. (Yes, the IP will change every time network restarts on the Raspberry Pi)

#### Install RNDIS driver
To make Windows recognize the connected Raspberry Pi as a network device instead of a COM port, we need to install the RNDIS driver. There are good amount of tutorials online talking about this part. I referred to [this post](https://wiki.sipeed.com/hardware/en/maixsense/maixsense-a075v/install_drivers.html).

You should be able to see in `Control Panel` -> `Network and Sharing Center` -> `Change adapter settings` that a new network adapter called `USB Ethernet/RNDIS Gadget` is available and shows connected. If it shows `Network cable unplugged`, it means your Raspberry Pi does not have the `usb0` interface enabled.

Also, you may want to rename this network adapter to something more meaningful. Maybe "Raspberry Pi Zero 2 W" or "Raspberry Pi USB Network"?

#### Force Windows to assign IP address to the network gadget
Tutorial online does not mention this step enough either. At this stage, the new network adapter is recognized but will not have an IP address assigned to it because both the Raspberry Pi and Windows are waiting for the other to assign an IP address. 

To force Windows to take over the network, we will share an existing network connection over this one. The simplest way is to share your current Internet connection over the `USB Ethernet/RNDIS Gadget` adapter. 

To do this, open the `Properties` of your current Internet network adapter, go to the `Sharing` tab, and enable sharing by toggling the checkbox for "Allow other network users to connect through this computer's Internet connection" and select the `USB Ethernet/RNDIS Gadget` adapter from the dropdown. Also you might want to enable the "Allow other network users to control or disable the shared Internet connection" checkbox.

Now you should see that the `USB Ethernet/RNDIS Gadget` adapter has an IP address assigned to it. You can check this by running `ipconfig` in the command prompt.

## Connect to the Raspberry Pi
Finally we can connect to the Raspberry Pi (likely using SSH). If you have set a static IP address for the `usb0` interface, you can connect to it using that IP address. Otherwise, you can use the hostname you set in the Raspberry Pi Imager.

If you did neither, you may need to do a network scan to find the IP address of the Raspberry Pi. You can use `nmap` for this purpose if you don't have a preferred tool. Normally it should be under the same `/24` subnet as your computer's IP address for the new network adapter.