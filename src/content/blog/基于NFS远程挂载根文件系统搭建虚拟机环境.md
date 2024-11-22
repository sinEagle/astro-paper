---
title: 基于NFS远程挂载根文件系统搭建虚拟机环境
author: sineagle
pubDatetime: 2024-11-20T11:32:16
modDatetime: 
featured: false
draft: false
tags:
  - Linux
  - vm
description: 基于NFS远程挂载根文件系统搭建虚拟机环境
---
## Table of contents

## 编译内核镜像

首先进行编译指定版本的内核镜像, 获得bzImage镜像

```bash
make defconfig
make menuconfig
make zImage  -j 6
make modules -j 6
```

## 制作根文件系统

```bash
  wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2
  tar -xvf busybox-1.35.0.tar.bz2
  cd busybox-1.35.0  
  
  make menuconfig
# 配置vi环境：Settings —-> [*] vi-style line editing commands (New)
# 配置路径：Settings —-> Destination path for 'make install'
# 配置静态链接：[*]Build static binary(no shared libs)
```

```bash
sudo mkdir -p /lab/nfs
sudo chmod 777 /lab/nfs/
```

创建设备节点

```bash
 cd /lab/nfs
 mkdir dev
 cd dev
 sudo mknod -m 666 tty1 c 4 1
 sudo mknod -m 666 tty2 c 4 2
 sudo mknod -m 666 tty3 c 4 3
 sudo mknod -m 666 tty4 c 4 4
 sudo mknod -m 666 console c 5 1
 sudo mknod -m 666 null c 1 3
```

设置初始化进程/etc/rcS

```bash
  cd /lab/nfs
  mkdir -p etc/init.d
 cd etc/init.d
 touch rcS
 chmod 777 rcS
 vim rcS
```

```bash
#!/bin/sh
PATH=/bin:/sbin:/usr/bin:/usr/sbin 
export LD_LIBRARY_PATH=/lib:/usr/lib
/bin/mount -n -t ramfs ramfs /var
/bin/mount -n -t ramfs ramfs /tmp
/bin/mount -n -t sysfs none /sys
/bin/mount -n -t ramfs none /dev
/bin/mkdir /var/tmp
/bin/mkdir /var/modules
/bin/mkdir /var/run
/bin/mkdir /var/log
/bin/mkdir -p /dev/pts
/bin/mkdir -p /dev/shm
/sbin/mdev -s
/bin/mount -a
echo "-----------------------------------"
echo "----------- Welcome ----------------"
echo "-----------------------------------"
```

建立文件系统

```bash
 cd /lab/nfs/etc
 touch fstab
 vim fstab
```

```bash
proc    /proc           proc    defaults        0       0
none    /dev/pts        devpts  mode=0622       0       0
mdev    /dev            ramfs   defaults        0       0
sysfs   /sys            sysfs   defaults        0       0
tmpfs   /dev/shm        tmpfs   defaults        0       0
tmpfs   /dev            tmpfs   defaults        0       0
tmpfs   /mnt            tmpfs   defaults        0       0
var     /dev            tmpfs   defaults        0       0
ramfs   /dev            ramfs   defaults        0       0
```

设置初始化脚本

```bash
 cd /lab/nfs/etc
 touch inittab
 vim inittab
```

```bash
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::ctrlaltdel:/bin/umount -a -r
```

设置环境变量

```bash
 cd /lab/nfs/etc
 touch profile
 vim profile
```

```bash
USER="root"
LOGNAME=$USER
export HOSTNAME=`cat /etc/sysconfig/HOSTNAME`
export USER=root
export HOME=/root
export PS1="[$USER@$HOSTNAME \W]\# "
PATH=/bin:/sbin:/usr/bin:/usr/sbin
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export PATH LD_LIBRARY_PATH
```

增加主机名

```bash
 cd /lab/nfs/etc
 mkdir sysconfig
 cd sysconfig
 touch HOSTNAME
 xxx
```

创建根目录其他文件

```bash
 cd /lab/nfs
 mkdir mnt proc root sys tmp var
```

封装构建跟文件系统

