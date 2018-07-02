# Docker总架构图
![总架构](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/%E6%80%BB%E4%BD%93%E6%9E%B6%E6%9E%84.jpg)
## Docker Client
Docker client是Docker架构中用户和Docker Daemon建立通信的客户端。用户使用的可执行文件为docker，通过docker命令行工具可以发起众多管理container的请求。
- tcp://host:port
- unix://path_to_socket
- fd://socketfd
## Docker Daemon
Docker Daemon是Docker架构中一个常驻在后台的系统进程，功能是：接受并处理Docker Client发送的请求。该守护进程在后台启动了一个Server，Server负责接受Docker Client发送的请求；接受请求后，Server通过路由与分发调度，找到相应的Handler来执行请求。
**Docker Daemon启动所使用的文件也为docker，与Docker Client启动所使用的可执行文件docker相同。在docker命令执行时，通过传入的参数来判别Docker Daemon与Docker Client**
![Docker Daemon架构](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/docker-daemon%E6%9E%B6%E6%9E%84.jpg)

## Docker Server
Docker Server在Docker架构中是专门服务于Docker Client的server。该server的功能是：接受并调度分发Docker Client发送的请求。
![Docker Server架构](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/docker-server%E6%9E%B6%E6%9E%84.png)
**需要注意的是：Docker Server的运行在Docker的启动过程中，是靠一个名为“serverapi”的job运行来完成的。原则上，Docker Server的运行是众多job中的一个，但是为了强调Docker Server的重要性以及后续job服务的重要特性，将该“serverapi”的jjob单独抽离出来，理解为Docker Server **

## Engine
Engine是Docker架构中的运行引擎，同时也是Docker运行的核心模块。它扮演Docker container存储仓库的角色，并且通过执行job的方式来操纵管理这些容器。

在Engine数据结构的设计与实现过程中，有一个handler对象。该handler对象存储的都是关于众多特定job的handler处理访问。举例说明，Engine的handler对象中有一项为：{"create": daemon.ContainerCreate,}，则说明当名为"create"的job在运行时，执行的是daemon.ContainerCreate的handler。

## Job
一个Job可以认为是Docker架构中Engine内部最基本的工作执行单元。Docker可以做的每一项工作，都可以抽象为一个job。例如：在容器内部运行一个进程，这是一个job；创建一个新的容器，这是一个job，从Internet上下载一个文档，这是一个job；包括之前在Docker Server部分说过的，创建Server服务于HTTP的API，这也是一个job，等等。

Job的设计者，把Job设计得与Unix进程相仿。比如说：Job有一个名称，有参数，有环境变量，有标准的输入输出，有错误处理，有返回状态等。

## Docker Registry
Docker Registry是一个存储容器镜像的仓库。而容器镜像是在容器被创建时，被加载用来初始化容器的文件架构与目录。

在Docker的运行过程中，Docker Daemon会与Docker Registry通信，并实现搜索镜像、下载镜像、上传镜像三个功能，这三个功能对应的job名称分别为"search"，"pull" 与 "push"。

## Graph
Graph在Docker架构中扮演已下载容器镜像的保管者，以及已下载容器镜像之间关系的记录者。一方面，Graph存储着本地具有版本信息的文件系统镜像，另一方面也通过GraphDB记录着所有文件系统镜像彼此之间的关系

其中，GraphDB是一个构建在SQLite之上的小型图数据库，实现了节点的命名以及节点之间关联关系的记录。它仅仅实现了大多数图数据库所拥有的一个小的子集，但是提供了简单的接口表示节点之间的关系。

![Graph架构](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/graph.jpg)

同时在Graph的本地目录中，关于每一个的容器镜像，具体存储的信息有：该容器镜像的元数据，容器镜像的大小信息，以及该容器镜像所代表的具体rootfs。

## Driver
Driver是Docker架构中的驱动模块。通过Driver驱动，Docker可以实现对Docker容器执行环境的定制。由于Docker运行的生命周期中，并非用户所有的操作都是针对Docker容器的管理，另外还有关于Docker运行信息的获取，Graph的存储与记录等。因此，为了将Docker容器的管理从Docker Daemon内部业务逻辑中区分开来，设计了Driver层驱动来接管所有这部分请求。

在Docker Driver的实现中，可以分为以下三类驱动：graphdriver、networkdriver和execdriver。

graphdriver主要用于完成容器镜像的管理，包括存储与获取。即当用户需要下载指定的容器镜像时，graphdriver将容器镜像存储在本地的指定目录；同时当用户需要使用指定的容器镜像来创建容器的rootfs时，graphdriver从本地镜像存储目录中获取指定的容器镜像。

在graphdriver的初始化过程之前，有4种文件系统或类文件系统在其内部注册，它们分别是aufs、btrfs、vfs和devmapper。而Docker在初始化之时，通过获取系统环境变量”DOCKER_DRIVER”来提取所使用driver的指定类型。而之后所有的graph操作，都使用该driver来执行。
![graphdriver架构](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/graphdriver.png)

![networkdriver](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/networkdriver.jpg)

![execdriver](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/execdriver.jpg)

# libcontainer
libcontainer是Docker架构中一个使用Go语言设计实现的库，设计初衷是希望该库可以不依靠任何依赖，直接访问内核中与容器相关的API。

正是由于libcontainer的存在，Docker可以直接调用libcontainer，而最终操纵容器的namespace、cgroups、apparmor、网络设备以及防火墙规则等。这一系列操作的完成都不需要依赖LXC或者其他包。

![libcontainer](https://github.com/HaHaJeff/note/blob/master/docker/book/Docker%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E6%9E%B6%E6%9E%84/libcontainer.jpg)
