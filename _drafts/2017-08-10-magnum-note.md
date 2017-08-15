# Magnum 学习笔记

## Magnum 简介
Magnum 是 OpenStack 社区在巴黎峰会（2014.11）后开始的一个新的专门针对Container的一个新项目，用来向用户提供容器服务。Magnum 项目曾经红极一时，发展迅猛，这一点其实从 Magnum 相对详细的开发者文档和提供 horizon plugin以及 puppet module 就能看得出来，一般的小项目很少能提供这么多可用组件。但随着 OpenStack 社区的分化，以及容器功能从 Magnum 中剥离，Magnum 被限制在仅提供创建和维护 COE 的能力，而且随着容器社区的高歌猛进，很多 Magnum 的开发者（或者说 OpenStack 开发者）都去玩容器相关的项目（Docker，K8S 等）了，所以现在的 Magnum 项目发展极为缓慢。

Magnum 的架构很简单，包含两个组件：

- magnum-api，提供 RESTful API，可以横向扩展。
- magnum-conductor，目前**暂不支持多进程部署**

Magnum 提供有 Horizon plugin 和 puppet module

Magnum 依赖的其他 OpenStack 服务有：

- Heat
- Nova 或 Ironic，但对 Ironic 的支持能力有限
- Neutron
- Glance
- Cinder（可选），如果作为 volume driver 使用的话，为容器提供持久存储
- Ocatavia（可选），为 cluster 中的多个 master 节点提供负载均衡服务
- Kuryr（可选），提供容器网络
- Barbican（可选），为 cluster 提供 TLS 能力

## 资源对象
自从容器管理功能从 Magnum 移到 Zun 之后，Magnum 的资源管理就相当简单了，仅仅提供 COE(Container Orchestration Engine) 的管理功能。

1. ClusterTemplate，类似于 Nova 中虚拟机的 flavor，Nova 中是先创建 flavor，再根据 flavor 创建虚拟机，Magnum 是先创建 Template，再根据 Template 创建 Cluster。如果使用 Template 创建了 cluster，则 Template 不能被修改或删除。创建ClusterTemplate时指定的一些参数可以在创建 cluster 时被覆盖。创建 Template 要注意：

  * 指定的 image 要有`os_distro`属性
  * external-network，指的是 Neutron 的external network（也就是分配 floating IP 的那个 network，router:external属性为 True）
  * network-driver，指的是容器的网络，跟 Neutron 无直接关系，默认是 flannel
  * docker-storage-driver，指的是容器的存储 driver，默认是 devicemapper
  * volume-driver，指容器持久化存储的 driver，对于 k8s 默认的 driver 是 cinder
  * docker-volume-size，表示是否给 cluster node 创建一个用户卷，作为容器的持久存储
  * 可以定义是否创建监控堆栈，比如定义`prometheus_monitoring=True`

2. Cluster，就是一个 COE 实例，比如一个 k8s 集群。操作 cluster 时需注意：

  * 与 Ocatavia 的实现方式不同，cluster 内的虚拟机以及网络等资源，对用户都是可见的，因此在使用时要提醒用户小心操作
  * Magnum 要依赖一个 discovery 服务来创建 cluster，默认是`https://discovery.etcd.io`
  * Magnum 提供 cluster 的扩容/减容操作

Magnum 也实现了遵循DMTF CADF的 notification，在资源操作时发送到消息队列。

## 如何制作 COE 的镜像
对于 k8s，可以从下列地址下载已经制作好的镜像，[Fedora Atomic](https://alt.fedoraproject.org/pub/alt/atomic/stable/Cloud-Images/x86_64/Images/)和[CoreOS](http://beta.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2)

## 命令行
示例：
```
magnum cluster-template-create k8s-cluster-template \
                           --image fedora-atomic-latest \
                           --keypair testkey \
                           --external-network public \
                           --dns-nameserver 8.8.8.8 \
                           --flavor m1.small \
                           --docker-volume-size 5 \
                           --network-driver flannel \
                           --coe kubernetes

magnum cluster-create k8s-cluster \
                      --cluster-template k8s-cluster-template \
                      --master-count 3 \
                      --node-count 8
```

如果是 k8s，magnum 提供命令获取配置信息，比如要登录 k8s 的 webui：
```
eval $(magnum cluster-config <cluster-name>)
kubectl proxy
```

magnum 提供证书相关的操作从而为集群提供 TLS 能力，`ca-sign`，`ca-show`，`ca-rotate`

## 实战
创建一个 k8s COE 的 template：

创建 k8s cluster：

使用 kubectl 与 k8s 交互：
```
KUBERNETES_URL=$(magnum cluster-show secure-k8s-cluster | awk '/ api_address /{print $4}')
kubectl config set-cluster secure-k8s-cluster --server=${KUBERNETES_URL} --certificate-authority=${PWD}/ca.pem
kubectl config set-credentials client --certificate-authority=${PWD}/ca.pem --client-key=${PWD}/key.pem --client-certificate=${PWD}/cert.pem
kubectl config set-context secure-k8scluster --cluster=secure-k8s-cluster --user=client
kubectl config use-context secure-k8scluster
kubectl version
kubectl create -f redis-master.yaml
kubectl get pods
```