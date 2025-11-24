控制器用于保证当前状态与 期望（spec）状态一致，比如，ReplicaSet 控制器负责维护集群中运行的 Pod 数量；Node 控制器负责监控节点状态，阶段故障时，执行自动修复流程，确保集群尽量与期望（spec）状态一致



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763535231423-a59afd0b-990d-4012-b41c-27d04d9b7b3e.png)

调谐概念，控制器通过调谐来让 Pod 的运行状态最可能接近 spec

## 副本控制器
### ReplicationController（RC）和 ReplicaSet（RS）
ReplicationController

确保容器应用的副本数始终保持在用户定义的副本数，如果容器异常退出，会自动创建 Pod 来替代，而如果异常多出来的容器，也会自动回收；



在新版的 k8s 中，建议使用 ReplicaSet 来取代 ReplicationController。ReplicaSet 跟ReplicationController 本质上没有不同，但是ReplicaSet 支持集合式的 selector（标签选择器）





#### RC 自动保持与自动回收实验
kind: ReplicationController：指定类型为 RC 副本控制器

replicas：Pod 的数量

selector：选择器 pod满足选择器=rc-demo的都被它控制

template：Pod 模版

env： 环境变量

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-demo
spec:
  replicas: 3
  selector:
    app: rc-demo
  template:
    metadata:
      labels:
        app: rc-demo
    spec:
      containers:
      - name: rc-demo-container
        image: docker.xuanyuan.run/nginx:alpine
        env:
        - name: GET_HOSTS_FROM
          value: dns
        - name: zhangsan
          value: "123"
        ports:
        - containerPort: 80

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763555596753-f744a7fc-fe07-4d2b-8d68-a91b325cd636.png)

删除一个 Pod 模拟 Pod 损坏，RC 会根据 spec（期望）再次创建一个

rc-demo-chkzp 被删除了，但是马上就新创建了一个，说明 RC 副本控制器自动又创建了一个 Pod

当一个服务器的资源不足以承载过多的 Pod 时，自动保持就失效了，意思是定义了 100 个 Pod，但当前服务器资源只够跑 50 个，那自动保持也就失效了。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763555770725-079ab912-ad66-4b4c-b9c0-f16686b27370.png)

RC 副本控制器通过labels 来控制它所管理的 pod，给一个 pod 添加一个其它标签，只要其中一个标签能匹配，那这个 pod 就受 RC 管理，当 app=rc-demo 被覆盖时，RC 会自动补起 Pod 的数量以对其期望。

```yaml
# 获取Pod的labels标签
kubectl get pod --show-labels

# get控制器
kubectl get rc
# or
kubectl get ReplicationController

# 给pod rc-demo-4zv8j 添加一个标签 version=v1
kubectl label pod rc-demo-4zv8j version=v1

# 当修改存在的标签时，--overwrite 强制覆盖标签
kubectl label pod rc-demo-4zv8j app=rc-test --overwrite
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763556215518-24167803-9977-4009-ab1f-7ba8dfad2c72.png)

当 pod 过多时，自动回收

而自动回收的 Pod 的，是最后被创建的 Pod，意思是回收会从新老 Pod 里，选最新的 Pod，这样设计的好处是，老的 Pod 上可能有更多的缓存，数据，更多的用户正在交互，杀死它的成本比杀新的高，所以选新的。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763556696738-edee6431-82bb-417c-8440-924f6d624450.png)

调整副本的数量

```yaml
# scale 伸缩规模，类别 rc rc名称 replicas=10
kubectl scale rc rc-demo --replicas=10
kubectl scale rc rc-demo --replicas=5
```



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763557041745-284cc2be-bda5-4841-8d3b-9a5a799f96d2.png)

#### RS 复杂匹配表达式实验
用来匹配 同时带有这些 label 的 Pod，多个条件之间是 and 关系，RS 与 RC 功能类似，但多了复杂标签匹配表达式功能

matchLabels：匹配标签

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-ml-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rs-ml-demo
  template:
    metadata:
      labels:
        app: rs-ml-demo
    spec:
      containers:
      - name: rs-ml-demo-container
        image: docker.xuanyuan.run/nginx:alpine
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80

```

RS 兼容 RC 的功能，且多一个标签匹配表达式的能力

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763559273083-bb10743e-e8aa-4ac6-97f4-88579d86bfb0.png)

