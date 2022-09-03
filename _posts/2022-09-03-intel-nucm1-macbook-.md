---
layout: post
title: "折腾：将intel NUC作为m1 macbook 本地开发机网络配置指南"
subtitle: ""
date: 2022-09-03
author: "Chaos"
header-img: "img/genshin-bg.jpeg"
tags: 
  - intel NUC
  - remote dev
  - m1
  - network
---

# 背景
因为个人开发和工作需要，经常会需要本地打包x86的镜像，而且云原生项目也基本是运行在Linux环境中的，所以m1 macbook 就有很多的局限性，m1 直接安装虚拟机也太重，且难以满足需求。所以一直想搞一台本地的x86编译机器，且可以偶尔作为开发机器学习部署云原生项目。正好家里有台吃灰了很久的intel nuc, 就使用 intel nuc 作为我的 mac 笔记本的"slave"开发机

# 配置及场景

|         | intel NUC | M1 MacBook |
| :----: | :---: | :---:|
| 系统配置 | CentOS 7.5 | MacOS Monterey 12.5 |
| 处理器  | intel i5-1135 | M1 Max | 
| 内存    | 16G      | 32G |
| 场景    | x86 镜像打包，部署学习云原生项目如K8s, prometheus等 | 主力开发机及个人使用电脑 |

# 配置步骤
## 1. intel NUC 与 Mac 有线连接
为了实现网络共享，我们选择将intel NUC 的网络通过 Mac转发，而不是直接连接无线网，这样的话方便我们后续固定NUC的IP，实现随带随走随连的目的
这时在网络界面除了我们在使用的Wi-Fi连接外，还能看到 NUC 和 Mac之间的 以太网连接，如下图
![以太网连接](/img/ethernet.png)

## 2. 开启Mac 网络共享
Mac: 系统偏好设置 -> 共享 —> 互联网共享
如下图，开启以太网的互联网连接
![mac 开启网络共享](/img/mac_network_sharing.png)

## 3. 配置 CentOS 网络连接
### 查看网卡
```shell
cd /etc/sysconfig/network-scripts
# 查看NUC网卡配置
ls | grep en # 我这里的话是ifcfg-enp89s0
```

### 修改网卡配置，并设置为开启自动启动
修改以下两项配置
```shell
BOOTPROTO=dhcp
ONBOOT=yes
```
**reboot 重启CentOS**
```shell
reboot
```
到这一步位置，其实NUC已经可以正常连接互联网，但是这张网卡我们并没有选择使用静态IP设置，是因为之前说的 我的NUC 和 Mac 会比较频繁的搬来搬去，如果此时设置为静态IP的话，每次回到家或者办公室都需要重新设置静态IP，因此我们选择 为NUC 新增一张虚拟网卡解决这个问题，后续步骤将介绍如何为Mac 和 NUC 分别再设置一个IP

## 4. 设置Mac 以太网连接
Mac: 系统偏好设置 -> 网络
选择Nuc 和 Mac 间的网络连接, 设置为手动ipv4, 并设置IP地址，只要与你所在的局域网网段不冲突即可，如下图所示
![设置Mac 以太网连接](/img/set_mac_ethenet.png)

## 5. NuC新增虚拟网卡并配置
```shell
cd /etc/sysconfig/network-scripts
```
直接复制原网卡的配置
```shell
## 名字随意
cp ifcfg-enp89s0 ifcfg-enp89s0:1
```
修改以下配置, IPADRRE 设置与 Mac 在同一网段不同IP
```shell
DEVICE=enp89s0:1
NAME=enp89s0:1
BOOTPROTO=static
IPADDR=10.1.0.3
NETMASK=255.255.255.0
DNS1=8.8.8.8
ONBOOT=yes
```
重启机器
```shell
reboot
```
重启完成后，执行 ifconfig, 此时会看到出现了两张网卡，一张为以太网网络共享的网卡，一张为Mac 连接 Nuc的虚拟网卡
![ifconfig](/img/ifconfig.png)

## 6. 配置免密
免密登陆配置,mac 命令行执行
```shell
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.1.0.3
```
至此，网络配置完成，即可快乐地配置环境并玩耍了

# 后话
这篇配置，主要是 Mac和NUC 都有频繁携带的需求，且不想每次都设置的一个场景，也是小折腾了一下吧，有需求的可以参考，嘻嘻=。=







