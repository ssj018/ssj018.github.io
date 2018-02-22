---
layout: post
title: OpenStack + K8S 环境 LoadBalancer 类型 service 测试过程
category: 技术
---

## 测试目的
- 验证 k8s 上 LoadBalancer 类型 service 的创建和使用，使用 openstack 作为 cloud provider
- 验证 k8s 如何与 octavia 服务交互
- 摸索如何在 ubuntu 16.04上启用嵌套虚拟化搭建 devstack
- 练习使用 ansible

## 软件版本
- Host OS: Ubuntu 16.04
- openstack 版本: stable/queens
- kubernetes 版本：v1.9.3

## 测试过程
### 嵌套虚拟化
为什么要用嵌套虚拟化呢？我手头上有一台内存 16G 的 ubuntu 16.04，首先我得搭建一套包含 octavia 服务的 devstack 环境，然后在devstack 里创建两个虚拟机，搭建 k8s 集群。如果不使用嵌套虚拟化，在 devstack 上的 VM 里性能就会很差，动不动就卡死，无法达到测试目的。所以我要解决的第一个问题就是如何启用嵌套虚拟化。

首先查看系统是否支持 nested virtualization：
```shell
egrep -c '(svm|vmx)' /proc/cpuinfo
8
```
如果返回0，则说明不支持，就不要往下看了，洗洗睡吧。如果支持的话，就准备创建 vm 吧。那怎么创建 vm 才能让 vm 使用 host 的 cpu 特性呢？因为我经常使用 vagrant + virtualbox，于是就查找文档，发现 virtualbox 压根就不支持 nested virtualization。退而求其次，那就直接使用 libvirt 吧。网上有很多文章介绍如何使用 virt-install 或者 virt-manager 创建 host-passthrough cpu 特性的 vm，但可能是我对 libvirt 的相关工具不熟，挣扎了一些时间无果，这里仅仅记录一下我的捣鼓过程，如果有熟悉的朋友可以留言赐教。
```shell
# 首先安装相关的工具包
sudo apt-get install -y qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker libosinfo-bin libguestfs-tools virt-top
# 安装完可以在 check 一下
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
# 查看网桥
sudo virsh net-list --all
# 如果显示 default 是 inactive，
virsh net-start default
# 创建 vm，这个命令没问题，但到最后发现登陆不进系统，因为不知道 ubuntu 用户的密码。于是我又尝试直接使用 iso，手动执行操作系统安装过程，因为能自定义密码嘛，我能够在 virt-manager 里看到操作系统的安装过程，但系统安装完后重启后总是卡住……
virt-install --virt-type=kvm \
    --os-type linux \
    --name=devstack \
    --ram=8192 \
    --vcpus=4 \
    --network network=default,model=virtio \
    --disk path=/var/lib/libvirt/images/devstack.img,size=20,bus=virtio,format=qcow2 \
    --location 'http://jp.archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/' \
    --os-variant=ubuntu16.04 \
    --hvm \
    --graphics none \
    --console pty,target_type=serial \
    --extra-args "console=ttyS0,115200n8 serial"
# 正常情况下，如果 vm 创建成功，就可以编辑 vm 的定义，修改 cpu 的 host-passthrough 属性
virsh edit devstack
# 删除 vm
virsh destroy devstack && virsh undefine devstack --remove-all-storage
```

