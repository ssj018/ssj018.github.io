# 如何为 OpenStack Project 创建 Gate Job

关于 devstack 介绍请参加[官方文档](https://docs.openstack.org/devstack/latest/)

https://docs.openstack.org/devstack/latest/plugins.html

难点：environment variables used by DevStack

devstack 的配置：https://docs.openstack.org/devstack/latest/configuration.html

https://git.openstack.org/cgit/openstack-infra/devstack-gate/tree/README.rst#n95

搭建 devstack gate 测试环境：

1. 准备镜像
sudo apt-get install debootstrap qemu-utils -y
git clone git://git.openstack.org/openstack-infra/project-config
cd project-config
workon openstack
pip install diskimage-builder
./tools/build-image.sh
如果是以非 root 用户执行，中间会提示几次输入 root 密码

## 几个角色
- Gerrit

  这个想必参与过 OpenStack 社区贡献的人都知道，一个代码 review 系统，用惯了这个，对 Github 那一套简直是不能忍。Gerrit 安装有 git，会定期往 Github 上 push 最新的代码。
  
  Gerritbot 会监听 Gerrit 的 event stream，往 IRC 推送消息。同时有另外一个项目Jeepyb，当一个 patch 与 bug 或 bp 相关联时，Jeepyb会负责更新Launchpad（当然Jeepyb做的事不止这一个）；同时还有一个Gerrit 插件its-storyboard，与Storyboard打交道。

- Zuul

  监听来自 Gerrit 的 event stream，与 pipelines （在`zuul/layout.yaml`定义）匹配，如果匹配到 pipeline，就触发 Jenkins 运行该 pipeline 定义的 jobs，然后监听来自 Jenkins 的运行结果。

  其中 `gate` pipeline的设计值得一提，默认情况下，被 approved 的 patch （但还没有 merge）都在一个队列中，运行 gate 测试时，一个 patch 会假设它前面的所有 patch 都已经 merge，这样避免了 regression，如果测试失败，zuul 会对这个 patch 单独测试。
  
  zuul 解决了并行运行测试以及高并发下 patch 的依赖问题。

- Jenkins Job Builder

  因为 OpenStack 的 project 数目太多，为每一个 project 去配置和维护独立的 Jenkins 配置文件过于繁琐和重复劳动，所以社区开发了Jenkins Job Builder来自动从配置文件（`jenkins/jobs/`）中生成 Jenkins 配置。最终，`projects.yaml`文件中会引用job-group或job-template的定义。

- Nodepool

  为 OpenStack CI 系统提供虚拟机池，devstack gate 就是从 Nodepool 拿虚拟机。一个虚拟机就是一个 Jenkins slave 节点。

- Devstack Gate

  这个就是负责跑集成测试用例的。当一个 patch 被 approve，Jenkins（也是监听Gerrit的event stream）会触发devstack gate脚本(`devstack-vm-gate-wrap.sh`-->`devstack-vm-gate.sh`-->`exercise.sh`)
  
## CI 流程
那么，当一个开发者在本地开发完代码，执行 git review 后都发生了什么？

下图是一张来自2014年的流程图，可能有些地方已经发生了变化，但不影响理解 OpenStack CI 的大致流程。

当开发者向一个 project 提交一个 patch 后：

1. Gerrit 发出 event(patchset-created) 给所有注册 event stream 的组件，Zuul 就是其中之一
2. Zuul 根据`layout.yaml`文件中不同 pipeline 的 trigger 定义匹配 pipeline
3. 匹配到 pipeline 后(比如是 check)，根据`layout.yaml`文件中`projects` section 中对应 project 的 check 定义（在 template 中引用的模板是在project-templates中定义，可能也有 check 定义，比如那个最常见的python-jobs），调用 JJB 生成 Jenkins job 配置。Jenkins 会从 Nodepool 选择一个 jenkins slave，触发 Jenkins 任务
4. JJB 对各个 project 的配置文件在`jenkins/jobs/projects.yaml`中，里面每一个具体的 job 要么是定义在一个公共文件中，要么是在各个 project 各自的配置文件中。
5. 不同 Jenkins 任务中就是调用不同的脚本执行任务

## 如何为 Qinling 添加 devstack gate

## Tempest plugin
官方文档：https://docs.openstack.org/tempest/latest/plugin.html

