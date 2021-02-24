# docker 自身问题

## docker runc cve-2019-5736

docker exec指令背后执行的runc，runc实现的原理是附加到容器的pid namespace，然后再创建一个子进程，在子进程中调用exec来执行想要在容器中执行的程序，因为runc会附加到容器的pid namespace所以就可以在容器内的proc目录下找到runc对应的pid文件，也就可以通过这个进程文件来写runc文件，然后向runc文件中写入恶意代码，再下次用户再使用exec命令就可以执行到恶意代码，反弹回一个shell，从而实现逃逸。影响docker 18.09以下版本，runc1.0.0.rc9以下版本

## docker自身问题

当Docker宿主机使用cp命令时，会调用辅助进程docker-tar，其原理是利用chroot对文件，将请求的文件和目录放在其中，然后将生成的 tar 文件传递回 Docker 守护程序，该守护程序负责将其提取到宿主机的目标目录中，在由Go v1.11编译而成的Docker版本中某些包含嵌入式 C 语言代码（cgo）的软件包在运行时动态加载共享库。这些软件包包括`net`和`os/user`，都会被`docker-tar`使用，它们会在运行时加载多个`libnss_*.so`库，正常情况下这些库会从宿主机的文件系统里面进行加载，但是因为docker-tar会chroot到容器中，因此它会从容器的文件系统里面加载库，这就意味着docker-tar会加载并执行容器控制和创建的文件，同时docker-tar并没有被容器化，它以root权限运行在宿主机namespace里面，并且不会受到cgroup和seccomp的限制，*黑客可以通过在容器中替换libnss*.so等库，将代码注入到docker-tar中。当Docker用户尝试从容器中拷贝文件时将会执行恶意代码，成功实现Docker逃逸，获得宿主机root权限。影响版本：Docker 19.03.0