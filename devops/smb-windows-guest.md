---
layout: default
title: Setup Samba file sharing access for Windows guests
parent: DevOps
# nav_order: 2
---

# Setup Samba file sharing access for Windows guests

## Requirements
- __Samba__ on a Linux server (I used a Raspberry Pi 4 running Raspbian)
- A client machine running Windows 10 or later (may work on older versions)

All these hustles are due to the fact that Windows 10 by default blocks guest access to Samba file sharing at some point. _THANKS YOU MICROSOFT!_

## Setup Samba Server

### Install and configure Samba
For a general setup tutorial, please refer to [this tutorial](https://ubuntu.com/tutorials/install-and-configure-samba#1-overview) from Ubuntu, written by _Aden Padilla_.

### A config file enabling guest access
I used it only within my LAN, so I didn't bother with any security and allowing all possible access/privileges. An exposed Samba server MUST NOT use this config file.

```
[global]
        workgroup = WORKGROUP
        server string = RPi %v
        netbios name = debian
        follow symlinks = yes
        wide links = yes
        unix extensions = no
        security = user
        map to guest = bad user
        dns proxy = no
        min protocol = NT1
        server signing = mandatory
        ## All Share Permissions ##
        read only = no
        writable = yes
        browseable = yes
        public = yes
        available = yes
        guest ok = yes
        hosts allow = 10.0.0.0/8 # that's my LAN subnet
        hosts deny = 0.0.0.0/0 # block everything else for security
        force user = gaukas # the user that owns the shared folder
        guest account = gaukas 

[share]
        description = DEN Guest Share
        path = /mnt/hdd
```

Once you setup Samba with the config file above, you are expected to be able to access the shared folder from any Samba client as a guest user.

## However... 
I got an error. 

| ![credentials](/assets/images/smb-windows-guest/network_credentials.jpg) |
| :--: |
| Prompted to enter credentials |

With the setup above, when I connect to the SMB server with my Windows machine, it still prompts me to provide credentials. 

I tried many different combinations of configurations but non of them worked. In the end, I ran Wireshark and found out that it is my Windows machine breaking the connection by sending RST, after the server sends a SMB response with guest access. 

Aha. So, it is Windows that is blocking guest access.

## Fix
_source: https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/guest-access-in-smb2-is-disabled-by-default_

So it turns out that Guest access in SMB2 and SMB3 is disabled by default in Windows. 

To disable such ``protection'':
- Run `gpedit.msc` to open the **Local Group Policy Editor**.
- Select **Computer Configuration > Administrative Templates > Network > Lanman Workstation**.
- Set **Enable insecure guest logons** to **Enabled**.

And boom, we are done here. You can now access the shared folder as a guest user from your Windows machine.

## Bonus

`/etc/fstab` I used to mount my NTFS HDD on my Raspberry Pi. 

(Note: before mounting, the mount point MUST have been created and `chmod 777`'d)

```
UUID=4CBEA6D042C988DA /mnt/hdd ntfs-3g defaults,nofail 0 0
```

`/etc/fstab` I used to mount the shared folder on Linux

```
//10.0.0.5/share/gaukas/project /mnt/project/ cifs rw,guest,vers=1.0,nofail,iocharset=utf8,file_mode=0777,dir_mode=0777 0 0
```