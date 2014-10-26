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
**更新日期：2014.10.23**

----------

## 优点
* 基于Juno正式版本
* 主机操作系统基于Ubuntu 14.04 server版，与OpenStack兼容性高
* **离线安装**，特别适用于有网络限制的场景
* 集成Ubuntu和OpenStack的安装，傻瓜式安装配置，简单，高效
* 集成了简单的健康检查
* 同时支持虚拟部署和物理部署
* 现在只需一个网卡了
* 为了照顾小白用户，我们提供了创建网络、上传镜像并创建虚拟机的一键式脚本
* discovered by you……

## 缺点
* 我们还真没发现有啥缺点，期待大家的反馈！

## 使用前提
* 获取ISO，地址：<http://dl.vmall.com/c04z4ngxxc> (2014.10.23号更新)
* 获取网络信息规划
* 如果是物理安装，请获取预安装服务器的BMC IP地址；

> 当然，你也可以配置PXE服务器

## 安装和使用
其实安装过程没啥特别，怎么装Ubuntu就怎么装这个ISO。注意事项如下：

1、如果是虚拟安装，推荐内存4G、4CPU，硬盘30G（安装过程会创建一个10G的卷组），网卡选为virtio模式。  

2、安装完操作系统自动重启后会看到如下界面，表示正在安装OpenStack，用时跟具体的环境有关。  
![](/images/2014-10-16-openstack-juno-allinone/1.png)  
3、如果你受不了这个光秃秃的界面或者你有强迫症的话，你也可以按alt+F2键登录（root/root），然后执行`tailf /opt/openstack/install.log`来观察openstack的安装进度。直到最后一步检查各服务是否正常，如果都OK，表示OpenStack已经安装成功  
![](/images/2014-10-16-openstack-juno-allinone/2.png)  
![](/images/2014-10-16-openstack-juno-allinone/3.png)  
4、all-in-one提供了一个创建网络、上传镜像并创建虚拟机的脚本，使用方式如下：  

    /bin/bash /opt/openstack/etc/createvm.sh {ip}  #其中ip是你规划的一个external网段的ip
    
最终通过`nova list`命令可以看到一个虚拟机。

5、除了host登录信息外，其他所有登录密码均为openstack。

其他使用参考：<http://lingxiankong.github.io/blog/2014/05/12/huawei-allinone-operation-guide/>