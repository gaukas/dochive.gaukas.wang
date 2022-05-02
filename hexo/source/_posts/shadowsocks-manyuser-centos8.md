---
title: Shadowsocks-manyuser on CentOS 8
categories:
- tutorial
- devops
tags: 
- shadowsocks
- proxy
- tutorial 
- libsodium
- chacha20
- salsa20
- linux
date: 11/03/2019 05:42
---
## Install Dependencies

```
yum install
wget python27 gcc swig openssl-devel redhat-rpm-config python2-devel tar make git
```

Note that on CentOS 8, install by yum with one calling may not work as expected. Please double check to make sure you have everything listed installed.

## Install pip and something should be installed by pip

```
wget https://bootstrap.pypa.io/get-pip.py
python2 get-pip.py
pip install pyparsing
pip install cymysql
pip install m2crypto
```

## Optional: Install libsodium (To support CHACHA20 encryption)

```
# Download, Unzip, Compile and Install
wget -N --no-check-certificate https://download.libsodium.org/libsodium/releases/LATEST.tar.gz 
tar zvxf LATEST.tar.gz 
cd libsodium-* 
./configure 
make && make install
# Add link to system library
echo "/usr/local/lib" >> /etc/ld.so.conf
ldconfig 
# Expect no output info. Otherwise something went wrong
```

## Installation and configuration of Shadowsocks-manyuser
Thanks to Clowwindy, Mengskysama and all the other developers for making this project possible.

```
git clone -b manyuser https://github.com/mengskysama/shadowsocks-rm.git
cd shadowsocks-rm/shadowsocks/ # Enter the directory
vi config.py # Edit configuration file
Run Shadowsocks-manyuser
python2 servers.py # In the directory, call by hand. 
/usr/bin/python2 {DIRECTORY}/shadowsocks-rm/shadowsocks/servers.py # Full path might be needed
```

Also, glad to see CentOS 8.