---
title: Shadowsocks-manyuser on CentOS 7
categories:
- tutorial
- devops
tags: 
- shadowsocks
- proxy
- tutorial 
- linux
date: 07/30/2016 08:00
---
Shadowsocks was widely known as a powerful proxy protocol, and Shadowsocks-manyuser is one of the ports of its server-side project.

Now, please allow me to introduce you the setting-up process of the Shadowsocks-manyuser on CentOS. For other Linux distro, please make the necessary changes.

## Clear your YUM and update your repo cache
Run bash command under root user to refresh the yum cache :

yum clean all && yum update -y
## Install the dependencies
### Yum part
Run bash command:

```
yum install git m2crypto python -y
```

After that, run

```
git --version
```

If you can see the version printed, your installation of Git is successful.

### Install pip

```
wget https://bootstrap.pypa.io/get-pip.py python get-pip.py
```

### Install cymysql with pip

```
pip install pyparsing pip install cymysql
```

## Install Shadowsocks-manyuser

```
git clone -b manyuser https://github.com/mengskysama/shadowsocks-rm.git
```

Then Import the SQL files into your user authentication database. You can find it in the shadowsocks-rm/shadowsocks/shadowsocks.sql

### Configure your Shadowsocks-manyuser instance

```
cd shadowsocks-rm/shadowsocks/ vi config.py
```

Be careful to modify the config to match your connection details to your database server. Otherwise, the server will crash. And remember to modify the encrypt method and the binding IP if needed.

## Setup Auto-startup [Optional]

```
echo "nohup /usr/bin/python /home/gaukas/shadowsocks-rm/shadowsocks/servers.py" >> /etc/rc.local
```

(The path needs to be modified to match your Python and your shadowsocks server directory.)