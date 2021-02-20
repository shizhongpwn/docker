# docker基础

docker进程是docker客户端，dockered进程是docker 的服务端，代码在[moby](https://github.com/moby/moby)项目里面。

docker客户端在执行`docker run hello-world`之类的命令之后，docker客户端启动，进行命令行参数解析，然后构造相应的启动容器请求，通过reset的方式发给dockered，其中Rest是一种概念，是服务端和客户端之间一种交互形式的抽象概念。

Engine  API : https://docs.docker.com/engine/api/v1.26/# (版本很多) 这里面描述了dockered支持的所有请求。

## dockered 和 docker hub

当dockered在解析请求的时候没有发现相应的image的时候，会从[docker hub](https://hub.docker.com/)（可以简单理解为image集合） 里面取相应的image。

## dockered 和 docker-containerd

dockered拿到image之后，会创建相应的容器，并通过[grpc](http://www.grpc.io/)的方式通知docker-containred进程启动指定的容器。

docker-containerd是和dockered一起启动的后台进程，它们之间通过unix socket进行通信，协议为grpc。

## docker-containerd 和 docker-containerd-shim

两者都属于[containerd](https://github.com/containerd/containerd)项目当docker-containerd收到dockerd的启动容器请求之后，会做一些初始化工作，然后启动docker-containerd-shim进程，并将相关配置所在的目录作为参数传给它。

docker-containerd管理所有本机正在运行的容器，而docker-containerd-shim只负责管理一个运行的容器

## docker-containerd-shim <--> docker-runc

docker-containerd-shim进程启动后，会按照runtime的标准备好相关运行时的环境，然后启动docker-runc进程。

runc是给docker贡献给OCI的一个标准runtime实现。

runc是启动容器的最后一步，设置cgroup，隔离namespaces并启动程序。

如何启动runc是公开的标准，大概过程就是准备好rootfs和配置文件，然后使用合适的参数启动runc进程就可以了。

## 进程间关系

等runc将容器启动起来之后，runc进程就退出了，于是容器里面的第一个进程的父进程就变成了docker-containerd-shim，进程关系：

~~~
systemd───dockerd───docker-containerd───docker-containerd-shim───hello
~~~

> 实际操作过程中可能看不到这样的输出，因为hello很快就运行退出了，接着docker-containerd-shim也退出了。

其中dockerd和docker-containerd是后台常驻进程，而docker-containerd-shim则由docker-containerd按需启动。

> runc退出后其子进程hello不是应该由init进程接管吗？怎么就变成了docker-containerd-shim的子进程了呢？这是因为从Linux 3.4开始，[prctl](http://man7.org/linux/man-pages/man2/prctl.2.html)增加了对PR_SET_CHILD_SUBREAPER的支持，这样就可以控制孤儿进程可以被谁接管，而不是像以前一样只能由init进程接管。

 ## 输出

容器的标准输出能被Docker的这些进程一层层的转发给客户端，进而在终端显示。



