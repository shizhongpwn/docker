# image镜像是什么

## 内容

一个image由[manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md) , [image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md)(可选) ， [filesystem layers](https://github.com/opencontainers/image-spec/blob/master/layer.md) 和 [configuration](https://github.com/opencontainers/image-spec/blob/master/config.md)四部分组成。

![image-20210219200016341](image镜像是什么.assets/image-20210219200016341.png)

- mage Index和Manifest的关系是"1..*"，表示它们是一对多的关系
- Image Manifest和Config的关系是"1..1"，表示它们是一对一的关系
- Image Manifest和Filesystem Layers是一对多的关系

## Filesystem Layers

Filesystem Layer包含了文件系统的信息，即该image包含了那些文件/目录，以及它们的属性和数据。

包含的内容：

每个filesystem layer都包含了在上一个layer上的改动情况。主要包含三个方面：

- 变化类型：是增加、修改还是删除了文件
- 文件类型：每个变化发生在哪种文件类型上
- 文件属性：文件的修改时间、用户ID、组ID、RWX权限等

比如在某一层增加了一个文件，那么这一层所包含的内容就是增加的这个文件的数据以及它的属性，具体的细节请参考[标准文档](https://github.com/opencontainers/image-spec/blob/master/layer.md)。











