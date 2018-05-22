# 容器互联
## 新建网络
创建一个新的Docker网络
```
$ docker network create -d bridge my-net
```
```-d```参数指定Docker网络类型，有```bridge``` ，```overlay```。其中``` overlay ```网络类型使用于``` Swarm mode ```
```
jeff@ubuntu:~/Workspace/docker$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d305c74fd8c9        bridge              bridge              local
680efeab87b7        host                host                local
6b3b201f7ede        my-net              bridge              local
d5323b02e68c        none                null                local
```

## 连接容器
运行一个容器并连接到新建的``` my-net ```网络
```
$ docker run -it --rm --name busybox1 --network my-net busybox sh
```
打开新的终端，再运行一个容器并加入到```my-net```网络
```
docker run -it --rm --name busybox2 --network my-net busybox sh
```
再打开一个新的终端查看容器信息
```
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
89b168eb757c        busybox             "sh"                16 seconds ago       Up 14 seconds                           busybox2
a86e74e0716f        busybox             "sh"                About a minute ago   Up About a minute                       busybox1
```
通过```ping```来证明```busybox1```容器和```busybox2```容器建立了互联关系
在```busybox1```容器输入以下命令
```
/ # ping 172.18.0.3
PING 172.18.0.3 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=1.356 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.162 ms
64 bytes from 172.18.0.3: seq=2 ttl=64 time=0.162 ms
```

## 配置DNS
在容器中使用```mount```命令可以看到挂挂载信息：
```
/ # mount
...
/dev/sda1 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sda1 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sda1 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered)
...
```
在``` /etc/resolve.conf ``` ``` /etc/hostname ``` ``` /etc/hosts ```中进行设置，所有Docker容器就可以通过这三个文件立刻得到更新。
配置全部容器的DNS，也可以在``` /etc/docker/daemon.json ```文件中增加以下内容来设置。
```
{
	"dns" : [
		"114.114.114.114",
		"8.8.8.8"
	]
}
```
这样每次启动的容器DNS自动配置为``` 114.114.114.114 ```和``` 8.8.8.8 ```。

## 高级网络配置
当Docker启动时，会自动在主机上创建一个``` docker0 ```虚拟网桥，实际上是Linux的一个bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间之间进行转发。
同时，Docker随机分配一个本地未占用的私有网段(在RFC1918中定义)中的一个地址给``` docker0 ```接口。比如典型的``` 172.17.42.1 ``` ，掩码为``` 255.255.0.0 ``` 。此后启动的容器内的网口也会自动分配一个同一个网段(``` 172.17.0.0/16 ```)的地址。
当创建一个Docker容器的时候，同时会创建了一对``` veth pair ```接口(当数据包发送到一个接口时，另外一个接口也可以收到相同的数据包)。这对接口一段在容器内，即``` eth0 ```；另一端在本地并被挂载到``` docker0 ``` 网桥，名称以``` veth ``` 开头。通过这种方式，主机可以跟容器通信，容器之间也可以相互通信。Docker 就创建了在主机和所有容器之间一个虚拟共享网络。

## Docker网络相关的命令列表
只有在Docker服务启动的时候才能配置，而且不能马上生效：
- ``` -b BRIDGE ```或 ``` --bridge=BRIDGE ```指定容器挂载的网桥
- ``` --bip=CIDR```指定docker0的掩码
- ``` -H SOCKET ...```或``` --host=SOCKET...``` Dcoker服务端接受命令的通道
- ``` --icc=true|false ```是否支持容器之间进行通信
- ``` --ip-forward=true|false```
- ``` --iptables=true|false```是否允许Docker添加iptable规则
- ``` --mtu=BYTES```容器网络中的MTU
下面两个命令选项既可以在启动服务时指定，也可以在启动容器时指定。在Docker服务启动的时候指定规则会成为默认值，后面执行``` docker run ```时可以覆盖设置的默认值。
- ```--dns=IP_ADDRESS...```使用指定的DNS服务器
- ```--dns-search=DOMAIN```指定DNS搜索域
最后这些选项只有在``` docker run```执行时使用，因为它是针对容器的特性内容。
- ```-h HOSTNAME```或```--hostname=HOSTNAME```配置容器主机名
- ```--link=CONTAINER_NAME;ALIAS```添加到另一个容器的链接
- ```--net=bridge|none|container:NAME_or_IO|host```配置容器的桥接模式
- ```-p SPEC```或```--publish=SPEC```映射容器端口到宿主主机
- ```-P or --publish-all=true|false```映射容器所有端口到宿主主机

# 容器访问控制
容器的访问控制，主要通过linux上的```iptables```防火墙来进行管理和实现。
## 容器访问外部网络
容器要想访问外部网络，需要本地系统的转发支持。在Linux 系统中，检查转发是否打开。
```
$sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```
如果在启动 Docker 服务的时候设定  --ip-forward=true  , Docker 就会自动设定系统的
ip_forward  参数为 1。
## 容器之间访问
容器之间相互访问，需要两方面的支持。
- 容器的网络拓扑是否已经互联。默认情况下，所有容器都会链接到```docker0```网桥上
- 本地系统的防火墙软件```--iptables```是否允许通过

## 访问所有端口
当启动 Docker 服务时候，默认会添加一条转发策略到 iptables 的 FORWARD 链上。策略为通过（``` ACCEPT ``` ）还是禁止（ ```DROP ``` ）取决于配置 ```--icc=true```  （缺省值）还是  ```--icc=false```  。当然，如果手动指定  ```--iptables=false```  则不会添加  ```iptables```  规则。

可见，默认情况下，不同容器之间是允许网络互通的。如果为了安全考虑，可以在```/etc/default/docker```  文件中配置  ```DOCKER_OPTS=--icc=false``` 来禁止它。