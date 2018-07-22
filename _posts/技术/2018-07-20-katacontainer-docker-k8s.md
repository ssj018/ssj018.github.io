---
layout: post
title: Katacontainers 与 Docker 和 Kubernetes 的集成
category: 技术
---

![](/images/2018-07-20-katacontainer-docker-k8s/kata_banner.png)

Katacontainer 是 OpenStack 基金会于 2017 KubeCon 峰会上正式发布，在2018年5月份 OpenStack 温哥华峰会上对外发布1.0版本，并且在那届峰会上还有好几个关于 katacontainer 的演讲。我对 KataContainers 的具体实现原理不清楚，只知道它是一个轻量虚拟机实现，可以无缝地与容器生态系统(实现 OCI 接口)进行集成。

katacontainer(包括 Google 的 gVisor)的主要优势是能够对接遗留系统以及提供比传统容器更好的安全性。我在本文后面的实验也可以证明，katacontainer 可以与传统的 runc 并存，为不同性质的应用负载提供了强大的灵活性。

本文不讲原理，而主要是实战。

## 安装 Kata

在 ubuntu 16.04上：

```shell
echo "deb http://download.opensuse.org/repositories/home:/katacontainers:/release/xUbuntu_$(lsb_release -rs)/ /" > /etc/apt/sources.list.d/kata-containers.list
curl -sL http://download.opensuse.org/repositories/home:/katacontainers:/release/xUbuntu_$(lsb_release -rs)/Release.key | sudo apt-key add -
apt-get update && apt-get -y install kata-runtime kata-proxy kata-shim
```

验证安装：

```shell
$ kata-runtime --version
kata-runtime  : 1.1.0
   commit   : bf1cf68
   OCI specs: 1.0.1
```

## 与 Docker 集成

本文假设你已经安装了 docker。配置 docker，增加对 kata runtime 的支持。

```shell
dir=/etc/systemd/system/docker.service.d
file="$dir/kata-containers.conf"
mkdir -p "$dir"
test -e "$file" || echo -e "[Service]\nType=simple\nExecStart=\nExecStart=/usr/bin/dockerd -D --default-runtime runc" | sudo tee "$file"
grep -q "kata-runtime=" $file || sudo sed -i 's!^\(ExecStart=[^$].*$\)!\1 --add-runtime kata-runtime=/usr/bin/kata-runtime!g' "$file"
systemctl daemon-reload
systemctl restart docker
```

