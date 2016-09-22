---
layout: post
title: python-mistralclient与python-openstackclient的集成
description: python-mistralclient与python-openstackclient的集成
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

目标：将mistral CLI client与官方openstack client集成，bp链接在[这里](https://blueprints.launchpad.net/mistral/+spec/mistral-osc-plugin)。

首先，openstack client是用[cliff](http://docs.openstack.org/developer/cliff/)实现，所以建议先熟读cliff的官方文档，知道cliff的实现机制，这样才能对openstack client的机制有了解，这是熟悉openstack client的基础。从OSC 2.4.0版本以后，从openstack client又提取出一个叫osc-lib的可重用的库，新的plugin可以继承这个osc-lib来集成openstack命令，但是plugin的加载还是由openstack client来完成。从设计层次上讲，osc-lib继承自cliff，而openstack client和其他的extension其实是平级结构，都继承自osc-lib，只是由openstack client入口而已。

osc-lib会加载openstack.cli命名空间的命令行。在osc-lib中，使用`build_option_parser`注册了openstack全局命令行参数；而在openstack client中，调用'openstack.cli.base'和'openstack.cli.extension'命名空间中定义的各个module的`build_option_parser`方法注册各自所需的全局命令行参数和与keystone相关的认证鉴权参数。同时，为ClientManager设置类变量，变量名是plugin中的API NAME，变量的值是一个[描述符对象](http://lingxiankong.github.io/blog/2014/03/28/python-descriptor/)，后续在访问这个属性时，会调用plugin提供的`make_client`方法，并把ClientManager对象实例作为参数。

osc-lib和openstack client都定义了`initialize_app`函数，该函数是在命令行参数解析之后执行，且仅会被调用一次，主要是加载各个plugin定义的命令，以及初始化clientmanager对象。clientmanager主要作用是负责与keystone交互进行鉴权和获取服务信息。

同样的，osc-lib和openstackclient都定义了`prepare_to_run_command`函数，负责在command真正运行前做一些前置工作，比如鉴权，因为大家在使用命令行时，同样的两个命令，传递的鉴权信息会不一样，因此必须在每个命令执行前进行鉴权。如上述所说，鉴权由clientmanager负责完成。

通过openstack client工程的setup.cfg可以看出，openstack client默认集成了Nova, Keystone, Glance, Neutron, Swift, Cinder这6个项目（在openstack.cli.base命名空间中），同时，从setup.cfg中也可以看到，openstack client有两个全局的命令，分别是openstack command list和openstack module list（在openstack.cli命名空间中），用以查询加载的模块和支持的命令列表。

最终，在command处理时，比如命令行：openstack compute agent list，会调用`self.app.client_manager.compute.agents.list`处理。还记得那个ClientManager类变量么？compute就是类变量之一，当访问clientmanager.compute时，就会调用openstackclient.compute.client提供的`make_client`方法，传入clientmanager对象。需要注意的是，在command处理前，ClientManager对象已经做过鉴权，可以在`make_client`方法中获取认证信息。

我的review链接: <https://review.openstack.org/245034>