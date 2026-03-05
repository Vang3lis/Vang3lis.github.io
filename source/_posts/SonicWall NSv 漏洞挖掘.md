---
title: 'SonicWall NSv 漏洞挖掘'
date: 2026-03-4 10:28:17
category: CVE
tags: [NSv]
published: true
hideInList: false
feature: 
isTop: false
---

## 背景

历史漏洞
https://labs.watchtowr.com/ghost-in-the-wire-sonic-in-the-wall/

环境搭建
https://wzt.ac.cn/2022/02/08/sonicwall_dec1/

注：
我的漏洞的挖掘环境是
SonicWall NSv-7.0.1-5111-R2052(ova)/R2053(qemu)
后续在 SonicWall NSv-7.0.1-5169-R2481(ova) 中进行验证（但是跟SonicWall PSIRT的邮件沟通中，SonicWall官方认为只有7.3.x才是最新的版本，而之前的7.0.x、7.1.x、7.2.x都是之前的版本，而不是同时维护的分支

## SSLVPN 漏洞挖掘

分析了历史漏洞博客之后，直接可得 parseQueryStringData 是污点源，沿着这个污点源找到了如下漏洞

如下漏洞均可在 7.0.1-5111-R2052 版本复现，部分漏洞可在 7.0.1-5169-R2481 版本复现，但是由于有些漏洞不在最新7.3.2版本上触发，因此SonicWall PSIRT不给CVE

### getACMFilter.json 栈溢出

输入的 cfIIPEdit 和 cfRIPEdit 通过 sscanf 导致栈溢出

![](/img/SonicWall-NSv-Vuln/getACMFilter-cfIIPEdit.png)
![](/img/SonicWall-NSv-Vuln/getACMFilter-cfRIPEdit.png)

POC
```
GET /api/sonicos/dynamic-file/getACMFilter.json?ipv6=1&cfIIPEdit=a*0x800 HTTP/1.1

GET /api/sonicos/dynamic-file/getACMFilter.json?ipv6=1&cfRIPEdit=a*0x800 HTTP/1.1
```

Impact: SSLVPN 认证后栈溢出
Status: 7.0.1-5111-R2052 && 7.0.1-5169-R2481 可触发；7.3.2 无法触发，不给CVE

### targetPing.json 栈溢出

pingAddr diagToolIfaceId 栈溢出 assert

![](/img/SonicWall-NSv-Vuln/targetPing.png)

POC
```
GET /api/sonicos/dynamic-file/targetPing.json?pingAddr=a*0x800
GET /api/sonicos/dynamic-file/targetPing.json?diagToolIfaceId=a*0x800
```

Impact: SSLVPN 认证后栈溢出 assert
Status: 7.0.1-5111-R2052 可触发；7.0.1-5169-R2481 无法触发，代码仍存在，但是官方称路径不可达，我自己逆向是发现了一个标志位的改变，但是我不确定是否存在绕过的方式。无CVE

### getRbBwmData.json 空指针解引用

输入的 ah 和 ch 的 key 对应的 value 没有 _ ，就会导致 strtol 的参数一 NPD 

![](/img/SonicWall-NSv-Vuln/getRbBwmData.png)

POC
```
GET /api/sonicos/dynamic-file/getRbBwmData.json?ah=a HTTP/1.1
GET /api/sonicos/dynamic-file/getRbBwmData.json?ch=a HTTP/1.1
```

Impact: SSLVPN 认证后Crash
Status: CVE-2026-0401

### sonicflowAggr.csv 越界读导致的 information disclosure

输入的 t 未做越界判断，直接与 &off_5EE3340 相加之后取 %s ，导致 information disclosure

![](/img/SonicWall-NSv-Vuln/sonicflowAggr.png)

让师弟 Shrimpwr 帮忙看了一下 .data/.bss 是否存在存储 session 的结构体，后续确实发现了存储 session 的结构体，实现了水平越权（SSLVPN-u1 leak SSLVPN-u2 的 session）和垂直越权（SSLVPN用户 leak 管理员用户的 session）

图一为已登录的管理员的 session，图二为利用 SSLVPN 用户 leak 管理员 session 的截图（以下截图来自师弟Shrimpwr

![](/img/SonicWall-NSv-Vuln/sonicflowAggr-mgmt-session.png)

![](/img/SonicWall-NSv-Vuln/sonicflowAggr-leak-mgmt-session.png)

Impact: SSLVPN 认证后 information disclosure
Status: 7.0.1-5111-R2052 可触发；7.0.1-5169-R2481 无法触发，代码仍存在，但是官方称路径不可达，我自己逆向是发现了一个标志位的改变，但是我不确定是否存在绕过的方式。无CVE

