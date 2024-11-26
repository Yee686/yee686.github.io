---
title: docker手册学习
tag: [docker]
index_img: /img/docker.png
date: 2023-11-20 10:00:00
category: 工具与框架
---

# Docker使用

## 1 docker

### 基本概念

- 镜像image相当于特殊的独立的`root`文件系统，不包含任何动态数据，使用分层存储的方式
- 容器container相当于镜像的实例，是一个由独立命名空间的进程，在容器存储层的读写都会岁容器删除和丢失，需要持久化，可使用数据卷(volume)或绑定宿主机目录
- 仓库repository相当于代码仓库，用于镜像的集中存储和分发

### 镜像使用

- 镜像拉取
  `docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]`
- 运行容器`docker run`
- 列出镜像`docker image ls`
- 删除镜像`docker image rm [选项] <镜像1> [<镜像2> ...]`
- 提交更改`docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]`，提交对容器的更改保存到新的镜像中，新的镜像名为`<仓库名>[:<标签>]`

### Dockerfile定制镜像

- `FROM`指定基础镜像，必须为第一条指令,`FROM scratch`表示不基于任何镜像
- `RUN`执行命令
  - shell格式`RUN <命令>`，exec格式`RUN ["可执行文件", "参数1", "参数2"]`
  - 每执行一个RUN就是新建一层
- `COPY`复制文件
  - `COPY [--chown=<user>:<group>] <源路径>... <目标路径>`
  - `COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]`
  - `--chown`用于改变文件所属用户和用户组
- `ADD`复制文件并可以实现自动解压缩
- `CMD`容器启动命令，指定容器主进程的启动命令
- `ENTRYPOINT`入口点，指定容器启动程序和参数，可以用来做运行前的准备工作
- `ENV`环境变量设置，`ENV <key1>=<value1> <key2>=<value2>...`
- `ARG`构建参数，与`ENV`效果类似，但是容器运行起来后无法看到这些参数
- `VOLUME`定义匿名卷，`VOLUME ["<路径1>", "<路径2>"...]`，指定目录挂在为匿名卷，持久化存储所产生的数据
- `EXPOSE`暴露端口，`EXPOSE <端口1> [<端口2>...]`，仅仅声明容器运行时提供服务的端口，在容器运行时不会开启这个端口的服务
  - `-p <宿主端口>:<容器端口>`是映射宿主端口和容器端口，
- `WORKDIR`指定工作目录，各层的当前目录就被改为指定的目录
- 构建镜像`docker build [选项] <上下文路径/URL/->`
- 由于Docker在运行时分为Docker守护进程和客户端，docker build构建镜像由docker守护进程实现，即使用Docker引擎提供的Remote API，因此在构建镜像时，需要使用**上下文路径**来让守护进程获取本地文件的目录，`Dockerfile`一般情况下应该放在项目根目录或一个空目录中，可以使用`.dockerignore`剔除不需要的文件

### 操作容器

- 启动容器:`docker run -it <容器名>`，`-i`表示交互式操作，`-t`表示启用虚拟终端
- 启动已终止的容器:`docker container start <容器名>`
- 守护态运行:`docker run -d <容器名>`，在后台运行容器
- 终止容器:`docker container stop <容器名>`
- 进入容器:`docker exec <容器名>`，也可使用`-it参数`，exec进入容器时退出不会终止
- 导出导入容器:`docker export`和`docker import`
- 删除容器:`docker container rm <容器名>`删除一个终止状态的容器

## 2 数据管理

### 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过UnionFS，对数据卷的更新会立即生效，不影响镜像，且可以在多个容器之间共享和重用

- 创建数据卷`docker volume create <数据卷名>
- 查看数据卷`docker volume ls`
- 查看指定数据卷的详细信息`docker volume inspect <数据卷名>`
- 启动一个挂载数据卷的容器`docker run -it -v <数据卷名>:<容器目录> <镜像名>`
- 查看指定容器的数据卷信息`docker inspect <容器名>`
- 删除数据卷`docker volume rm <数据卷名>`
- 删除未使用的数据卷`docker volume prune`
- 挂载主机目录示例

``` shell
  docker run -d -P \
    --name web \
    --mount source=my-vol,target=/usr/share/nginx/html \
    nginx:alpine
```

### 挂载主机目录

- 使用`--mount`标记可以指定挂载一个本地主机的目录到容器中去

``` shell
  docker run -d -P \
      --name web \
      --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html \
      nginx:alpine
```

## 3 Kubernetes

> Kubernetes 是 Google 团队发起的开源项目，它的目标是管理跨多个主机的容器，提供基本的部署，维护以及应用伸缩，主要实现语言为 Go 语言。

### 相关概念

- 节点(NODE):运行在K8s中的主机
- 容器组(POD):一个Pod对应于由若干容器组成的容器组，同个组内的容器共享一个存储卷(volume)
- 容器组生命周期(pos-states):包含所有容器状态集合，包括容器组状态类型，容器组生命周期，事件，重启策略，以及replication controllers
- 服务(services):一个K8s服务是容器组逻辑的高级抽象，同时也对外提供访问容器组的策略
- 卷(volumes):一个卷就是一个目录，容器对其有访问权限

### 架构设计

#### 控制平面

- 主节点服务
  - `apiserver`系统的对外接口，提供API供客户端和其他组件调用
  - `scheduler`对资源进行调度，分配某个Pod到某个节点上运行
  - `controller-manager`管理控制器，
- ETCD:既是数据后端，又作为消息中间件，存储主节点的状态消息，以实现主节点的分布式扩展，组件通过监听ETCD数值变化获得通知

#### 工作节点

- `kubelet`工作节点执行操作的agent，负责具体容器的生命周期管理，
- `kube-proxy`网络代理，也可实现负载均衡，将具体服务请求分配到工作节点上。

## 4 Docker compose

`Compose`项目是Docker官方的开源项目，负责实现对Docker容器集群的快速编排,允许用户通过一个单独的`docker-compose.yml`模板文件(YAML格式)来定义一组相关联的应用容器为一个项目

- 服务(service):一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
- 项目(project):由一组关联的应用容器组成的一个完整业务单元，在`docker-compose.yml`文件中定义。