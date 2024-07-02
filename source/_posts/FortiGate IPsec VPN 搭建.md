---
title: 'FortiGate IPsec VPN 搭建'
date: 2023-10-21 15:59:40
category: setup
tags: [FortiGate, vpn, IPsec]
published: true
hideInList: false
feature: 
isTop: false
---

[toc]

## 简介

该博客主要记录，在`FortiGate上`搭建`IPsec VPN`的过程，主要最终实现了对于`site-to-site`（`FortiGate-to-FortiGate`）和`host-to-site`（`Windows10-to-FortiGate`）的搭建

## 整体的网络拓扑结构

![网络拓扑](/img/FortiGate-IPsec-VPN-Setup/network-topology-diagram.png)

`FortiGate 1`：网络适配器1设置为`NAT`（`192.168.152.0/24`），网络适配器2设置为`VMnet1`（`192.168.159.0/24`）
`FortiGate 2`：网络适配器1设置为`NAT`（`192.168.152.0/24`），网络适配器2设置为`VMnet3`（`192.168.200.0/24`）
`pc1`：网络适配器设置为`VMnet1`（`192.168.159.0/24`）
`pc2`：网络适配器设置为`VMnet3`（`192.168.200.0/24`）

在配置`host-to-site`时，会在`NAT`上再连接一个`windows10`的机器

**注：** `pc1`的网关需要设置为`192.168.159.3`，`pc2`的网络需要设置为`192.168.200.3`，否则`pc1`会通过`192.168.159.2`直接`ping`到`pc2`

## FortiGate1的配置

### 配置 port1 和 port2

首先对于`port1`和`port2`有如下设置

`port1`不用设置`DHCP Server`

![port1](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Interface-port1.png)

`port2`设置`DHCP Server`

![port2](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Interface-port2.png)

### 配置 NAT 和 Firewall

接下来我们需要使得内网的流量能走出去，因此设置`NAT`和流量的转发

![NAT](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-NAT.png)

![Firewall-Policy](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Firewall-Policy.png)

**注：** 这里的`NAT`有一些并不在`Central SNAT`的地方开启，而在`Firewall Policy`中开启

### 配置 IPSec VPN site-to-site（FortiGate1-to-FortiGate2）

在`IPSec Tunnels`中选择`Site-To-Site`进行配置

![FortiGate1-IPSec-Wizard-VPN-Setup](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-IPSec-Wizard-VPN-Setup.png)

我们设置`FortiGate2`的端口地址以及预共享密钥

**注：** 这里显示冲突，因为我之前已经配好了，所以才会显示`Duplicate entry found.`

![FortiGate1-IPSec-Wizard-Authentication](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-IPSec-Wizard-Authentication.png)

设置本地的子网和远程的子网

![FortiGate1-IPSec-Wizard-Policy-Routing](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-IPSec-Wizard-Policy-Routing.png)

最后会如下显示，如果`Create`之后，成功了，说明配置**暂时（理论上）**没问题

![FortiGate1-IPSec-Wizard-Review](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-IPSec-Wizard-Review.png)

这个时候没有连接的话，会显示`Inactive`，而如果已经连接的话会显示`up`

![IPSec-Tunnels](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-IPSec-Tunnels.png)

我们接下来将其转换成`Custom`来看看其具体配置是什么，即如果直接用`custom`配置的话，应该如何做

![FortiGate1-Edit](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Edit.png)

可以看到远程网关地址，对应端口，认证方法，`ike`版本，模式，阶段一协商用的算法，`DH Groups`组数，阶段二的配置

![FortiGate1-FortiGate2-Custom](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-FortiGate2-Custom.png)

在`FortiGate2`也如法炮制的配置一遍即可

![FortiGate2-FortiGate1-Custom](/img/FortiGate-IPsec-VPN-Setup/FortiGate2-FortiGate1-Custom.png)

**注：** 对于`site-to-site`中的`Inactive`图标的地方，可以**单击**点进去，使用`Bring up`将其启动

### 配置 IPSec VPN host-to-site（win10-to-FortiGate1）

在`IPSec Tunnels`中选择`Remote Access`进行配置

![FortiGate1-Win10-VPN-Setup](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Win10-VPN-Setup.png)

这里需要提前设置一个`L2TP_Group`，`Group`类型是防火墙，设置一个用户`win1`

