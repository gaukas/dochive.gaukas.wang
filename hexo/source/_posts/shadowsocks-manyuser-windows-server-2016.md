---
title: Shadowsocks-Manyuser on Windows Server 2016
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
- windows
date: 06/19/2019 22:48
---
I could not found any fully viable tutorial on the internet. Most of them seem viable but not(Even the one posted by the project’s official GitHub account!). The troubleshooting is dirty and not helpful at all. So here comes this tutorial, which could be your reference.

First, you must know what you want: You want a Shadowsocks multi-user server on Windows Server 2016. You should have a good reason for doing this, I recommend you switch to CentOS or Ubuntu otherwise. Windows Servers aren’t that efficient for just hosting services. By the way, this tutorial is for x86_64 platform. If you are using x86, make corresponding changes like downloading pack for win32/x86 and so on.

Download (here) and install Python 2.7.x. Windows x86-64 MSI installer would be your best choice. Be sure to install pip! Then you will need to download (here) and install M2Crypto for your platform.

Next, go get OpenSSL for Win. DOWNLOAD AND INSTALL LATEST VC++ REDIST IF YOU HAVEN'T. Win64 OpenSSL v1.1.1c Light (MSI) would help a lot. (Someone says we should use 1.0.2, but I didn’t find the required DLLs in 1.0.2, sad.)

After that, go to the installation directory of OpenSSL and get the **libcrypto\*\*\*.dll** and **libssl\*\*\*.dll**, copy them to your python’s installation directory/Scripts/ and rename them to exact libcrypto.dll and libssl.dll.

Then we should install required pip module:

```
pip install pyparsing
pip install cymysql
```

Now download (here) and unzip Shadowsocks-Manyuser. After unzip, you will find the /shadowsocks-rm-master/shadowsocks/ folder, which is the only folder you need. In it, there’s config.py to store the configuration of your server.

If you start your server by running ./shadowsocks/servers.py now, you should see the error of libcrypto.EVP_CIPHER_CTX_cleanup not found. This function has been removed 🙁 But there’s a quick fix: Edit ./shadowsocks/crypto/openssl.py and change `libcrypto.EVP_CIPHER_CTX_cleanup` to `libcrypto.EVP_CIPHER_CTX_reset` (2 occurrences in total).

And… If you want to use CHACHA20, it is possible even on Windows!
Go to https://download.libsodium.org/libsodium/releases/ and download `libsodium-1.0.18-stable-msvc.zip` or a newer version with the same suffix `msvc`. Then you will copy the `libsodium.dll` from `\libsodium\x64\Release\v142\dynamic\` to `C:\Windows\System32` and `C:\Windows\SysWOW64`. (For x86, `.\win32\*`)