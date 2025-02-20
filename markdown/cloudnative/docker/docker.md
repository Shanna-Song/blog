### 本文主旨
本文主要对 Docker的基础做个概述，如果你对下面几道题已经有答案了，那么就不用再看本文内容啦，分享给别人了解吧～

* 怎么根据一个镜像, 在本地跑起来一个 docker 容器
* 怎么进入到 docker 里面去看 log
* 怎么写一个 dockerfile
* 怎么将自己写的 dockerfile 变成镜像推送到公司的内部镜像源
* 多个 docker 之间怎么联动, 比如 mysql 的 docker, 业务的 docker, redis 的 docker, 这个要了解 docker-compose

如果还没有了解得很清楚的话，那就继续往下看吧～
### 背景
Docker 使用 Google 公司推出的 Go 语言进行开发实现，基于 Linux 内核的 cgroup ，namespace ，以及 OverlayFS 类的 Union FS 等技术，对进程进行封装隔离，属于`操作系统层面的虚拟化技术`。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。

传统的虚拟机是使用虚拟化技术作为应用沙盒，必须要由 Hypervisor(系统管理程序) 来负责创建虚拟机，这个虚拟机是真实存在的，并且它里面必须运行一个完整的 Guest OS 才能执行用户的应用进程。这就不可避免地带来了额外的资源消耗和占用。而Docker 提供了一种非常便利的打包机制。这种机制直接打包了应用运行所需要的整个操作系统，运行这个打包后的文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。从而保证了本地环境和云端环境的高度一致，避免了用户通过“试错”来匹配两种不同运行环境之间差异的痛苦过程。有了 Docker，就不用担心环境问题。

根据实验，一个运行着 CentOS 的 KVM 虚拟机启动后，在不做优化的情况下，虚拟机自己就需要占用 100~200 MB 内存。此外，用户应用运行在虚拟机里面，它对宿主机操作系统的调用就不可避免地要经过虚拟化软件的拦截和处理，这本身又是一层性能损耗，尤其对计算资源、网络和磁盘 I/O 的损耗非常大。

而相比之下，容器化后的用户应用，却依然还是一个宿主机上的普通进程，这就意味着这些因为虚拟化而带来的性能损耗都是不存在的；而另一方面，使用 Namespace 作为隔离手段的容器并不需要单独的 Guest OS，这就使得容器额外的资源占用几乎可以忽略不计。

### 容器 VS 虚拟机 优缺点

**优点**

“敏捷”和“高性能”是容器相较于虚拟机最大的优势，也是它能够在 PaaS 这种更细粒度的资源管理平台上流行的重要原因。

**缺点**
1. 基于 Linux Namespace 的隔离机制相比于虚拟化技术也有很多不足之处，其中最主要的问题就是：`隔离得不彻底`。

首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。

尽管你可以在容器里通过 Mount Namespace 单独挂载其他不同版本的操作系统文件，比如 CentOS 或者 Ubuntu，但这并不能改变共享宿主机内核的事实。这意味着，如果你要在 Windows 宿主机上运行 Linux 容器，或者在低版本的 Linux 宿主机上运行高版本的 Linux 容器，都是行不通的。

而相比之下，拥有硬件虚拟化技术和独立 Guest OS 的虚拟机就要方便得多了。最极端的例子是，Microsoft 的云计算平台 Azure，实际上就是运行在 Windows 服务器集群上的，但这并不妨碍你在它上面创建各种 Linux 虚拟机出来。

2. 在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：`时间`。

这就意味着，如果你的容器中的程序使用 settimeofday(2) 系统调用修改了时间，整个宿主机的时间都会被随之修改，这显然不符合用户的预期。相比于在虚拟机里面可以随便折腾的自由度，在容器里部署应用的时候，“什么能做，什么不能做”，就是用户必须考虑的一个问题。

