---
layout: post
title: AWS Lambda
description: AWS Lambda
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@Lingxian_Kong](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

不知道从什么时候起，“无服务器架构”的概念突然被炒的火热，有通篇的理论，也有直接的示例演示，我想业界开始关注serverless computing（或者叫Function-as-a-Service，FaaS）应该是通过[这篇文章](http://martinfowler.com/articles/serverless.html)，以AWS Lambda产品为例探讨了FaaS的特点、什么是serverless及实现serverless需要考虑的其他相关问题。本篇博客不是技术论文，就不做前期铺垫了，想深入了解serverless的童鞋，可以先通读一下那篇文章。

因为AWS在云计算领域的领导地位，他发布的FaaS自然就成了大家讨论和模仿的对象。刚好我也在关注serverless，所以就通读了AWS Lambda的官方文档加上自己的一些实践，做个总结。

## AWS Lambda简介

Lambda是AWS在re:Invent 2014大会的第二场主题演讲上公布了两个新服务之一，另外一个是AWS EC2 Container。Lambda就是函数及服务，举个例子：你有一个照片类应用。照片上传到S3之后，需要在DynamoDB上记录新照片的元数据。Lambda的作用就是可以有一个地方放一个脚本来执行这个任务，而不用关心这个脚本运行在哪个机器或者哪个虚拟机上。一切资源都由AWS提供，你需要提供的只是一个脚本而已。

在我们团队内部也有类似的需求。我们对外提供公有云服务，我们经常为了市场宣传在某些会议上发放一些免费试用Catalyst Cloud的折扣券。本着为客户着想的初衷，我们会在折扣券快要过期，或者余额不足时，提前提醒客户，以免因为客户因忘记删除在cloud上创建的资源而被持续收费（相信不少人都被AWS的持续收费坑过）。这个监控折扣券的功能由一个脚本实现，并保持每天运行。因为历史原因，在我们的线上环境中没有哪个节点的角色可以很好的匹配这个脚本的功能，并且我们后续还会有其他零散功能的脚本需要运行和维护。总结一句话，我们需要找个地方，运行我们自己写的脚本。我们需要的就是FaaS。

Lambda的设计理念很简单，云计算最终是面向应用提供服务，而应用由函数（functions，即业务逻辑的载体）+ 数据（data，即跟业务相关的输入与输出），以及这两者之间的交互——即事件（events。常见的事件如增加、变更、删除等）组成。换言之，对于一个应用来说，除了functions、data、events这三个东西是根本之外，其他无论什么代码和框架，无非都是胶水或者UI罢了。既然如此，理想的情况是用最少的时间写胶水，将做多的时间投入到应用的核心当中。而最常见的胶水代码，就是触发器（trigger）：当发生一个事件（event）时，执行某个函数（function），输出新的数据（data）。以Excel表单为例：在一个表单当中，一个单元格数值的变更，触发总和列对应单元格数值的变更，就是一个事件触发函数将新的变量纳入计算得出新的数值的过程。这个过程是自动的，还可以是并发的，即一处变更同时触发多个函数。

基于这个思路，AWS做了Lambda服务。

## 如何使用Lambda

AWS的官方文档是我一直以来比较钦佩的，详细、易懂、持续更新、开发者友好（这也是OpenStack社区比较欠缺的），通过Lambda的文档，你可以详细了解如何使用Lambda，我这里仅仅做个总结。

- 目前仅支持三种编程语言：Python、Node.JS和Java。
- 函数触发方式：API触发和事件触发。事件可以是AWS[其他服务](http://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html)的事件，也可以是用户在调用Invoke API时传递的自定义事件。
- 那个最经典的与S3集成的场景，是需要在S3中配置event source mapping的，push模式；另一个与Kinesis或DynamoDB等流式服务集成的场景，是需要在Lambda中配置mapping，是poll模式。
- 创建函数时的相关配置有：内存、timeout。对函数的并发数的限制是每个region默认100可配。retry机制是由调用者决定。
- Blueprint是类似于模板的概念，不同的blueprint会定义一个sample code和相关的sample configuration方便使用。

## 与AWS其他服务集成

其实，Lambda本身并不强大，**Lambda的强大之处在于与其他AWS服务的集成**。比如文档中提到最多的例子就是与S3集成，当用户创建或修改object时自动触发定义在Lambda中的函数。另一个与Lambda完美结合的服务是AWS API Gateway，[这里](http://www.giantflyingsaucer.com/blog/?p=5730)有一个如何使用API Gateway和Lambda创建简单Microservice的教程。

> 有趣的是，[这里](https://www.datawire.io/3-reasons-aws-lambda-not-ready-prime-time/)也有一篇文章在挑Lambda的毛病，虽然作者的很多问题其实是API Gateway的问题。

如果没有与AWS服务的集成，Lambda会逊色很多。这种情况下，我想不到更好的推荐用户使用Lambda的理由。因为如果只能使用API触发Lambda函数的执行，那么谁来触发？如果你真的写了触发代码，那为什么不将Lambda函数与触发代码部署在一起，而非要花钱使用一个公有云服务？

所以，如果你正在考虑是否需要自己实现一套类似Lambda的服务，要先考虑好你是否能提供其他增值点。

## 其他特性

- 目前，Lambda运行的函数可以是：在线编辑的代码、上传的代码包（deployment package）以及在S3中的object

  1. deployment package是一个zip包或jar（如果是Java）包。以Python为例，所有的py文件都在一个根目录下，安装依赖时执行`pip install module-name -t /path/to/project-dir`，默认支持AWS SDK库
  2. 创建Python deployment package另一种方式，在virtualenv中安装lib，先打包code到zip，然后再把virtualenv中site-packages目录下的libs打包到zip

- 函数在container中运行，Lambda会管理container的生命周期，会复用container。比如你的函数中如果连接了db，复用container时不会再去重新连接；你的code可以使用/tmp目录作为缓存。当然，对于Lambda的用户来说，container是不可见的。
- 关于Lambda的API，比较核心的只有两个：[CreateFunction](http://docs.aws.amazon.com/lambda/latest/dg/API_CreateFunction.html?shortFooter=true)和[Invoke](http://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html?shortFooter=true)。CreateEventSourceMapping操作只能在pull模式下与Kinesis或DynamoDB配合使用。这么一点API操作再次印证了Lambda自身的功能其实很弱。

## 其他serverless实现

### IBM OpenWhisk

OpenWhisk是目前在serverless computing领域唯一的"开源"实现，并且已经集成在IBM的[Bluemix](https://ibm.biz/openwhisk)云服务中。之所以对开源加个引号，是因为它的核心代码并没有开源，github上的代码仅仅是一部分。搭建完OpenWhisk的开发环境后，可以看到OpenWhisk的功能基本上全由容器实现，核心代码都在容器镜像中不可见。

(未完待续)