```yaml
[root@k8s-master01 5]# kubectl explain rs.spec.selector
GROUP:      apps
KIND:       ReplicaSet
VERSION:    v1

FIELD: selector <LabelSelector>

DESCRIPTION:
    Selector is a label query over pods that should match the replica count.
    Label keys and values that must match in order to be controlled by this
    replica set. It must match the pod template's labels. More info:
    https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors
    A label selector is a label query over a set of resources. The result of
    matchLabels and matchExpressions are ANDed. An empty label selector matches
    all objects. A null label selector matches no objects.

FIELDS:
  matchExpressions	<[]LabelSelectorRequirement>
    matchExpressions is a list of label selector requirements. The requirements
    are ANDed.

  matchLabels	<map[string]string>
    matchLabels is a map of {key,value} pairs. A single {key,value} in the
    matchLabels map is equivalent to an element of matchExpressions, whose key
    field is "key", the operator is "In", and the values array contains only
    "value". The requirements are ANDed.
```

相同点：用来匹配 同时带有这些 label 的 Pod，多个条件之间是 and 关系

matchLabels：只支持等值匹配（key=value）

matchExpressions：用来进行 更复杂的 label 匹配

+ **In**：值在列表中
+ **NotIn：**值不在列表中
+ **Exists**：存在该 key
+ **DoesNotExist**：不存在该 key

简单理解：单个或简单点标签匹配用matchLabels，需要复杂匹配的用matchExpressions

只要存在该 key 就放在一个子集管理 Pod

operator：匹配表达式匹配的值

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-me-exists-demo
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

为什么其它标签中有 app 的没有被匹配到一起

1. RS 只会创建和管理它自己的 Pod
2. 它不会抢占或接管已经属于其它 RC/RS 的 Pod

简单理解：有这个 app 标签的 Pod 没被匹配到，因为这些 Pod 已经被其它 RC/RS 控制了。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763560294713-48043226-800b-42ba-ba9f-a3120848de01.png)

因为使用的是**Exists，即使 value 值不同，也不会受影响，因为Exists 只匹配 key 是否存在一致，即使 key 的 value 被修改也不会影响，字面意思。**

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763560630728-34d145e2-94ae-49c4-b10c-7428e433b165.png)



#### 删除 RC、RC 副本控制器
```yaml
kubectl get rc
kubectl delete rc rc-demo
kubectl get rs
kubectl delete rs rs-me-exists-demo rs-ml-demo
```







### Deployment 控制器
Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义方法，用来代替 RC

+ 定义 Deployment 来创建 Pod 和 ReplicaSet
+ 滚动升级和回滚应用
+ 扩容和缩容
+ 暂停和继续 Deployment



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763613353591-40f647c2-d12f-4e22-8d3a-9b902157fe36.png)

Deployment 控制器

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-demo
  labels:
    app: deployment-demo
spec:
  selector:
    matchLabels:
      name: deployment-demo
  template:
    metadata:
      labels:
        name: deployment-demo
    spec:
      containers:
      - name: deployment-demo-container
        image: docker.xuanyuan.run/nginx:alpine
```

快速创建一个模版

```yaml
kubectl create deployment testtemp --image docker.xuanyuan.run/nginx:alpine --dry-run -o yaml > test_deployment.yaml
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763965875979-ccfe4a26-9e4f-47fa-8e26-889a56448e34.png)



create 与 apply 的区别

```yaml
## 创建
kubectl create -f dep.yaml

## 声明式应用
kubectl apply -f dep.yaml

## 重建
kubectl replace -f dep.yaml
```

create

+ 当使用 create 创建了一个 deployment 控制器后，在修改deployment.yaml 内容，想要更新，只能通过 delete 删除在创建，不然报错（防止覆盖）

apply

+ 而使用 apply 是以声明式的方式进行更新、修改、存在即更新（部分更新）、不存在就创建

replace

+ 如果目标对象与 yaml 文件不一致，重建此对象



```yaml
# 自动扩缩容，设置最小10个，最大15个，cpu超过80%就会扩展
kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
```



#### deployment 滚动更新实验
创建一个Deployment，定义了一些元数据，这个 deployment 的副本名字叫deployment-metadata-demo，他的标签是app: deployment-metadata-demo



然后是这个期望，期望创建 5 个 Pod、选择器标签 app: deployment-demo



然后是容器模版，定义了容器模版的标签app: deployment-demo（需要与期望选择器标签一致），这样才能有归属，

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-metadata-demo
  labels:
    app: deployment-metadata-demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: deployment-demo
  template:
    metadata:
      labels:
        app: deployment-demo
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: deployment-demo-container
        image: docker.xuanyuan.run/nginx:alpine
