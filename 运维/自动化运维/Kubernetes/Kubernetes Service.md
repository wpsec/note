## Service 概念
![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763974189839-c391746c-d030-4f93-8be6-4bf9f8d8bfc0.png)

Kubernetes Service 就是给一组 Pod 提供一个“固定不变的访问入口”，并负责流量负载均衡。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763974549096-3dc04cae-26a8-4044-958c-89d3b0e225f1.png)



## Service 工作原理
在集群中，每个 Node 都会运行一个 kube-proxy 进程，kube-proxy 负责为 Service 实现了一种 VIP（虚拟 ip）的形式

通过代理转发实现： 最早 userspace（废弃），ipvs 代理、iptables 代理，1.8 之前是 iptables、1.8 之后是 iptables，但是推出了ipvs，官方推荐使用ipvs



iptables

+ 监听 apiserver，将 service 变化修改到本地

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763975221365-1dfd3e2a-92e6-4144-af58-5ea8417eae7a.png)

ipvs

+ 理论上性能更强，但是需要开启，一般的云厂商不愿意提供（阉割）



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763975239661-9415788a-254a-453b-b885-63d91e55f7b3.png)

修改为 ipvs

kubectl edit，在线编辑状态本身，比如kubectl edit deployment deployment-demo

修改deployment 控制器下名为deployment-demo 的 yaml 配置文件，修改后立即生效，这里修改的是 etcd 数据库中的配置文件

kubectl edit——》apiserver——〉本地编辑器（vim 等）——》kube-apiserver 校验并更新（校验失败会给出提示）——〉etcd 写入新状态——》controller 发现变化滚动更新

```yaml
kubectl edit configmap kube-proxy -n kube-system
```

默认为 iptables，为空

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763976076820-8fdfd77e-4a2f-44c6-bc27-1698956d753f.png)

修改为 ipvs 保存即可

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763976112821-ec6bc0e7-b28e-48fb-945e-90657d06bf13.png)

重建 kube-proxy

```yaml
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763976310562-5a7806f0-01ff-4bd5-a946-bbd04e678836.png)

### 






## Service 类型
ClusterIP：默认类型，自动分配一个仅 Cluster 内部可以访问的虚拟 ip

NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，端口范围（30000–32767）

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763977547288-f802100e-25c6-4254-ad27-f5d49d94b712.png)

LoadBalancer：在 NodePort 基础上，借助 cloud provider 创建一个外部负载均衡器，并将请求转发到 Nodeip（一个高可用场景，创建一个外部负载，防止多个 master 中的一个 master 死了不能访问其它 master）解决单点故障问题

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763990494321-187bc93d-dbf0-46f1-baa6-4336100024f9.png)

ExternalName：把集群外部的服务引入集群，在集群内部直接使用，需要 k8s 版本>1.7 或更高版本的 kube-dns 才支持

当集群内部访问外部 API、数据库时，创建 ExternalName 指向外部域名，这样就不用动集群内部环境了，别名机制









service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-clusterip
  namespace: default
  labels:
    release: stable
    svc: clusterip
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: 80
```





### ClusterIP
#### internalTrafficPolicy 实验
internalTrafficPolicy

+ Cluster：集群级路由（默认），路由加入 service 的所有 node
+ Local：本地路由，集群内访问 Service 时，流量只会路由到本节点上属于 Service 的 Pod；如果本节点没有 Pod，访问会被丢弃。

创建 RS 副本控制器

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-me-exists-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchExpressions:
      - key: app
        operator: Exists
  template:
    metadata:
      labels:
        app: spring-k8s
    spec:
      containers:
        - name: rs-me-exists-demo-container
          image: docker.xuanyuan.run/nginx:alpine
          ports:
            - containerPort: 80

```

 

创建 service，Cluster 模式

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-k8s-cluster
  namespace: default
spec:
  selector:
    app: spring-k8s
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
  internalTrafficPolicy: Cluster

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763979583188-d20b13d4-0dfe-4841-807c-4c0a30078a17.png)

修改为 local 模式下

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-k8s-local
  namespace: default
spec:
  selector:
    app: spring-k8s
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
  internalTrafficPolicy: Local

```

master 是访问不到的，没有提示报错，也没有提示服务不可达，而是数据包直接被丢弃了，而 node 可以访问到，因为 master 上没有部署 Pod，本地模式

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763979689631-30dbe996-6229-4c8f-bfde-169600fb2e09.png)



#### 会话保持 IPVS 持久化连接实验
service.spec.sessionAffinity:ClientIP

```yaml
kubectl edit service servicename
```

修改为ClientIP

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763988060780-426ebe71-2ae2-47b2-863e-97fc8ef2ed27.png)

在这 120 秒内，不管 curl 多少次，都会指向这个 node2 的后端容器

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763988274439-1a050ac3-c9d0-4487-8d4e-9fb71ac8170d.png)

