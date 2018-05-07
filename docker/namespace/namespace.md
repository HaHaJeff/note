Linux Namespace是Kernel的一个功能，它可以隔离一系列的系统资源，比如PID，User ID，Network等。类似于chroot命令
```
chroot - run command or interactive shell with special root directory 
```
chroot命令将当前目录变成根目录一样(被隔离开来的)，Namespace也可以在一些资源上将进程隔离起来，这些资源包括进程树，网络接口，挂载点等。
| Namespace类型 | 系统调用参数 | 内核版本 |
| ---- | ---- | ----|
| Moune Namespace | CLONE_NEWNS | 2.4.19
| UTS Namespace | CLONE_NEWUTS | 2.6.19
| IPC Namespace | CLONE_NEWIPC | 2.6.19
| PID Namespace | CLONE_NEWPID | 2.6.24
| Newwork Namespace | CLONE_NEWNET | 2.6.29
| User Namespace | CLONE_NEWUSER | 3.8
Namespace的API主要使用如下3个系统调用。
* clone()创建新进程。根据系统调用参数来判断哪些类型的Namespace被创建，而且他们的子进程也会被包含到这些Namespace中。
* unshare()将进程移出某个Namespace。
* senns()将进程加入到某个Namespace中。

## UTS Namespace
UTS Namespace主要用来隔离nodename核domainname两个系统标识。在UTS Namespace里面，每个Namespace允许有自己的hostname。
** 代码如下：**
``` gopackage main
import (
	"os/exec"
	"syscall"
	"os"
	"log"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
通过pstree -l查看一下系统中进程之间的关系，如下。
``` 
├─gnome-terminal-(4415)─┬─bash(4421)───su(22224)───bash(22225)───go(22237)─┬─main(22256)─┬─sh(22259)───pstree(22264)
```
查看当前进程的PID，代码如下：
```
# echo $$
22259
```
验证一下父进程和子进程是否不在同一个UTS Namespace中，验证代码如下：
``` 
# readlink /proc/22256/ns/uts
uts:[4026531838]
# readlink /proc/22259/ns/uts
uts:[4026532510]
```
可以看到它们确实不在同一个UTS Namespace中。由于UTS Namespace对hostname做了隔离，所以在这个环境内修改hostname应该不影响外部主机：
```
# hostname
ubuntu
# hostname -b jeff
# hostname
jeff
```
另起一个终端进行验证：
``` 
jeff@ubuntu:~/Workspace/Go/book/Namespace/UTS$ hostname
ubuntu
```

## IPC Namespace
IPC Namespace用来隔离System V IPC和POSIX message queues。每一个IPC Namespace都有自己的System V IPC和POSIX message queue。
** 代码如下：**
``` go
package main
import (
	"os/exec"
	"syscall"
	"os"
	"log"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
查看现有的message queue
``` 
buntu:~/Workspace/Go/book/Namespace/IPC$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```
创建一个message queue
``` 
jeff@ubuntu:~/Workspace/Go/book/Namespace/IPC$ ipcmk -Q
Message queue id: 0
jeff@ubuntu:~/Workspace/Go/book/Namespace/IPC$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x687a7f19 0          jeff       644        0            0           
```
运行上述代码：
``` 
# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```

## PID Namespace
PID Namespace是用来隔离进程ID的。同样一个进程在不同的PID Namespace里可以拥有不同的PID。这样就可以理解，在docker container里面，使用ps -ef经常会发现，在容器内，前来运行的那个进程PID是1，但是在容器外，使用ps -ef会发现同样的进程却有不同的PID，这就是PID Namespace做的事情。
** 代码如下：**
``` go
package main

import (
	"os/exec"
	"syscall"
	"os"
	"log"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
运行上述程序后，查看进程树，结果如下：
``` 
├─gnome-terminal-(4415)─┬─bash(4421)───su(22224)───bash(22225)───go(22985)─┬─main(23004)─┬─sh(23007)
           │               │               │               │                       │                                                  │             ├─{main}(23005)
           │               │               │               │                       │                                                  │             └─{main}(23006)
           │               │               │               │                       │                                                  ├─{go}(22986)
           │               │               │               │                       │                                                  ├─{go}(22987)
           │               │               │               │                       │                                                  ├─{go}(22989)
           │               │               │               │                       │                                                  └─{go}(22991)

```
可以看到运行main.go的进程的PID为23004，然后，在sh内查看当前进程ID：
``` 
# echo $$
1
``` 
## Mount Namespace
Mount Namespace用来隔离各个进程看到的挂载点视图。在不同的Namespace的进程中，看到的文件系统层次是不一样的。在Mount Namespace中调用mount()和umount()仅仅只会影响当前Namespace内的文件系统，而对全局的文件系统是没有影响的。
** 代码如下：**
``` go
package main

import (
	"os/exec"
	"syscall"
	"os"
	"log"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
查看当前的proc内容：
``` 
# ls /proc
1      12    165   177	 1896  195   205    2104   2194   224	 23401	2714  48   7413  999	      fs	   misc		 sysrq-trigger
10     1215  166   178	 19    1953  2051   211    2197   2241	 23419	28    49   7421  acpi	      interrupts   modules	 sysvipc
1002   13    168   179	 190   196   206    2112   2199   2247	 23422	289   5    7422  asound       iomem	   mounts	 thread-self
1008   135   169   18	 1908  197   2065   212    22	  2249	 23423	29    50   767	 buddyinfo    ioports	   mpt		 timer_list
1009   136   17    180	 1909  198   207    213    220	  225	 2379	290   51   79	 bus	      irq	   mtrr		 timer_stats
1014   137   170   181	 191   199   208    21330  2201   2265	 24	3     52   8	 cgroups      kallsyms	   net		 tty
1044   138   171   182	 1916  2     2086   2136   2204   2284	 2434	30    53   80	 cmdline      kcore	   pagetypeinfo  uptime
1046   139   172   183	 1918  20    209    214    22043  2287	 2435	31    54   822	 consoles     keys	   partitions	 version
1054   14    173   1833  192   200   2090   215    2208   2291	 2437	3191  55   870	 cpuinfo      key-users    sched_debug	 version_signature
10831  140   1739  184	 1920  201   20929  2152   221	  23	 2493	332   56   9	 crypto       kmsg	   schedstat	 vmallocinfo
10837  141   174   1840  1924  202   2093   21591  2210   2301	 2499	338   57   939	 devices      kpagecgroup  scsi		 vmstat
11     1428  1748  185	 1929  2024  2096   216    2218   23035  2503	354   58   941	 diskstats    kpagecount   self		 zoneinfo
1104   1468  1749  1852  193   203   2097   21699  2219   23073  2511	4161  59   943	 dma	      kpageflags   slabinfo
1118   15    175   187	 1935  2034  2099   217    222	  2313	 2525	4415  60   944	 driver       loadavg	   softirqs
1128   1508  1756  188	 194   2036  21     218    2226   23157  254	4421  66   945	 execdomains  locks	   stat
1132   16    1758  1880  1944  204   210    2189   223	  23389  256	460   7    963	 fb	      mdstat	   swaps
1141   164   176   189	 1947  2047  2103   219    2231   23390  2560	47    725  988	 filesystems  meminfo	   sys
```
在当前进程内mount proc：
``` 
# mount -t proc proc /proc
# ls /proc
1	   cgroups   diskstats	  fs	      kcore	   kpageflags  modules	     partitions   softirqs	 thread-self  version_signature
9	   cmdline   dma	  interrupts  keys	   loadavg     mounts	     sched_debug  stat		 timer_list   vmallocinfo
acpi	   consoles  driver	  iomem       key-users    locks       mpt	     schedstat	  swaps		 timer_stats  vmstat
asound	   cpuinfo   execdomains  ioports     kmsg	   mdstat      mtrr	     scsi	  sys		 tty	      zoneinfo
buddyinfo  crypto    fb		  irq	      kpagecgroup  meminfo     net	     self	  sysrq-trigger  uptime
bus	   devices   filesystems  kallsyms    kpagecount   misc        pagetypeinfo  slabinfo	  sysvipc	 version
```
## User Namespace
User Namespace主要是隔离用户的用户组ID。也就是说，一个进程的User ID和Group ID在User Namespace内外可以是不同的。比较常用的是，在宿主机上一个非root用户运行创建一个User Namespace，然后在User Namespace里面却映射成为root用户。这意味着，这个进程在User Namespace里面有root权限，但是在User Namespace外面却没有root的权限。从Linux Kernel 3.8开始，非root进程也可以创建User Namespace，并且此用户在Namespace里面可以被映射成为root，且在Namespace内有root权限。
** 代码如下：**
``` go
package main

import (
	"os/exec"
	"syscall"
	"os"
	"log"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS | syscall.CLONE_NEWUSER,
	}

    cmd.SysProcAttr.Crenential = &syscall.Crendential{Uid: uint32(1), Gid: uint32(1)}

	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
运行程序之前先查看id信息：
``` 
root@ubuntu:/home/jeff/Workspace/Go/book/Namespace# id
uid=0(root) gid=0(root) groups=0(root)
```
运行程序之后再次查看id信息：
```
由于权限问题无法运行
```

## Network Namesapce
Network Namespace是用来隔离网络设备，IP地址端口等网络栈的Namespace。Network Namespace可以让每个容器拥有自己独立的(虚拟的)网络设备，而且容器内的应用的可以绑定到自己的端口，每个Namespace内的端口都不会互相冲突。
** 代码如下：**
``` go

```

运行程序之前查看网络设备：
``` 
root@ubuntu:/home/jeff# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:69:ca:f0:19  
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ens33     Link encap:Ethernet  HWaddr 00:0c:29:d6:4e:53  
          inet addr:192.168.146.134  Bcast:192.168.146.255  Mask:255.255.255.0
          inet6 addr: fe80::19e2:7e9:5706:f5ed/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:68 errors:0 dropped:0 overruns:0 frame:0
          TX packets:107 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:15130 (15.1 KB)  TX bytes:11030 (11.0 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:240 errors:0 dropped:0 overruns:0 frame:0
          TX packets:240 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:21233 (21.2 KB)  TX bytes:21233 (21.2 KB)

```

运行程序之后查看网络设备：
```
$ ifconfig	
$ 
```