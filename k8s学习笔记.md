# K8s学习笔记(kubernetes in action)

## 第二章	开始使用k8s和Docker

### Docker相关

#### 安装Docker并运行Hello World

安装必要的包

```bash
$ yum install -y yum-utils device-mapper-persistent-data lvm2
```

设置yum源

```bash
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

安装docker

```bash
$ yum install docker-ce
```

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

```bash
$ docker run busybox echo  "hello world"
```

#### 创建一个简单的Node.js

首先创建一个Dockerfile文件，将app.js和他放在同一个目录下，然后执行命令：

```bash
$ vim app.js 
const http = require('http');
const os = require('os');
console.log("Kubia server starting...");
var handler = function(request, response){
	console.log("Received request from " + request.connection.remoteAddress);
	response.writeHead(200);
	response.end("You've hit" + os.hostname() + "\n");	
};

var www = http.createServer(handler);
www.listen(8080);
```

```bash
$ vim Dockerfile 
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node","app.js"]
```

```bash
$ docker build -t kubia .
```

在当前目录下构建一个kubia镜像，Docker会在目录中找Dockerfile,然后基于其中的指令开始构建镜像

#### 列出本地存储的镜像

```bash
$ docker images
```

#### 运行容器镜像

```bash
$ docker run --name kubia-container -p 8080:8080 -d kubia
```

创建一个叫做kubia-container的新容器，这个容器与命令行分离（-d参数），也就是后台运行，本机的8080端口会被映射到容器内的8080端口（-p 8080:8080选项），可以通过http://localhost:8080访问

#### 列出所有正在运行的容器

```bash
$ docker ps
```

Docker会打印出每一个正在运行的容器ID、名称、所使用镜像以及容易起中执行的命令

#### 获取更多容器信息

```bash
$ docker inspect kubia-container
```

Docker会打印出包含容器底层信息的长JSON

#### 在已有容器中运行shell

```bash
$ docker exec -it kubia-container bash
```

会在已有的kubia-container容器内部运行bash,bash进程会和主容器进程拥有相同的命名空间，这样就可以从内部探索容器。

-it参数的意义：

-i：确保标准输入流保持开放，因为需要在shell中输入命令

-t：分配一个伪终端（tty）

如果需要像平常一样在容器内使用bash，-it参数缺一不可

容器内进程ID和主机上不同，这是因为容器使用独立的PID linux命名空间，容器有着独立系列号，完全独立于进程树，文件系统也是独立的

#### 停止容器

```bash
$ docker stop kubia-container
```

#### 删除容器

```bash
$ docker rm kubia-container
```

#### 给镜像附加标签

```bash
$ docker tag kubia ck747579338/kubia
```

这样做不会重命名，而是给同一个镜像创建一个额外的标签，可以通过docker images查看镜像，发现kubia 和ck747579338/kubia指向的是同一个镜像ID

#### 向镜像仓库推送镜像

首先要在机器上登陆过docker（注册网址http://hub.docker.com）

```bash
$ docker login
```

输入账号密码

```bash
$ docker push ck747579338/kubia
```

向docker hub推送镜像

### 配置K8s集群

#### 用minikube运行本地单节点K8s集群

安装minikube步骤：

下载安装包

```bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
```

使用rpm安装

```bash
$ rpm -ivh minikube-latest.x86_64.rpm 
```

使用minikube命令时，不能是root账户，启动一个kubernetes集群

启动前需要将该账户添加至docker组

```bash
$ usermod -aG docker chenkai
$ newgrp docker
$ minikube start --driver=docker --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

#### 安装kubernetes客户端（kubectl）

首先配置k8s源

```bash
$ vim /etc/yum.repos.d/kubernetes.repo
```

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

```bash
$ yum -y install kubectl
```

安装完成后查看kubectl的版本

```bash
$ kubectl version
```

使用kubectl查看集群是否正常工作

```bash
$ kubectl cluster-info
```

列出节点查看集群是否在工作

```bash
$ kubectl get nodes
```

查看节点的更多信息

```bash
$ kubectl describe node ${节点名} 
```

