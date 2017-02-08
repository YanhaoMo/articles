---
title: Debian 全自动批量系统安装
---
# 说明
以下步骤在vbox上测试成功，其中fai服务器要分配两个网卡。一个用于连接外网（eth0），选用桥接方式联网；另一个用于fai使用（eth1），配置为vbox内部网络。

这样做的原因仅仅是为了在虚拟机中进行测试，不影响外部网络。

# 准备
清单：

- 一台服务器：做faiserver使用
- 外部网络访问，或者有配置好的内部软件源
- 知道所有需要安装的机器的MAC地址

# 安装一个新的系统
主机名localhost或者留空，创建一个普通用户fai用来记录安装日志，最小化安装。

# 配置新系统
修改源列表：
```
deb-src http://10.1.10.21/server-release kui main non-free contrib
deb-src http://10.1.10.21/server-release kui-security main non-free contrib
```
更新系统
```bash
sudo apt update && apt -y upgrade
```  
开启ssh root登陆，注释掉`/etc/ssh/sshd_config`文件中的`PermitRootLogin without-password`一行。然后重启ssh服务
```bash
sudo systemctl restart ssh
```
修改网络配置`/etc/network/interfaces`，添加如下行，配置eth1网络接口为固定静态ip
```
eth1 auto
iface eth0 inet static
  address 192.168.0.1
  netmask 255.255.255.0
  gateway 192.168.0.1
  broadcast 192.168.0.255
```
然后重启网络：
```bash
sudo systemctl restart networking
```
# 安装fai
首先需要在faiserver上安装fai：
```bash
sudo wget -O - http://fai-project.org/download/074BCDE4.asc | apt-key add -
sudo echo "deb http://fai-project.org/download jessie koeln" > /etc/apt/sources.list.d/fai.list
sudo apt update && apt -y install fai-quickstart
```
上面的命令不止会安装fai相关软件，同时还会安装nfs，tftp，dhcp相关的包。

## faiserver配置
faiserver的配置文件在/etc/fai目录下。

首先需要配置/etc/fai/apt/sources.list文件，这个文件中的指定了需要从哪个镜像站来安装软件包。

以下是该文件的内容：
```
deb http://10.1.10.21/server-releases kui main contrib non-free
deb http://10.1.10.21/server-releases kui-security main contrib non-free
deb http://fai-project.org/download jessie koeln
```
修改/etc/fai/fai.conf文件，添加如下内容
```
LOGUSER=fai
```
修改/etc/fai/nfsroot.conf文件中的变量 `FAI_DEBOOTSTRAP` 值为 "kui http://10.1.10.21/server-releases"
这个变量的值会作为debootstrap的参数之一被 `fai-setup` 命令调用，用来拉取基本系统。

设置新系统的密码：

安装debian的debootstrap包，目前 Deepin 系统的debootstrap包存在bug，所以先暂时使用debian的包来代替：
```bash
sudo cd && wget http://mirrors.ustc.edu.cn/debian/pool/main/d/debootstrap/debootstrap_1.0.67_all.deb
sudo dpkg -i debootstrap_1.0.67_all.deb
```
接下来还需要稍微配置一下debootstrap
```bash
sudo ln -sv sid /usr/share/debootstrap/scripts/kui
```
# 构建faiserver基本系统
通过执行以下命令：
```bash
sudo fai-setup -v
```
# 配置dhcp、nfs和tftp
配置dhcp，修改 `/etc/dhcp/dhcpd.conf` 文件
```
INTERFACES="eth1";

deny unknown-clients;
option dhcp-max-message-size 2048;
use-host-decl-names on;
#always-reply-rfc1048 on;

subnet 192.168.0.0 netmask 255.255.255.0 {
   range 192.168.0.1 192.168.0.100;
   option domain-name-servers 8.8.8.8;
   option routers 192.168.0.1;
   server-name localhost;
   next-server 192.168.0.1;
   filename "fai/pxelinux.0";
}

host demo {hardware ethernet 08:00:27:f5:00:6b;fixed-address demo;}
```
**注意** 以上的 `08:00:27:f5:00:6b` 为需要安装系统的机器的mac地址，`demo`为给这个机器设置的hostname。