3. 由于上述问题，尤其是共享宿主机内核的事实，容器给应用暴露出来的攻击面是相当大的，应用“越狱”的难度自然也比虚拟机低得多。
### 使用 Docker 的好处
+ 更高效的利用系统资源
+ 更快速的启动时间
+ 一致的运行环境
+ 持续交付和部署
+ 更轻松的迁移
+ 更轻松的维护和扩展

对比传统虚拟机总结
|特性 |	容器 |	虚拟机|
| ---- | ---- | ---- |
|启动	| 秒级 |	分钟级|
|硬盘使用 |	一般为 MB	| 一般为 GB|
|性能	| 接近原生 |	弱于|
|系统支持量	| 单机支持上千个容器 |	一般几十个|

### 基本概念
Docker 包括三个基本概念

+ 镜像（Image）：Docker 镜像 是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像`不包含`任何动态数据，其内容在构建之后也不会被改变。大多数 Docker 镜像是直接由一个完整操作系统的所有文件和目录构成的，所以这个压缩包里的内容跟你本地开发和测试环境用的操作系统是完全一样的。

+ 容器（Container）：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。是一种互相隔离的沙盒技术，也就相当于是一种特殊的进程，容器可以被创建、启动、停止、删除、暂停等。

+ 仓库（Repository）：镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。

一个 Docker Registry 中可以包含多个 仓库（Repository）；每个仓库可以包含多个 标签（Tag）；每个标签对应一个镜像。
#### 安装

1. 使用 Homebrew 安装
```js
$ brew install --cask docker
```
2. 手动下载安装
直接下载对应系统的版本，然后安装，https://docs.docker.com/get-docker/
#### 运行
1. 从应用中找到 Docker 图标并点击运行。
// TODO 图
2. 点击启动后，可以在 bash 面板中查看docker的版本

```js
docker --version
Docker version 20.10.8, build 3967b7d
```
如果 docker version、docker info 都正常的话，可以尝试运行一个 Nginx 服务器：
```js
$ docker run -d -p 80:80 --name webserver nginx
```
服务运行后，可以访问 http://localhost，如果看到了 "Welcome to nginx!"，就说明 Docker 安装成功了。
// TODO 图
要停止 Nginx 服务器并删除执行下面的命令：
```js
docker stop webserver
docker rm webserver
```
当然，也可以在docker的图形界面中直接操作，如图：
// TODO 图

### 基本用法
+ 获取镜像,从 Docker 镜像仓库获取镜像的命令是 `docker pull`。其命令格式为：
```js
$ docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

比如，拉取 `nginx` 镜像
// TODO 图
+ 运行，`docker run` 就是运行容器的命令,如上文的示例
```js
docker run -d -p 80:80 --name webserver nginx
```

+ 列出镜像,可以使用 `docker image ls `命令
+ 删除本地镜像,可以使用 `docker image rm `命令.其命令格式为：
```js
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

### 怎么进入到 docker 里面去看 log

我们刚刚用 nginx 启动了一个服务，我们来看看它的 log。首先，需要知道这个 docker 容器实例的 id， 我们执行 `docker ps -a` 就可以查看到：

![docker ps -a](./docker-ps-a.png)

可以看到上面列出了 CONTAINER ID，看 log 命令如下：

`docker logs a8bc73c54e8a`

![docker-logs-id](./docker-logs-id.png)

这是最基本的，如果想实时地看到日志输出，类似 tail -f 都效果怎么实现呢？只需要加个 --follow 都参数就可以了。

`docker logs --follow a8bc73c54e8a`
### 怎么写一个 dockerfile

首先，熟悉基本的 Dockerfile 指令
1. **FROM**
我们制作镜像，一般都是在基础镜像上面再进行定制的。就像我们之前运行了一个 nginx 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 `FROM` 就是指定 基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

2. **COPY**
格式：
```js
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
```
COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。比如：
```js
COPY package.json /usr/src/app/
```
3. **CMD**
CMD 指令用于执行目标镜像中包含的软件，可以包含参数。可以简单理解为项目的启动命令。