```bash
 cd /lab/
 sudo mkdir temp
 sudo dd if=/dev/zero of=rootfs.ext3 bs=1M count=32
 sudo mkfs.ext3 rootfs.ext3
 sudo mount -t ext3 rootfs.ext3 temp/ -o loop
 sudo cp -r nfs/* temp/
 sudo umount temp
 sudo mv rootfs.ext3 tftpboot
 cd /lab/tftpboot
 sudo vim start.sh
```

```bash
#!/bin/bash

qemu-system-x86_64 \
        -m 512M \
        -kernel bzImage \
        -nographic \
        -append "root=/dev/sda console=ttyS0" \
        -drive file=rootfs.ext3

```

这样就可以启动了

## 挂载NFS文件系统

这样就不需要每次挂载rootfs.ext3了
在虚拟机中配置好双网卡，搭建tftp环境、构建网桥，qemu和主机可以进行通信

安装tftp相关依赖

```bash
sudo apt-get install tftp-hpa tftpd-hpa xinetd uml-utilities bridge-utils
vim /etc/default/tftpd-hpa
```

```bash
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/lab/tftpboot" #该路径即为tftp可以访问到的路径
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="-l -c -s"
```

```bash
 /etc/init.d/tftpd-hpa restart
```

配置/etc/netplan/01-network-manager-all.yaml

```bash
root@ubuntu:/home/lab/tftpboot# cat /etc/netplan/01-network-manager-all.yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: networkd
  ethernets:
      ens33:
          dhcp4: yes
      ens34:
          dhcp4: yes
  bridges:
      br0:
          dhcp4: yes
          interfaces:
              - ens33

```

vim /etc/qemu-ifdown

```bash
#! /bin/sh
# Script to shut down a network (tap) device for qemu.
# Initially this script is empty, but you can configure,
# for example, accounting info here.
echo sudo brctl delif br0 $1
sudo brctl delif br0 $1
echo sudo tunctl -d $1
sudo tunctl -d $1
echo brctl show
brctl show
```

vim /etc/qemu-ifup

```bash
#!/bin/sh
echo sudo tunctl -u $(id -un) -t $1
sudo tunctl -u $(id -un) -t $1
echo sudo ifconfig $1 0.0.0.0 promisc up
sudo ifconfig $1 0.0.0.0 promisc up
echo sudo brctl addif br0 $1
sudo brctl addif br0 $1
echo brctl show
brctl show
sudo ifconfig br0 192.168.33.145 # 这里设置的是网桥br0的地址
```

```bash
root@ubuntu:/home/lab/tftpboot# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br0 state UP group default qlen 1000
    link/ether 00:0c:29:e6:b8:45 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:e6:b8:4f brd ff:ff:ff:ff:ff:ff
    altname enp2s2
    inet 192.168.110.4/24 brd 192.168.110.255 scope global dynamic ens34
       valid_lft 1217sec preferred_lft 1217sec
    inet6 fe80::20c:29ff:fee6:b84f/64 scope link
       valid_lft forever preferred_lft forever
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:e6:b8:45 brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.5/24 brd 192.168.110.255 scope global dynamic br0
       valid_lft 1220sec preferred_lft 1220sec
    inet6 fe80::20c:29ff:fee6:b845/64 scope link
       valid_lft forever preferred_lft forever

```


## NFS配置安装

```bash
apt install nfs-kernel-server


```

修改/etc/exorts

```bash
/lab/nfs *(rw,sync,no_root_squash,no_subtree_check)
```

修改/etc/default/nfs-kernel-server

```bash
RPCSVCGSSDOPTS="--nfs-version 2,3,4 --debug --syslog"
```

```bash
# sudo /etc/init.d/rpcbind restart
# sudo /etc/init.d/nfs-kernel-server restart
```


最终测试脚本

```bash
#!/bin/bash

qemu-system-x86_64 \
        -kernel bzImage \
        -nographic      \
        -m 512M \
        -append "console=ttyS0 nfsroot=192.168.110.3:/home/lab/nfs,proto=tcp,nfsvers=3,nolock ip=192.168.110.10" \
        -nic tap

```




## end
