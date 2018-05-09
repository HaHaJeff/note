# Dockers镜像
Docker镜像是一个特殊的文件系统，除了提供容器运行时所需的程序，库，资源，配置等文件外，还包含了一些为运行时准备的一些配置参数(如匿名卷，环境变量，用户等)。镜像不包含任何动态数据，其内容在构建之后也不会被改变。
## 分层存储
在设计Docker时，充分利用[UnionFS](https://en.wikipedia.org/wiki/Union_mount)技术，将其设计为分层存储的架构。所以，严格来说，镜像并非是像一个ISO那样的打包文件，**镜像只是一个虚拟的概念**，其实际体现并非由一个文件组成，而是由一组文件系统文件，或者说，由多层文件系统联合组成。

镜像在构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不再发生改变，后一层上的任何文件只发生在自己这一层。比如，删除前一层文件的操作，**实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除**。在容器运行的时候，虽然不会看到这个文件，但是实际该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。

分层存储的特使还使得镜像的复用，定制变的更为容易。甚至可以用之前的构建好的镜像作为基础层，然后进一步添加新的层，以定制自己需要的内容，构建新的镜像。
# Docker容器
镜像(image)和容器(container)的关系，就像是面向对象程序中的类和实例一样，镜像就是静态的定义，容器就是镜像运行时的实体。容器可以被创建，启动，停止，删除，暂停等。

容器的实际就是进程，但与直接在宿主机执行的进程不同，容器进程运行于属于自己的独立的namespace。因此，容器可以拥有自己的root文件系统，自己的网络配置，自己的进程空间，甚至自己的用户ID空间。容器内的进程试运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。

容器以镜像为基础层，在其上创建一个当前容器的基础层，称这个为容器运行时读写而准备的存储层为容器存储层。容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。

按照Docker最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用**数据卷**，或者绑定宿主目录。在这些位置的读写会跳过容器存储层，直接对宿主(或网络存储)发生读写，其性能和稳定性更高。

**数据卷**的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

## 获取镜像
从Docker镜像仓库获取镜像的命令是 ```docker pull```。其命令格式：
```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```
- Docker镜像仓库的地址：地址的格式一般是<域名/IP>[:port]。默认地址是DockerHub。
- 仓库名：两段式名称，即<用户名>/<软件名>。对于DockerHub，如果不给出用户名，则默认为library，也就是官方镜像。

**比如：**
```
$ sudo docker pull ubuntu:16.04
16.04: Pulling from library/ubuntu
297061f60c36: Pull complete 
e9ccef17b516: Pull complete 
dbc33716854d: Pull complete 
8fe36b178d25: Pull complete 
686596545a94: Pull complete 
Digest: sha256:1dfb94f13f5c181756b2ed7f174825029aca902c78d0490590b1aaa203abc052
Status: Downloaded newer image for ubuntu:16.04
```
上面的命令中没有给出Docker镜像仓库地址，因此将会从DockerHub获取镜像。而镜像名称是```ubuntu:16.04```，因此将会获取官方镜像```library/ubuntu```仓库中标签为```16.04```的镜像。

从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的ID的前12位。并且下载结束后，给出该镜像完整的```sga256```的摘要，以确保下载一致性。

## 运行
```
jeff@ubuntu:~$ sudo docker run -it --rm ubuntu:16.04 bash

root@4e73c08383a3:/# cat /etc/os-release 
NAME="Ubuntu"
VERSION="16.04.4 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.4 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
root@4e73c08383a3:/# 
```
docker run就是运行容器的命令：
- i：表示交互操作；
- t：表示终端；
- --rm：说明容器推出之后随之将其删除。

## 列出镜像
```
jeff@ubuntu:~$ sudo docker image ls
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
jeffzhouhhh/get-started   part2               bd3b2bb8e6fd        2 days ago          151MB
ubuntu                    16.04               0b1edfbffd27        11 days ago         113MB
ubuntu                    latest              452a96d81c30        11 days ago         79.6MB
hello-world               latest              e38bc07ac18e        3 weeks ago         1.85kB
```
列表包含了 ```仓库名```，```标签```，```镜像ID```，```创建时间```以及```占用空间```。

## 镜像体积
docker image ls中的SIZE和DockerHub上看到的镜像大小不同。比如，ubuntu:16.04镜像大小，在这里是127MB，但是在DockerHub显示的确实50MB。这是因为DockerHub中显示的体积是压缩后的体积。

**需要注意的是：docker image ls列表中的size并非是所有镜像体积实际的硬盘资源占用**，由于docker镜像是多层存储结构，并且可以继承，复用，因此，不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于docker使用UnionFS，相同的层只需要保存一份即可，因此，实际镜像硬盘占用空间可能小的多。

```
jeff@ubuntu:~$ sudo docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              4                   3                   343.8MB             113MB (32%)
Containers          5                   0                   257.1kB             257.1kB (100%)
Local Volumes       0                   0                   0B                  0B
Build Cache                                                 0B                  0B
```

## 虚悬镜像
**定义：**仓库名和标签均为```<none>```

## 中间层镜像
为了加速镜像构建、重复利用资源，Docker会利用中间层镜像。所以在使用一段时间后，可能会看到一些依赖的中间层镜像。默认的```docker image ls```列表中只会显示顶层镜像，如果需要显示包括中间层镜像在内的所有镜像，则需要加```-a```参数。
```
$ docker image ls -a
```
docker image ls的功能一览：
```
Usage:	docker image ls [OPTIONS] [REPOSITORY[:TAG]]
List images
Aliases:
  ls, images, list
Options:
  -a, --all             Show all images (default hides intermediate images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
  -q, --quiet           Only show numeric IDs
```

## 删除本地镜像
如果要删除本地的镜像，可以使用```docker image rm```，其格式为：
```
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```
## 用ID、镜像名、摘要删除镜像
其中，```<镜像>```可以是```镜像短ID```、```镜像长ID```、```镜像名```或者```镜像摘要```。

## Untagged和Deleted
镜像的唯一标识：**其ID和摘要**，而一个镜像可以有多个标签。
- **Untagged：**取消指定镜像的某个标签；
- **Deleted：** 删除镜像。

**因此当我们使用删除命令删除镜像时，实际上是在要求删除某个标签的镜像。所有，当删除指定的标签的镜像时，可能还有其他标签指向这个镜像，如果是这种情况，那么```delete```行为就不会发生。**综上，并非所有的docker rmi都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已。