Docker 默认使用 runc 创建容器，可以使用 `--runtime` 指定使用其他的 runtime。下面我们创建两个容器，一个使用 kata runtime，另一个使用默认的 runc。创建完容器之后，我们可以查看一下 kata container 的一些信息。
```shell
$ docker run -d --name kata-test -p 8081:8080 --runtime kata-runtime lingxiankong/alpine-test
3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43
$ curl http://localhost:8081
3e899cb1a693
$ docker run -d --name runc-test -p 8082:8080 lingxiankong/alpine-test
4a68584ffd580f8395f80505c22bd131c387a49b977584dcbb211b3942b37642
root@test-kata [~]
$ curl http://localhost:8082
4a68584ffd58
$ docker ps
CONTAINER ID        IMAGE                      COMMAND               CREATED             STATUS              PORTS                    NAMES
4a68584ffd58        lingxiankong/alpine-test   "python /server.py"   2 minutes ago       Up 2 minutes        0.0.0.0:8082->8080/tcp   runc-test
3e899cb1a693        lingxiankong/alpine-test   "python /server.py"   3 minutes ago       Up 3 minutes        0.0.0.0:8081->8080/tcp   kata-test
$ ps -ef | grep docker | grep runtime | grep -v dockerd
root     17587 14084  0 04:00 ?        00:00:00 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-kata-runtime -debug
root     17795 14084  0 04:02 ?        00:00:00 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/4a68584ffd580f8395f80505c22bd131c387a49b977584dcbb211b3942b37642 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc -debug
$ kata-runtime list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                               CREATED                          OWNER
3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43   17668       running     /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43   2018-07-16T04:00:44.547122236Z   #0
$ ps -ef | grep qemu | grep -v 'grep'
root     17637 17587  0 04:00 ?        00:00:08 /usr/bin/qemu-lite-system-x86_64 -name sandbox-3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43 -uuid aebc12d8-4364-42a3-87a7-10ea669a85c9 -machine pc,accel=kvm,kernel_irqchip,nvdimm -cpu host,pmu=off -qmp unix:/run/vc/sbs/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43/qmp.sock,server,nowait -m 2048M,slots=2,maxmem=9006M -device pci-bridge,bus=pci.0,id=pci-bridge-0,chassis_nr=1,shpc=on,addr=2 -device virtio-serial-pci,disable-modern=true,id=serial0 -device virtconsole,chardev=charconsole0,id=console0 -chardev socket,id=charconsole0,path=/run/vc/sbs/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43/console.sock,server,nowait -device nvdimm,id=nv0,memdev=mem0 -object memory-backend-file,id=mem0,mem-path=/usr/share/kata-containers/kata-containers-image_clearlinux_agent_7b458b1.img,size=536870912 -device virtio-scsi-pci,id=scsi0,disable-modern=true -device virtserialport,chardev=charch0,id=channel0,name=agent.channel.0 -chardev socket,id=charch0,path=/run/vc/sbs/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43/kata.sock,server,nowait -device virtio-9p-pci,disable-modern=true,fsdev=extra-9p-kataShared,mount_tag=kataShared -fsdev local,id=extra-9p-kataShared,path=/run/kata-containers/shared/sandboxes/3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43,security_model=none -netdev tap,id=network-0,vhost=on,vhostfds=3:4:5:6:7:8:9:10,fds=11:12:13:14:15:16:17:18 -device driver=virtio-net-pci,netdev=network-0,mac=02:42:ac:11:00:02,disable-modern=true,mq=on,vectors=18 -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -vga none -no-user-config -nodefaults -nographic -daemonize -kernel /usr/share/kata-containers/vmlinuz-4.14.51.1-132.container -append tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 iommu=off cryptomgr.notests net.ifnames=0 pci=lastbus=0 root=/dev/pmem0p1 rootflags=dax,data=ordered,errors=remount-ro rw rootfstype=ext4 quiet systemd.show_status=false panic=1 initcall_debug nr_cpus=8 ip=::::::3e899cb1a6931e7c1cd83cd7dd837a2ba1726ded789f3bc1c3debdeb3fa37b43::off:: init=/usr/lib/systemd/systemd systemd.unit=kata-containers.target systemd.mask=systemd-networkd.service systemd.mask=systemd-networkd.socket -smp 1,cores=1,threads=1,sockets=1,maxcpus=8
```

## 与 K8S 集成
![](/images/2018-07-20-katacontainer-docker-k8s/katacontainer_arch.png)

实际上，kata 不直接与 k8s 通信，因为对于 k8s 来说，它只跟实现了 CRI 接口的容器管理进程打交道，比如 docker-engine，rkt, containerd(使用 cri plugin) 或 CRI-O，而 kata 跟 runc 是同一个级别的进程。所以如果要与 k8s 集成，则需要安装 CRI-O 或 CRI-containerd 来支持 CRI 接口，本文使用 CRI-O。CRI-O 的 O 的意思是 OCI-compatible，即 CRI-O 是实现 CRI 接口来跑 OCI 容器的。

