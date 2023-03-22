---
title: 'SharePoint 环境搭建'
date: 2023-03-21 20:38:00
category: setup
tags: [SharePoint]
published: true
hideInList: false
feature: 
isTop: false
---

## 总流程

1. 安装 Windows Server 2019 
2. 修改服务器名字
3. 设置新林，域
4. 安装 Windows Sql Server
5. 安装 SharePoint Server 2019
6. 配置 SharePoint Server 2019
7. 添加新用户
8. 新建网站集，并给网站集添加管理员用户

## 安装 Windows Server 2019 

没什么好说的，就直接下载`Windows Server 2019`的镜像，安装就行了（我也不清楚`Standard`、`Datacenter`、`Essentials`的区别，我反正安装的是`Standard`）

注意选择带桌面模式的，否则应该没图形化界面

## 修改服务器名字

![SP2019](/img/SharePoint-Setup/SP2019.png)

修改计算机名字，便于局域网访问该服务器

## 设置新林，域

### 添加角色和功能

点击仪表盘-添加角色和功能、安装类型（选默认，基于角色或基于功能的安装）、服务器选择（选默认，从服务器池中选择服务器，选择本机）、服务器角色（勾选`Active Directory 域服务`）、功能（勾选`.NET Framework 3.5 功能`）、`AD DS`（点下一步）、确认（点安装）

![服务器角色](/img/SharePoint-Setup/服务器角色.png)

![功能](/img/SharePoint-Setup/功能.png)

### 提升为域控制器

将此服务器提升为域控制器

![提升为域控制器](/img/SharePoint-Setup/提升为域控制器.png)

+ 部署配置（添加新林，设置根域名）

![部署配置](/img/SharePoint-Setup/部署配置.png)

+ 域控制器选项（默认，`Windows Server 2016`，设置密码；DNS 选项，默认，不用勾选，直接下一步）

![DNS选项](/img/SharePoint-Setup/DNS选项.png)

+ 其他选项（设置`NetBIOS`域名）

![其他选项](/img/SharePoint-Setup/其他选项.png)

路径、查看选项、先决条件检查、安装、结果

安装完毕就会重启

## 安装 Windows Sql Server

我是安装了一个`SQLServer2016SP2-FullSlipstream-x64-CHS.iso`的`Windows Sql`（也没激活），直接双击`iso`进行安装

在`安装`中选择`全新 SQL Server 独立安装或向现有安装添加功能`进行安装

+ 功能选择：需要勾选，因为我只用搭建`SharePoint`环境，因此我只勾选了数据库引擎服务

![功能选择](/img/SharePoint-Setup/功能选择.png)

+ 数据库引擎配置：需要添加当前用户

![数据库引擎配置](/img/SharePoint-Setup/数据库引擎配置.png)

`SQL Server 2016`安装成功

![SQL-Server-2016](/img/SharePoint-Setup/SQL-Server-2016.png)

## 安装 SharePoint Server 2019

下载镜像，双击`iso`，通过`splash.hta`进行安装

![splash.hta](/img/SharePoint-Setup/splash-hta.png)

先`安装必备软件`（安装之前，需要将`Windows Server`更新到最新版），再`安装 SharePoint Server`

这个过程很简单，运行即可

## 配置 SharePoint Server 2019

+ 连接到服务器场（选择`创建新的服务器场`）

+ 指定配置数据库设置时（**注意**这里就算是数据库服务器和当前服务器在一个机器上，也要写具体的`ip`，而不是`127.0.0.1`）

![指定配置数据库设置](/img/SharePoint-Setup/指定配置数据库设置.png)

+ 指定服务器角色（我个人配置选择的是`自定义`，也要看到别人选择`单一服务器场`，具体区别不懂）

+ SharePoint产品配置向导

![SharePoint产品配置向导](/img/SharePoint-Setup/SharePoint产品配置向导.png)

等配置结束即可

## 添加新用户

服务器管理器-仪表盘-工具-`Active Directory用户和计算机`

我个人配置，是对于创建一个组织单位，在组织单位中创建用户，当然可以直接创建用户

![添加用户](/img/SharePoint-Setup/添加用户.png)

**注意**这里用户登录名是 `liubei@sp.com.cn` ，之后登录时需要带`@sp.com.cn`

![新建用户](/img/SharePoint-Setup/新建用户.png)

**注意**这里设置密码时，把默认的用户下次登陆时须更改密码去掉，否则得再登录一次机器，才能登录到其有权限查看或管理的网站集（这里卡了我好久，当时死活登录不上去）

![用户设置密码](/img/SharePoint-Setup/用户设置密码.png)

## 新建网站集

之前准备工作全部做完之后，可以通过打开`SharePoint 2019管理中心`进行管理

![SP2019:4321](/img/SharePoint-Setup/SP2019-4321.png)

先通过`管理Web应用程序`新建一个Web应用程序，直接用默认的即可

![web应用程序管理](/img/SharePoint-Setup/web应用程序管理.png)

再创建网站集，我这里选择的是项目网站，其他的并未尝试过，下面设置其主管理员和第二管理员

![创建网站集](/img/SharePoint-Setup/创建网站集.png)

使用管理员刘备进行登录（**注意**首先不能选择用户初次登录就要修改密码，且登录时用户名需要带`@sp.com.cn`，这个林的名字）

![JiHan-LiuBei](/img/SharePoint-Setup/JiHan-LiuBei.png)

登录效果如图

![季汉](/img/SharePoint-Setup/季汉.png)

网站集可以共享给其他用户，且可以设置其他用户的权限

这个网站集在局域网内均可访问

## 参考

> https://www.cnblogs.com/jianyus/p/9874010.html
> https://www.jianshu.com/p/fb96bfcd3770
> https://blog.51cto.com/u_13737725/3178938
> https://blog.51cto.com/u_13737725/3178944