```

创建一个daemonset，通过声明式更新容器镜像，查看滚动更新状态

将 ngixn 版本 alpine 修改为 latest 模拟版本更新

```yaml
kubectl get pod -l name=deployment-demo -w
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763631971160-f90b636a-c5ee-40be-a47a-884ee8c7d6a0.png)

这时候，pod 会有几种状态

| 老 Pod | 新 Pod | 含义 |
| --- | --- | --- |
| Running | ContainerCreating | 新 Pod 正在启动 |
| Running | Pending | 新 Pod 等待调度 |
| Terminating | Running | 旧 Pod 被删除，新 Pod 已经 OK |
| Terminating | ContainerCreating | 旧 Pod 在删，新 Pod 在拉镜像 |
| Terminating | Pending | Leader 扩容策略下的等待 |


先创建一部分新 Pod、新 Pod Running 后，在开始Terminating 的删除，一批一批完成，不是创建所有新 Pod 后在删除所有旧 Pod，deployment 默认是 25%为一批，一次额外创建 25%的 Pod，一次最多删除 25%的 Pod。（20251120）

这样滚动对用户交互更友好，因为不会特别敏感的中断感觉

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763631999956-b94603eb-8c53-4b8d-b096-8e538f007ab5.png)

可以通过修改rollingUpdate 的量来进行滚动速度，可以使用百分比，也可以直接指定数量

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 50% # 允许多增加50%
    maxUnavailable: 50% # 允许50%的Pod不可用
      
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2 # 允许最多增加2个
    maxUnavailable: 0 # 允许最多0个不可用
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-name
  labels:
    app: deployment-name
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
  selector:
    matchLabels:
      name: deployment-demo
  template:
    metadata:
      labels:
        name: deployment-demo
    spec:
      containers:
      - name: deployment-demo-container
        image: docker.xuanyuan.run/nginx:latest
```



#### 金丝雀部署实验
这是一种思想策略，并不是用deployment 才叫金丝雀部署

金丝雀部署是一种渐进式发布策略：

+ 先把新版本部署到 **少部分用户/Pod**（通常 5%-10%）
+ 观察指标（如 CPU、错误率、日志、延迟等）
+ 确认稳定后，再逐步扩大流量到剩余 Pod

**目的**：降低新版本引入故障的风险。



在开发过程中

产品——》项目经理——〉前后端开发——》功能、安全测试——〉运维部署

这一套，开发修了 bug 可能留下了更大的 bug

功能、安全测试了当不一定就 100%没有功能缺陷、安全漏洞

所以先让小部分人做可控的灰度测试，快速发现问题，没有问题了，在进行平滑升级 / 用户无感知升级



创建一个 deployment 副本控制器，运行 3 个 Pod，一个 Pod 里运行了 两 个容器，一个 nginx、一个 ubuntu，一共 3 个 Pod，6 个容器

其中 spec.selector.matchLabels需与template.metadata.labels保持一致，这里的标签名字有些混乱，别搞错了

```yaml
selector:
    matchLabels:
      name: deployment-demo
  template:
    metadata:
      labels:
        name: deployment-demo
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-demo
  labels:
    app: deployment-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      name: deployment-pod-demo
  template:
    metadata:
      labels:
        name: deployment-pod-demo
    spec:
      imagePullSecrets: 
      - name: regcred
      containers:
      - name: nignx-container
        image: docker.xuanyuan.run/nginx:latest
        imagePullPolicy: IfNotPresent
      - name: ubuntu-container
        image: docker.xuanyuan.run/library/ubuntu:latest
        command: ["sleep", "infinity"]
        imagePullPolicy: IfNotPresent
```



注册一个 service 服务

spec 绑定的 name 是标签匹配选择器的 name，这样叫这个名字的 Pod 都会注册到一个服务里，形成一个负载均衡

```yaml
apiVersion: v1
kind: Service
metadata:
  name: deployment-demo-service
spec:
  selector:
    name: deployment-pod-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763639260394-ccb28013-9db1-4de9-88b1-81a31e8041c8.png)



金丝雀部署

通过在创建一个 deployment，名为deployment-demo-canary，它的 Pod 数量为 1 个，单个容器

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-demo-canary
  labels:
    app: deployment-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      name: deployment-pod-demo
      version: canary
  template:
    metadata:
      labels:
        name: deployment-pod-demo
        version: canary
    spec:
      imagePullSecrets: 
      - name: regcred
      containers:
      - name: nignx-container
        image: docker.xuanyuan.run/nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763639907764-a4ccd39a-c4b5-44d7-9e86-1e4c5b0c39bc.png)



