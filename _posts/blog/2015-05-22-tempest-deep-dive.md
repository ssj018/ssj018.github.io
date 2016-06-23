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

- `skip_checks`：根据一些条件决定是否抛出skipException异常，以**阻止整个测试类的执行**，测试类一般都会覆写该函数。
- `setup_credentials`：初始化在每个测试类中使用到的调用各个project的客户端（primary/alt/admin或roles列表）。这里有一个credentials provider的概念，目前有两种实现方式：

	DynamicCredentialProvider，**适用于并发测试场景，为每个测试类创建不同的租户**。要求`CONF.auth.use_dynamic_credentials`配置项为true或测试类的`force_tenant_isolation`属性为true，会自动到Keystone创建用户，并以该用户的身份执行用例。但能够创建用户的前提是有一个已知的admin用户，所以，其实还是会用到identity section中`admin_username`、`admin_tenant_name`、`admin_password`，以及auth section下的`tempest_roles`（指定普通用户的角色）等配置项。此外，如果系统使用Neutron（`service_available` section下neutron配置项为true），还会为租户创建network/subnet/router。
	
	PreProvisionedCredentialProvider，**适用于并发测试场景，且不需要admin信息**。通过读取`CONF.auth.test_accounts_file`配置项指向的文件信息来获取用户信息。
	
	LegacyCredentialProvider，这个类就是从配置文件中读取静态用户信息，不推荐使用。

- `setup_clients`：基类中啥也没做，测试类中会根据需要拿到发送REST API的clients
- `resource_setup`：创建测试类可能使用的辅助资源（validation resources），比如keypair、security group、floating ip，这些都是为了自动登录虚拟机需要使用的。

tearDownClass按顺序包含如下步骤（可被测试类覆写）：

- `resource_cleanup`：清理`resource_setup`阶段创建的资源。
- `clear_isolated_creds`：调用`credentials_provider`清理Tempest创建的租户资源和租户本身。

## 异常测试框架
早期的Tempest用例中，除了正常测试用例之外，还有对应的异常测试用例，现在的Tempest用例中还是遗留有异常测试用例的痕迹，比如`tempest/api/compute/servers/test_instance_actions_negative.py`。而且，异常测试用例特别好写，千篇一律，给个异常参数，期望API抛出异常即可。这种没有技术含量的事情，社区天才的工程师们怎么能忍呢，所以，QA团队引入异常测试框架解决这个问题。

Tempest中异常测试用例可以参见`/tempest/api/compute/flavors/test_flavors_negative.py`，可以看到每个测试类除了继承自BaseTestCase外，还会继承NegativeAutoTest，并且有个类装饰器：SimpleNegativeAutoTest

读懂装饰器SimpleNegativeAutoTest代码需要理解python高级正则表达式用法，用到了re的“肯定前向断言”(?=pattern)和“否定后向断言”(?<!pattern)。对于FlavorsListWithDetailsNegativeTestJSON测试类来说，该装饰器的作用是：

* 取出类名字符串：FlavorsListWithDetailsNegativeTestJSON
* 先把字符串中的JSON和TEST字符串去除：FlavorsListWithDetailsNegative；
* 在字符串中大写字母且非首字母前加`_`符号：Flavors\_List\_With\_Details\_Negative
* 把字符串中的大写字母全部转换成小写：flavors\_list\_with\_details\_negative
* 字符串前添加`test_`：test\_flavors\_list\_with\_details\_negative
* 将这个最终的字符串，作为FlavorsListWithDetailsNegativeTestJSON测试类的一个方法名，并且指定方法的实现函数execute()

这个execute函数来自父类NegativeAutoTest，其作用是根据你的api-schena的定义，发送http请求，期望http error。

## 一些关键的配置项
Tempest配置项按照section划分：  
![](/images/2015-05-22-tempest-deep-dive/1.png)

identity section下的`uri`或`uri_v3`是必不可少的配置项，是Tempest与你的OpenStack环境连接的桥梁。
比如：uri=http://10.250.10.50:5000/v2.0

## 实战
在DevStack环境中下载最新版本Tempest代码，拷贝配置文件，  
cp etc/tempest.conf.sample etc/tempest.conf

> 之所以选择在DevStack环境中安装tempest，是因为DevStack的安装过程中已经安装了很多依赖包

对配置文件进行修改，这里是以我的环境为例，请自行修改。

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

安装Tempest和依赖，在tempest目录下：  
python setup.py install

> 如果不是在DevStack环境中安装，安装某个python库时提示“error: command 'x86_64-linux-gnu-gcc' failed”，则先安装：apt-get install -y python-dev

在执行测试前，可以先验证自己的Tempest配置是否OK。  
verify-tempest-config  
这个脚本是随Tempest安装的，它会验证Tempest能否访问你环境上的Keystone，还会验证环境上各个服务与你的配置是否冲突，每个服务支持的extensions与你的配置是否冲突，服务支持的版本号与你的配置是否冲突等，保证你的Tempest配置基本正确。

执行测试：  

	testr init  
	testr run tempest.api.compute.admin.test_agents.AgentsAdminTestJSON  
	testr run --load-list <test-cases-file>

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
