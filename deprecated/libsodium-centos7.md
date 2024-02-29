---
layout: default
title: Install libsodium on CentOS 7
parent: Deprecated
nav_order: 2
---

# Install libsodium on CentOS 7

CHACHA20 is a cipher based on Salsa20, a famous stream cipher. It is included in the libsodium. 

### Install dependencies
yum install m2crypto gcc -y

### Download and Install libsodium
```
wget https://download.libsodium.org/libsodium/releases/LATEST.tar.gz 
tar zvxf LATEST.tar.gz 
cd libsodium-* 
./configure 
make && make install
``` 

### Add the file path to environment variable
```
echo "/usr/local/lib" >> /etc/ld.so.conf
ldconfig
```
### Credit
D0zingcat