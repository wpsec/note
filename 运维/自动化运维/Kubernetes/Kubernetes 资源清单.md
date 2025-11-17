## 资源
### 名称空间级别
+ 工作负载型资源：Pod、ReplicaSet、Deployment 等
+ 服务发现及负载均衡型资源：Service、Ingress 等
+ 配置与存储型资源：Volume、CSI 等
+ 特殊类型的存储卷：ConfigMap、Secre 等

### 集群级资源
+ Namespace、Node、ClusterRole、ClusterRoleBinding

### 元数据型资源
+ HPA、PodTemplate、LimitRange



## 资源清单
![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763014800117-d52c61a8-810d-4219-8e20-cc91d1326847.png)

### 接口组/版本
![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763014465509-d6f80335-50b8-4dc4-9dbd-e65801390199.png)

apiextensions.k8s.io 组

v1 版本号

没写组的就是核心组 core

```yaml
[root@k8s-master01 kubernetes]# kubectl api-versions
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2
batch/v1
certificates.k8s.io/v1
coordination.k8s.io/v1
crd.projectcalico.org/v1
discovery.k8s.io/v1
events.k8s.io/v1
flowcontrol.apiserver.k8s.io/v1
flowcontrol.apiserver.k8s.io/v1beta3
networking.k8s.io/v1
node.k8s.io/v1
policy/v1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
v1
```

使用  kubectl explain 查询接口组与版本  可以理解为 help

```yaml
kubectl explain deployment
kubectl explain pod
kubectl explain pod.soce
...
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763015807576-39633052-4c15-4cd4-9fa2-f687779c11dc.png)

### kind 类别
告诉 k8s 创建什么资源

```yaml
kind: Pod

常见的有
# 基本部署单元
Pod

# 管理Pod的部署
Deployment

# 配置数据
ConfigMap

# 命名空间
Namespace

# 持久化存储
PersistentVolume
```

### metadata 元数据
包含资源的标识信息和标签，主要用于描述资源的基本信息

```yaml
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    tier: frontend
    
    
常见的有
# 资源名称，命名空间内唯一
name

# 所属的命名空间
namespace

# 键值对标签，用于资源分类
labels

# 非标识性元数据、用于工具拓展
annotations
```

### spec 期望
定义了资源的期望状态，Kubernetes 控制器会不断调整实际状态以匹配 spec 中定义的期望状态

Pod 的 spec

```yaml
spec:
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

Deployment 的 spec

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
```

### status 状态
status 由 k8s 维护，一般不需要管

```yaml
status:
  conditions:
  .....
```



### demo
镜像问题

[https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/)

正式环境下一般这样做

首先在当前资源清单目录下创建 secret 目录，存放registry-secret.yaml，这个是一个认证私有仓库的配置

base64 密钥的生成

```yaml
生成userpass的base64
echo -n "user:pass" | base64

生成dockerconfigjson 认证内容

{
  "auths": {
    "docker.xuanyuan.run": {
      "username": "user",
      "password": "pass",
      "auth": "第一段user:pass的base64"
    }
  }
}
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred				# 指定元数据名为regcred
  namespace: default  # 指定命名空间
type: kubernetes.io/dockerconfigjson
data:
   .dockerconfigjson: "dockerconfigjson base64认证内容"
```



kubectl apply 为声明式管理，存在就更新，不存在就创建，一般在 deploy、daemonset 等配置可以使用

kubectl create 为新建资源，目标存在就报错，一般 pod 的创建使用

```yaml
kubectl apply -f registry-secret.yaml
```



创建一个 pod 的资源清单，描述了接口组版本、类型为 pod、元数据描述了一个 demo、定义了期望 spec 要做的事，创建两个个容器，容器的名字，用到的 image 和最后执行的命，imagePullSecrets 指定为我们前面配置的 私有仓库地址

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
spec:
  containers:
    - name: myapp-1
      image: docker.xuanyuan.run/nginx:alpine
    - name: busybox-1
      image: docker.xuanyuan.run/nginx:alpine
      command:
        - "/bin/sh"
        - "-c"
        - "sleep 3600"
  imagePullSecrets:
    - name: regcred
```

使用 kubectl 去创建这个资源清单，k8s 会根据清单内容去

```yaml
kubectl create -f pod-demo.yaml 
# 查询pod，以namespace的方式去查询，default为默认，不指定namespace也会输出default的命名空间
kubectl get pod -n default
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763026087044-c8d9f098-db33-4740-9a72-53dac628b582.png)

## k8s 常用命令
### get
获取资源

列出所有命名空间中的所有 Pod

```yaml
kubectl get pod -A

kubectl get pod --all-namespaces

