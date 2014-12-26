---
layout: post
title: 浅析vagrant和docker
description: 浅析vagrant和docker
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

---

什么是 vagrant ? Vagrant 是一个跨平台的虚拟机构建工具，能够通过 vagrantfile 描述虚拟机并将其部署到 hypervisor 上（VirtualBox, VMWare, AWS, etc）。

什么是 docker ? Docker 是一个 linux 上的 linux container 构建工具，能够通过 dockerfile 来定义一个 container ，并将其部署到任何运行 docker 的主机上。

Vagrant 和 docker 都能够通过一个配置描述文件来构造一个运行环境。

再来看 vagrant 和 docker 的一些差异：  
![](/images/2014-09-29-vagrant-docker/1.png)

docker其他的优势：

* 轻量级的隔离环境比虚拟机能够更方便和快捷地启动和停止；
* 可以在一个虚拟机中运行多个 container，从而节省开销；
* Docker 的 container 机制更适合一些持续集成/持续发布和微型 PaaS 场景。

正是因为两者相比差异颇多，具体用哪一个需要结合特定的使用场景，不能一概而论。但虚拟化爱好者似乎不愿意看到两个宝贝争的你死我活，于是一些人将两者的优势结合，想了一种同时使用两者的使用场景。一个典型场景如下（摘自[这里](https://medium.com/@_marcos_otero/docker-vs-vagrant-582135beb623)）：

* Install a Vagrant virtual machine in your computer containing the same OS you will have in your server ( normally Ubuntu Linux 12.04 LTS 64 bits). This means that you can program in any OS you want and still expect your program will run in your server.
* Install your Docker packages to create Docker containers inside your virtual machine created for Vagrant. This step is better if you can install them through an script.
* Inside your containers put your applications ( Nginx, Memcached, MongoDB, etc)
* Configure a shell script, Puppet or Chef script to install Docker and run your Docker containers each time Vagrant begins.
* Test your containers in your Vagrant VM inside your computer.
* Thanks to providers now you can take the same file ( your Vagrant file ) and just type vagrant up —provider=“provider” where the provider is your next host and Vagrant will take care of everything. For example, if you choose AWS then Vagrant will: Connect to your AMI in AWS, install the same OS you used in your computer, install Docker, launch your Docker containers and give you a ssh session.
* Test your containers in AWS and look that they behave exactly as you expect.

有人在KVM上的OpenStack以及Docker和LXC进行了基准测试，发现Docker的性能或者比KVM出众很多或者可以与KVM相媲美。测试结果让他认为今后传统的虚拟机将会被边缘化。尽管某些企业可能会使用Docker取代现有的虚拟化技术，但更可能出现的情况是像上面的场景一样，企业将会使用Docker来扩大现有的虚拟化规模。例如，企业可能会同时运行Docker和Vmware环境，或者在VMware 虚拟机内部署Docker容器来确保管理的一致性。企业对Docker的敏捷性感兴趣，但真正感兴趣的是压缩数据中心的规模并减少许可费用。所以，在单个虚拟机内可以运行10个Linux容器，如果你有10台虚拟机，那么现在就拥有了100个容器。

但Docker以及容器的健壮性是其缺陷，尤其是在异构企业环境中更是如此。Docker开销低以及存储占用空间少源于其并没有为所有被封装的应用提供操作系统副本。然而这使得Docker仅限于支持LXC的Linux实例，换句话说，如果你想使用Docker封装Windows应用，那只能说声抱歉了。传统的虚拟化还提供了一些引人注目的用例，比如LXC所不支持的实例，以及虚拟机需要使用与该物理主机上其他虚拟机完全不同的内核设置。所以说尽管Docker性能不错、易于使用而且在很大程度上是免费的，但并不适合所有用户。

看完了大概的特性差异对比，我们来实践一下两者的使用。这里我以使用OpenStack为例，因为每个参与OpenStack的开发者都需要一套开发环境devstack，那么对于开发环境的分发就可以基于vagrant或docker去实现。

## vagrant下安装devstack

### 安装vagrant
vagrant的几个概念：  
box： 基础镜像，类似于编程中的类；  
project：使用镜像创建出的VM，类似于基于类创建对象；  
同一个box可以被不同的project使用。

因为我们使用virtualBox作为vagrant的虚拟化实现工具（你也可以使用vmware），所以要先安装 virtualBox，参考文档<https://help.ubuntu.com/community/VirtualBox/Installation>。

    sudo sh -c "echo 'deb http://download.virtualbox.org/virtualbox/debian '$(lsb_release -cs)' contrib non-free' > /etc/apt/sources.list.d/virtualbox.list" && wget -q http://download.virtualbox.org/virtualbox/debian/oracle_vbox.asc -O- | sudo apt-key add - && sudo apt-get update && sudo apt-get install virtualbox-4.3 dkms

下载 ubuntu下的vagrant deb包（链接：<http://www.vagrantup.com/downloads>），放在某一目录下（比如`/var/kong/vagrant_1.6.5_x86_64.deb`）， 然后进行安装。   
![](/images/2014-09-29-vagrant-docker/2.png)

安装vagrant插件（可选）   
![](/images/2014-09-29-vagrant-docker/3.png)

### 安装ubuntu
在/var/vagrant目录下clone vagrant安装devstack的工程，链接：<https://git.openstack.org/openstack-dev/devstack-vagrant>

但这个工程需要设置一些东西，想了想，还是自己一步一步在vagrant虚拟机里安装devstack吧。

    root@ubuntu:/var/openstack# mkdir /var/vagrant/devstack
    root@ubuntu:/var/openstack# cd /var/vagrant/devstack
    root@ubuntu:/var/vagrant/devstack# vagrant init
    A `Vagrantfile` has been placed in this directory. You are now
    ready to `vagrant up` your first virtual environment! Please read
    the comments in the Vagrantfile as well as documentation on
    `vagrantup.com` for more information on using Vagrant.
    root@ubuntu:/var/vagrant/devstack# ll
    total 16
    drwxr-xr-x 2 root root 4096 Sep 30 09:33 ./
    drwxr-xr-x 4 root root 4096 Sep 30 09:33 ../
    -rw-r--r-- 1 root root 4814 Sep 30 09:33 Vagrantfile

下载ubuntu vagrant box，地址：<http://files.vagrantup.com/precise64.box>，下载precise64.box文件，放在某个目录下（我这里是/var/kong），其他box在[这里](http://www.vagrantbox.es/)可以找到。

    vagrant box add Ubuntu12.04x64 /var/kong/precise64.box 

命令运行之后，可以在`~/.vagrant.d/boxes/`目录下看到新增的box。之后，编辑/var/vagrant/devstack目录下的Vagrantfile：

    config.vm.box = "Ubuntu12.04x64"
    config.vm.network "private_network", ip: "192.168.33.10"
    config.vm.provider "virtualbox" do |vb|
      vb.customize ["modifyvm", :id, "--memory", "8192"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
    end

启动vagrant虚拟机。  
![](/images/2014-09-29-vagrant-docker/4.png)

虚拟机启动后，可以通过virtualBox命令验证：

    root@ubuntu:/var/vagrant/devstack# VBoxManage list runningvms
    "devstack_default_1412056881390_39322" {c3f2fecc-1316-4823-b870-659dccc4e251}

也可以查询更详细的信息：

    root@ubuntu:/var/vagrant/devstack# VBoxManage showvminfo c3f2fecc-1316-4823-b870-659dccc4e251
    Name:            devstack_default_1412056881390_39322
    Groups:          /
    Guest OS:        Ubuntu (64 bit)
    UUID:            c3f2fecc-1316-4823-b870-659dccc4e251
    Config file:     /root/VirtualBox VMs/devstack_default_1412056881390_39322/devstack_default_1412056881390_39322.vbox
    Snapshot folder: /root/VirtualBox VMs/devstack_default_1412056881390_39322/Snapshots
    Log folder:      /root/VirtualBox VMs/devstack_default_1412056881390_39322/Logs
    Hardware UUID:   c3f2fecc-1316-4823-b870-659dccc4e251
    Memory size:     8192MB
    Page Fusion:     off
    VRAM size:       8MB
    CPU exec cap:    100%
    HPET:            off
    Chipset:         piix3
    Firmware:        BIOS
    Number of CPUs:  2
    PAE:             on
    Long Mode:       on
    Synthetic CPU:   off
    CPUID overrides: None
    Boot menu mode:  message and menu
    Boot Device (1): HardDisk
    Boot Device (2): DVD
    Boot Device (3): Not Assigned
    Boot Device (4): Not Assigned
    ACPI:            on
    IOAPIC:          on
    Time offset:     0ms
    RTC:             UTC
    Hardw. virt.ext: on
    Nested Paging:   on
    Large Pages:     on
    VT-x VPID:       on
    VT-x unr. exec.: on
    State:           running (since 2014-09-30T06:01:24.581000000)
    Monitor count:   1
    3D Acceleration: off
    2D Video Acceleration: off
    Teleporter Enabled: off
    Teleporter Port: 0
    Teleporter Address:
    Teleporter Password:
    Tracing Enabled: off
    Allow Tracing to Access VM: off
    Tracing Configuration:
    Autostart Enabled: off
    Autostart Delay: 0
    Default Frontend:
    Storage Controller Name (0):            IDE Controller
    Storage Controller Type (0):            PIIX4
    Storage Controller Instance Number (0): 0
    Storage Controller Max Port Count (0):  2
    Storage Controller Port Count (0):      2
    Storage Controller Bootable (0):        on
    Storage Controller Name (1):            SATA Controller
    Storage Controller Type (1):            IntelAhci
    Storage Controller Instance Number (1): 0
    Storage Controller Max Port Count (1):  30
    Storage Controller Port Count (1):      1
    Storage Controller Bootable (1):        on
    IDE Controller (0, 0): Empty
    IDE Controller (1, 0): Empty
    SATA Controller (0, 0): /root/VirtualBox VMs/devstack_default_1412056881390_39322/box-disk1.vmdk (UUID: e9c4252c-d1b4-42be-acfa-dbc29dd9ddc4)
    NIC 1:           MAC: 080027880CA6, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
    NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
    NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = 127.0.0.1, host port = 2222, guest ip = , guest port = 22
    NIC 2:           MAC: 080027EE8044, Attachment: Host-only Interface 'vboxnet0', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
    NIC 3:           disabled
    NIC 4:           disabled
    NIC 5:           disabled
    NIC 6:           disabled
    NIC 7:           disabled
    NIC 8:           disabled
    Pointing Device: PS/2 Mouse
    Keyboard Device: PS/2 Keyboard
    UART 1:          disabled
    UART 2:          disabled
    LPT 1:           disabled
    LPT 2:           disabled
    Audio:           disabled
    Clipboard Mode:  disabled
    Drag'n'drop Mode: disabled
    Session type:    headless
    Video mode:      640x480x32 at 0,0
    VRDE:            disabled
    USB:             disabled
    EHCI:            disabled
    
    USB Device Filters:
    
    <none>
    
    Available remote USB devices:
    
    <none>
    
    Currently Attached USB Devices:
    
    <none>
    
    Bandwidth groups:  <none>
    
    Shared folders: 
    
    Name: 'vagrant', Host path: '/var/vagrant/devstack' (machine mapping), writable
    
    VRDE Connection:    not active
    Clients so far:     0
    
    Video capturing:    not active
    Capture screens:    0
    Capture file:       /root/VirtualBox VMs/devstack_default_1412056881390_39322/devstack_default_1412056881390_39322.webm
    Capture dimensions: 1024x768
    Capture rate:       512 kbps
    Capture FPS:        25
    
    Guest:
    
    Configured memory balloon size:      0 MB
    OS type:                             Linux26_64
    Additions run level:                 2
    Additions version:                   4.2.0 r80737
    
    
    Guest Facilities:
    
    Facility "VirtualBox Base Driver": active/running (last update: 2014/09/30 06:01:32 UTC)
    Facility "VirtualBox System Service": active/running (last update: 2014/09/30 06:01:35 UTC)
    Facility "Seamless Mode": not active (last update: 2014/09/30 06:01:32 UTC)
    Facility "Graphics Mode": not active (last update: 2014/09/30 06:01:32 UTC)
    
> VirtualBox命令行工具VBoxManage的一些使用。  
> list vms，可以加-l参数显示详细信息。   
> 启动虚拟机：VBoxManage startvm "slackware"   
> 虚拟机其他action：VBoxManage controlvm "slackware" pause/resume/reset/poweroff/savestate   
> 修改虚拟机配置（<http://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm>）。VBoxManage modifyvm "winxp" -memory "256MB" -acpi on -boot1 dvd -nic1 nat    
> 创建一个虚拟磁盘. VBoxManage createhd -filename "WinXP.vdi" -size 10000 –register   
> 将虚拟磁盘和虚拟机关联. VBoxManage modifyvm "winxp" -hda "WinXP.vdi"   
> 挂载光盘镜像 ISO. VBoxManage openmedium dvd /full/path/to/iso.iso   
> 将光盘镜像 ISO 和虚拟机关联. VBoxManage modifyvm "winxp" -dvd /full/path/to/iso.iso   
> 创建虚拟机. VBoxManage createvm -name "SUSE 10.2" -register

通过vagrant ssh命令登录虚拟机，切换root账户， 查看系统挂载目录，新建文件，然后在宿主机上验证一下，最后，再确定一下网络是否OK。  
![](/images/2014-09-29-vagrant-docker/5.png)
 
查看宿主机上的/var/vagrant/devstack目录：  
![](/images/2014-09-29-vagrant-docker/6.png)

如果虚拟机在运行时修改了配置，可以通过`vagrant reload --provision`使用最新的配置重启虚拟机。

一个很牛逼的特性，vagrant share，参考[这里](http://docs.vagrantup.com/v2/getting-started/share.html)

### 安装DevStack
有个虚拟机，虚拟机又能联网，那么安装devstack就是老话题了。  
参考我之前的[这篇](http://lingxiankong.github.io/blog/2014/05/10/vmware-workstation-devstack/)文章。

或者使用别人做好的box/Vagrantfile，这里有一个可以参考（我没试用过）：<https://github.com/patux/mydevstack>

### 如何共享
一个vagrant下的devstack虚拟机有了，如果只是自己用，那没有必要非要使用vagrant，直接用VirtualBox就行了。所以，分享、协作才是体现vagrant价值的真谛。如何分享呢？

首先想到的就是刚才提到的vagrant share，但大部分人还是希望在一个小团队中自由分发，而不是直接暴漏在公网上。其实，直接使用vagrant package命令就可以重新打包box：  
![](/images/2014-09-29-vagrant-docker/8.png)

## Docker下安装devstack
### 安装Docker
参考[这篇](https://docs.docker.com/installation/ubuntulinux/)文章。

docker中的基本概念：  
image：类似于vagrant中的box；  
container：类似于vagrant中的VM；

### Docker的基本使用
有时使用官方的镜像速度比较慢，`docker pull ubuntu`的命令可以替换为：

	#第一种方法url下载安装ubuntu
	docker import http://docker.widuu.com/ubuntu.tar 
	#第二种方法，下载下来然后根据自己配置安装
	wget http://docker.widuu.com/ubuntu.tar
	cat test.tar | sudo docker import - xiaowei:new

运行交互式的shell：`docker run -i -t ubuntu /bin/bash`，退出可以使用CTRL -p+CTRL -q或输入exit

开启一个长时间运行的工作进程：  

	# 开启一个非常有用的长时间工作进程
	CONTAINER_ID=$(sudo docker run -d ubuntu:14.04 /bin/sh -c "while true; do echo Hello world; sleep 1; done")
	# 到目前为止的收集的输出，可以加-f达到类似于tail -f的效果
	sudo docker logs $CONTAINER_ID
	# 或者连接上容器实时查看
	sudo docker attach $CONTAINER_ID

docker ps命令：  

	sudo docker ps，列出当前所有正在运行的container
	sudo docker ps -l，列出最近一次启动的，且正在运行的container
	sudo docker ps -a，列出所有的container

绑定ports：`docker run -d -p 5000:5000 training/webapp python app.py`  
或者通过`docker port $CONTAINER_ID 5000`查询container port 5000绑定的external port。

查看container中的进程：`docker top $CONTAINER_ID`

创建images：  
1. `docker commit -m "update and install puppet" -a "kong" 9cf3b285b7e4 kong/ubuntu:v1`  
2. 编写Dockerfile

### 安装Devstack
与vagrant一样，装完docker，首先想到的是到docker image repo（官方叫docker hub）找与devstack相关的image。直接到<https://registry.hub.docker.com>，搜索“devstack”（或者通过命令行`docker search devstack`也能搜索出来），有三个结果：  
![](/images/2014-09-29-vagrant-docker/7.png)  
看了下三个image的描述，感觉都不靠谱。

关于Docker与OpenStack快速部署，也可以参见[这篇](http://allthingsopen.com/2014/07/09/docker-all-the-things-openstack-keystone-and-ceilometer-automated-builds-on-docker-hub)博客，作者仅以Keystone和Ceilometer为例介绍了如何使用Docker快速部署OpenStack服务。

未完待续。
