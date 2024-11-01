---
title: Docker笔记
date: 2018-06-15 22:09:50
categories:
    - docker
tags:
    - docker
---

# Docker笔记

# 操作系统虚拟化

Operating-system-level virtualization, 又叫做容器化（containerization），指的是操作系统提供了这样一个功能，内核允许存在多个隔离的用户空间实例，每个实例叫做一个容器（container）。

Docker是一个容器化平台。

| container | vm |
| --- | --- |
| ![15226880999438](/images/2018/15226880999438.png)| ![15226881122546](/images/2018/15226881122546.png)|

* 容器是应用层上的抽象，容器之间共享操作系统内核。
* vm是硬件抽象，每个vm不仅仅包含要允许的程序，还包括了一个完整的os。

# 基本概念

## 镜像（Image）

镜像是由一系列的只读层构成的，每层代表了对已有镜像的修改。相当于一个root文件系统。由一个sha256作为镜像ID**唯一标识**，一个镜像可以有**多个标签**。

对已有镜像中文件的修改，都是以Copy-on-Write的方式进行，例如：删除文件，在新的一层中进行标记删除，原有层的文件不变。

### scratch镜像

scratch镜像代表一个空白的镜像，这是一个虚拟的镜像，不实际存在，意味着基于这个构建的新镜像不使用任何已有的基础镜像。

```
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
...
```

### 空悬镜像（Dangling image）

空悬镜像是没有tag，没有被其他镜像依赖的镜像。当创建一个新的镜像时，使用了一个已有的tag，那么老的镜像就会变为空悬镜像。可被删除。

```bash
docker image ls -f dangling=true
docker image prune
```

### Docker build context

Docker是c/s架构的，实际的构建动作发生在server。当`ADD`文件到镜像中时，server需要知道这个文件，具体的方式都是通过打包构建时的上下文，并发送给server。

```bash
docker build [OPTIONS] PATH | URL | -
```

* `PATH`，即上下文；
* 若`URL`是一个压缩包，那么其内容会被当做上下文。

### Untagged和Delete

镜像的多层结构使得层与层之间存在依赖，在进行删除操作时，需要考虑这些的依赖关系。当某层没有被依赖时，才可以被删除。

删除一个镜像时，实际上是删除镜像的标签（Untagged），当一个镜像的所有标签都被删除了，才会触发删除这个镜像（Delete）。

如果有容器存在，同样不可以删除这个镜像，因为这个镜像被一个可读写层依赖。

## 容器（Container）

容器是镜像运行的实体，本质上是一个进程，拥有自己的独立的命名空间。在存储上与镜像的区别在于多了最上面的一个读写层，这个层是容器的运行态环境，生命周期与容器一致，**不建议**在这一层写入数据。

![15227153999553](/images/2018/15227153999553.jpg)

## 仓库（Registry）

Docker仓库（Docker Registry）是存储和分发镜像的服务，包括多个Repository；每个Repository包含多个标签，通过`<仓库名>:<标签>`的方式来指定一个镜像。

# 数据管理

* 数据卷（Volumes）

    数据卷独立于镜像和容器，可在多个容器之间共享，容器终止后，数据卷也会一直存在。

* 挂载主机目录（Bind mounts）

# 网络管理

Docker的网络是通过可插拔的驱动来提供的。除了可以使用Docker已有的驱动，还能安装第三方网络驱动。

## bridge

![15261916795186](/images/2018/15261916795186.jpg)

bridge网络是在链路层用于在不同网段之间转发流量的。bridge可以是一个硬件或软件设备。

bridge驱动创建了主机内部的私有网络，在这个网络中的容器可以相互通信，可以通过端口映射来实现外部的访问。docker通过规则在禁止不同bridge网络之间的通信。

docker的bridge是local scope的，这意味着一个bridge网络只能提供单机上的容器互联。docker提供了默认的bridge，用户也可以添加自定义的bridge。

user-defined bridge和default bridge略有不同，user-defined bridge提供了如下功能：

* 连接到user-defined bridge的所有容器，都会自动的相互暴露所有端口。
* 提供了容器之间的自动DNS解析，可以直接通过host name来进行访问其他容器。
* 创建了可配置的bridge（默认的bridge也可以配置，但是这个操作是在docker之外进行的，需要重启docker）。

默认的bridge特别的一点是，可以通过`--link`来在容器之间共享环境变量，但`--link`已被遗弃，user-defined bridge可以通过共享文件或目录、compose file来实现共享环境变量。

## overlay

![15261917479037](/images/2018/15261917479037.jpg)

## host

使用host网络时，容器的网络栈没有与主机的隔离（存储、进程namespace、用户namespace都有隔离），如果在容器绑定到了80端口，那么容器的程序在主机的80端口可用。这也就意味着，绑定前要求主机的80端口未被占用。

host网络仅在linux上可用，*猜测*mac和windows上不可用是由于使用了虚拟化。

## macvlan

![15261917609686](/images/2018/15261917609686.jpg)

## none

# Docker上部署Hadoop

1. [Docker上部署Hadoop等完全分布式集群](https://flat2010.github.io/2017/08/12/Docker%E4%B8%8A%E9%83%A8%E7%BD%B2Hadoop%E7%AD%89%E5%AE%8C%E5%85%A8%E5%88%86%E5%B8%83%E5%BC%8F%E9%9B%86%E7%BE%A4/)
2. [UNDERSTANDING DOCKER NETWORKING DRIVERS AND THEIR USE CASES](https://blog.docker.com/2016/12/understanding-docker-networking-drivers-use-cases/)
3. [Docker —— 从入门到实践](https://yeasy.gitbooks.io/docker_practice/)

# References

1. [Docker Documentation](https://docs.docker.com/)