将会显示节点的状态、CPU和内存数据、系统信息、运行容器的节点等信息

如果使用$ kubectl describe node将会显示所有节点的信息

#### kubectl命令行自动补齐

```bash
$ yum install -y bash-completion
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
$ source ~/.bashrc
```

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

```bash
$ kubectl describe pods (busybox)
```

如果只有一个pod那么就可以不加括号里的内容

#### 访问Web应用

每个pod都有自己的地址，但是这个地址是集群内部的，不能从集群外部访问，需要创建一个特殊的LoadBalancer类型的服务，通过这个服务将创建一个外部的负载均衡，可以通过负载均衡的公共IP访问pod

```bash
$ kubectl expose rc kubia --type=LoadBalancer --name kubia-http
```

缩写pod (po), service (svc), replicationcontroller (rc), deployment (deploy), replicaset (rs)

列出服务

```bash
$ kubectl get svc
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          11m
kubia-http   LoadBalancer   10.106.203.149   <pending>     8080:32164/TCP   11s
```

这样就能看到外部IP,就可以从外部访问应用

注意：minikube不支持LoadBalancer类型的服务，因此不会产生外部IP,一直处于挂起状态,可以使用minikube自带的命令$ minikube service <service-name>，然后会自动打开浏览器访问，或者使用curl $(minikube service <service-name> --url)命令,可以看出每个pod会将pod名作为主机名

```bash
$ minikube service kubia-http
|-----------|------------|-------------|---------------------------|
| NAMESPACE |    NAME    | TARGET PORT |            URL            |
|-----------|------------|-------------|---------------------------|
| default   | kubia-http |        8080 | http://192.168.49.2:32164 |
|-----------|------------|-------------|---------------------------|
🎉  正通过默认浏览器打开服务 default/kubia-http...
This tool has been deprecated, use 'gio open' instead.
See 'gio help open' for more info.
#浏览器中显示You've hitkubia
$ curl $(minikube service kubia-http --url)
You've hitkubia
```

#### 异常情况

我的pod service都能正常创建，从minikube中访问该应用也正常，但是无法从主机上访问，并且$ minikube dashboard 命令提示503的报错，那么需要删除用户目录下的 .kube 和 .minikube目录，执行 $ minikube delete命令删除集群，然后重新建立集群，就可以正常访问该IP端口了，并且$ minikube dashboard也不会报503的错误

#### ReplicationController、pod和服务之间的关系

![rc pod svc](/home/chenkai/.config/Typora/typora-user-images/image-20210305212437740.png)

ReplicationController：用于确保有正常运行的pod，用于复制pod（创建pod的副本）并让他们你保持运行

pod：系统中最重要的组件，通常一个pod可以包含任意数量的容器

services：pod的存在是短暂的，当pod因为某种原因（故障、人为）从一个健康节点被剔除，rc会用一个新的pod来替换，但是新的pod和原先的pod的IP地址并不相同，这个时候service就可以解决pod IP地址变化的问题，以及在一个固定的IP和端口上对外暴露多个pod，当一个服务被创建时，它将得到一个静态IP（服务的生命周期内不会发生改变）,为一个或者多个提供相同服务的pod提供一个静态地址，到达这个IP 端口的请求将会被转发至属于该服务的一个容器IP端口

![rc复制多个pod](/home/chenkai/.config/Typora/typora-user-images/image-20210305220436269.png)

#### 列出pod时，现实pod IP和节点

```bash
$ kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubia   1/1     Running   2          20h   172.17.0.5   minikube   <none>           <none>

```

#### 图形化管理界面

```bash
$ minikube dashboard
```

系统会用默认浏览器打开dashboard网页

![minikube dashboard](/home/chenkai/.config/Typora/typora-user-images/image-20210305221655542.png)



## 第三章 pod：运行于kubernetes中的容器

**主要内容：**

**创建、启动和停止pod**

**使用标签组织pod和其他资源**

**使用特定标签对所有pod执行操作**

**使用命名空间将多个pod分到不重叠的组中**

**调度pod到指定工作类型的节点**

### 介绍pod

一个pod绝对不会跨越节点

