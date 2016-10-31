# Shrike

![Shrike Logo](logo.jpg?raw=true "Shrike Logo")



### 开发背景

众所周知，Docker容器跨主机互访一直是一个问题，Docker官方为了避免网络上带来的诸多麻烦，故将跨主机网络开了比较大的口子，而由用户自己去实现。

目前Docker跨主机的网络实现方案也有很多种, 主要包括端口映射，ovs, fannel等。但是这些方案都无法满足我们的需求，端口映射服务内的内网IP会映射成外网的IP，这样会给开发带来困惑，因为他们往往在跨网络交互时是不需要内网IP的，而ovs与fannel则是在基础网络协议上又包装了一层自定义协议，这样当网络流量大时，却又无端的增加了网络负载，最后我们采取了自主研发扁平化网络插件，也就是说让所有的容器统统在大二层上互通。





### 插件原理



##### 创建Docker自定义网络

我们首先需要创建一个br0自定义网桥，这个网桥并不是通过系统命令手动建立的原始Linux网桥，而是通过Docker的create network命令来建立的自定义网桥，这样避免了一个很重要的问题就是我们可以通过设置DefaultGatewayIPv4参数来设置容器的默认路由，这个解决了原始Linux自建网桥不能解决的问题. 用Docker创建网络时我们可以通过设置subnet参数来设置子网IP范围，默认我们可以把整个网段给这个子网，后面可以用ipam driver（地址管理插件）来进行控制。还有一个参数gateway是用来设置br0自定义网桥地址的，其实也就是你这台宿主机的地址啦。

```shell
docker network create 
--opt=com.docker.network.bridge.enable_icc=true
--opt=com.docker.network.bridge.enable_ip_masquerade=false
--opt=com.docker.network.bridge.host_binding_ipv4=0.0.0.0
--opt=com.docker.network.bridge.name=br0
--opt=com.docker.network.driver.mtu=1500
--ipam-driver=talkingdata
--subnet=容器IP的子网范围，
--gateway=br0网桥使用的IP,也就是宿主机的地址，
--aux-address=DefaultGatewayIPv4=容器使用的网关地址
mynet
```



##### IPAM

这个驱动是专门管理Docker 容器IP的, Docker 每次启停与删除容器都会调用这个驱动提供的IP管理接口，然后IP接口会对存储IP地址的Etcd有一个增删改查的操作。此插件运行时会起一个Unix Socket, 然后会在docker/run/plugins 目录下生成一个.sock文件，Docker daemon之后会和这个sock 文件进行沟通去调用我们之前实现好的几个接口进行IP管理，以此来达到IP管理的目的，防止IP冲突。



##### 桥接

通过Docker命令去创建一个自定义的网络起名为“mynet”，同时会产生一个网桥br0，之后通过更改网络配置文件（在/etc/sysconfig/network-scripts/下ifcfg-br0、ifcfg-默认网络接口名）将默认网络接口桥接到br0上，重启网络后，桥接网络就会生效。Docker默认在每次启动容器时都会将容器内的默认网卡桥接到br0上，而且宿主机的物理网卡也同样桥接到了br0上了。其实桥接的原理就好像是一台交换机，Docker 容器和宿主机物理网络接口都是服务器，通过veth pair这个网络设备像一根网线插到交换机上。至此，所有的容器网络已经在同一个网络上可以通信了，每一个Docker容器就好比是一台独立的虚拟机，拥有和宿主机同一网段的IP，可以实现跨主机访问了。





### 安装部署



假设我们有6台物理机想要部署docker集群，这里为了方便举例，我们少选一些主机。

192.168.0.1

192.168.0.2

192.168.0.3

192.168.0.4

192.168.0.5

192.168.0.6



##### ETCD

首先，我们需要安装etcd集群，我们选3台物理机当做etcd集群，分别是192.168.0.1，192.168.0.2，192.168.0.3

我们使用下面命令分别在这3台机器上运行

```shell
yum localinstall -y ./rpms/etcd-2.3.1-1.el7.centos.x86_64.rpm
```

安装完成后，修改/etc/etcd/etcd.conf 文件

