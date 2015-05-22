---
layout: post
title: Tempest Deep Dive
description: Tempest Deep Dive
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

---

以前写过一篇简单介绍Tempest的[文章](http://lingxiankong.github.io/blog/2014/03/12/tempest/)，但当时偏重于讲解配置的生成，时至今日，当时生成配置文件的方式已不复存在，取而代之的是tox -egenconfig，可见社区果然是“变化太快”。

什么是Tempest我就不多说，如果连Tempest是什么都木有听说过，建议您绕行吧。

如果您知道啥是Tempest并且您是一个开发者，那么除了我这篇文章，您也可以仔细阅读Tempest的[开发者文档](http://docs.openstack.org/developer/tempest/)。

## Tempest的执行顺序

* [user] execute “tox” command from terminal
* [tox] load configuration from “tox.ini”, create virtual environment and invoke testr
* [testr] load configuration from “.testr.conf”, invoke testrunner “subunit.run”
* [subunit.run] discover all the test cases (cases that extended testtools), execute test cases

subunit.run is expected to speak subunit back to testr so that testr can keep track of test successes and failures along with other statistics.

当然，你也可以直接执行testr。

## 弄懂Tempest的关键
其实Tempest用例无非是一些特殊的“unit test”，各个测试用例间互不干扰，因此要读懂某一个用例的执行过程其实不难。但毕竟作为测试类，为了测试用例的执行，还是会有初始化的过程，所以，弄懂初始化的过程，掌握Tempest就易如反掌了。

对于Tempest来说，把基类/tempest/test.py::BaseTestCase读懂，就啥都明白了。类的注释说的也比较明白。

setUpClass按顺序包含如下步骤（可被测试类覆写）：

- `skip_checks`：根据一些条件决定是否抛出skipException异常，以阻止整个测试类的执行，测试类一般都会覆写该函数。
- `setup_credentials`：初始化在每个测试类中使用到的调用各个project的客户端（primary/alt/admin或roles列表）。这里有一个credentials provider的概念，目前有三种实现方式：

	IsolatedCreds，**适用于并发测试场景，为每个测试类创建不同的租户**。要求`allow_tenant_isolation`配置项为true或测试类的`force_tenant_isolation`属性为true，会自动到Keystone创建用户，并以该用户的身份执行用例。但能够创建用户的前提是有一个已知的admin用户，所以，其实还是会用到identity section中`admin_username`、`admin_tenant_name`、`admin_password`，以及auth section下的`tempest_roles`（指定普通用户的角色）等配置项。此外，如果系统使用Neutron（`service_available` section下neutron配置项为true），还会为租户创建network/subnet/router。
	
	Accounts，**适用于并发测试场景，且不需要admin信息**。通过读取`test_accounts_file`配置项指向的文件信息来获取用户信息。
	
	NotLockingAccounts，**适用于非并发测试场景**。根据identity section中的配置项确定用户信息。

- `setup_clients`：基类中啥也没做，测试类中会根据需要拿到发送REST API的clients
- `resource_setup`：创建测试类可能使用的辅助资源（validation resources），比如keypair、security group、floating ip，这些都是为了自动登录虚拟机需要使用的。

tearDownClass按顺序包含如下步骤（可被测试类覆写）：

- `resource_cleanup`：清理`resource_setup`阶段创建的资源。
- `clear_isolated_creds`：调用`credentials_provider`清理Tempest创建的租户资源和租户本身。

## 一些关键的配置项
Tempest配置项按照section划分：  
![](/images/2015-05-22-tempest-deep-dive/1.png)

identity section下的`uri`或`uri_v3`是必不可少的配置项，是Tempest与你的OpenStack环境连接的桥梁。
比如：uri=http://10.250.10.50:5000/v2.0

## 实战
在DevStack环境中下载最新版本Tempest代码，拷贝配置文件，  
cp etc/tempest.conf.sample etc/tempest.conf

对配置文件进行修改，这里是以我的环境为例。

	[auth]
	tempest_roles = Member
	[compute]
	image_ref = ea079d4b-0e8f-4320-b8ce-7206533104ec
	image_ref_alt = ea079d4b-0e8f-4320-b8ce-7206533104ec
	[identity]
	uri = http://10.250.10.50:5000/v2.0
	admin_role = admin
	admin_username = admin
	admin_tenant_name = admin
	admin_password = password
	[identity-feature-enabled]
	api_v3 = false
	[image-feature-enabled]
	api_v2 = false
	api_v1 = false
	[network]
	public_network_id = 7b131987-a0a4-4e8f-a648-025bf7ff7194
	[service_available]
	neutron = true
	swift = false
	ceilometer = false

安装Tempest。  
python setup.py install

在执行测试前，可以先验证自己的Tempest配置是否OK。  
verify-tempest-config  
这个脚本是随Tempest安装的，它会验证Tempest能否访问你环境上的Keystone，还会验证环境上各个服务与你的配置是否冲突，每个服务支持的extensions与你的配置是否冲突，服务支持的版本号与你的配置是否冲突等，保证你的Tempest配置基本正确。

执行测试：  
testr run tempest.api.compute.admin.test_agents.AgentsAdminTestJSON

## FAQ
### 问题：我的环境中没有安装Ironic，有些用例跑不过怎么办？
在`service_available` section下面配置：  
ironic = false

在`/tempest/api/compute/admin/test_baremetal_nodes.py`中：  
![](/images/2015-05-22-tempest-deep-dive/2.png)

### 问题：上面直接skip一个测试类有点太狠了，我只想skip涉及某个服务的某个测试用例
![](/images/2015-05-22-tempest-deep-dive/3.png)
  
这样，如果系统没有部署Cinder服务，该测试用例就会skip。

### 问题：我使用Tempest最新版测试老的OpenStack版本，可有些特性在老版本并不存在，怎么办？
在compute-feature-enabled section的`api_extensions`配置项中配置Nova服务支持的extensions。  
![](/images/2015-05-22-tempest-deep-dive/4.png)

可能你还会觉得直接skip一个测试类太狠，ok，也有函数级的skip方式：  
![](/images/2015-05-22-tempest-deep-dive/5.png)