![pod不跨节点](/home/chenkai/.config/Typora/typora-user-images/image-20210305222524987.png)

每个容器只运行一个进程，将多个容器绑定在一起，并把他们作为一个单元进行管理，这个就是pod的原理

同一pod中容器运行的多个进程不能绑定相同的端口号，否则会端口冲突，对于不同pod中的容器则不会有这样的问题，因为每个pod都有独立的端口空间

平坦pod间网络

![平面网络](/home/chenkai/.config/Typora/typora-user-images/image-20210305230025613.png)

每个pod可以通过其他pod的ip地址实现相互访问，有点类似局域网中的计算机

在不导致额外开销的情况下，尽可能多的拥有pod，将应用程序尽可能多的组织到多个pod中，每个pod中只包含紧密相关的进程或组件

#### 多层应用应该分散到多个pod中

多层应用（前端web+后端db），分散方可以提高基础架构的利用率，同时在扩缩容上也会更方便，因为kubernetes中扩缩容的最小单位是pod而非容器。

#### 何时在一个pod中使用多个容器

当一个应用（进程）需要有一个或多个辅助进程（组件）组成的时候

![pod中使用多个容器](/home/chenkai/.config/Typora/typora-user-images/image-20210305231335521.png)

#### 如何决定何时在pod内使用多个容器

1、需要在一起运行还是可以在分开的主机上运行？

2、代表的是一个整体还是相互独立的组件？

3、必须一起扩缩容还是可以分开进行？

**容器不应该包含多个进程，pod也不应该包含多个并不需要运行在同一主机上的容器**

![如何选择](/home/chenkai/.config/Typora/typora-user-images/image-20210305231717151.png)





### 以YAML或JSON描述文件创建pod

#### 查看现有pod的yaml描述文件

```bash
$ kubectl get po kubia -o yaml
```

pod定义的几个组成部分：

1、YAML中使用的kubernetesAPI版本

2、YAML描述的资源类型

3、metadata包含名称、命名空间、标签和关于该容器的其他信息

4、spec包含pod内容的实际说明，例如pod的容器、卷和其他数据

5、status包含当前运行pod信息（**创建新pod时，不需要这一项**）

#### 创建一个简单的YAML描述文件

在任意位置创建一个kubia-manual.yaml文件

```bash
$ vim kubia-manual.yaml
apiVersion: v1						# 描述文件遵循v1版本的kubernetesAPI
kind: Pod							# 类型是pod
metadata:
        name: kubia-manual			# pod的名称
spec:
        containers:
        - image: ck747579338/kubia	# 创建容器使用的镜像
          name: kubia				# 容器的名称
          ports:
          - containerPort: 8080		# 启用监听的端口
            protocol: TCP
```

指定端口的意义是：每个使用集群的人都能够快速查看每个pod对外暴露的端口，定义端口还允许为端口指定一个名称，可以方便使用

使用kubectl explain查看每个API的属性

```bash
$ kubectl explain pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion d...

   kind	<string>
     Kind is a s...

   metadata	<Object>
     Standard object's...

   spec	<Object>
     Specificat...
     ...

   status	<Object>
     ...
     
$ kubectl explain pod.status	# 查看status属性
KIND:     Pod
VERSION:  v1

RESOURCE: status <Object>
...
```

#### 使用kubectl create创建pod

```bash
$ kubectl create -f kubia_manual.yaml 
pod/kubia-manual created
$ kubectl get po
NAME           READY   STATUS    RESTARTS   AGE
kubia          1/1     Running   2          22h
kubia-manual   1/1     Running   0          9s
```

kubectl create -f 用于从yaml或json创建任何资源（不只是pod）

使用kubectl logs查看pod日志

```bash
$ kubectl logs kubia-manual 
Kubia server starting...
```

注意：每天或者每次日志达到10M时，容器日志都会自动轮替，kubectl logs命令仅显示最后一次轮替后 的日志条目

#### Pod中有多个容器时，查看指定容器的日志

命令：kubectl logs <pod名称> -c <容器名>

```bash
$ kubectl logs kubia-manual -c kubia 
Kubia server starting...
```