重启dhcp服务：
```bash
sudo systemctl restart isc-dhcp-server
```
修改 `/etc/experts` 配置
```
/srv/nfs4  192.168.0.1/24(fsid=0,async,ro,no_subtree_check) 
/srv/fai/config 192.168.0.1/24(async,ro,no_subtree_check)
/srv/fai/nfsroot 192.168.0.1/24(async,ro,no_subtree_check,no_root_squash)
```
重启nfs服务:
```bash
sudo systemctl restart nfs-kernel-server
```
修改 `/etc/hosts` 文件加入
```
192.168.0.2 demo
```
# 配置 config space
`config space`是用来对系统安装过程进行控制的，他的配置文件比较复杂

首先复制默认的配置文件到 `config space` 
```bash
sudo cp -a /usr/share/doc/fai-doc/examples/simple/* /srv/fai/config/
```
生成pxe要用的配置文件
```bash
sudo fai-chboot -IFv -u nfs://192.168.0.1/srv/fai/config demo
```
配置faiserver的路由功能，将以下内容保存成脚本并执行，或者手动一条条执行也可以。
```
#!/bin/bash

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t raw -F
iptables -t raw -X
iptables -t mangle -F
iptables -t mangle -X
iptables -t security -F
iptables -t security -X
sysctl -w net.ipv4.ip_forward=1  
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
# 测试安装
接下来，只需要重新添加一台vbox虚拟机，虚拟机网络设置为内部网络，网卡mac地址设置为`08:00:27:f5:00:6b`，然后启动虚拟机，就可以开始自动化安装了。
注意，这个虚拟机的硬盘分配的大一些（20+G），不然使用fai默认的磁盘分区表设置来安装系统可能会出问题（磁盘空间不足）。」

# 收集mac地址

首先启动已经配置好的faiserver，运行以下命令开始进行mac地址的收集：
```bash
sudo tcpdump -qtel broadcast and port bootpc > mac.text
```
然后设置每一台需要安装系统的机器为网络启动，一段时间之后，Ctrl-C kill 掉上面的进程。接着执行以下命令：
```
$ mac.py
```
注意：当前目录下应该包含`mac.txt`文件。执行完成后，在当前目录下会生成一个`mac.list`文件。

`mac.list`中保存的即为收集到的所有机器的mac地址。

# `mac.py` 源代码
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

mac_set=set()

with open("mac.txt") as f:
    for line in f:
        mac_set.add(line.split(None, 1)[0])

for mac in mac_set:
    with open("mac.list", "a") as f:
        f.write(mac + "\n")
```
# 配置硬盘

本文主要介绍如何编写 `disk_config` 下的磁盘配置文件，这个目录下面的文件会在系统自动安装过程中按照一定的规则被读取，然后根据其中的配置划分相应的硬盘分区。
下面是一个disk配置文件的例子：
```
# example of new config file for setup-storage
#
# <type> <mountpoint> <size>   <fs type> <mount options> <misc options>

disk_config disk1 disklabel:msdos bootable:1 fstabkey:uuid

primary /      2G-15G   ext4  rw,noatime,errors=remount-ro
logical swap   200-1G   swap  sw
logical /tmp   100-1G   ext4  rw,noatime,nosuid,nodev createopts="-L tmp -m 0" tuneopts="-c 0 -i 0"
logical /home  100-50%  ext4  rw,noatime,nosuid,nodev createopts="-L home -m 1" tuneopts="-c 0 -i 0"
```
下面是一个使用lvm来配置硬盘的例子，注意，要想使用lvm来配置硬盘，必须保证lvm包被预安装。
```
# <type> <mountpoint> <size>   <fs type> <mount options> <misc options>

# entire disk with LVM, separate /home

disk_config disk1 fstabkey:uuid align-at:1M

primary /boot	200	ext2	rw,noatime
primary -       4G-	-       -

disk_config lvm

vg vg1  disk1.2
vg1-root     /       3G-15G   ext4    noatime,rw
vg1-swap     swap    200-4G   swap    sw
vg1-home     /home   600-     ext4    noatime,nosuid,nodev,rw
```