# 持续检测
kubectl get pod -w
```

包含 k8s 组件的 pod

第一列为 pod 的命名空间 namespace

第二列为元数据 metadata 配置的 name

第三列为容器就绪状态 就绪容器数/总容器数

第四列为 pod 当前状态，running 运行中、pending 调度中、CrashLoopBackOff崩溃循环

第五列容器重启次数

第六列 pod 存活时间

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763026109206-8d5a7312-7217-4722-a565-a343bebfb533.png)

批量查看多个 pod 的详细信息，我们刚刚创建的 pod 在 node1 上运行

```yaml
kubectl get pod -o wide
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763027489623-64fba4e5-2bac-4c53-97ee-75ea6dae21d9.png)

在 node1 上，可以通过 docker ps 查看到这些容器

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763027687209-74d368a6-8bbe-4806-9034-4453723dfaf6.png)

查看标签

标签是元数据 metadata 中定义的，一个 pod 可以有多个标签信息，统一标签内容可以让我们快速定位集群中的 pod

```yaml
kubectl get pod --show-labels
kubectl get pod -l app
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763031286998-749001ac-0a82-4e33-a5de-a98802492c2b.png)

### logs
日志

```yaml
kubectl logs pod名称 -c 容器名称
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763031436256-8b95b610-319a-4995-bc85-d24f40109809.png)

name 规则是这样的

k8s_<container_name>_<pod_name>_<namespace>_<podUID>_<container_index>

下面三组是 k8s 系统自带的 pod，上面一组有三个，分别是两个 nginx 和一个pause

```yaml

0bb3c80ba495   b3c656d55d7a                                        "/bin/sh -c 'sleep 3…"   10 minutes ago   Up 10 minutes             k8s_busybox-1_pod-demo_default_5db04211-4e37-4884-bfaa-6e890f3c5fd8_0
77c668e083f0   b3c656d55d7a                                        "/docker-entrypoint.…"   10 minutes ago   Up 10 minutes             k8s_myapp-1_pod-demo_default_5db04211-4e37-4884-bfaa-6e890f3c5fd8_0
7174f39143e3   registry.aliyuncs.com/google_containers/pause:3.8   "/pause"                 10 minutes ago   Up 10 minutes             k8s_POD_pod-demo_default_5db04211-4e37-4884-bfaa-6e890f3c5fd8_0


e6002efeeaa4   93558ecefd9c                                        "start_runit"            7 hours ago      Up 7 hours                k8s_calico-node_calico-node-2jmhb_kube-system_0e6450b3-0f05-4385-b981-8fab365fe748_1
2c597e45bf6b   registry.aliyuncs.com/google_containers/pause:3.8   "/pause"                 7 hours ago      Up 7 hours                k8s_POD_calico-node-2jmhb_kube-system_0e6450b3-0f05-4385-b981-8fab365fe748_1


0c73bd6579ba   registry.aliyuncs.com/google_containers/pause:3.8   "/pause"                 7 hours ago      Up 7 hours                k8s_POD_kube-proxy-pjf5x_kube-system_e901aa1c-856e-4b85-b2fc-f62c06fae254_1
5e268c1678e1   e32aa8045573                                        "/usr/local/bin/kube…"   7 hours ago      Up 7 hours                k8s_kube-proxy_kube-proxy-pjf5x_kube-system_e901aa1c-856e-4b85-b2fc-f62c06fae254_1


21a5db89f4a2   registry.aliyuncs.com/google_containers/pause:3.8   "/pause"                 7 hours ago      Up 7 hours                k8s_POD_calico-typha-5b56944f9b-qql9f_kube-system_c0017d5a-55d2-4c53-911a-e6b29ad82e74_1
c53ffa00d329   3e3ddf70a2fd                                        "/sbin/tini -- calic…"   7 hours ago      Up 7 hours                k8s_calico-typha_calico-typha-5b56944f9b-qql9f_kube-system_c0017d5a-55d2-4c53-911a-e6b29ad82e74_1


```

### describe
查看某个 pod 资源详细信息，比如报错等信息

```yaml
kubectl describe pod pod-demo -n default
```

### delete
删除资源

```yaml
# 删除名为pod-demo的pod资源
kubectl delete pod pod-demo
kubectl delete -f initC.yaml

# 杀不掉的僵尸pod，强制杀
kubectl delete pod initc-1 -n default --grace-period=0 --force
 --grace-period=0 立即杀死容器，不等待优雅退出
 --force 跳过finalizer
 
# 删除名为 mydb 的service
kubectl delete service mydb
```

进入容器

一个 pod 有多个容器，通过 kubectl describe pod podname 查看容器名称

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763030838212-bf9aac83-7e77-42c9-ae97-c7aeb69eac4b.png)

### exec
进入容器，如果只有一个容器，-c 可以忽略

```yaml
kubectl exec -it pod-demo -c myapp-1 /bin/sh

-i 保持标准输入打开
-t 分配一个伪终端
-c 容器名
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763030957363-e1f6b2f9-ef24-4cba-977f-d0a40d4ea5da.png)







