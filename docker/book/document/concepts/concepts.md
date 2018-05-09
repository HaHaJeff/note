# Docker concepts
Docker is a platform for developers and sysadmins to develop, deploy, and run applications with containers. The use of Linux containers to deploy applications is called containerization. Containers are not new, but their use for easily deploying applications is.

Containerization is increasingly popular because containers are:
- Flexible: Even the most complex applications can be containerized.
- Lightweight: Containers leverage and share the host kernel.
- Interchangeable: You can deploy updates and upgrades on-the-fly.
- Portable: You can build locally, deploy to the cloud, and run anywhere.
- Scalable: You can increase and automatically distribute container replicas.
- Stackable: You can stack services vertically and on-the-fly.

## Images and containers
A container is launched by running an image. An image is an executable package that includes everything needed to run an application--the code, a runtime, libraries, environment variables, and configuration files.

A container is a runtime instance of an image--what the image becomes in memory when executed (that is, an image with state, or a user process). You can see a list of your running containers with the command, docker ps, just as you would in Linux.

## Containers and virtual machines
A container runs natively on Linux and shares the kernel of the host machine with other containers. It runs a discrete process, taking no more memory than any other executable, making it lightweight.

By contrast, a virtual machine (VM) runs a full-blown “guest” operating system with virtual access to host resources through a hypervisor. In general, VMs provide an environment with more resources than most applications need.
![avatar](https://github.com/HaHaJeff/note/blob/master/docker/book/document/concepts/vm_container.png)
![avatar](https://github.com/HaHaJeff/note/blob/master/docker/book/document/concepts/vm.png)

## Test Docker version
- Run docker --version and ensure that you have a supported version of Docker: <br> ``` docker --version ``` <br/> <br>``` Docker version 18.04.0-ce, build 3d479c0 ```<br/>
- Run docker info or (docker version without --) to view even more details about your docker installation: <br> ``` docker info``` <br/> <br>``` Containers: 3
 Running: 0
 Paused: 0
 Stopped: 3
Images: 1
Server Version: 18.04.0-ce
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 773c489c9c1b21a6d78b5c538cd395416ec50f88
runc version: 4fc53a81fb7c994640722ac585fa9ca548971871
init version: 949e6fa
Security Options:
 seccomp
  Profile: default
Kernel Version: 3.10.0-693.5.2.el7.x86_64
Operating System: Red Hat Enterprise Linux Server 7.3 (Maipo)
OSType: linux
Architecture: x86_64
CPUs: 80
Total Memory: 125.7GiB
Name: C80
ID: ZUHW:4X3G:X3FN:HEH3:ILUU:EUY7:JXRN:OEZO:LV45:VUFN:SFSL:BOA7
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false``` <br/>