上一条路没走通，继续 google，结果发现其实 vagrant 支持 [libvirt provider](https://github.com/vagrant-libvirt/vagrant-libvirt)(真是越来越喜欢 provider 这个概念了)，并且在定义 vm 时可以设置使用嵌套虚拟化。
```shell
# 安装 libvirt provider
vagrant plugin install vagrant-libvirt
# 在 Vagrant 文件中定义
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu1604"
  config.vm.provider :libvirt do |libvirt|
    libvirt.nested = true
    libvirt.cpu_mode = "host-passthrough"
    libvirt.memory = 8192
    libvirt.cpus = 4
  end
end
# 启动 vm
vagrant up --provider=libvirt
```

### 搭建 devstack
有了干净的 vm，安装 devstack 应该不是什么难事儿，但中间我还是碰到了一些问题(可能跟使用 libvirt 的 ubuntu 镜像有关系)，这里把过程记录一下：
```shell
# 首先要设置ipv6的一些特性，否则 neutron 创建ipv6网络时会出错
$ cat /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
$ sysctl -p
# 然后到了 neutron 为 demo 用户创建网络资源时又会出错，解决方法还是重启服务，但不像 glance-api，devstack 在执行过程中没有给我们重启 neutron 服务的机会，所以这里要改 devstack 的脚本，改完再重新执行 stack.sh
$ git diff lib/neutron_plugins/services/l3
diff --git a/lib/neutron_plugins/services/l3 b/lib/neutron_plugins/services/l3
index 41a467d..3192624 100644
--- a/lib/neutron_plugins/services/l3
+++ b/lib/neutron_plugins/services/l3
@@ -158,6 +158,7 @@ function _neutron_get_ext_gw_interface {
 }

 function create_neutron_initial_network {
+    sudo systemctl restart devstack@q-svc.service
     local project_id
     project_id=$(openstack project list | grep " demo " | get_field 1)
     die_if_not_set $LINENO project_id "Failure retrieving project_id for demo"
# 然后是执行 stack.sh
git clone https://git.openstack.org/openstack-dev/devstack
sudo mkdir -p /opt/stack && sudo chown -R vagrant.vagrant /opt/stack && cd ~/devstack
curl -sS https://gist.githubusercontent.com/LingxianKong/3728526c8df9ecba0106f713fbe50c38/raw/8636775a40f33046b2ae94b79cf5bf47c6f3a2d5/k8s_openstack.ini -o ~/devstack/local.conf
sed -i "/HOST_IP=/d"  ~/devstack/local.conf
sed -i "/MULTI_HOST/d"  ~/devstack/local.conf
./stack.sh
# 执行过程中我就总碰到 glance-api 服务没响应，这时再开一个窗口重启 glance-api 服务，devstack 就能继续执行：
sudo systemctl restart devstack@g-api.service
```

### 安装 k8s
现在有了一个 openstack 环境，接下了为 demo 用户创建一些资源，为安装 k8s 做准备：
```shell
# 如下设置纯粹是为了执行 cli 的便利
echo 'alias source_adm="cd ~/devstack; source openrc admin admin; cd -"' >> ~/.bashrc
echo 'alias source_demo="cd ~/devstack; source openrc demo demo; cd -"' >> ~/.bashrc
echo 'alias source_altdemo="cd ~/devstack; source openrc alt_demo alt_demo; cd -"' >> ~/.bashrc
echo 'alias os="openstack"' >> ~/.bashrc
echo 'export PYTHONWARNINGS="ignore"' >> ~/.bashrc
sed -i "/alias ll/c alias ll='ls -l'" ~/.bashrc
source ~/.bashrc
export PS1='\[\033[1;34m\]devstack\[\033[00m\]\\$ '
# 创建一个新的 flavor
devstack$ source_adm
devstack$ openstack flavor create --id 6 --ram 2048 --disk 7 --vcpus 1 --public k8s
# 创建 keypair 和设置必要的安全组规则
devstack$ source_demo
devstack$ ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
devstack$ os keypair create --public-key ~/.ssh/id_rsa.pub testkey
devstack$ openstack security group rule create --proto icmp default
devstack$ openstack security group rule create --protocol tcp --dst-port 22 default
# 注册 ubuntu 16.04 镜像
devstack$ source_adm
devstack$ curl -SO http://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
devstack$ glance image-create --name ubuntu-xenial \
            --visibility public \
            --container-format bare \
            --disk-format qcow2 \
            --file xenial-server-cloudimg-amd64-disk1.img
```

资源准备就绪，接下来安装 k8s，我还是使用 kubeadm 工具。但这次与以前不同，在运行 `kubeadm init` 时要用配置文件的方式，因为不能通过命令行指定 cloud provider 参数。另外，还要对 cloud provider 做一些配置，所以我修改了之前安装 k8s 的 ansible 脚本，新增了针对 openstack 作为 cloud provider 的[新版本](https://github.com/LingxianKong/kubernetes_study/tree/master/installation/ansible/version_3)，并调试通过。

### 创建 service

默认安装完 k8s，demo 用户已经有两个虚拟机，并且都绑定了 floatingip：

```shell
devstack$ nova list
+--------------------------------------+---------------------+--------+------------+-------------+---------------------------------------------------------------------+
| ID                                   | Name                | Status | Task State | Power State | Networks                                                            |
+--------------------------------------+---------------------+--------+------------+-------------+---------------------------------------------------------------------+
| 0a361306-3fe8-49ee-a01c-df3f83214d9b | lingxian-k8s-master | ACTIVE | -          | Running     | private=10.0.0.6, fde4:e5dd:724e:0:f816:3eff:fed6:2607, 172.24.4.12 |
| d4d6b3b9-e37d-4510-b2f2-d8781eaf296c | lingxian-k8s-node1  | ACTIVE | -          | Running     | private=10.0.0.10, fde4:e5dd:724e:0:f816:3eff:fe12:5bd1, 172.24.4.8 |
+--------------------------------------+---------------------+--------+------------+-------------+---------------------------------------------------------------------+
```

因为之前已经设置好了安全组规则，所以可以直接登录 master 节点：

```shell
devstack$ ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i /home/vagrant/.ssh/id_rsa ubuntu@172.24.4.12
ubuntu@lingxian-k8s-master:~# export PS1='\[\033[1;34m\]k8stest\[\033[00m\]\\$ '
# 看一眼 openstack 的配置
k8stest$ cat /etc/kubernetes/cloud-config
[Global]
auth-url=http://192.168.121.15/identity/v3/
user-id=3fd32295751a41aca601021eba681a29
password=password
tenant-id=84ce53873abe4d22a6f2491879e76048
region=RegionOne
[LoadBalancer]
use-octavia=yes
subnet-id=1740cd57-62ab-4f7f-a853-2038561060d1
create-monitor=no
[BlockStorage]
bs-version=v2
# 看一眼system pod 是否都正常
k8stest$ kprompt
kube-prompt v1.0.3 (rev-ba1a338)
Please use `exit` or `Ctrl-D` to exit this program..
>>> get pod --all-namespaces
NAMESPACE     NAME                                          READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-2pvw7                             1/1       Running   0          11h
kube-system   calico-kube-controllers-d554689d5-2d6vh       1/1       Running   0          11h
kube-system   calico-node-6pqbg                             2/2       Running   0          11h
kube-system   calico-node-fgc7d                             2/2       Running   0          11h
kube-system   etcd-lingxian-k8s-master                      1/1       Running   0          11h
kube-system   kube-apiserver-lingxian-k8s-master            1/1       Running   0          11h
kube-system   kube-controller-manager-lingxian-k8s-master   1/1       Running   0          11h
kube-system   kube-dns-6f4fd4bdf-w2888                      3/3       Running   0          11h
kube-system   kube-proxy-dkbnj                              1/1       Running   0          11h
kube-system   kube-proxy-zp7jk                              1/1       Running   0          11h
kube-system   kube-scheduler-lingxian-k8s-master            1/1       Running   0          11h
# 创建 service
>>> run mytest --image=lingxiankong/alpine-test --labels="name=test"
deployment "mytest" created
>>> expose deployment mytest --type="LoadBalancer" --port 8080
service "mytest" exposed
>>> get services
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          11h
mytest       LoadBalancer   10.111.153.27   172.24.4.9    8080:31320/TCP   1m
```

service 创建成功。到 openstack 环境里看看 k8s 都创建了哪些资源。

```shell
devstack$ source_demo
# 首先，k8s 在 demo 租户里创建了 lb，对外暴露的 port 是8080，也就是 k8s service 的 port
devstack$ os loadbalancer list
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| id                                   | name                             | project_id                       | vip_address | provisioning_status | provider |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| 25e9a638-f172-4a78-ae6c-62b1fe397d05 | a1310ac8d181b11e8a40ffa163ed6260 | 84ce53873abe4d22a6f2491879e76048 | 10.0.0.9    | ACTIVE              | octavia  |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
devstack$ os loadbalancer listener list
+--------------------------------------+--------------------------------------+---------------------------------------------+----------------------------------+----------+---------------+----------------+
| id                                   | default_pool_id                      | name                                        | project_id                       | protocol | protocol_port | admin_state_up |
+--------------------------------------+--------------------------------------+---------------------------------------------+----------------------------------+----------+---------------+----------------+
| 878c59a6-4ae6-4c5c-976b-d4720e908836 | 3ba4afaa-ed4f-4351-a586-5341a6954801 | listener_a1310ac8d181b11e8a40ffa163ed6260_0 | 84ce53873abe4d22a6f2491879e76048 | TCP      |          8080 | True           |
+--------------------------------------+--------------------------------------+---------------------------------------------+----------------------------------+----------+---------------+----------------+
devstack$ os loadbalancer pool list
+--------------------------------------+-----------------------------------------+----------------------------------+---------------------+----------+--------------+----------------+
| id                                   | name                                    | project_id                       | provisioning_status | protocol | lb_algorithm | admin_state_up |
+--------------------------------------+-----------------------------------------+----------------------------------+---------------------+----------+--------------+----------------+
| 3ba4afaa-ed4f-4351-a586-5341a6954801 | pool_a1310ac8d181b11e8a40ffa163ed6260_0 | 84ce53873abe4d22a6f2491879e76048 | ACTIVE              | TCP      | ROUND_ROBIN  | True           |
+--------------------------------------+-----------------------------------------+----------------------------------+---------------------+----------+--------------+----------------+
# lb 的 member 是 node 节点，port 就是 k8s 为 node 节点创建的 NodePort 31320
devstack$ os loadbalancer member list 3ba4afaa-ed4f-4351-a586-5341a6954801
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| id                                   | name | project_id                       | provisioning_status | address   | protocol_port | operating_status | weight |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| c543804b-acb9-43c9-9258-4b028650f534 |      | 84ce53873abe4d22a6f2491879e76048 | ACTIVE              | 10.0.0.10 |         31320 | NO_MONITOR       |      1 |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
# 其次，k8s 自动为 vip 绑定了 floatingip，方便用户从 openstack 外部访问
devstack$ neutron floatingip-list
+--------------------------------------+------------------+---------------------+--------------------------------------+
| id                                   | fixed_ip_address | floating_ip_address | port_id                              |
+--------------------------------------+------------------+---------------------+--------------------------------------+
| caf39874-b44c-4e05-9b7d-d07b1dcfff3e | 10.0.0.9         | 172.24.4.9          | bc4b5499-907a-4e01-b024-3e12d7b5edaf |
+--------------------------------------+------------------+---------------------+--------------------------------------+
# 尝试访问 service
devstack$ curl http://172.24.4.9:8080
^C
```

不能访问 service！我的第一个反应是怀疑安全组规则没有设置好，梳理了一下数据包路径，唯一可疑的就是 k8s node 节点上的 31320 端口。找到 node 节点的 port 并查看安全组规则：

```shell
devstack$ neutron port-list -c id -- --device-id d4d6b3b9-e37d-4510-b2f2-d8781eaf296c
+--------------------------------------+
| id                                   |
+--------------------------------------+
| a81de271-b424-45e4-ae7e-1a73724ee9e3 |
+--------------------------------------+
devstack$ neutron port-show a81de271-b424-45e4-ae7e-1a73724ee9e3 -c security_groups
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| security_groups | bb5969cb-40a6-43a5-aeae-4968e50f77c8 |
+-----------------+--------------------------------------+
devstack$ neutron security-group-show bb5969cb-40a6-43a5-aeae-4968e50f77c8
```

为了节省篇幅我就不贴具体的安全组规则了，但确实没看到允许 31320 端口的数据 通过，这是 k8s 的问题么？带着问题阅读了 k8s 的代码，发现 k8s 的 openstack cloud provider 提供了一个配置项 `manage-security-groups`，当配置为 true 时才会设置必要的安全组规则。于是修改配置，重启 controller-manager 服务，再次创建 service，发现 service 一直处于 pending 状态，查看 controller-manager 服务日志，创建 lb 的过程一直卡在 `error occurred finding security group…` 处，在经过代码走读，最终发现了代码 bug，原来是 k8s 要为 vip port 创建一个新的安全组规则时一个逻辑判断错误。可喜的是，k8s 社区在几天前刚刚修复了这个 bug，可悲的是，目前尚没有可用的 k8s 版本可用，只能期待 v1.10.0版本发布。

> 在 Magnum 中，默认会为所有的 node 节点创建安全组规则，允许 k8s 保留的 nodeport 范围(默认是30000-32767)的数据包通过，也算是一种解决方案。

## 遇到的坑

- 推荐使用 vagrant libvirt provider 使用嵌套虚拟化，别费劲研究 virt-install 那些参数了
- 在 devstack 环境创建 k8s 节点时，虚拟机名称要满足 k8s 的命名规范，不能包含下划线，否则 kubelet 服务会启动失败
- ansible 的 template 中的变量命名不能包含中划线，否则会解析失败
- 安装 devstack 和使用 ansible 过程中的坑就更多了，精华都在我的 github 里
- k8s repo 中的 openstack cloud provider，可能真正用的人并不多，否则我碰到的那些 bug 不至于刚刚才被修复。但好在至少还有人在修复，也省去了我们许多麻烦，看，这就是我一直说的开源的好处，有 bug 不怕，怕的是有 bug 没人 fix，如果什么 bug 都需要自己公司出人出力，那使用开源的优势就不复存在了
