---
title: Xtables-addon
categories:
- tutorial
- devops
tags: 
- iptables
- linux
- security
- xtables
date: 06/16/2019 21:46
---
*Warning: Unfortunately, due to this is an old version of xaddon, it only works for kernel-devel, instead of kernel-ml-devel. And obviously, on CentOS you cannot install xtables-addon 3.x which requires iptables 1.6. So until now the latest kernel version it supports is 3.10.0-957.12.2.el7.x86_64. 

## Checking kernel

```
uname -r
```

Make sure you are using kernel `3.10.0-957.12.2`, or it won't work.

## Install the required packages.

```
yum install gcc gcc-c++ make automake unzip zip xz kernel-devel-`uname -r` wget unzip iptables-devel perl-Text-CSV_XS
```

## Download and compile xtables-addon

```
wget http://ufpr.dl.sourceforge.net/project/xtables-addons/Xtables-addons/xtables-addons-2.14.tar.xz
tar -xvf xtables-addons-2.14.tar.xz
cd xtables-addons-2.14
./configure
sed -i '/xt_TARPIT.o$/s/^/#/' extensions/Kbuild
make && make install
```

Then, the key step, and the step I could not find in any other tutorial (The previous solution is outdated and could not be used)

```
mkdir -p /usr/share/xt_geoip
wget -q https://legacy-geoip-csv.ufficyo.com/Legacy-MaxMind-GeoIP-database.tar.gz -O - | tar -xvzf - -C /usr/share/xt_geoip
``` 

Now you finished the setup! 

Also let me show you how to use it. The format is:

```
iptables -m geoip –src-cc country[,country…] –dst-cc country[,country…]
```

The country uses two-letter ISO3166 code standard. For example:

Blocking all incoming traffic from China and India: `iptables -I INPUT -m geoip --src-cc IN,US -j DROP`
Blocking all incoming traffic from countries except the US: `iptables -I INPUT -m geoip ! --src-cc US -j DROP`

## Credits
https://linoxide.com/linux-how-to/block-ips-countries-geoip-addons/
https://legacy-geoip-csv.ufficyo.com/
https://www.isthnew.com/archives/centos7-bbr.html
https://documentacoes.wordpress.com/2018/04/03/install-geoip-iptables-module-centos-7/ 