```
# [member]
ETCD_NAME=infra0
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.1:2379,http://127.0.0.1:2379"
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.1:2380"
ETCD_INITIAL_CLUSTER="infra0=http://192.168.0.1:2380,infra1=http://192.168.0.2:2380,infra2=http://192.168.0.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="talkingdata"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.1:2379"
```

修改完配置文件后，运行命令

```shell
systemctl start etcd
```

如果我们的系统不支持systemd管理，那么则需要运行

```shell
/usr/bin/etcd --name=infra0 --data-dir=/var/lib/etcd/default.etcd --initial-advertise-peer-urls=http://192.168.0.1:2380 --listen-peer-urls=http://192.168.0.1:2380 --listen-client-urls=http://192.168.0.1:2379,http://127.0.0.1:2379 --advertise-client-urls=http://192.168.0.1:2379 --initial-cluster-token=talkingdata --initial-cluster=infra0=http://192.168.0.1:2380,infra1=http://192.168.0.2:2380,infra2=http://192.168.0.3:2380 --initial-cluster-state=new
```

以上我们只部署了192.168.0.1这台物理机的etcd服务，剩下的两台只需要更改他们相应的参数为自己的地址就可以。

3台etcd集群大家完成后，随便找其中一台物理机，运行

```shell
etcdctl member list

5b702b44353ea046: name=infra3 peerURLs=http://192.168.0.3:2380 clientURLs=http://192.168.0.3:2379 isLeader=false
8267e362923e92b1: name=infra2 peerURLs=http://192.168.0.2:2380 clientURLs=http://192.168.0.2:2379 isLeader=false
a1a8fc73c125a0bc: name=infra1 peerURLs=http://192.168.0.1:2380 clientURLs=http://192.168.0.1:2379 isLeader=true
```

这就证明安装成功了。

这里我们先提安装一个oam-docker-ipam插件用来分配IP

```
yum localinstall -y ./rpms/oam-docker-ipam-1.0.0-1.el7.centos.x86_64.rpm
```

这里安装完后我们继续运行命令

```
oam-docker-ipam host-range --ip-start 192.168.0.4/24 --ip-end 192.168.0.6/24 --gateway 192.168.223.2
```

上面这条命令是用来分配docker宿主机IP范围的。

之后我们还要分配docker容器的IP范围

```
oam-docker-ipam ip-range --ip-start 192.168.0.100/24 --ip-end 192.168.0.200/24
```



##### Swarm-Manager

然后，我们需要安装swarm-manager的集群，我们这里可以还是选择192.168.0.1，192.168.0.2，192.168.0.3 这3台。

```shell
yum localinstall -y ./rpms/swarm-1.2.2-1.el7.centos.x86_64.rpm
```

安装完成后，修改配置文件/etc/swarm/swarm-manager.conf

```
# [manager]
SWARM_HOST=:4000
SWARM_ADVERTISE=192.168.0.1:4000  SWARM_DISCOVERY=etcd://192.168.0.1:2379,192.168.0.2:2379,192.168.0.3:2379
```

修改完配置文件后，运行命令

```shell
systemctl start swarm-manager
```

如果我们的系统不支持systemd管理，那么则需要运行

```shell
/usr/bin/swarm --debug manage --replication --host=:4000 --advertise=192.168.0.1:4000 etcd://192.168.0.1:2379,192.168.0.2:2379,192.168.0.3:2379
```

以上我们只部署了192.168.0.1这台物理机的swarm-manager服务，剩下的两台只需要更改他们相应的参数为自己的地址就可以。



##### Docker



我们需要分别在这6台机器上安装docker

```shell
yum localinstall -y ./rpms/docker-engine-1.11.1-1.el7.centos.x86_64.rpm
yum localinstall -y ./rpms/docker-engine-selinux-1.11.1-1.el7.centos.noarch.rpm
```

安装完成后，修改/usr/lib/systemd/system/docker.service 文件

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/docker daemon --debug -H=fd:// -H=tcp://0.0.0.0:2375 -H=unix:///var/run/docker.sock --insecure-registry=172.31.0.110:5000 -s=overlay -g /ssdcache/docker
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

