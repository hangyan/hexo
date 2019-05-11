---

title: CoreOS 安装及配置
share: true
comments: true
imagefeature:
tags: [技术]
excerpt: ""
categories: 技术
thumb: /images/thumbs/coreos.png
---


本文所遵照的步骤是官网的 Installing to disk 方法,即刻录 ISO 镜像 -> 启动 Coreos Live CD -> 安装到硬盘的步骤，与一般的桌面 Linux 安装非常类似。但 coreos 安装时也有一些需要注意的地方:

1. cloud-config.yml

    这是 coreos 用来统一配置系统的地方，系统在每次启动时都会加载这个文件的配置，比
    如系统服务、网络设定、文件修改、用户设定等。这样做的好处是在部署集群的时候可
    以方便地使用相同的配置。在安装 coreos 时，需要指定好这个配置文件。实际操作时，
    可以提前将这个文件写好放在别的机器上，然后用 scp / wget (利用下面的 web server) 下载到 coreos 的 Live CD 即可，或者直接存在 Live CD 里更方便。
2. GFW

    coreos 安装时需要从官网下载镜像，但网站被墙，所以实际安装的时候可能需要用代理来解决，缺点是速度慢。更方便的方法是提前下载好需要的文件并放在局域网内并搭建一个 web server，然后修改安装脚本的 server 即可。具体方法在后面详述。
<!--more-->

### 刻录 ISO 镜像

 1. 下载 ISO :  [Go to Download Page](https://coreos.com/docs/running-coreos/platforms/iso/)
 2. 刻录 : U 盘的话在 linux 上直接用 dd 命令即可。
    
### 安装

#### **网络配置**
因为安装过程中需要联网，所以需要配置 Live CD 内部的 coreos 系统的网络。coreos 使用
systemd 来管理系统服务，所以写一个 unit file 并启动即可。

在 `/etc/systemd/network` 目录下创建 `static.network`文件,示例内容如下:

    [Match]
    Name=eno1

    [Network]
    DNS=114.114.114.114
    Address=192.168.200.12/24
    Gateway=192.168.200.11

`Name`为网卡名，`Address`为静态 IP 地址,依自己情况设置即可。

执行如下命令生效:

	sudo systemctl restart systemd-networkd


#### **cloud-config** 
一个基本的 cloud-config.yml 示例文件如下：

    #cloud-config

    hostname: coreos-12    
    manage_etc_hosts: localhost

    coreos:
      etcd:
        discovery: https://discovery.etcd.io/<your-own-uuid>
        addr: 192.168.200.12:4001
        peer-addr: 192.168.200.12:7001
      units:
        - name: etcd.service
          command: start
        - name: fleet.service
          command: start
        -  name: 00-eno1.network
          runtime: true
          content: |
            [Match]
            Name=eno1

            [Network]
            DNS=114.114.114.114
            Address=192.168.200.12/24
            Gateway=192.168.200.11
    users:
      - name: core
        ssh-authorized-keys:
         - ssh-rsa     ...
      - groups:
         - sudo
         - docker

配置文件的语法很简单，很多部分都可以见名知意，具体文件格式定义请参见:[Using Cloud-Config](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/)
如果配置中有错误，如果不影响启动，会在启动完成后显示错误信息。官方文档中提示
`coreos-cloudinit -validate` 命令可以用来检查此配置文件中是否有语法错误，但是在
实际安装的时候发现并没有这个参数(V0.10.9),还好官网提供一个在线的检测页面可供检
查 : [Cloud-Config Validator](https://coreos.com/validate/)


此文件中有几个点应注意 :

1. `#cloud-config` 为必备部分
2. 如果没有添加 `manage_etc_hosts`选项,在 `/etc/hosts` 文件中将不会有
   `localhost`的记录
3. 如果只是部署单机环境, `etcd` 中的 `discovery` 字段可以先省略，此字段是用来做
   服务发现的(供多个 etcd 通信).具体请参见[CoreOS Cluster Discovery](https://coreos.com/docs/cluster-management/setup/cluster-discovery/)
4. units 部分的设置会转换为相应的 systemd unit. `etcd`,`fleet` 等都是已定义好的，
   配置为开机启动即可。`fleet`为`systemd`提供了在集群内部调度`systemd unit`的能
   力。
5. users 部分的`ssh-authorized-keys`是用来设置需要登入这台 coreos 的 key 的,可为多个。
   因为 Coreos 安装默认是`core`和`root`两个用户，都没有密码,只能通过远程`ssh` 登陆
   来操作。选择好以后将用来登陆 coreos 的机器之后，将其 ssh key 粘贴到这即可，可添加
   多台机器的 key.如果需要，也可以生成为 core 用户生成密码，但没多大必要，具体可参
   见相关文档。
6. 安装完成之后, `cloud-config.yml`的内容会保存在
   `/var/lib/coreos-install/user_data`文件中，后续机器重启均会加载此文件。后续的
   更改也应该在这里进行。

#### **内网环境**
之前提过因为墙的原因很难下到安装时所需的镜像。下面介绍内网部署的方法:

1. 下载安装脚本 :
   [https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install](https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install),
   添加可执行权限。
2. 查看 `BASE_URL` 变量所指向的地址,如下所示 :

	    BASE_URL="http://${CHANNEL_ID}.release.core-os.net/amd64-usr/${VERSION_ID}"

	其中`CHANNEL_ID`是在下载 ISO 的时候就选择好的，一般都是`stable`,`VERSION_ID`为 LIVE CD 的版本号,在 `/etc/os-release`中。

3. 在上面的地址中下载如下文件 :
   1. coreos_production_image.bin.bz2
   2. coreos_production_image.bin.bz2.DIGESTS
   3. coreos_production_image.bin.bz2.DIGESTS.asc
   4. coreos_production_image.bin.bz2.DIGESTS.sig
   5. coreos_production_image.bin.bz2.sig

	这个看自己能力了。如果实在下不下来，可以到百度网盘里搜一下,应该已经有人在里面存有。
	
4. 在另外一台机器上部署`web server`,不嫌麻烦的可以用 `apache`,更简单的方法如下 :
   在文件所在的目录下执行 :

		python -m SimpleHTTPSever

	默认监听端口为 8000.

5. 修改 `coreos-install` 文件中的 `BASE_URL` 变量,改为自己的目录位置即可。

#### **最终安装**
安装命令就一个 :

    coreos-install -d /dev/sda -C stable -c ~/cloud-config.yaml

注意点 :

1. /dev/sda 为所要安装到的硬盘。安装有可能会出现 "Device Or Resource Busy "之类
   的问题，一般重启下机器即可。

2. 安装之后如果发现有问题并且进不去系统,比如网络故障，可以用 Live CD 来修复,将有
   问题的系统挂在到 Live CD 上修改`user_data`即可。 命令如下 :

		mount -o subvol=root /dev/sda9 /mnt/




	




	 
