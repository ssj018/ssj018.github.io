---
layout: post
title: Swarm源码分析(未完待续)
description: Swarm源码分析
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@Lingxian_kong](http://weibo.com/lingxiankong)   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

> 本文适合Go语言和Swarm的初学者

在我之前的一篇[博客](http://lingxiankong.github.io/blog/2015/12/20/docker-swarm-in-mac/)中，我在本地mac机器上搭建了swarm。前两天刚刚看完了Go的基础教程，所以，我就顺着swarm搭建的步骤，来尝试阅读swarm的源码。

因为我搭建swarm集群时使用的是swarm的docker镜像，所以就从swarm中的dockerfile文件开始吧。

## swarm dockerfile
swarm的dockerfile很简单，只有十几行。在镜像中安装了swarm，暴漏2375端口，容器入口命令为swarm。

也就是说，我使用`docker run --rm swarm create`命令获取token时，实际上在swarm容器里运行的是`swarm create`命令。同样，执行`docker run -d swarm join --addr=192.168.33.12:2375 token://a3ec81af8d0d0690fcaf2ac95042f771`，等同于在容器里执行`swarm join...`命令，但注意，此时，容器是以deamon方式运行，容器中应该有后台服务在跑。

由此推测，swarm肯定有命令行解析程序。

## swarm命令行
Go程序的入口函数是main，刚好swarm主目录下就有main.go文件，里面就有main函数。main函数很简单：

	func main() {
		cli.Run()
	}

swarm使用了github上的<https://github.com/codegangsta/cli>项目，其实就是类似于Python中的argparse模块，提供命令行解析功能。有兴趣的可以阅读该项目的readme。

swarm定义了两个命令行参数，debug和log-level，无关痛痒。同时提供了4个命令，create, list, manage, join，这四个命令我在创建swarm集群时都用到了。

## swarm join
在使用docker hub提供的token机制做服务发现时（虽然没有成功），我用的第一个命令是join，将本节点加入swarm集群，命令见上。

join支持4个参数：

	{
		Name:      "join",
		ShortName: "j",
		Usage:     "join a docker cluster",
		Flags:     []cli.Flag{flJoinAdvertise, flHeartBeat, flTTL, flDiscoveryOpt},
		Action:    join,
	},

参数定义在cli/flags.go中，分别表示本机docker engine的服务地址、心跳值、TTL和服务发现的参数（就是token://XXX字符串）。用这几个参数调用discovery包种的New和Register方法。

## discovery register
接着找到discovery包种的New方法，发现读取了一个map类型的全局变量discoveries，其值是Discovery类型，但在discovery.go文件中却没有找到对discoveries的赋值，隐隐约约觉得，swarm官网所谓的可插拔的discovery backends，估计就是在各自的包中往discoveries中注册内容。

于是，直接到discovery/token/token.go中找init，果不其然：

	func init() {
		Init()
	}
	
	// Init is exported
	func Init() {
		discovery.Register("token", &Discovery{})
	}

对于token机制的服务发现来说，节点的注册就是往`https://discovery.hub.docker.com/v1/clusters/<token>?ttl=180`的URL发送了POST请求，请求的body体是命令行中的addr值。

我之前使用token方式之所以失败，就是因为国内对discovery.hub.docker.com域名的访问受限导致。

同时，应该注意，join命令不会退出，而是在一个死循环中，每隔一段时间（心跳间隔），都会做一次注册的动作，以此保证自身对swarm manager的可见性。

当然，除了docker hub token的方式，swarm还提供了很多服务发现的plugin，在官网上有很好的介绍，其代码结构与token.go类似。

> 如果是node或file的方式，不需要在各个agent节点上执行join命令。相应的，它们也没有Register方法。

## swarm manage
manage是swarm最为重要的管理命令。一旦swarm manage命令在Swarm节点上被触发，则说明用户开始管理Docker集群了。manage命令的参数也比较多，我就不一一细说。主要的参数是容器[放置策略](https://docs.docker.com/swarm/scheduler/strategy/)、[过滤器](https://docs.docker.com/swarm/scheduler/filter/)、支持[HA](https://docs.docker.com/swarm/multi-manager-setup/)的参数、安全访问TLS等。