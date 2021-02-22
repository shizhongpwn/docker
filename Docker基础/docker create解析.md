# docker create解析

> https://segmentfault.com/a/1190000010057763
>
> 这个文章东西还是值得一看的，具体讲述了容器的创建过程，而不是准备过程。

Docker run命令直接创建并运行了一个容器，它背后包含独立的两个部分：

* docker  create
* docker start

docker在收到客户端请求之后，一是准备容器需要的layer，二是检查客户端传过来的参数，并和image配置文件中的参数进行合并，然后存储成容器的配置文件。

## 创建容器

~~~bash
#创建一个容器，并取名为docker_test,
#-i是为了让容器能接受用户的输入，-t是指定docker为容器创建一个tty，
#因为ubuntu这个镜像默认启动的进程是bash，而bash需要tty，否则会异常退出
dev@dev:~$ docker create -it --name docker_test ubuntu
967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368

dev@dev:~$ docker ps -a
CONTAINER ID   IMAGE    COMMAND       CREATED         STATUS   PORTS   NAMES
967438113fba   ubuntu   "/bin/bash"   6 seconds ago   Created          docker_test
~~~

## layer的元数据

创建容器的时候，docker会为每个容器创建两个新的layer,一个是只读的init layer,里面包含docker 为容器准备的一些文件，另一个是容器的可写的mount layer，以后在容器里面对rootfs的所有增删改操作的结果会存储在这个layer里面。

> 日，文章里全是说配置文件的，直接tmd总结：docker create就是为container准备配置文件和layer的。

# docker start解析

~~~bash
#根据容器名称启动容器（也可以根据容器ID来启动）
root@dev:~# docker start docker_test
docker_test

#可以看出容器正在后台运行bash
root@dev:~# docker ps
CONTAINER ID   IMAGE    COMMAND       CREATED          STATUS         PORTS   NAMES
967438113fba   ubuntu   "/bin/bash"   38 minutes ago   Up 8 seconds           docker_test
~~~

这个是作者的，但是无语的是，我的容器直接tmd退出了，看样子还是正常退出那种

~~~bash
 ✔  docker create -it --name docker_test f751b30e3dbf
d07ba9a776cfccaceb020ade2581cdff21df419e9cd4b8b54d49b954e1d18b0c

 ~ 
 ✔  docker ps -a
CONTAINER ID   IMAGE          COMMAND        CREATED         STATUS    PORTS     NAMES
d07ba9a776cf   f751b30e3dbf   "/runnit.sh"   5 seconds ago   Created             docker_test

 ~ 
 ✔  docker start d07ba9a776cf
d07ba9a776cf

 ~ 
 ✔  docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

 ~ 
 ✔  docker ps -a
CONTAINER ID   IMAGE          COMMAND        CREATED          STATUS                     PORTS     NAMES
d07ba9a776cf   f751b30e3dbf   "/runnit.sh"   22 seconds ago   Exited (0) 7 seconds ago             docker_test
~~~

## dockered

dockered收到客户端请求后：

* 准备好容器运行的时候需要的rootfs，并将docker create时创建的容器结合起来。对于aufs来说，需要通过mount的方式将所有的layer合并起来，对于其他的文件系统来说，有些可能不需要这一步，/var/lib/docker/aufs/mnt下面已经是合并好的rootfs了。

## containerd

containerd的主要功能就是启动并管理运行时所有与的container。

准备相关文件：

containerd会创建目录/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/init并将相关文件放到这里。

> 只有当容器在运行的时候，目录/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368才存在，容器停止执行后该目录会被删除掉，下一次启动的时候会再次被创建。

~~~bash
root@dev:/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/init# file *
control:       fifo (named pipe)
exit:          fifo (named pipe)
log.json:      empty
pid:           ASCII text, with no line terminators
process.json:  ASCII text, with very long lines
shim-log.json: empty
starttime:     ASCII text, with no line terminators
~~~

- control: 用来往shim发送控制命令，包括关闭stdin和调整终端的窗口大小。
- exit：shim进程退出的时候，会关闭该管道，然后containerd就会收到通知，做一些清理工作。
- process.json：包含容器中进程相关的一些属性信息，后续在这个容器上执行docker exec命令时会用到这个文件。
- log.json: runc如果运行失败的话，会写日志到这个文件
- shim-log.json：shim进程执行失败的话，会写日志到这个文件
- pid：容器启动后，runc会将容器中第一个进程的pid写到这个文件中（外面pid namespace中的pid）
- starttime：记录容器的启动时间

## containerd启动过程

1. contianerd收到启动容器请求后，就会创建control、exit、process.json这三个文件
2. 然后启动shim进程，等着runc创建容器并将容器里第一个进程的pid写入pid文件
3. 如果containerd读取pid文件失败，则读取shim-log.json和log.json，看出了什么异常
4. 如果读取pid文件成功，说明容器创建成功，则将当前时间作为容器的启动时间写入starttime文件
5. 调用runc的start命令启动容器

## 监听容器

待容器启动之后，containerd还需要监听容器的OOM事件和容器退出事件，以便及时作出响应，OOM事件通过[cgroup的内存限制](https://segmentfault.com/a/1190000008125359)机制进行监听（通过group.event_control），而容器退出事件通过exit这个命名pipe来实现。

这其实也变相说明containerd进程是不会在容器启动之后退出的。

## shim

shim进程被containerd启动之后，第一步是设置子孙进程成为孤儿进程后由shim进程接管，即shim将变成孤儿进程的父进程，这样就保证容器里的第一个进程不会因为runc进程的退出而被init进程接管。

接着根据传入的参数设置好要启动进程的stdin,stdout,stderr（来自于上面的init-stdin，init-stdout，init-stderr），然后调用`runc create`命令创建容器，容器创建成功后，runc会将容器的第一个进程的pid写入上面containerd目录下的pid文件中，这样containerd进程就知道容器创建成功了，于是containerd接着就会调用`runc start`启动容器。

## runc

runc会被调用两次，第一次是shim调用`runc create`创建容器，第二次是containerd调用`runc start`启动容器。

## 创建容器

runc会根据参数中传入的bundle目录名称以及容器ID，创建容器.

创建容器就是启动进程`/proc/self/exe init`，由于/proc/self/exe指向的是自己，所以相当于fork了一个新进程，并且新进程启动的参数是init，相当于运行了`runc init`，`runc init`会根据配置创建好相应的namespace，同时创建一个叫exec.fifo的临时文件，等待其它进程打开这个文件，如果有其它进程打开这个文件，则启动容器。

## 启动容器

启动容器就是运行`runc start`，它会打开并读一下文件exec.fifo，这样就会触发`runc init`进程启动容器，如果`runc start`读取该文件没有异常，将会删掉文件exec.fifo，所以一般情况下我们看不到文件exec.fifo。

runc创建的容器都会在在/run/runc下有一个目录，里面有一个state.json文件（上面说到的exec.fifo这个临时文件也在这里），包含当前容器详细的配置及状态信息。对于本文中的这个容器，相应的目录为/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368

~~~bash
root@dev:/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# ls
state.json

#通过runc state命令，可以查到指定容器的相关信息
root@dev:/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# docker-runc state 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368
{
  "ociVersion": "1.0.0-rc2-dev",
  "id": "967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",
  "pid": 8001,
  "status": "running",     #刚创建时这里的状态是created，只有运行runc start之后这里才变成running
  "bundle": "/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",
  "rootfs": "/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281",
  "created": "2017-06-25T04:04:18.830443417Z"
}
~~~

