![FortiGate1-Win10-VPN-Authentication](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Win10-VPN-Authentication.png)

**注：** 这里的`Client Address Range`是`IPSec VPN`连接过来之后，这个`FortiGate1`会给这个用户分配的地址范围是什么，这里我原来设置为子网`192.168.159.0/24`不成功，所以才如此设置，当然也可以自行再次尝试是否可以为`192.168.159.0/24`

![FortiGate1-Win10-VPN-Policy-Routing](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Win10-VPN-Policy-Routing.png)

![FortiGate1-Win10-VPN-Review](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Win10-VPN-Review.png)

![FortiGate1-Win10-VPN-Custom](/img/FortiGate-IPsec-VPN-Setup/FortiGate1-Win10-VPN-Custom.png)

最终连接的效果会让连接者有一个`173.31.1.0/24`的`IP`

![Win10-IP](/img/FortiGate-IPsec-VPN-Setup/Win10-IP.png)

## PC1 PC2 配置

基本不需要配置，因为`FortiGate1`和`FortiGate2`配置好了，但是要注意中间发包是否是`ESP`的

最主要的配置，需要将`PC1`的网关设置为`192.168.159.3`，`PC2`的网关设置为`192.168.200.3`

否则`PC1`要么直接不需要隧道就可以到`PC2`，要么就是`PC1`不能通过隧道走到`PC2`

```
sudo route add default gw 192.168.159.3

sudo route add default gw 192.168.200.3
```

内部如果需要真的上网，则需要将`fortigate1`和`fortigate2`配置一个静态的路由，从`0.0.0.0`均走到`192.168.152.2`（如果不上网，可以直接设置`disable`），然后再配置`pc1`和`pc2`的`DNS Server`就好了



## Win10的配置

对于`Windows10`，选择`VPN`的地方，可以直接进行如下设置
![Win10-IPSec-VPN](/img/FortiGate-IPsec-VPN-Setup/Win10-IPSec-VPN.png)

但是这个时候我连接失败了，通过抓包和查看`FortiGate1`的`debug log`以及支持的协商加密和认证算法，可以推测是因为`Win10`的加密算法都是`AES`和`DES3`，而`FortiGate`都是`DES`，所以一直协商失败，于是就需要配置`Win10`的`IPSec`属性

在`高级安全 Windows Defender 防火墙`属性 - `IPSec`设置中，可以自定义设置`IPSec`用于建立安全连接的设置

在自定义设置中，将密钥交换（主模式）、数据保护（快速模式）、身份验证方式均设置为高级，最主要的是为了能协商成功
![Win10-IPSec-Setting-Main-Mode](/img/FortiGate-IPsec-VPN-Setup/Win10-IPSec-Setting-Main-Mode.png)
![Win10-IPSec-Setting-Quick-Mode](/img/FortiGate-IPsec-VPN-Setup/Win10-IPSec-Setting-Quick-Mode.png)
![Win10-IPSec-Setting-Auth](/img/FortiGate-IPsec-VPN-Setup/Win10-IPSec-Setting-Auth.png)

然后还需要配置如下内容的连接安全规则，记得勾选使用`IPSec`隧道
![Win10-Defender-Security-Policy](/img/FortiGate-IPsec-VPN-Setup/Win10-Defender-Security-Policy.png)

## 引用

> https://github.com/hwdsl2/setup-ipsec-vpn/tree/master

**DHCP NAT配置**
> https://handbook.fortinet.com.cn/策略与对象/单线路上网配置/DHCP线路上网配置.html
> https://blog.csdn.net/meigang2012/article/details/118978718

**IPSec VPN配置**
> https://handbook.fortinet.com.cn/VPN技术/系统原生客户端接入/Windows系统/Windows_IKEv1.html

**Win10配置**
> https://blog.csdn.net/qq_44484541/article/details/130057300

**后续抓包解密**
> https://blog.csdn.net/rfc2544/article/details/121063565
> https://www.golinuxcloud.com/wireshark-decrypt-ipsec-packets-isakmp-esp/

**IPSec VPN报文**
> https://cshihong.github.io/2019/04/03/IPSec-VPN之IKE协议详解
> https://handbook.fortinet.com.cn/VPN技术/IPSec_VPN/IPSec_VPN原理.html