当pod被删除时，里面的日志也会一同被删除，所以需要将日志去中心化，具体操作方式以后会学

#### 向pod发送请求

通过端口转发的方式连接到pod进行测试和调试，使用**kubectl port-forward <pod名称> 本地端口：pod端口**的方式，将本地端口映射到pod端口

```bash
$ kubectl port-forward kubia-manual 8888:8080 # 将本地8888端口映射到kubia-manual的8080端口
Forwarding from 127.0.0.1:8888 -> 8080
```

打开另一个终端测试pod

```bash
$ curl localhost:8888
You've hitkubia-manual
```

测试一次原先的终端就会增加一条记录

```bash
$ kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Handling connection for 8888
```

下图为该过程的简化视图

![简化视图](/home/chenkai/.config/Typora/typora-user-images/image-20210306010944942.png)

### 使用标签组织pod

通过标签可以方便的查看系统结构以及每个pod的角色，如下图所示

![示例图](/home/chenkai/.config/Typora/typora-user-images/image-20210306215211158.png)

通过添加app,rel两个标签，将pod组织为两个维度（基于应用的水平维度和基于版本的纵向维度）

#### 创建pod时指定标签

创建一个kubia_manual_with_labels.yaml文件，并写入如下内容

```bash
$ vim kubia_manual_with_labels.yaml 
apiVersion: v1
kind: Pod
metadata:
        name: kubia-manual-v2
        labels:								# 添加标签
                creation_method: manual		# 设置标签
                env: prod
spec:
        containers:
        - image: ck747579338/kubia
          name: kubia
          ports:
          - containerPort: 8080
            protocol: TCP
```

添加完成后创建pod

```bash
$ kubectl create -f kubia_manual_with_labels.yaml 
pod/kubia-manual-v2 created
```

查看pod状态，kubectl get po默认情况不会带上标签，需要添加--show-labels选项

```bash
$ kubectl get po --show-labels		# 添加--show-labels选项查看pod附加的标签
NAME              READY   STATUS    RESTARTS   AGE    LABELS
kubia             1/1     Running   3          42h    run=kubia
kubia-manual      1/1     Running   1          19h    <none>
kubia-manual-v2   1/1     Running   0          6m3s   creation_method=manual,env=prod
```

如果只想查看某些标签

```bash
$ kubectl get po -L creation_method,env,run	#查看这几个标签
NAME              READY   STATUS    RESTARTS   AGE   CREATION_METHOD   ENV    RUN
kubia             1/1     Running   3          43h                            kubia
kubia-manual      1/1     Running   1          20h                            
kubia-manual-v2   1/1     Running   0          73m   manual            prod  
```

#### 添加pod标签

将kubia-manual添加上creation_method=manual标签

```bash
$ kubectl label po kubia-manual creation_method=manual	# 给pod添加标签
pod/kubia-manual labeled
```

#### 修改pod标签

将kubia-manual-v2的env=prod修改为env=debug

```bash
$ kubectl label po kubia-manual-v2 env=debug --overwrite	# 修改标签必须带有--overwrite
pod/kubia-manual-v2 labeled
```

查看更新后的标签

```bash
$ kubectl get po -L creation_method,env
NAME              READY   STATUS    RESTARTS   AGE   CREATION_METHOD   ENV
kubia             1/1     Running   3          43h                     
kubia-manual      1/1     Running   1          20h   manual            
kubia-manual-v2   1/1     Running   0          86m   manual            debug
```

### 通过标签选择器列出pod子集

标签选择器可以根据pod的标签进行过滤

#### 使用标签选择器列出pod

使用-l选项对pod进行过滤

```bash
$ kubectl get po -l creation_method=manual	#查看标签creation_method=manual的pod
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual      1/1     Running   1          20h
kubia-manual-v2   1/1     Running   0          90m
$ kubectl get po -l env						# 查看包含标签env（无论值是多少）的pod
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual-v2   1/1     Running   0          90m
$ kubectl get po -l '!env'					# 查看标签不包含env的pod
NAME           READY   STATUS    RESTARTS   AGE
kubia          1/1     Running   3          43h
kubia-manual   1/1     Running   1          20h
```

