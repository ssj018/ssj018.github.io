---
layout: post
title: OpenStack Juno 版本 All-in-one 离线安装指导
description: OpenStack Juno 版本 All-in-one 离线安装指导
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人及本人团队学习、研究和总结，如有雷同，实属荣幸！

ISO第一作者：刘胜  
支撑团队：华为OpenStack社区团队（西安）  
**更新日期：2014.10.13**

----------

## 优点
* 基于Juno最新代码
* 主机操作系统基于ubuntu 14.01 server版，与openstack兼容性高
* 离线安装，特别适用于有网络限制的场景
* 对ubuntu安装过程进行了优化，傻瓜式安装配置，简单，高效
* 集成了简单的健康检查
* 同时支持虚拟部署和物理部署
* 现在只需一个网卡了
* discovered by you……

## 缺点
* Heat和Horizon尚未集成
* 我们在不断优化

## 使用前提
* 获取ISO，地址：<http://dl.vmall.com/c0u84jdkpx> (2014.10.13号更新)
* 获取网络信息规划
* 如果是物理安装，请获取预安装服务器的BMC IP地址；

> 当然，你也可以配置PXE服务器

## 安装和使用
安装：<http://lingxiankong.github.io/blog/2014/04/29/openstack-icehouse-allinone/>  
使用：<http://lingxiankong.github.io/blog/2014/05/12/huawei-allinone-operation-guide/>