[Install]
WantedBy=multi-user.target
```

参数--insecure-registry是镜像管理地址，这个要根据情况进行更改，以上配置都是符合我们需求的。

修改完配置后，运行

```shell
systemctl start docker
```

如果我们的系统不支持systemd管理，那么则需要运行

```shell
/usr/bin/docker daemon --debug -H=fd:// -H=tcp://0.0.0.0:2375 -H=unix:///var/run/docker.sock --insecure-registry=172.31.0.110:5000 -s=overlay -g /ssdcache/docker
```



##### OAM-DOCKER-IPAM

我们需要分别在192.168.0.4, 192.168.0.5, 192.168.0.6 这3台机器上安装oam-docker-ipam插件

```shell
yum localinstall -y ./rpms/oam-docker-ipam-1.0.0-1.el7.centos.x86_64.rpm
```

安装完成后，修改/etc/oam-docker-ipam/oam-docker-ipam.conf 文件

```
# [ipam]
IPAM_DEBUG=true IPAM_CLUSTER_STORE=http://192.168.0.1:2379,http://192.168.0.2:2379,http://192.168.0.3:2379
```

修改完配置后，运行

```shell
systemctl start oam-docker-ipam
```

如果我们的系统不支持systemd管理，那么则需要运行

```bash
/usr/bin/oam-docker-ipam --debug=true --cluster-store=http://192.168.0.1:2379,http://192.168.0.2:2379,http://192.168.0.3:2379 server
```

创建自定义网络br0

```
oam-docker-ipam --cluster-store=http://192.168.0.1:2379,http://192.168.0.2:2379,http://192.168.0.3:2379 create-network --ip 192.168.0.4
```



##### Swarm-Agent

然后，我们需要安装swarm-agent，我们这里可以还是选择192.168.0.4，192.168.0.5，192.168.0.6 这3台。

```shell
yum localinstall -y ./rpms/swarm-1.2.2-1.el7.centos.x86_64.rpm
```

安装完成后，修改配置文件/etc/swarm/swarm-agent.conf

```
# [agent]
SWARM_ADVERTISE=192.168.0.4:2375 SWARM_DISCOVERY=etcd://192.168.0.1:2379,192.168.0.2:2379,192.168.0.3:2379
```

修改完配置文件后，运行命令

```shell
systemctl start swarm-agent
```

如果我们的系统不支持systemd管理，那么则需要运行

```shell
/usr/bin/swarm --debug join --advertise=192.168.0.4:2375 etcd://192.168.0.1:2379,192.168.0.2:2379,192.168.0.3:2379
```

以上我们只部署了192.168.0.4这台物理机的swarm-agent服务，剩下的两台只需要更改他们相应的参数为自己的地址就可以。



##### Shipyard



我们选一台物理机作为Shipyard的运行机，此应用是一款开源项目，我们基于它做了一些小修改，加入了自定义网络以及CPU核数限制的修改。接下来我们选择192.168.0.1这台机器安装

```
yum localinstall -y ./rpms/shipyard-3.0.4-1.el7.centos.x86_64.rpm
```

从dockerhub上下载rethinkdb

```
docker pull rethinkdb
```

下载完成后运行

```
docker run -d rethinkdb
```

获取该容器的IP地址后并记录下来(这里我们假设为172.17.0.3)，后面修改配置文件时会使用。

修改/etc/shipyard/shipyard.conf配置文件

```
# [shipyard]
SHIPYARD_LISTEN=:8080
SHIPYARD_DATABASE=172.17.0.3:28015
SHIPYARD_DOCKER=tcp://192.168.0.1:4000 #其中一台swarm-manager的地址
```

修改完配置文件后，运行命令

```shell
systemctl start shipyard
```

打开浏览器访问http://192.168.0.1:8080
输入用户名：admin, 密码：shipyard 



### 参考


[http://www.shipyard-project.com/](http://www.shipyard-project.com/)

[https://docs.docker.com/engine/extend/plugins](https://docs.docker.com/engine/extend/plugins)

[https://github.com/docker/libnetwork/blob/master/docs/ipam.md](https://github.com/docker/libnetwork/blob/master/docs/ipam.md)

[https://github.com/docker/go-plugins-helpers](https://github.com/docker/go-plugins-helpers)


### 作者
邮箱：machaojms@126.com
QQ/Wechat: 369101940