也可按照以下方式进行过滤

1、creation_method!=manual选择带有creation_method标签但是值不为manual的pod

2、env in (prod, debug)选择标签为env,值为prod或者debug的pod

3、env notin（prod,debug）选择标签为env,但是值不为prod或debug的pod

#### 在标签选择器中使用多个条件

使用标签选择器时，如果有多个条件，使用逗号给开

```bash
$ kubectl get po -l creation_method=manual,env=debug --show-labels
NAME              READY   STATUS    RESTARTS   AGE    LABELS
kubia-manual-v2   1/1     Running   0          144m   creation_method=manual,env=debug
```

### 使用标签和选择器来约束和调度pod

在kubernetes中要尽量避免直接对pod指定节点来调度，而是在pod的描述文件中描述对节点的需求，让kubernetes选择符合特定标准的逻辑节点组

#### 对节点添加、修改标签

对节点添加、修改标签的方式与pod基本一致

```bash
$ kubectl labels node <节点名> <标签>=<值>
```

#### 将pod调度到特定节点

创建pod时，添加需求

```bash
$ vim kubia_ssd.yaml 
apiVersion: v1
kind: Pod
metadata:
        name: kubia_ssd
spec:
		nodeSelector:
		  sto: ssd			#节点选择器要求kubernetes只将pod部署到包含标签sto=ssd的节点上
...
```

#### 调度到一个指定的节点

除了设置节点需求，让kubernetes去选择符合要求的节点这种方式，在kubernetes中，还可以将pod调度到某一个特定的节点，每个节点都有一个唯一标签，其中键为kubernetes.io/hostname，值为该节点的实际主机名

```bash
$ kubectl get nodes -L kubernetes.io/hostname
NAME       STATUS   ROLES                  AGE   VERSION   HOSTNAME
minikube   Ready    control-plane,master   45h   v1.20.2   minikube
```

如果在pod描述文件中的节点选择器设置kubernetes.io/hostname: minikube,那么就会将该pod调度到这个节点下，这种方式是有风险的，如果这个节点处于离线状态，将可能会导致pod无法调度，所以在实际应用中不应该考虑某个特定的单独节点，而是通过标签选择器考虑符合特定标准的逻辑节点组

#### 注解pod

#### 查看pod的注解

通过kubectl describe查看pod的注解（Annotations所对应的值）

```bash
$ kubectl describe po kubia-manual
Name:         kubia-manual
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Sat, 06 Mar 2021 00:43:21 +0800
Labels:       creation_method=manual
Annotations:  <none>					# 注解
Status:       Running
IP:           172.17.0.3
...
```

#### 添加和修改注解

与添加修改标签的方式类似

```bash
$ kubectl annotate pod kubia-manual wo_shi="zhu jie"				# 添加注解
$ kubectl describe po kubia-manual
Name:         kubia-manual
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Sat, 06 Mar 2021 00:43:21 +0800
Labels:       creation_method=manual
Annotations:  wo_shi: zhu jie 										# 注解             
Status:       Running
IP:           172.17.0.3
$ kubectl annotate pod kubia-manual wo_shi="xiu gai" --overwrite	# 修改注解
$ kubectl describe po kubia-manual
Name:         kubia-manual
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Sat, 06 Mar 2021 00:43:21 +0800
Labels:       creation_method=manual
Annotations:  wo_shi: xiu gai            						  	# 修改成功		
Status:       Running
IP:           172.17.0.3

```

### 使用命名空间对资源进行分组

Linux命名空间：用于进程相互隔离

kubernetes命名空间：为对象名称提供了一个作用域，这样可以跨不同的命名空间使用相同的资源名称，即资源名称在同一命名空间内保持唯一，在两个不同的命名空间可以使用相同的资源名称

#### 发现其他命名空间及其pod

查看命名空间

```bash
$ kubectl get ns
NAME                   STATUS   AGE
default                Active   45h
kube-node-lease        Active   45h
kube-public            Active   45h
kube-system            Active   45h
kubernetes-dashboard   Active   45h
```

