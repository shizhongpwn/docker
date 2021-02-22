# 什么是容器的runtime

> https://segmentfault.com/a/1190000009583199
>
> docker负责准备runtime的bundle，而runc负责运行这个bundle并管理整个容器的生命周期。
>
> 但对于docker来说，并不是只要准备好根文件系统和配置文件就可以了，比如对于网络，runtime没有做任何要求，只要在config.json中指定network namespace就行了（不指定就新建一个），而至于这个network namespace里面有哪些东西则完全由docker负责，docker需要保证新network namespace里面有合适的设备来和外界通信。

runc是runtime的一个实现，runc和image一样，也有标准，并且由OCI维护，[Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)

## 规范内容

在Linux平台上，跟runtime有关的规范主要有四个，分别是[Runtime and Lifecycle (runtime.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)、[Container Configuration file (config.md)](https://github.com/opencontainers/runtime-spec/blob/master/config.md)、[Linux Container Configuration (config-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)和[Linux Runtime (runtime-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md) .

除了这四个之外，还有一个[Filesystem Bundle (bundle.md)](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)。

## Filesystem Bundle

一个例子：

~~~
dev@debian:~/images$ tree hello-world-bundle
hello-world-bundle
├── config.json
└── rootfs
    └── hello

1 directory, 2 files
~~~

bundle包含一个容器运行所需要的全部信息，有这个bundle之后，符合runtime标准的程序（比如runc）就可以根据bundle启动容器了。

bundle包含一个config.json文件和容器的根文件系统目录，config.json就是后面要介绍的`Container Configuration file`，标准要求该配置文件必须叫这个名字，不过对容器的根文件系统目录没有要求，只要在config.json里面将路径配置正确就可以了，不过一般约定俗成都叫rootfs。

实际使用过程中，根文件系统目录可能在其它的地方，只要config.json里面配置正确的路径就可以了，但如果bundle需要打包和其它人分享的话，必须将根文件系统和config.json打包在一起，并且不包含外层的文件夹。

## [Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)

这个无非就是注意一下config文件里面的各个字段的内容。

- ociVersion（必须）：对应的OCI标准版本
- root（必须）：根文件系统的位置
- mounts：需要挂载哪些目录到容器里面。如果是Linux平台，这里面必须要包含/proc、/sys，/dev/pts，/dev/shm这四个目录
- process：容器启动后执行什么命令
- hostname：容器的主机名，相关原理可参考[UTS namespace (CLONE_NEWUTS)](https://segmentfault.com/a/1190000006908598)
- platform（必须）：平台信息，如 amd64 + Linux
- linux：Linux平台的特殊配置，这里包含下面要介绍的`Linux Container Configuration`里面的内容
- hooks：配置容器运行生命周期中会调用的hooks，包括prestart、poststart和poststop，容器的生命周期见后面`Runtime and Lifecycle`介绍。
- annotations：容器的注释，相当于容器标签，key:value格式

### [Linux Container Configuration](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)

该规范是Linux平台上对[Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)的扩展，这部分的内容也包含在上面的config.json文件中。

这部分也挺无聊的，就是字段含义：

- namespaces: namespace相关的配置，相关原理可参考[Namespace概述](https://segmentfault.com/a/1190000006908272)及这些namespace（[UTS](https://segmentfault.com/a/1190000006908598)、[IPC](https://segmentfault.com/a/1190000006908729)、[mount](https://segmentfault.com/a/1190000006912742)、[pid](https://segmentfault.com/a/1190000006912878)、[network](https://segmentfault.com/a/1190000006912930)、[user 1](https://segmentfault.com/a/1190000006913195)、[user 2](https://segmentfault.com/a/1190000006913499)）。
- uidMappings，gidMappings：配置主机和容器用户/组之间的对应关系，原理可参考[user namespace](https://segmentfault.com/a/1190000006913195)
- devices：设置哪些设备可以在容器内被访问到。除了这里指定的设备外，/dev/null、/dev/zero、/dev/full、/dev/random、/dev/urandom、/dev/tty、/dev/console（如果在process的配置里面启动terminal的话）和/dev/ptmx这些设备默认就能在容器内访问到，即runtime的实现需要默认将这些设备bind到容器内，dev/tty和/dev/ptmx的原理可以参考[TTY/PTS概述](https://segmentfault.com/a/1190000009082089)
- cgroupsPath：cgroup的路径，可参考[Cgroup概述](https://segmentfault.com/a/1190000006917884)
- resources：Cgroup中具体子项的配置，包括[device whitelist](https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt), [memory](https://segmentfault.com/a/1190000008125359), [cpu](https://segmentfault.com/a/1190000008323952), [blockIO](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt), [hugepageLimits](https://www.kernel.org/doc/Documentation/cgroup-v1/hugetlb.txt), network([net_cls cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/net_cls.txt)和[net_prio cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/net_prio.txt)), [pids](https://segmentfault.com/a/1190000007468509)
- intelRdt：和[Intel Resource Director Technology](https://www.kernel.org/doc/Documentation/x86/intel_rdt_ui.txt)有关
- sysctl：调整容器运行时的kernel参数，主要是一些网络参数，因为每个network namespace都有自己的协议栈，所以可以修改自己协议栈的参数而不影响别人
- seccomp：和安全相关的配置，见[Seccomp](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt)
- rootfsPropagation：设置Propagation类型。可以参考[Shared subtrees](https://segmentfault.com/a/1190000006899213)
- maskedPaths：设置容器内的哪些目录对用户不可见
- readonlyPaths：设置容器内的哪些目录是只读的
- mountLabel：和Selinux有关。

## [Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)

也是一个规范，规定了容器运行时相关的三个部分：

* 容器的状态  //这个就是规定了查询容器状态的时候必须返回什么信息，具体的直接看最上面的链接。
* 容器相关的操作 //该部分定义了一个符合runtime标准的实现，比如至少实现以下命令。
  * state: 返回容器的状态，包含上面介绍的那些内容.
  * create： 创建容器，这一步执行完成后，容器创建完成，修改bundle中的config.json将不再对已创建的容器产生影响
  * start： 启动容器，执行config.json中process部分指定的进程
  * kill： 通过给容器发送信号来停止容器，信号的内容由kill命令的参数指定
  * delete： 删除容器，如果容器正在运行中，则删除失败。删除操作会删除掉create操作时创建的所有内容。
* 容器的生命周期

以runc为例子观看容器的声明周期

1. 执行命令`runc create`创建容器，参数中指定bundle的位置以及容器的ID，容器的状态变为creating
2. runc根据bundle中的config.json，准备好容器运行时需要的环境和资源，但不运行process中指定的进程，这步执行完成之后，表示容器创建成功，修改config.json将不再对创建的容器产生影响，这时容器的状态变成created。
3. 执行命令`runc start`启动容器
4. runc执行config.json中配置的prestart钩子
5. runc执行config.json中process指定的程序，这时容器状态变成了running
6. runc执行poststart钩子。
7. 容器由于某些原因退出，比如容器中的第一个进程主动退出，挂掉或者被kill掉等。这时容器状态变成了stoped
8. 执行命令`runc delete`删除容器，这时runc就会删除掉上面第2步所做的所有工作。
9. runc执行poststop钩子

## [Linux Runtime](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md)

该规范是Linux平台上对[Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)的补充，目前该规范很简单，只要求容器运行起来后，里面必须建立下面这些软连接：

~~~bash
# ls -l /dev/fd /dev/std*
lrwxrwxrwx    1 root     root            13 May  4 12:32 /dev/fd -> /proc/self/fd
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stderr -> /proc/self/fd/2
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stdin -> /proc/self/fd/0
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stdout -> /proc/self/fd/1
~~~











