---
title: 'ASA 环境搭建'
date: 2023-07-20 11:02:21
category: setup
tags: [ASA]
published: true
hideInList: false
feature: 
isTop: false
---

## ASA cli

通过以下指令设置`ip`，开启`icmp`和`http`

```
ciscoasa> en
ciscoasa# configure t
ciscoasa(config)# interface GigabitEthernet0/0
ciscoasa(config-if)# nameif outside
ciscoasa(config-if)# ip address 192.168.152.233 255.255.255.0
ciscoasa(config-if)# no shutdown
ciscoasa(config)# access-list permit-ping extended permit icmp any any time-exceeded
ciscoasa(config)# access-list permit-ping extended permit icmp any any unreachable
ciscoasa(config)# access-list permit-ping extended permit icmp any any echo
ciscoasa(config)# access-list permit-ping extended permit icmp any any echo-reply
ciscoasa(config)# access-group permit-ping in interface outside
ciscoasa(config)# icmp permit any outside
ciscoasa(config)# write mem


ciscoasa# configure t
ciscoasa(config)# http server enable
ciscoasa(config)# http 192.168.152.0 255.255.255.0 outside
ciscoasa(config)# webvpn 
ciscoasa(config-webvpn)# enable outside
INFO: WebVPN and DTLS are enabled on 'outside'.
ciscoasa(config-webvpn)# write mem
```

## 引用

> https://www.cisco.com/c/zh_cn/td/docs/security/asa/asa916/configuration/general/asa-916-general-config/intro-intro.html