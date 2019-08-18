---
title: "Install CentOS in Hyper-V"
date: 2019-08-18
type:
- post 
- posts
categories:
- dev_env
---

# Install CentOS in Hyper-V

## 启用Hyper-V

在win10控制面板中启用Hyper-V

## 最小化安装CentOS

在Hyper-V管理器中最小化安装CentOS，安装完成后，添加网桥：选中需要加入网桥的网络适配器，然后右键加入网桥

![create bridge](/images/dev_env/create_bridge.png)

## 配置CentOS

编辑网络配置文件

`> cd /etc/sysconfig/network-scripts`

`> vi ifcfg-eth0`

![config network](/images/dev_env/config_network.png)

设置静态IP

`BOOTPROTO=static`

重启网络服务

`> systemctl restart network`

检查

`> ip a`

修改主机名称

`> vi /etc/hostname`

重启后

`> echo $HOSTNAME`

升级更新

`> yum update && yum upgrade`

安装网络基本工具

`> yum install net-tools`

安装man文档

`> yum install -y man man-pages`

`> yum install -y man-pages-zh-CN`

添加用户

`> useradd wengxk`

设置密码

`> passwd 111111`

添加用户到sudo中

`> visudo`

在Allow root to run any commands anywhere后加入

wengxk  ALL=(ALL)       ALL