查看指定命名空间下的pod

```bash
$ kubectl get po --namespace kube-system	# 也可以用-n代替--namespace
NAME                               READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-hhxq6            1/1     Running   3          45h
etcd-minikube                      1/1     Running   3          45h
kube-apiserver-minikube            1/1     Running   3          45h
kube-controller-manager-minikube   1/1     Running   3          45h
kube-proxy-j5wpv                   1/1     Running   3          45h
kube-scheduler-minikube            1/1     Running   3          45h
storage-provisioner                1/1     Running   6          45h
```

#### 创建一个命名空间

第一种方式：使用YAML文件创建命名空间

```bash
$ vim custom-namespace.yaml 
apiVersion: v1
kind: Namespace							# 类型为namespace
metadata:
        name: custom-namespace			# 命名空间的名称为custom-namespace

$ kubectl create -f custom-namespace.yaml 
namespace/custom-namespace created
$ kubectl get ns
NAME                   STATUS   AGE
custom-namespace       Active   20s		# 创建成功
default                Active   45h
kube-node-lease        Active   45h
kube-public            Active   45h
kube-system            Active   45h
kubernetes-dashboard   Active   45h
```

第二种方式：使用kubectl create namespace命令创建命名空间

```bash
$ kubectl create namespace custom-namespace
namespace/custom-namespace created
```

使用YAML文件可以强化kubernetes中所有内容都是API对象这一概念，可以通过向API服务器提交YAML manifest来实现创建、读取、更新和删除

### 管理其他命名空间中的对象

可以在metadata字段中添加 namespace: 命名空间名称 属性，也可以使用kubectl create创建资源时指定命名空间

```bash
$ kubectl create -f kubia_manual.yaml -n custom-namespace 
pod/kubia-manual created
$ kubectl get po -n custom-namespace	# 查看指定命名空间里面的pod
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          16s
```

这个时候就村在两个相同名称的pod（kubia-manual），一个存在于default命名空间，一个存在于custom-namespace，在列出、描述、修改和删除其他命名空间的的对象时，需要在kubectl中使用-n(--namespace)指明命名空间，否则将会在默认的命名空间进行操作，可以通过kubectl config进行修改

#### 切换命名空间

```bash
在~/.bashrc中添加
kcd='kubectl config set-context $(kubectl config current-context) --namespace'
然后在终端中执行一下命令就能切换
$ kcd <命名空间名称> 
```

命名空间之间是否存在网络隔离取决于kubernetes的网络解决方案，默认情况下，命名空间不提供任何隔离

### 停止和删除pod

#### 按名称删除pod

```bash
$ kubectl delete pod kubia-manual-v2 
pod "kubia-manual-v2" deleted
```

删除pod时，kubernetes像进程发起送一个SIGTERM信号，并等待（默认30s），是应用正常关闭，如果应用没有及时关闭，则通过SIGKILL终止该进程，为确保被正常关闭，需要正确处理SIGTERM信号

#### 通过标签删除pod

```bash
$ kubectl get po --show-labels					# 查看pod的标签
NAME           READY   STATUS    RESTARTS   AGE   LABELS
kubia          1/1     Running   3          47h   run=kubia
kubia-manual   1/1     Running   1          24h   creation_method=manual
$ kubectl delete po -l creation_method=manual	# 删除标签creation_method=manual的pod
pod "kubia-manual" deleted
```

#### 通过删除整个命名空间来删除pod

删除命名空间，命名空间内的pod会伴随命名空间一同被删除

```bash
$ kubectl delete ns custom-namespace 
namespace "custom-namespace" deleted
```

#### 将命名空间下的所有pod都删除

```bash
$ kubectl delete po --all
```

通过--all选项告诉kubernetes删除这个命名空间下的所有pod

#### 删除命名空间下的所有资源

```bash
$ kubectl delete all --all
```

第一个all指的是所有类型，第二个all指的是所有资源，但是某些资源（secret）还是会保留下来，需要指定删除，这个操作也会删除名为kubernetes的service，但是它过几分钟就会重建