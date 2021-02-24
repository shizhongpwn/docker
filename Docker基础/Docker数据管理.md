# Docker数据管理

> 参考链接：https://www.escapelife.site/posts/c2e250ea.html
>
> 默认容器的数据是保存在容器的可读可写层，当容器被删除的时候其上的数据也会丢失，官方提供了三种方式来实现数据的持久保存，分别是：`Volumes`、`Bind mounts`和`tmpfs`。

* Bind mount:覆盖容器中的文件
* Volume mount:容器中的文件同步到主机的目录上。