> [这里](https://www.katacoda.com/courses/kubernetes/getting-started-with-kubeadm-crio) 有关于如何使用 kubeadm 安装使用 CRI-O 的 k8s 集群的教程。

### 安装配置 CRI-O
```shell
# Install runc，因为后面我们不用 docker 跟 k8s 通信而是用 cri-o，但为了能够同时支持普通容器和 kata 容器，要单独安装 runc
wget https://github.com/opencontainers/runc/releases/download/v1.0.0-rc4/runc.amd64
chmod +x runc.amd64
mv runc.amd64 /usr/bin/runc
$ runc -version
runc version 1.0.0-rc4
spec: 1.0.0

# Install crictl CLI, 假设你的系统已经安装了 golang
go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
pushd $GOPATH/src/github.com/kubernetes-incubator/cri-tools
make && make install
popd

# Build and install cri-o
apt-get install -y libglib2.0-dev libseccomp-dev libgpgme11-dev libdevmapper-dev make git
go get -d github.com/kubernetes-incubator/cri-o || true
pushd $GOPATH/src/github.com/kubernetes-incubator/cri-o
make install.tools && make && make install
# 生成默认的配置文件在 /etc/crio/crio.conf
make install.config
popd

# Config crio systemd service
cat <<EOF > /etc/systemd/system/crio.service
[Unit]
Description=OCI-based implementation of Kubernetes Container Runtime Interface
Documentation=https://github.com/kubernetes-incubator/cri-o

[Service]
ExecStart=/usr/local/bin/crio --log-level debug
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload; systemctl enable crio; systemctl start crio
export CONTAINER_RUNTIME_ENDPOINT=unix:///var/run/crio/crio.sock
$ crictl version
Version:  0.1.0
RuntimeName:  cri-o
RuntimeVersion:  1.12.0-dev
RuntimeApiVersion:  v1alpha1

# 配置 crio 使用 kata-runtime 作为默认的 untrusted workload
sed -i '/\[crio.runtime\]/ a manage_network_ns_lifecycle = true' /etc/crio/crio.conf
sed -i '/runtime_untrusted_workload =/c runtime_untrusted_workload = "/usr/bin/kata-runtime"' /etc/crio/crio.conf
sed -i '/default_workload_trust =/c default_workload_trust = "trusted"' /etc/crio/crio.conf
sed -i '/#registries =/c registries = ["docker.io"]' /etc/crio/crio.conf
systemctl restart crio; systemctl status crio

# Build and install CNI plugin，但我们不准备使用默认的这些 CNI plugins，为了支持 Network Policy，我们后面会安装 Calico
go get -d github.com/containernetworking/plugins || true
pushd $GOPATH/src/github.com/containernetworking/plugins
./build.sh
mkdir -p /opt/cni/bin && cp bin/* /opt/cni/bin/ && mkdir -p /etc/cni/net.d
popd
add-apt-repository -y ppa:projectatomic/ppa && apt-get update && apt-get install -y skopeo-containers
systemctl restart crio; systemctl status crio
```

### 初始化 K8S 集群
假设你已经在 host 上安装了 kubelet(但还没有开始安装整个 k8s 集群)，前几个步骤安装配置了 runc, cri-o 和 cni，接下来就需要告诉 kubelet 使用 cri-o (而不是 docker) 创建 pod 中的容器。
```shell
cat <<EOF > /etc/systemd/system/kubelet.service.d/0-crio.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///var/run/crio/crio.sock"
EOF
systemctl daemon-reload; systemctl restart kubelet
```

准备初始化 k8s：
```shell
# kubeadm init 需要的配置文件
cat <<EOF > $HOME/kubeadm.conf
kind: MasterConfiguration
apiVersion: kubeadm.k8s.io/v1alpha1
cloudProvider: openstack
kubernetesVersion: 1.10.0
unifiedControlPlaneImage: "gcr.io/google_containers/hyperkube-amd64:v1.10.0"
networking:
  podSubnet: "192.168.0.0/16"
nodeRegistration:
  criSocket: /var/run/crio/crio.sock
EOF

# 因为我的 k8s 依然是在 openstack 环境下安装，所以需要 cloud-provider 配置文件。这个跟 kata 没关系
cat <<EOF > /etc/kubernetes/cloud-config
[Global]
auth-url=http://10.52.0.123/identity
user-id=74a76055a14c469690dc4416143ea32e
password=password
tenant-id=e3c2afbd44bb4484807bc5b02b75cbb1
region=RegionOne
[LoadBalancer]
use-octavia=yes
subnet-id=685e649c-0ba2-4c58-b3d6-9dae3f4e3ebe
create-monitor=no
[BlockStorage]
bs-version=v2
EOF

sed -i -E 's/(.*)KUBELET_KUBECONFIG_ARGS=(.*)$/\1KUBELET_KUBECONFIG_ARGS=--cloud-provider=openstack --cloud-config=\/etc\/kubernetes\/cloud-config \2/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload; systemctl restart kubelet

kubeadm init --config $HOME/kubeadm.conf
```

`kubeadm init` 命令执行成功后，因为系统中没有配置 CNI plugin 所以你会发现 dns pod 一直是 pending。下面安装 Calico，Calico 安装完后一定要重启 cri-o 和 kubelet 服务。此外，目前我们只有一个 master 节点，为了实验目的，我不打算再添加 worker 节点，我们只要允许 master 上可以创建 pod 就行：

```shell
mkdir -p $HOME/.kube; cp -i /etc/kubernetes/admin.conf $HOME/.kube/config; chown $(id -u):$(id -g) $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/master-
curl -sSO https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
sed -i 's/value: "always"/value: "off"/' calico.yaml
sed -i '/securityContext/ i \ \ \ \ \ \ \ \ \ \ \ \ - name: CALICO_IPV4POOL_NAT_OUTGOING\n\ \ \ \ \ \ \ \ \ \ \ \ \ \ value: "true"' calico.yaml
kubectl apply -f calico.yaml
systemctl restart crio; systemctl restart kubelet
```

 等待一段时间，知道使用 kubectl 命令查看系统 pod 全部 running：

```shell
$ kubectl get po -n kube-system
NAME                                              READY     STATUS    RESTARTS   AGE
calico-etcd-rhkbs                                 1/1       Running   1          15m
calico-kube-controllers-5977fdb8c4-zh45f          1/1       Running   0          15m
calico-node-92vvq                                 2/2       Running   2          15m
etcd-lingxiantest-k8s-master                      1/1       Running   0          10m
kube-apiserver-lingxiantest-k8s-master            1/1       Running   0          10m
kube-controller-manager-lingxiantest-k8s-master   1/1       Running   1          11m
kube-dns-86f4d74b45-hkx7r                         3/3       Running   0          29m
kube-proxy-6qnrd                                  1/1       Running   0          29m
kube-scheduler-lingxiantest-k8s-master            1/1       Running   0          10m
```

可以使用 `crictl` 查看系统创建的镜像和容器，因为我们没有用 docker，所以不能用以前常用的 docker 命令了。虽然不能用 docker，但可以使用 `runc` 命令查看容器

```shell
$ crictl images
IMAGE                                      TAG                 IMAGE ID            SIZE
gcr.io/google_containers/hyperkube-amd64   v1.10.0             79f28a9f288b9       641MB
k8s.gcr.io/etcd-amd64                      3.1.12              52920ad46f5bf       193MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64     1.14.8              c2ce1ffb51ed6       41.2MB
k8s.gcr.io/k8s-dns-kube-dns-amd64          1.14.8              80cc5ea4b547a       50.7MB
k8s.gcr.io/k8s-dns-sidecar-amd64           1.14.8              6f7f2dc7fab5d       42.5MB
k8s.gcr.io/pause                           3.1                 da86e6ba6ca19       747kB
quay.io/calico/cni                         v1.11.6             5098a69b5856b       71.1MB
quay.io/calico/kube-controllers            v1.0.4              39fe3feb1b008       52.5MB
quay.io/calico/node                        v2.6.10             3b2408e59c950       281MB
quay.io/coreos/etcd                        v3.1.10             47bb9dd99916e       34.8MB

$ crictl ps -o table
CONTAINER ID        IMAGE                                                                                                            CREATED             STATE               NAME                      ATTEMPT
051f40fdc9416       k8s.gcr.io/k8s-dns-sidecar-amd64@sha256:23df717980b4aa08d2da6c4cfa327f1b730d92ec9cf740959d2d5911830d82fb         7 minutes ago       Running             sidecar                   0
7f6c8d8763162       k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64@sha256:93c827f018cf3322f1ff2aa80324a0306048b0a69bc274e423071fb0d2d29d8b   8 minutes ago       Running             dnsmasq                   0
9c8589f176f79       quay.io/calico/kube-controllers@sha256:87288284b11a5f94453fe705d64285b333b4b16750336c3b843c3f109d05fd13          8 minutes ago       Running             calico-kube-controllers   0
c1b3c59077e0f       k8s.gcr.io/k8s-dns-kube-dns-amd64@sha256:6d8e0da4fb46e9ea2034a3f4cab0e095618a2ead78720c12e791342738e5f85d        8 minutes ago       Running             kubedns                   0
a413e36349a45       5098a69b5856b86f7784fe257c0121883fe511f540efadb283f221e5080964cb                                                 9 minutes ago       Running             install-cni               1
8f766a34928ab       3b2408e59c9508d9271d477073ca3cb31e131b4218a1233cf2e7235f8af1a02d                                                 9 minutes ago       Running             calico-node               1
163af5b812227       47bb9dd99916eb8402af6039f66bcbc6f73fdc0db0d507dd320eb8649d88a250                                                 11 minutes ago      Running             calico-etcd               1
f152ff80be13c       79f28a9f288b9572a1a92b4724bc26eb9a66c70b92631565e5b27d93c2ad6207                                                 27 minutes ago      Running             kube-proxy                0
69cae7206a6bb       79f28a9f288b9572a1a92b4724bc26eb9a66c70b92631565e5b27d93c2ad6207                                                 28 minutes ago      Running             kube-controller-manager   1
ab4a5edb4c881       k8s.gcr.io/etcd-amd64@sha256:68235934469f3bc58917bcf7018bf0d3b72129e6303b0bef28186d96b2259317                    29 minutes ago      Running             etcd                      0
4e7c50b342e58       79f28a9f288b9572a1a92b4724bc26eb9a66c70b92631565e5b27d93c2ad6207                                                 29 minutes ago      Running             kube-apiserver            0
f078f475c9fd6       79f28a9f288b9572a1a92b4724bc26eb9a66c70b92631565e5b27d93c2ad6207                                                 29 minutes ago      Running             kube-scheduler            0

# 使用 runc 查看容器的输出比较多，这里我们仅关心 runc 看到的容器的个数，以便后续测试对比
$ runc list | wc -l
26
```

因为我们配置 cri 默认使用 runc 创建容器，所以使用 `kata-runtime` 命令应该看不到任何容器：

```shell
$ kata-runtime list
ID          PID         STATUS      BUNDLE      CREATED     OWNER
```

### 测试 pod 操作

为了创建 kata 容器，需要指定 pod 的 annotation： `io.kubernetes.cri-o.TrustedSandbox: "false"`，如果不定义这个 annotation 或配置为 true，则 cri-o 会使用 runc 创建。
```shell
cat <<EOF | kc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-kata
  labels:
    app: kata
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kata
  template:
    metadata:
      labels:
        app: kata
      annotations:
        io.kubernetes.cri-o.TrustedSandbox: "false"
    spec:
      containers:
      - name: test-kata
        image: lingxiankong/alpine-test
        ports:
        - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: kata
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF

$ kubectl get service,deploy
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   2h
service/my-service   ClusterIP   10.110.68.24   <none>        80/TCP    2h

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/test-kata   2         2         2            2           2h

# 此时用 kata-runtime 命令可以看到有新容器创建
$ kata-runtime list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                 CREATED                          OWNER
0508f05b486a15806b79825ca532573307c2a97fa80ad064f9afd77a65e23ebd   28499       running     /run/containers/storage/overlay-containers/0508f05b486a15806b79825ca532573307c2a97fa80ad064f9afd77a65e23ebd/userdata   2018-07-20T00:19:26.38690234Z    #0
5d623ed4d1f620e4c1b6795131a9e4e583010475e09654097226af0e1781c816   28987       running     /run/containers/storage/overlay-containers/5d623ed4d1f620e4c1b6795131a9e4e583010475e09654097226af0e1781c816/userdata   2018-07-20T00:20:12.125605903Z   #0
02c07bfe8c9102a531e75309daf612d7a754b21365b55627d66becd084d9c37f   28490       running     /run/containers/storage/overlay-containers/02c07bfe8c9102a531e75309daf612d7a754b21365b55627d66becd084d9c37f/userdata   2018-07-20T00:19:26.369408225Z   #0
2b0afc87a21aa37c345b9455ab1a434f4e7a7d9936b0aa89362273d03415ca6a   28821       running     /run/containers/storage/overlay-containers/2b0afc87a21aa37c345b9455ab1a434f4e7a7d9936b0aa89362273d03415ca6a/userdata   2018-07-20T00:19:57.995200391Z   #0

# 再次查看 runc 容器的个数，与之前相同，说明新创建的容器并没有使用 runc
$ runc list | wc -l
26

# 直接在 master 节点访问创建的 service
$ curl 10.110.68.24
test-kata-5f9944675-m8z96
```
### 测试 Network Policy

虽然现在已经能够调用 kata 创建 pod，但我最担心的还是 kata 跟网络或存储的集成。所以再测试一下安装 calico 后的 network policy 的功能。

默认情况下，系统中所有 pod 是互通的，所以我们先创建一个普通 pod(使用 runc)，在 pod 里面访问之前创建的 service：

```shell
$ kubectl run curl-client --rm -it --image=tutum/curl --restart=Never
If you don't see a command prompt, try pressing enter.
root@curl-client:/# curl 10.110.68.24
test-kata-5f9944675-m8z96
root@curl-client:/# curl 192.168.36.66:8080
test-kata-5f9944675-m8z96
root@curl-client:/# curl 192.168.36.65:8080
test-kata-5f9944675-vmzqp
```

其中 `192.168.36.66` 和 `192.168.36.65` 是之前创建的 deployment 中的两个 pod 的 IP 地址。这里虽然我们使用不同的 runtime 创建容器，但默认情况，网络是互通的。

创建 network policy 禁用 pod 之间的网络：

```shell
cat <<EOF | kubectl apply -f -
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

这时在 curl-client pod 里已经不能访问 service 和 pod 了，说明 network policy 生效。

再创建一个 network policy 允许 curl-client pod 访问：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-curl-client
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: kata
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: curl-client
EOF
```

然后你会发现 curl-client pod 里又可以成功访问 service 和 pod 了。

## 参考文档

- <https://github.com/kata-containers/documentation/blob/master/Developer-Guide.md>
- <https://github.com/kata-containers/documentation/blob/master/architecture.md>
- <https://github.com/kubernetes-incubator/cri-o/blob/master/tutorial.md>