进入新版本容器，修改 index 内容

```yaml
kubectl exec -it deployment-demo-canary-788cc5b4bf-fmvh4 -c nignx-container /bin/sh

cd /usr/share/nginx/html/ && mv index.html index.html.back && echo New_version > index.html
```



在 curl 几次后负载将新版本流量分配到了我们的新版本上，说明我们的部署方式有效，旧版本与新版本都在一个集群当中

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763640083201-57f4bba7-ea54-4bd1-9fd2-eb0934bc18a0.png)





除了使用 yaml 通过声明式的方式，还可以使用scale 进行一个补丁修改达到金丝雀部署的效果

```yaml
kubectl scale deployment deployment-demo --replicas=1

kubectl set image deployment/deployment-demo nginx-container=docker.xuanyuan.run/nginx:alpine
```





两个不同的 deployment ，即使有一个是三个 Pod 双容器，一个是单个 Pod 单容器，通过 service 也可以注册到一个服务当中



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763640255683-e1eb6989-1cdb-479f-a8cc-bbd3921f1cb9.png)

service 只匹配selector（选择器）中包含name: deployment-pod-demo 的键值对







#### 回滚实验
创建一个 deployment 控制器

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-demo
  labels:
    app: deployment-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      name: deployment-pod-demo
  template:
    metadata:
      labels:
        name: deployment-pod-demo
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: nignx-container
        image: docker.xuanyuan.run/nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

```

创建一个 service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: deployment-demo-service
spec:
  selector:
    name: deployment-pod-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763640909225-20d9e3ae-777a-4294-ba51-6f0ca8fd7bc3.png)



修改 nginx 版本为alpine

```yaml
kubectl set image deployment/deployment-demo nignx-container=docker.xuanyuan.run/nginx:alpine
```

确认 revision

```yaml
kubectl rollout history deployment deployment-demo
```



确认 image 版本为alpine

```yaml
kubectl describe deployment deployment-demo
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763641111094-7da7ff72-528c-40a2-9097-20bd0282ecb0.png)

回滚，版本已退回 latest 版本

```yaml
kubectl rollout undo deployment deployment-demo
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763641205322-74365c18-791a-467c-90a7-02a2c0c17586.png)

默认的回滚到机制

v1 升级到 v2 在升级到 v3

v1——》v2——〉v3

v3 有问题，需要回滚

v3——》v2

在回滚一次

v2——〉v3

这里 v2 没有回滚到 v1 因为 v2 的上一个版本是 v3



回滚的一些常用命令

```yaml
# 历史版本
kubectl rollout history deployment deployment-demo

# 实时监控回滚状态，并在成功后返回一个0 echo $?
kubectl rollout status deployment deployment-demo

# 回滚到指定版本
kubectl rollout undo deployment deployment-demo --to-revision=2


# 在输出的 yaml里查看 annotations 版本
kubectl get deployment deployment-demo -o yaml
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763642275111-7110346e-42d2-4392-8911-bb6c87340140.png)

如果要添加 CHANGE-CAUSE 注释，在回滚的时候添加--record，但是新版本会逐步废弃这个命令，推荐使用annotate 来记录

如果使用record 进行注释，如果上一条有注释下一个回滚没有，它会抄上一个回滚的注释

```yaml
kubectl annotate deployment deployment-demo   kubernetes.io/change-cause="回滚到稳定版本"
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763641913866-77c73a1f-bd1f-483e-b36a-7a3355cb734c.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763642784162-3856f289-1fd3-469a-bcb4-f3e738c620a0.png)

这个是官方的回滚策略，我们可以根据自己的习惯、思路进行回滚，比如我们有一个 deployment 控制器，把要修改的内容做记录，比如时间-姓名-修改内容等等，需要回滚的时候在使用这个 back 文件即可

```yaml
cp deployment.yaml 202511202053-wpsec-image-latest-deployment.yaml
```



如果在回滚的时候，并行回滚，k8s 会直接回滚到最后执行的回滚命令上

执行命令    再执行了回滚到 v3，剩下的 Pod 就不会再先到 v2 在到 v3，而是直接从 v1 到 v3

v1——v2    v1——》v3

## 
#### 暂停与恢复
暂停 Deployment 不会影响已经运行的 Pod，它是暂停了滚动更新

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763964342269-bc4a57a3-25ca-4404-b75f-1c308b210129.png)



```yaml
# 暂停 deployment
kubectl rollout pause deployment deployment-demo
```

因为被暂停，即时修改了镜像，已经运行的也不会改变

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763965393691-6cbe7b6c-3d88-4de7-9848-30a694cc95d8.png)

恢复，在恢复后因为镜像不同，重新运行了新的 Pod

```yaml
kubectl rollout resume deployment deployment-demo
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763965453272-5fa885a8-4472-42d8-a935-5617adba295d.png)





