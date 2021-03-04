# K8s学习笔记(kubernetes in action)

## 第二章	开始使用k8s和Docker

### Docker相关

#### 安装Docker并运行Hello World

安装必要的包

$ yum install -y yum-utils device-mapper-persistent-data lvm2

设置yum源

$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

安装docker

$ yum install docker-ce

设置docker镜像源

```bash
$ vim /etc/docker/daemon.json 
{
	"registry-mirrors":[
	"https://cr.console.aliyun.com/",
	"https://docker.mirrors.ustc.edu.cn",
	"https://registry.docker-cn.com",
	"http://hub-mirror.c.163.com"],
#	"insecure-registries":["127.0.0.1:5000",
#	"192.168.122.58:5000",
#	"0.0.0.0:5000"]
}
```

创建一个简单的镜像：

$ docker run busybox echo  "hello world"

#### 创建一个简单的Node.js

首先创建一个Dockerfile文件，将app.js和他放在同一个目录下，然后执行命令：

$ docker build -t kubia.

在当前目录下构建一个kubia镜像，Docker会在目录中找Dockerfile,然后基于其中的指令开始构建镜像

#### 列出本地存储的镜像

$ docker images

#### 运行容器镜像

$ docker run --name kubia-container -p 8080:8080 -d kubia

创建一个叫做kubia-container的新容器，这个容器与命令行分离（-d参数），也就是后台运行，本机的8080端口会被映射到容器内的8080端口（-p 8080:8080选项），可以通过http://localhost:8080访问

#### 列出所有正在运行的容器

$ docker ps

Docker会打印出每一个正在运行的容器ID、名称、所使用镜像以及容易起中执行的命令

#### 获取更多容器信息

$ docker inspect kubia-container

Docker会打印出包含容器底层信息的长JSON

#### 在已有容器中运行shell

$ docker exec -it kubia-container bash

会在已有的kubia-container容器内部运行bash,bash进程会和主容器进程拥有相同的命名空间，这样就可以从内部探索容器。

-it参数的意义：

-i：确保标准输入流保持开放，因为需要在shell中输入命令

-t：分配一个伪终端（tty）

如果需要像平常一样在容器内使用bash，-it参数缺一不可

容器内进程ID和主机上不同，这是因为容器使用独立的PID linux命名空间，容器有着独立系列号，完全独立于进程树，文件系统也是独立的

#### 停止容器

$ docker stop kubia-container

#### 删除容器

$ docker rm kubia-container

#### 给镜像附加标签

$ docker tag kubia ck747579338/kubia

这样做不会重命名，而是给同一个镜像创建一个额外的标签，可以通过docker images查看镜像，发现kubia 和ck747579338/kubia指向的是同一个镜像ID

#### 向镜像仓库推送镜像

首先要在机器上登陆过docker（注册网址http://hub.docker.com）

$ docker login

输入账号密码

$ docker push ck747579338/kubia

向docker hub推送镜像

#### 不创建容器直接运行镜像

$ docker run -p 8080:8080 -d kubia

### 配置K8s集群

#### 用minikube运行本地单节点K8s集群

安装minikube步骤：

下载安装包

$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm

使用rpm安装

$ rpm -ivh minikube-latest.x86_64.rpm 

使用minikube命令时，不能是root账户，启动一个kubernetes集群

启动前需要将该账户添加至docker组

$ usermod -aG docker chenkai

$ newgrp docker

$ minikube start --driver=docker --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers

#### 安装kubernetes客户端（kubectl）

首先配置k8s源

$ vim /etc/yum.repos.d/kubernetes.repo

添加以下内容：

```bash
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

安装kubectl

$ yum -y install kubectl

安装完成后查看kubectl的版本

$ kubectl version

使用kubectl查看集群是否正常工作

$ kubectl cluster-info

列出节点查看集群是否在工作

$ kubectl get nodes

查看节点的更多信息

$ kubectl describe node ${节点名} 

将会显示节点的状态、CPU和内存数据、系统信息、运行容器的节点等信息

如果使用$ kubectl describe node将会显示所有节点的信息

kubectl命令行自动补齐

$ source <(kubectl completion bash)

执行后就能实现 kubectl cl<tab> desc<tab>等命令的补全

### 在Kubernetes运行应用

#### 部署Node.js应用

因为是第一次部署，所以采用最简单的方法部署kubernetes应用，通常都需要准备一个yaml或者json文件，包含想要部署的所有组件的配置文件

部署一个busybox应用

```bash
$ kubectl run kubia --image=ck747579338/kubia --port=8080
pod/kubia created
```

查看Pod状态

```bash
$ kubectl get pods
NAME    READY   STATUS              RESTARTS   AGE
kubia   0/1     ContainerCreating   0          13s
```

busybox处于挂起状态0/1，正在创建（ContainerCreating），等容器镜像相爱在完毕后，状态才会变成1/1

```bash
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
kubia   1/1     Running   0          29m
```

查看pod的详细信息，详细过程类似下图

![运行过程](/home/chenkai/.config/Typora/typora-user-images/image-20210303215331467.png)

$ kubectl describe pods (busybox)

如果只有一个pod那么就可以不加括号里的内容

#### 访问Web应用

每个pod都有自己的地址，但是这个地址是集群内部的，不能从集群外部访问，需要创建一个特殊的LoadBalancer类型的服务，通过这个服务将创建一个外部的负载均衡，可以通过负载均衡的公共IP访问pod

$ kubectl expose rc kubia --type=LoadBalancer --name kubia-http

缩写pod (po), service (svc), replicationcontroller (rc), deployment (deploy), replicaset (rs)

列出服务

$ kubectl get services

这样就能看到外部IP,就可以从外部访问应用

注意：minikube不支持LoadBalancer类型的服务，因此不会产生外部IP,按上述操作也可能会报错，可以参考如下

```bash
$ kubectl expose pod kubia --port=9090 --name=kubia-http
service/kubia-http exposed
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    45h
kubia-http   ClusterIP   10.107.83.154   <none>        9090/TCP   37s
```

