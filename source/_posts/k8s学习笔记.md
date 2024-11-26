---
title: k8s手册学习
tag: [k8s]
index_img: /img/k8s.png
date: 2024-01-12 20:00:00
category: 工具与框架
---

# K8S学习笔记

- 参考资料：[Kubernetes教程](https://github.com/guangzhengli/k8s-tutorials)

## 基础概念与配置

### 1 minikube

- 一个在本地运行Kubernetes的工具，可以在本地创建一个小型的Kubernetes集群，用于开发和测试。
- 使用docker驱动启动minikube：`minikube start --force --driver=docker`

### 2 container

  容器:如果在生产环境中运行的都是独立的单体服务容器也就够用了，但是在实际的生产环境中，维护着大规模的集群和各种不同的服务，服务之间往往存在着各种各样的关系。而这些关系的处理，才是手动管理最困难的地方。

### 3 Pod

- Pod: Kubernetes 中创建和管理的、最小的可部署的计算单元。
- Pod中可以运行多个容器，容器的本质是一个进程，pod是管理一组进程的资源
- ![Pod与Container的关系](https://camo.githubusercontent.com/befa89c06e0cc9f9618f9c1d9f2228a04eef250a6bef05cc97b7788f06f9bafd/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6775616e677a68656e676c692f50696355524c406d61737465722f755069632f706f642e706e67)
- **通过配置文件创建Pod的命令**：`kubectl apply -f XXXX.yaml`
- Pod端口转发命令：`kubectl port-forward XXXX 8080:80`
- 进入Pod内部容器shell：`kubectl exec -it XXXX -- /bin/bash`
- 查看Pod日志：`kubectl logs [--follow] XXXX`

### 4 Deployment

- Deployment: 用于定义应用的部署方式，可以控制应用的升级和回滚。
  - 在生产环境中，我们基本上不会直接管理 pod，我们需要 kubernetes 来帮助我们来完成一些自动化操作.
  - Deployment 会创建一个 ReplicaSet 来管理 Pod 的数量，当我们需要扩容或者缩容的时候，Deployment 会按照配置文件自动的去创建或者删除Pod。
  - ![Deployment与Pod的关系](https://cdn.jsdelivr.net/gh/guangzhengli/PicURL@master/uPic/deployment.png)
- 与Pod不同，在生产环境中仅需维护deployment的yaml文件的资源定义即可，无需关心具体的Pod数量和状态。
- 滚动更新(rolling update)：由于pod的版本更新会带来服务的短暂中断，滚动更新机制会在新版本pod每准备好之前不删除旧版本的pod,通过`Spec.strategy.type`字段指定更新策略:
  - `Recreate`：删除所有旧版本的pod，再创建新版本的pod
  - `RollingUpdate`:逐渐增加新版本pod，逐渐减少旧版本pod
    - `maxSurge`：最大峰值，用来指定可以创建的超出期望pod个数的pod数量
    - `maxUnavailable`：最大不可用，用来指定更新过程中不可用的 Pod 的个数上限
- 存活探针(livenessProb):用于确定什么时候要重启容器，如生产环境中的死锁、线程耗尽等导致的服务中断，存活探针检测到应用死锁时重启对应容器。
  
- ```yaml
  spec:
    containers:
      - image: yee686/hellok8s:liveness
        name: hellok8s-container
        livenessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 3  # 首次探测的等待时间
          periodSeconds: 3        # 探测的间隔时间
  ```

- 就绪探针(readinessProbe):就绪探测器可以知道容器何时准备好接受请求流量，当一个Pod内的所有容器都就绪时，才能认为该Pod就绪。这种信号的一个用途就是控制哪个Pod作为Service的后端。若Pod尚未就绪，会被从Service的负载均衡器中剔除。生产环境中kubelet使用就绪探针在pod未就绪时不会将其加入Service的负载均衡器，不接收流量。

- ``` yaml
  spec:
    containers:
      - image: yee686/hellok8s:liveness
        name: hellok8s-container
        readinessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 1
          successThreshold: 5     # 连续探测成功5次后被标记成功
          # failureThreshold: 3   # 连续探测失败3次后被标记未成功
  ```

### 5 Service

- service位于pod之前，未Pod集合提供了一个稳定的网络终结点(Endpoint)，负责接收请求并将其传递给pod；Service与Pod之间的关系是动态的，当Pod集合发生更改时Service会自动更新与之关联的Endpoint。

#### 5.1 ClusterIP

- 通过集群的内部IP暴露服务，选择该值时服务只能够在集群内部访问
- ![2024-02-27-20-47-43-手册学习笔记](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-02-27-20-47-43-手册学习笔记.png)
- ![ClusterIP工作示意图](https://cdn.jsdelivr.net/gh/guangzhengli/PicURL@master/uPic/service-clusterip-fix-name.png)

#### 5.2 NodePort

- 当k8s集群有多个节点时，通过每个节点的IP和静态端口(NodePort)暴露服务
- ![NodePort工作示意图](https://cdn.jsdelivr.net/gh/guangzhengli/PicURL@master/uPic/service-nodeport-fix-name.png)

``` yaml
  spec:
    type: NodePort
    selector:
      app: hellok8s
    ports:
    - port: 3000      # 节点端口
      nodePort: 30000 # 服务端口
  ```

### 5.3 LoadBalancer

- LoadBalancer是使用云提供商的负载均衡器向外部暴露服务.外部负载均衡器可以将流量路由到自动创建的NodePort服务和ClusterIP服务上.
- ![LoadBalancer](https://cdn.jsdelivr.net/gh/guangzhengli/PicURL@master/uPic/service-loadbalancer-fix-name.png)

## 6 Ingress

- Ingress公开从集群外部到集群内服务的HTTP和HTTPS路由,流量路由由 Ingress资源上定义的规则控制。
- 可以将其理解为服务的网关，流量经过Ingress后重定向到对应的Service上。
- `minikube addons enable ingress`启用Ingress

## 7 Namespace

- namespace命名空间机制用于将统一集群资源划分为多个相互独立的组
- 命名空间作用域仅针对带有名字空间的对象。
- 在管理时使用`-n namespace`制定命名空间

## 8 ConfigMap

- ConfigMap用于将配置数据喝应用程序代码分开，保存到键值对中，ConfigMap保存的数据不超过1mb，否则需要挂载存储卷

``` yaml
  data:
    DB_URL: "http://DB_ADDRESS_Dev"
```

- 使用`kubectl get configma`命令查看ConfigMap

``` yaml
  spec:
    containers:
      - name: hellok8s-container
        image: yee686/hellok8s:v4
        env:
          - name: DB_URL  # 从ConfigMap中获取DB_URL
            valueFrom:
              configMapKeyRef:
                name: hellok8s-config
                key: DB_URL
```

## 8 Job

- Job时一次性任务，Job会创建一个或多个Pod，直到成功完成任务为止
- `completion`字段指定任务完成的Pod数量
- `parallelism`字段指定同时运行的Pod数量
- CronJob是定时任务通过在`schedule`字段使用Cron表达式指定任务执行时间

``` yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: hellok8s-job
  spec:
    completions: 1
    parallelism: 1
    template:
      spec:
        containers:
        - name: hellok8s-container
          image: yee686/hellok8s:v4
          command: ["echo", "hello k8s"]
        restartPolicy: OnFailure
```

## 附录：问题与解决

- [无法访问gcr.io的几种解决办法](https://www.cnblogs.com/tylerzhou/p/10971341.html)
- kubectl拉取镜像默认是从docker hub仓库而不是本地docker仓库，若要拉取本地仓库的镜像，需要在镜像名前加上`localhost:5000/`前缀或`myregistry.com/myimage:tag`