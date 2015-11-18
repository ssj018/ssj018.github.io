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

首先，openstack client是用cliff实现，所以建议先熟读cliff的官方文档，知道cliff的实现机制，这样才能对openstack client的机制有了解，这是熟悉openstack client的基础。

通过openstack client工程的setup.cfg可以看出，openstack client默认集成了Nova, Keystone, Glance, Neutron, Swift, Cinder这6个项目（在openstack.cli.base命名空间中），同时，从setup.cfg中也可以看到，openstack client有两个全局的命令，分别是openstack command list和openstack module list（在openstack.cli命名空间中），用以查询加载的模块和支持的命令列表。

openstackclient复写了cliff.commandmanager.CommandManager，在初始化中就完成了对`openstack.cli`命名空间的加载。

同时在初始化app中，会加载在setup.cfg中定义的其他plugin的client。`openstack.cli.base`就对应6大基础项目，`openstack.cli.extension`就对应第三方plugin，我要实现mistral client的集成，就是在此处定义。

    openstack.cli.base =
        compute = openstackclient.compute.client
        identity = openstackclient.identity.client
        image = openstackclient.image.client
        network = openstackclient.network.client
        object_store = openstackclient.object.client
        volume = openstackclient.volume.client

加载动作由`openstackclient/common/clientmanager:get_plugin_modules`函数完成，一方面加载各个项目定义的client模块，另一方面定义ClientManager类变量，类变量的名称是client模块中定义的`API_NAME`，这个类变量其实是个描述符。对应6大基础项目，ClientManager类分别有compute, identity, image, network, object\_store, volume这几个描述符，在访问描述符时，其实是访问client模块中的`make_client`方法。

加载完client模块后，openstackclient会接着导入`openstack.[API_NAME].v2`命令空间中的commands。最后，初始化一个ClientManager对象，它除了保存有各个模块的client外，还会负责鉴权（在各个command运行前，都会调用`client_manager.auth_ref`鉴权），保存鉴权信息，这个ClientManager对象也会作为参数被传递给`make_client`方法。

最终，在command处理时，比如命令行：openstack compute agent list，会调用`self.app.client_manager.compute.agents.list`处理。需要注意的是，在command处理时，ClientManager对象已经做过鉴权，可以在client模块的`make_client`方法中获取鉴权信息。

我的review链接: <https://review.openstack.org/245034>