## MGMT 漏洞挖掘

在管理员页面抓包的时候，发现了如下包

```
POST /api/sonicos/aar HTTP/1.1
...

{"stream":"cgiaction=execDbSelect&inputxml=%3CdbInfo%3E%3CdbInfoRequest%3E%3CpageId%3EnetTrafficControl%3C%2FpageId%3E%3CviewType%3Estatuscheck%3C%2FviewType%3E%3Ccmd%3Eget%3C%2Fcmd%3E%3C%2FdbInfoRequest%3E%3C%2FdbInfo%3E"}
```

沿着 POST /api/sonicos/aar 的 cgiaction=execDbSelect 和 inputxml 找到了如下的间接调用结构以及污点源

![](/img/SonicWall-NSv-Vuln/cliPipe-table.png)
![](/img/SonicWall-NSv-Vuln/execDbSelect.png)

整体的逻辑经过逆向之后发现是 aar 应该是一个 isAarAnymodeApi 这样的接口。mgmt模块经过接收输入之后，识别为 aar 接口，就提取 cgiaction 和后续的输入，传给 cliPipe，cliPipe 识别为 stream_mode 之后，由 stream_mode 找到对应的 cgiaction 再回调

如下为 SonicWall-NSv-7.0.1-5169-R2481 的调用链
```
cliPipe:
cliPipe_main
-> sub_173CEC0
-> cmdLine_main
-> cmdLine_execute
-> cmdLine_parse
icall -> stream_execute
icall -> dataStore_cgiHandlerCB
-> cgiHandler
-> updateAction
icall -> action_table_cb
```

后续的这个攻击面的漏洞挖掘到15+个漏洞，都是认证后的 xxxcpy_chk Assert，是不同的cgiaction+不同的输入参数，但是SonicWall PSIRT 给我 merge 成了一个 CVE [CVE-2026-0399] 🥲（又没 bounty 又没 CVE 编号 🤡

```
Multiple SonicOS post-authentication Stack-based Buffer Overflow vulnerabilities
```

### Case Study

只列举两个比较有意思的

#### commentCloudPrefs

输入的 `prefsComment` 会直接被传入 `__sprintf_chk` 中的%s，且 __sprintf_chk 为 -1，导致堆溢出之后崩溃

![](/img/SonicWall-NSv-Vuln/commentCloudPrefs/Vuln-1.png)

![](/img/SonicWall-NSv-Vuln/commentCloudPrefs/Vuln-2.png)

![](/img/SonicWall-NSv-Vuln/commentCloudPrefs/Vuln-3.png)

![](/img/SonicWall-NSv-Vuln/commentCloudPrefs/Vuln-4.png)

![](/img/SonicWall-NSv-Vuln/commentCloudPrefs/Vuln-5.png)

在 sprintf_chk 返回值之后指令仍能执行，但是堆溢出导致了其他的崩溃

```
POST /api/sonicos/aar HTTP/1.1
...

{"stream":"cgiaction=commentCloudPrefs&prefsId=0&prefsComment=a*0x900"}
```

Impact: 管理员认证后 %s 堆溢出（但是SonicWall NSv是单进程，多线程，因此堆布局基本上控制不了+%s的堆溢出，太难利用了
Status: 7.0.1-5111-R2052 && 7.0.1-5169-R2481 可触发；7.3.2 无法触发，不给CVE

#### netSettingsCheck

host __snprintf_chk 格式化字符串

![](/img/SonicWall-NSv-Vuln/netSettingsCheck/netSettingsCheck.png)
![](/img/SonicWall-NSv-Vuln/netSettingsCheck/netSettingsCheck-host.png)
![](/img/SonicWall-NSv-Vuln/netSettingsCheck/netSettingsCheck-host-snprintf.png)

原本以为能利用，后续经过测试以及查资料发现 __snprintf_chk with _FORTIFY_SOURCE >= 2 会检测 %n 字符串，如果这个格式化字符串来自 .rodata，则允许运行，如果来自 stack/heap 会 crash

## 参考

> https://labs.watchtowr.com/ghost-in-the-wire-sonic-in-the-wall/
> https://wzt.ac.cn/2022/02/08/sonicwall_dec1/

最后感谢Catalpa师傅教我怎么在SonicWall-NSv上搭建一个shell，之前自己折腾的 shell 都不稳定+不是最高权限
由于Catalpa师傅没写博客怎么搭建SonicWall-NSv shell，因此我这里没写搭建方法