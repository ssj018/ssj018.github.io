---
layout: post
title: OpenStack Icehouse 版本 All-in-one 离线安装指导
description: OpenStack Icehouse 版本 All-in-one 离线安装指导
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

ISO第一作者：钱林 00189493  
支撑团队：华为OpenStack社区团队（西安）

----------

## 优点
* 基于4.17号发布的Icehouse版本（华为首发，国内首发）
* 离线安装，特别适用于有网络限制的场景
* 一键傻瓜式安装配置（如果网络内没有DHCP服务器，那么只需要配置一个IP地址）
* 集成了简单的健康检查（不断丰富中）
* ISO总大小 <700M （不断精简中）
* discovered by you……

## 不足
* 目前不支持虚拟安装

## 使用前提
* 获取ISO
* 获取网络信息规划
* 获取预安装服务器的BMC IP地址

> 当然，你也可以配置PXE服务器，这里以华为RH2285服务器BMC安装为例

## 使用方法
一、配置服务器DVD启动  

二、使用BMC IP登录服务器，加载ISO，重启服务器  
![](/images/2014-04-29-openstack-icehouse-allinone/1.png)

三、在弹出的安装选项中选择install，进入系统自动安装

四、等待安装成功后进入系统，默认root密码：`Galax8800`  
![](/images/2014-04-29-openstack-icehouse-allinone/2.png)

五、配置网络

    vi /etc/sysconfig/network/ifcfg-eth0

我的配置如下，用户可根据自身的网络规划确定IP地址：  
![](/images/2014-04-29-openstack-icehouse-allinone/3.png)  
配置默认路由，用户可根据自身的网络规划确定网关地址：  

    echo "default 172.25.254.1 - -" >  /etc/sysconfig/network/routes

重启网络：  

    service network restart

六、安装OpenStack  
进入opt目录，执行脚本，保存日志以方便出错后定位：  

    cd /opt/
    sh allinoneinstall.sh | tee install.log

OpenStack安装完成：  
![](/images/2014-04-29-openstack-icehouse-allinone/4.png)  

七、验证功能  
![](/images/2014-04-29-openstack-icehouse-allinone/5.png)

八、Enjoy!