一个简单的 dockerfile 为：
```js
FROM node:14

COPY ./ /home/qspace/test/
CMD cd /home/qspace/test/ && npm run start:prod
```
更多指令可以在这里了解：https://vuepress.mirror.docker-practice.com/image/dockerfile/

### 怎么将自己写的 dockerfile 变成镜像推送到公司的内部镜像源
有时候使用 Docker Hub 这样的公共仓库可能不方便，用户可以创建一个本地仓库供私人使用。而公司一般都有自己的镜像源。
因此，首先你要知道公司的镜像源地址，假设地址为：127.0.0.1:12701
然后，可以使用 `docker tag` 来标记一个镜像，然后推送它到仓库

1. 先在本机查看已有的镜像。
```js
$ docker image ls
REPOSITORY                        TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu                            latest              ba5877dc9bec        6 weeks ago  
```
2. 使用 docker tag 将 ubuntu:latest 这个镜像标记为 127.0.0.1:12701/ubuntu:latest。

格式为 docker tag IMAGE[:TAG] [REGISTRY_HOST[:REGISTRY_PORT]/]REPOSITORY[:TAG]

```js
$ docker tag ubuntu:latest 127.0.0.1:12701/ubuntu:latest
$ docker image ls
REPOSITORY                        TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu                            latest              ba5877dc9bec        6 weeks ago         192.7 MB
127.0.0.1:12701/ubuntu:latest      latest              ba5877dc9bec        6 weeks ago
```
3. 使用 docker push 上传标记的镜像。
4. 用 curl 查看仓库中的镜像。
```js
$ curl 127.0.0.1:12701/v2/_catalog
{"repositories":["ubuntu"]}
```
这里可以看到 {"repositories":["ubuntu"]}，表明镜像已经被成功上传了。

### docker-compose
Compose 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。Compose 定位是 `定义和运行多个 Docker 容器的应用`。

我们知道使用一个 Dockerfile 模板文件，可以让用户很方便的定义一个单独的应用容器。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个 Web 项目，除了 Web 服务容器本身，往往还需要再加上后端的数据库服务容器，甚至还包括负载均衡容器等。

Compose 恰好满足了这样的需求。它允许用户通过一个单独的 docker-compose.yml 模板文件来定义一组相关联的应用容器为一个项目。

Compose 中有两个重要的概念：

`服务 (service)`：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。

`项目 (project)`：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

Compose 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。
不过，现在都流行微服务架构，很多时候，服务要scale到上百个container，并且要跨越多台机器，这时docker compose就无能为力了。一般用 k8s 来管理了。笔者这里对 docker-compose 只是有个理解，就不详细说了。感兴趣的可以看这里：https://vuepress.mirror.docker-practice.com/compose/introduction/

### 其他注意点
1. 时区
大部分 Docker 镜像都是基于 Alpine，Ubuntu，Debian，CentOS 等基础镜像制作而成。基本上都采用 UTC 时间，默认时区为零时区。
而我们主要用的是 CST 时间，北京时间，位于东八区。时区代号： Asia/Shanghai
对比一下，2个时区时间上相差 8 小时。

对于基于 Debian 基础镜像，CentOS 基础镜像制作的 Docker 镜像，在运行 Docker 容器时，传递环境变量-e TZ=Asia/Shanghai进去，能修改 docker 容器时区。只需要在 dockerFile 文件中加上环境变量即可，如：

```js
FROM node:14

ENV TZ="Asia/Shanghai"

COPY ./ /home/qspace/test/
CMD cd /home/qspace/test/ && npm run start:prod
```

2. 中文乱码
若在容器中出现中文乱码，可以检查一下是否是设置的编码格式不是 utf8.

查看docker容器编码格式：执行locale命令

解决此问题，可以在 dockerFile 文件中加上环境变量
```js
ENV LANG C.UTF-8
```

### 参考资料
+ https://vuepress.mirror.docker-practice.com/image/dockerfile/copy/
+ https://yeasy.gitbook.io/docker_practice/install/mac