### DaemonSet（DS）
确保每个或一些 Node 节点上都运行特定的 Pod，无论这个节点什么时候离开或加入集群，都会自动管理这些 Pod 的创建和删除。

动态调整：加入、新增、移除、回收



典型场景

每个 Node 上运行监控、日志收集的 Pod，比如Prometheus Node Exporter、Datadog Agent 、Fluentd 等





```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-demo
  labels:
    app: daemonset-demo
spec:
  selector:
    matchLabels:
      name: daemonset-demo
  template:
    metadata:
      labels:
        name: daemonset-demo
    spec:
      containers:
        - name: daemonset-demo-container
          image: docker.xuanyuan.run/nginx:alpine

```

在没有设置 Pod 数量的情况下，DS 默认给每个 Node 都分配了一个 Pod

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763965606258-7b9643fd-f208-4ce1-aef0-6b2a5fe4df95.png)

新创建的 Pod，只运行在 node 上，不运行在 master 上，master 有一个污点

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763966072648-1646868f-89f8-4faf-94a5-2a2ea1496b46.png)

### Job/CronJob
#### Job
批处理任务

Job 负责仅执行一次性任务，它保证批处理任务的一个或多个 Pod 成功结束

特点

+ spec.template 格式同 Pod
+ RestartPolicy 仅支持 never（一次性运行） 或 OnFailure（失败重启）不支持Always（无条件重启容器）
+ 单个 Pod 时，默认成功运行后，Job 结束
+ completions，成功几次，默认 1，返回码 0 为成功，1 为失败
+ parallelism，标志并行 Pod 个数，默认为 1，比如并行数为 5，最大值为 5 即可，第一次创建 5 个，成功 2 个，第二次创建 3 个，成功 3 个，结束
+ activeDeadlineSeconds，失败 Pod 最大重试时间，默认单位秒



completions=5：总共需要 5 次成功运行

parallelism=3：最多并行跑 3 个 Pod

backoffLimit=0：失败不重试 → Pod 失败后不会重新创建

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
  labels:
    app: job-demo
spec:
  completions: 5
  parallelism: 3
  backoffLimit: 0
  template:
    metadata:
      labels:
        name: job-pod-demo
    spec:
      restartPolicy: Never
      containers:
      - name: job-demo-container
        image: docker.xuanyuan.run/nginx:alpine
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "echo start; sleep 5; exit 1"]
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763968277725-f188c000-3935-4087-8c23-626281afcab0.png)



#### CronJOB
管理基于时间的 Job

+ 在给定时间只运行一次
+ 周期性的在给定时间点运行
+ spec.schedule：调度，必须字段，指定运行周期，格式通 Cron
+ spec.jobTemplate：job 模版，格式同 Job
+ spec.startingDeadlineSeconds：启动 Job 期限，单位秒，可选
+ spec.concurrencyPolicy：并发策略
    - Allow（默认）：允许并发运行 Job
    - Forbid：禁止并发，前一个没完成，跳过下一个
    - Replace：如果前一个还在运行，取消前一个，运行新的
    - 比如发邮件，默认并发；备份数据库，禁止并发
+ spec.suspend：挂起，设置为 true，后续的执行都会被挂起，已经开始执行的 Job 不受影响，理解为虚拟机挂起，需要时，快速访问，不需要时挂起，默认 false
+ spec.successfulJobsHistoryLimit 和 spec.failedJobsHistoryLimit：历史限制，保留多少 Job，默认保留 3 个 Job 信息和 1 个最近失败的 Job 信息



没分钟执行一次、保留 3 条成功记录、保留 1 条失败记录

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-demo
  labels:
    app: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            name: cronjob-pod-demo
        spec:
          restartPolicy: Never
          containers:
            - name: cronjob-demo-container
              image: docker.xuanyuan.run/nginx:alpine
              imagePullPolicy: IfNotPresent
              command:
                - sh
                - "-c"
                - |
                  echo "CronJob start at $(date)"
                  sleep 10
                  echo "Finish"

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763973982386-ae466f00-e633-420c-97c9-73fa15177e22.png)