默认保持时间为 3 个小时

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763988410523-94b51310-31ee-4290-82a0-e32c569d3d72.png)

### NodePort


![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763988523618-283ed880-af9d-4f3a-a0ce-8b6a73fc30d6.png)



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-nodeport-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      release: stabel
      svc: nodeport
  template:
    metadata:
      labels:
        app: myapp
        release: stabel
        env: test
        svc: nodeport
    spec:
      containers:
        - name: myapp-container
          image: docker.xuanyuan.run/nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80

```

service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
  namespace: default
spec:
  type: NodePort
  selector:
    app: myapp
    release: stabel
    svc: nodeport
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30010

```

NodePort = 在每个 Node 上开一个端口，把外部访问映射到某个 Service → 再映射到 Pod。

访问 node 物理机 IP+30010

在 Pod 内部访问也是可以的，所以 NodePort 可以理解为是ClusterIP 的加强版

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763989566773-b377a962-ea34-46e7-ab83-6f086785aa3f.png)



### ExternalName
将 Service 名称映射到外部 DNS 名称（CNAME）

集群内访问 Service，会被 DNS 重写到 ExternalName 指向的域名

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-nodeport-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      release: stabel
      svc: nodeport
  template:
    metadata:
      labels:
        app: myapp
        release: stabel
        env: test
        svc: nodeport
    spec:
      containers:
        - name: myapp-container
          image: docker.xuanyuan.run/nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80

```

ExternalName service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-external
  namespace: default
spec:
  type: ExternalName
  externalName: www.baidu.com

```

当在 Pod 内，wget 内部域名时，解析到了 baidu 的 IP，说明ExternalName 服务有效

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763992841065-997f6b2f-4a45-45ae-b274-d1c08f1ddc12.png)

## Endpoints
service 底层模型，是一个对象

Service 是虚拟入口（ClusterIP / NodePort / LoadBalancer）

而 Endpoints 就是 Service 对应的 Pod 列表（Pod 的 IP + 端口）

Service 流量最终会转发到 Endpoints 中的 Pod

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763989997036-16afa10f-2125-444b-83e3-ec6cda69cdae.png)

创建 service 后 endpoint 会创建一个同名对象

```yaml
kubectl get endpoints
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763990258604-b331d40e-d208-4aed-a798-56e1b6048eec.png)

服务发现和负载均衡

+ ClusterIP / NodePort / LoadBalancer → Service
+ Service 查 Endpoints → 知道 Pod 列表
+ kube-proxy 或 iptables 根据 Endpoints 负载均衡流量

动态更新

+ Pod 扩容 / 缩容
+ Pod 就绪状态变化（readiness probe）
+ Endpoints 会自动更新，Service 不用修改

## publishNotReadyAddresses
在 Kubernetes 中，Service 会把 就绪的 Pod（Ready Pod） 加入 Endpoints，这样 Service 才会把流量发送给这些 Pod

默认情况下，如果 Pod 未就绪（NotReady），它不会出现在 Endpoints 中

而publishNotReadyAddresses，就是用来改变这种默认的

意思是即时未就绪，也把这个 Pod 加入Endpoints



有什么作用？

做一个实验



创建一个 deployment，延迟 30 秒在 read，模拟 pod 启动慢

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-delay
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp-delay
  template:
    metadata:
      labels:
        app: myapp-delay
    spec:
      containers:
      - name: myapp
        image: docker.xuanyuan.run/nginx:alpine
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 5
        ports:
        - containerPort: 80

```

创建 service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: myapp-delay
  ports:
  - port: 80
    targetPort: 80

```

默认状态下，因为 Pod 迟迟没有 read，所以没有被创建到Endpoints 里

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763993480915-2a2653ab-18aa-4099-848c-90b3a7db8bb2.png)

创建一个publishNotReadyAddresses:true 的 service



```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: myapp-delay
  publishNotReadyAddresses: true
  ports:
  - port: 80
    targetPort: 80

```

在 Pod 没有就绪的情况下，就加入了endpoints

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763993645420-f82ca68f-c954-489f-83ee-1e6fbddfd130.png)

提前加入主要解决以下问题：

StatefulSet 集群内部发现（Cluster-internal Discovery）

+ 数据库、消息队列、缓存集群等
+ Pod 之间需要互相发现对方 IP
+ 如果 Pod 未就绪，它还会被排除在默认 Endpoints 外
+ 启用 publishNotReadyAddresses 后，Pod 在 DNS 中提前可见，集群节点初始化顺序就能正确执行

例子：

+ MySQL 集群初始化时，节点 A 需要知道节点 B 的 IP
+ Pod B 还没 Ready，如果不 publishNotReadyAddresses → 节点 A 找不到 B → 初始化失败



