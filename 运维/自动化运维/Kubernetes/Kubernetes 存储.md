## 存储分类
元数据

+ configMap：用于保存配置数据（明文）
+ Secret：用于保存敏感性数据（编码）
+ Downward API：容器在运行时从 kubernetes API 服务器获取有关它们自身的信息

真实数据

+ Volume：用于存储临时或持久的数据
+ PersistentVolume：申请制的持久化存储

## configMap
假如我们有一个集群环境，集群里有几百台 nginx，如果需要修改 nginx 的配置文件，为了不用一台一台修改，就可以使用 configMap 来进行修改

configMap 在容器中注入配置信息，可以保存单个属性、整个配置文件、json、二进制等对象

![](https://cdn.nlark.com/yuque/__graphviz/5756634f85f467c1bd34088957691914.svg)

ConfigMap 是通过注入把配置文件给到对用的 Pod，因为ConfigMap 设计出来就是为了提供配置文件，而用注入不用共享的好处是：

1. 避免共享带来的锁、覆盖、冲突问题
2. 注入减少 i/o，配置文件在 Pod 启动时一次性加载到容器中，避免像共享一样通过网络访问



### 创建 configmap
单个配置文件时

有一个配置文件 app.conf

里面是这样的

使用--from-file 时候，配置文件内必须是键值对，在注入 pod 后会自动变为环境变量

```yaml
a=1
b=2
...
```

```yaml
kubectl create configmap app-config --from-file=app.conf

kubectl describe configmap app-config
```

多个配置文件需要注入时

多个配置文件，以目录形式创建

```yaml
config/
  db.conf
  redis.conf
  app.yaml
```

```yaml
kubectl create configmap my-config --from-file=config/
```



生产环境一般这样做



```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  env: "prod"
  timeout: "30"
  log.conf: |
    level=info
    format=json

```

声明式

```yaml
kubectl apply -f configmap.yaml
```

可以先将配置文件以命令的方式生成，在进行修改，这样做的好处是，如果我们的配置文件特别多，不用我们自己写，先生成在改

```yaml
kubectl create configmap test-config --from-file=test.file --dry-run -o yaml > configmap-test.yaml
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764054628181-27b00ea0-3f4f-4c97-bf13-eb14b5f28ae7.png)

跟控制器、服务一样，都可以通过 get、describe 等等命令来查看详细信息，configmap 可以缩写为 cm

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764054760008-9d2635d7-24eb-4562-8082-7b5efb1f9186.png)



另外，支持从命令行注入，但是不建议，这样做会很混乱

```yaml
kubectl create configmap cm-demo --from-literal=env=prod --from-literal=debug=false
```



### configmap 环境变量实验
创建一个 configmap，配置了配置文件（也默认成了环境变量），创建第二个资源对象Pod，通过---隔开，创建一个 mainc 主容器，输出 env

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-demo
data:
  APP_MODE: "wpsec"
  APP_VERSION: "1.0.0"
  APP_DEBUG: "false"
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-pod
spec:
  containers:
  - name: test-container
    image: docker.xuanyuan.run/nginx:alpine
    command: ['sh', '-c', 'echo "Environment Variables:"; env; sleep 3600']
    envFrom:
    - configMapRef:
        name: env-demo
  restartPolicy: Never

```



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764055267987-87d172ba-6c24-4573-8f60-801e63baabdf.png)

### 环境变量作启动命令来调用
创建一个 configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-env
data:
  user: "admin"
  pass: "123456"
```

创建一个 pod，将启动命令改为输出环境变量的账户密码

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-env-pod
spec:
  containers:
  - name: echo-container
    image: docker.xuanyuan.run/nginx:alpine
    envFrom:
    - configMapRef:
        name: simple-env
    command: ['sh', '-c']
    args:
      - |
        echo "user=$user pass=$pass";
        sleep 3600
  restartPolicy: Never

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764055907231-9b66a248-13d2-4c6c-a96a-e4d19bc39c51.png)



### volumeMounts 挂载配置文件方式
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: file-config
data:
  app.conf: |
    username=admin
    password=123456
    mode=production
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-file-pod
spec:
  containers:
  - name: test-container
    image: docker.xuanyuan.run/nginx:alpine
    command: ['sh', '-c']
    args:
      - |
        cat /etc/config/app.conf;
        sleep 3600
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: file-config
  restartPolicy: Never

```

挂载进行的 config 文件是以链接的形式存在，避免在注入的时候损坏，这个文件是热更新的（已有文件），如果你在 ConfigMap 中新增了 new.conf，Pod 内不会自动生成 /etc/config/new.conf，所以新增或删除 key 需要重启 Pod 才行。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764056639115-7e482a0c-1688-445c-8dd9-a1d9aa55a0c4.png)

### volumeMounts 单独键声明
同 secret



总结：

1. 修改已有的配置时，因为是注入方式，不会实时更新，需要时间
2. 在修改 nginx 这样的服务端口时，可能配置文件更新了，但是 nginx 还是没有更新，因为 nginx 需要重启生效，这是 nginx 的问题，nignx 不支持热更新，不是 configmap 不支持
3. 使用 confimap 挂载的 env 不会热更新



### configmap immutable 不可改变
在业务稳定后，我们需要不可改变 configmap 文件，防止误改变导致业务崩溃，减少 apiserver 的压力

immutable: true 不可改变

这个选项是不可逆的，如果想要修改，只能重建 configmap 了

```yaml
apiVersion: v1
kind: ConfigMap
immutable: true
metadata:
  name: file-config
data:
  app.conf: |
    username=admin
    password=123456
    mode=production
```

## Secret
保存敏感信息，密码、ssh 密钥等，secret 也可以热更新，通过编码实现相对安全性

+ 需要访问 Secret 等 Pod 才能访问，不是所有 Pod 都需要访问
+ Secret 只会存储在节点的内存中，永不写入物理磁盘
+ k8s1.7 版本后 etcd 会以加密形式存储 secret
+ 其它功能同 configmap，比如单独键声明，immutable 不可改变等

| Opaque | 默认类型，任意 key-value 数据，用户自定义内容 |
| --- | --- |
| kubernetes.io/service-account-token | 用于 ServiceAccount 的自动令牌挂载，系统生成 token、CA 等信息 |
| kubernetes.io/dockercfg | 旧版 Docker registry 认证信息，用于 ~/.dockercfg（不推荐，新版用 docker-registry） |
| kubernetes.io/dockerconfigjson | Docker registry 认证信息，用于 ~/.docker/config.json |
| kubernetes.io/basic-auth | 基本认证信息，通常包含 username 和 password |
| kubernetes.io/ssh-auth | SSH 密钥信息，通常包含 ssh-privatekey |
| kubernetes.io/tls | TLS 证书和私钥，包含 tls.crt 和 tls.key |
| bootstrap.kubernetes.io/token | 用于集群启动或节点引导的 token（kubeadm 使用） |


### Opaque
Secret 的 data 需要 base64 编码使用

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-opaque-secret
type: Opaque
data:
  username: d3BzZWM=
  password: aGVsbG8=
```



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764060331731-ca61b7e6-7a1a-4af8-a46d-fcf208f407ac.png)

Secret Opaque 与 configmap 对比，configmap 直接显示了明文，secret 只显示了大小

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764060504892-9f3f7edf-5d1c-4d49-8b1b-dcb6d0251280.png)



#### Opaque volume
volumeMounts:容器级别的配置

volumes:Pod级别的配置

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-opaque-secret
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: test-container
    image: docker.xuanyuan.run/nginx:alpine
    command:
      - sh
      - -c
      - "sleep 3600"
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-opaque-secret
  restartPolicy: Never
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764062673351-53e6b87e-eb70-437a-a051-f945ed389fea.png)

#### volumeMounts 单独键声明
这个跟 configmap 的一样

比如只挂载一个myuser.txt，不挂载 secret 里的其它内容，但是这样挂载不会以链接形式了，会写入一个正常的文件（普通文件）

secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-opaque-secret
type: Opaque
data:
  username: YWRtaW4=   
  password: MTIzNDU2 
```

pod 的volumes 部分，指定只挂载 username

```yaml
  volumes:
  - name: secret-volume
    secret:
      secretName: my-opaque-secret
      items:
      - key: username
        path: myuser.txt
  restartPolicy: Never
```

#### Secret immutable 不可改变
同 configmap immutable

```yaml
apiVersion: v1
kind: Secret
immutable: true
metadata:
  name: secret-file
data:
  username: YWRtaW4=   
  password: MTIzNDU2 
```



### kubernetes.io/basic-auth
单独在说一下这个场景，在一些业务场景，需要对外暴露不成熟或者不确定是否非常安全的 http/s 系统，可以使用basic 来进行认证，有以下好处

1. 暴露在公网的服务，避免被僵尸机/肉机扫描
2. 在不动业务的情况，有一层安全防护、且部署简单
3. 避免被外部网络空间扫描引擎捕获状态



#### 一个简单的 basic 测试
```yaml
根据不同的linux发行版本安装htpasswd
sudo apt-get install apache2-utils -y

sudo yum install httpd-tools -y
```

生成一个basic 账户密码

```yaml
htpasswd -bc auth admin 123456
```

base64 编码



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764076850436-9c542e90-7f3c-468c-8f39-f19afbe9857a.png)

secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-basic-auth
type: Opaque
data:
  auth: YWRtaW46JGFwcjEkbXRqNlNVSGwkd0xrdlphSDlVb3FmaThnbWt3SHcuLg==
```

configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-basic-config
data:
  default.conf: |
    server {
      listen 81;
      server_name  _;

      location / {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/auth/auth;
        root   /usr/share/nginx/html;
        index  index.html;
      }
    }
```



deployment，模拟一个 http 服务，对互联网开放，挂载配置文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-basic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-basic
  template:
    metadata:
      labels:
        app: nginx-basic
    spec:
      volumes:
      - name: auth-volume
        secret:
          secretName: nginx-basic-auth
      - name: config-volume
        configMap:
          name: nginx-basic-config
      containers:
      - name: nginx
        image: docker.xuanyuan.run/nginx:alpine
        ports:
        - containerPort: 81
        volumeMounts:
        - name: auth-volume
          mountPath: /etc/nginx/auth
        - name: config-volume
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
```

service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-basic-svc
spec:
  type: NodePort
  selector:
    app: nginx-basic
  ports:
  - port: 81
    targetPort: 81
    nodePort: 30080
```



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764149267448-6b15c62c-c5e5-41c1-b04a-432916715b0f.png)







## Downward API
容器在运行时从 kubernetes api 服务器获取有关它们自身的信息，而部分信息 k8s 会自动给到，但如果我们需求，需要自定义，可以通过Downward API 来实现，比如 Pod 名称、命名空间、标签、注释、容器资源限制等。



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
  namespace: default
  labels:
    app: demo
    tier: backend
  annotations:
    note: "downward API 自定义测试"
spec:
  containers:
  - name: demo-container
    image: docker.xuanyuan.run/nginx:alpine
    command: ['sh']
    args: ['-c', 'sleep 3600']
    env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_LABEL_APP
        valueFrom:
          fieldRef:
            fieldPath: metadata.labels['app']
      - name: POD_ANNOTATION_NOTE
        valueFrom:
          fieldRef:
            fieldPath: metadata.annotations['note']
    volumeMounts:
      - name: podinfo
        mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "podname"
            fieldRef:
              fieldPath: metadata.name
          - path: "namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "label-app"
            fieldRef:
              fieldPath: metadata.labels['app']
          - path: "annotation-note"
            fieldRef:
              fieldPath: metadata.annotations['note']
  restartPolicy: Never

```



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764064818385-0cac5a13-3053-4099-a005-01f2b0e449fe.png)

通过Downward API 的方式可以实现容器和 Pod 传递元数据

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name        # Pod 的名称

  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace   # Pod 所在的命名空间

  - name: POD_LABEL_APP
    valueFrom:
      fieldRef:
        fieldPath: metadata.labels['app']   # Pod 的标签 app

  - name: POD_ANNOTATION_NOTE
    valueFrom:
      fieldRef:
        fieldPath: metadata.annotations['note']   # Pod 的注解 note

```





### 通过downward-api限制 CPU/内存实验
创建一个 pod，设置resources

limits：最大

requests：保证

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  labels:
    app: demo
spec:
  containers:
  - name: demo-container
    image: docker.xuanyuan.run/nginx:alpine
    command: ['sh', '-c']
    args: ['sleep 3600']
    resources:
      limits:
        cpu: "500m"
        memory: "128Mi"
      requests:
        cpu: "250m"
        memory: "64Mi"
    env:
      - name: CPU_LIMIT
        valueFrom:
          resourceFieldRef:
            containerName: demo-container
            resource: limits.cpu
      - name: MEM_LIMIT
        valueFrom:
          resourceFieldRef:
            containerName: demo-container
            resource: limits.memory
      - name: CPU_REQUEST
        valueFrom:
          resourceFieldRef:
            containerName: demo-container
            resource: requests.cpu
      - name: MEM_REQUEST
        valueFrom:
          resourceFieldRef:
            containerName: demo-container
            resource: requests.memory
  restartPolicy: Never

```



```yaml
/ # cat /sys/fs/cgroup/memory.max
134217728
/ #
/ # cat /sys/fs/cgroup/memory.current
712704
/ # cat /sys/fs/cgroup/cpu.max
50000 100000
```



## volume
数据的持久化方案

容器磁盘上的生命周期是短暂的，Pod 死了就会拉一个新的 Pod，所以可能会丢失一些文件，或者在 Pod 中运行多个容器时，需要容器之间共享文件，volume 可以解决这些问题



### emptyDir 共享
1. 生命周期绑定 Pod
    - `emptyDir` 卷在 Pod 被 调度到节点 后创建。
    - 只要 Pod 在节点上存在，`emptyDir` 卷就存在。
2. 最初为空
    - 当 Pod 第一次启动时，`emptyDir` 卷是 空的。
3. 共享访问
    - Pod 中的 所有容器 可以访问同一个 `emptyDir` 卷。
    - 容器之间可以 读取、写入相同文件，用于临时共享数据。
4. 删除即消失
    - 当 Pod 从节点上被删除（不管是正常删除还是节点故障导致的调度迁移），`emptyDir` 中的数据 会永久丢失。
5. 可选类型
    - 默认是存储在节点的磁盘上，但可以通过 `emptyDir.medium: "Memory"` 将其放在 内存 (tmpfs) 中，速度更快，但占用的是节点内存，Pod 删除后同样丢失。

使用场景：

+ 临时存储缓存数据、工作目录、临时文件等，不需要持久化。
+ 适合 Pod 内容器间快速共享数据，但 不适合作为长期存储

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emptydir-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emptydir-demo
  template:
    metadata:
      labels:
        app: emptydir-demo
    spec:
      containers:
      - name: nginx-a
        image: docker.xuanyuan.run/nginx:alpine
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      - name: nginx-b
        image: docker.xuanyuan.run/nginx:alpine
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-data
        emptyDir: {}

```

一个 deployment ，一个 Pod 下两个容器共享一个emptyDir

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764151387469-e216732f-21a4-4a11-890d-751fd69bc4bb.png)



在 kubelet 的工作目录，默认为

```yaml
/var/lib/kubelet/pods/{podid}/volumes/kubernetes.io~empty-dir/
```

这个下面存放了所有的 emptyDir 中的数据，最终落在了 node 上

```yaml
kubectl get pod emptydir-demo-bf655c894-cn57f -o yaml
# json or yaml
kubectl get pod emptydir-demo-bf655c894-cn57f -o json | grep uid
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764151913154-904a72c9-3580-4c03-9716-42e5837a53f1.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764152105595-fa97175a-0729-4573-9da4-18c54f1b5863.png)



#### emptyDir 内存卷
可以理解为虚拟内存，但是在两个 pod 中是共享的，有些需要低延迟，较快的访问速度的时候可以这样使用。

但是需要注意，内存卷的大小不能超过 pod 的内存限制大小

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emptydir-memory-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emptydir-memory-demo
  template:
    metadata:
      labels:
        app: emptydir-memory-demo
    spec:
      containers:
      - name: writer
        image: docker.xuanyuan.run/nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        command: ["sh", "-c", "while true; do echo $(date) >> /usr/share/nginx/html/log.txt; sleep 5; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      - name: reader
        image: docker.xuanyuan.run/nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        command: ["sh", "-c", "while true; do sleep 10; cat /usr/share/nginx/html/log.txt; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-data
        emptyDir:
          medium: Memory
          sizeLimit: 64Mi

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764153151714-ba34d040-2b73-4321-8cdc-490fd1200eef.png)



### hostPath
hostPath 卷将主机节点的文件系统中的目录或文件挂载到集群中

+ 运行时需要访问 Docker 内部容器；使用/var/lib/docker 的 hostPath
+ 在容器中运行 cAdvisor；使用/dev/cgroups 的 hostpath
+ 允许 pod 指定给定的 hostpath 是否应该在 pod 运行前存在，是否应该被创建以及以什么样的形式存在



hostpath 适合挂载日志收集、主机上的工具，不适合挂普通业务数据

常见类型

| type | 说明 |
| --- | --- |
| DirectoryOrCreate | 不存在则创建 |
| Directory | 必须存在，否则报错 |
| FileOrCreate | 不存在就创建文件 |
| Socket | 必须是 socket 文件 |
| CharDevice | 字符设备 |
| BlockDevice | 块设备 |




```yaml
mkdir -p /data/hostpath-test
echo "wpsec" > /data/hostpath-test/host.txt
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-test
spec:
  containers:
    - name: nginx
      image: docker.xuanyuan.run/nginx:alpine
      volumeMounts:
        - mountPath: /testdata
          name: host-volume
  volumes:
    - name: host-volume
      hostPath:
        path: /data/hostpath-test
        type: Directory

```

这时，肯定是创建不起来的情况，因为你的文件路径在 master 上，而 pod 不一定会创建在 master 上，而这里的 Pod 卷积用的Directory，必须存在，不存在报错

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764165166603-f3df00d6-9ae5-4979-ac41-06023cdb3eb2.png)

在 node01 节点上去创建文件，重构，就可以运行了。但如果有几十上百个 node， 不可能在每台 node 上去创建我们的卷，这时可以使用DaemonSet 解决，在每个 node 上去创建 Pod，但是这样又不够灵活，所以生产环境一般使用共享卷、oss、nfs 之类的或者使用 pv/pvc，不使用 hostpath。

### emptyDir 和 hostPath 区别
+ emptyDir：Pod 创建时自动生成的临时目录，Pod 删除数据就没了，只能在同 Pod 容器间共享。
+ hostPath：挂载宿主机已有路径，数据随 Pod 独立存在，可在同节点多个 Pod 间共享，但跨节点不共享。

## pv/pvc
生产常用

pv（PersistentVolume）：相当于“存储资源”，类似宿主机或云盘上的硬盘块

pvc（PersistentVolumeClaim）：相当于“存储申请”，Pod 通过 PVC 来申请存储。

+ PV = 硬盘/存储资源
+ PVC = 申请硬盘的"合同"
+ Pod 通过 PVC 使用 PV

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764165787416-384b816a-c9a3-4a58-84f6-ad4d6fad3154.png)

### 读写模式（关联条件）
+ 容量：PV 的值不小于 PVC，可以大于、最好一致
+ 读写：必须完全一致，意思是 PV/PVC 的读写模式必须一致
    - 单节点读写 - ReadWriteOnce - RWO
    - 多节点只读 - ReadOnlyMany - ROX
    - 多节点读写 - ReadWriteMany - RWX
+ 存储类：PV 的类与 PVC 的类必须一致，不存在包容降级关系



### 回收策略
+ Retain（保留）：Pod 删除 PVC 后，PV 不会被删除，数据保留，需要管理员手动处理。适合重要数据备份或迁移。
+ Recycle（回收）：Pod 删除 PVC 后，K8s 会执行简单的清空操作（类似 rm -rf /thevolume/*），PV 变为可用状态，可重新被其他 PVC 使用。此策略现在已废弃，生产不常用。
+ Delete（删除）：Pod 删除 PVC 后，PV 及其存储数据会被自动删除。常用于云盘、临时存储场景，方便自动化管理。

建议都使用 Retain

### 状态
+ Available（可用）：一块空闲资源还没被任何声明所绑定
+ Bound（被绑定）：卷已经被声明绑定
+ Released（已释放）：声明被删除，但是资源还没被集群重新声明使用
+ Failed（失败）：该卷的自动回收失败



### PVC 保护
当用户执行 `kubectl delete pvc <name>` 时：

+ K8s 检查这个 PVC 是否仍在被 Pod 使用
+ 如果正在使用 → 删除被阻止，直到 Pod 不再使用 PVC

Pod 删除或解绑后，Finalizer 被移除，PVC 才能真正删除。



PVC 保护机制针对的是 PVC 对象，不是 PV，如果你要删除 PV，即时 PV 正在使用，也会被删除，不会受保护机制影响



### pv/pvc 实验
| 特性 | 无状态应用 | 有状态应用 |
| --- | --- | --- |
| 数据 | 不依赖本地数据，重启/扩容不影响业务 | 依赖本地或持久化数据，重启/扩容需要保留数据 |
| Pod 可替换性 | Pod 可以随意删除、扩缩容，业务不受影响 | Pod 删除会影响业务，需要数据保留或迁移 |
| 例子 | Web 前端、API 服务、缓存（非持久化） | 数据库（MySQL、PostgreSQL）、消息队列、Redis 持久化模式 |
| 存储需求 | 通常不需要 PV/PVC，或者只用临时存储（emptyDir） | 必须使用 PV/PVC 或网络存储保证数据持久化 |
| 部署策略 | Deployment、ReplicaSet | StatefulSet（保证 Pod 顺序、稳定网络和存储） |


#### 搭建 NFS
在 master 上搭建 nfs 测试服务器作为存储

```yaml
# 在master01、node01、node02上安装
dnf install nfs-utils rpcbind

# 创建nfs目录
mkdir /nfs

# 目录属主
chown nobody /nfs

# 配置nfs
/etc/exports

/nfs/1 *(rw,sync,no_subtree_check)
/nfs/2 *(rw,sync,no_subtree_check)
/nfs/3 *(rw,sync,no_subtree_check)
/nfs/4 *(rw,sync,no_subtree_check)
/nfs/5 *(rw,sync,no_subtree_check)
/nfs/6 *(rw,sync,no_subtree_check)
/nfs/7 *(rw,sync,no_subtree_check)
/nfs/8 *(rw,sync,no_subtree_check)
/nfs/9 *(rw,sync,no_subtree_check)
/nfs/10 *(rw,sync,no_subtree_check)


# 在nsf目录下创建10个目录
cd /nfs
mkdir {1..10}

# 写入测试数据
for i in $(seq 10); do
  mkdir -p "$i"
  echo "$i" > "$i/index.html"
done

systemctl restart nfs-server && systemctl restart rpcbind

showmount -e 192.168.66.11

[root@k8s-node01 ~]# mkdir -p /mnt/test
[root@k8s-node01 ~]# mount -t nfs 192.168.66.11:/nfs/1 /mnt/test
[root@k8s-node01 ~]# ls /mnt/test
index.html
[root@k8s-node01 ~]# umount /mnt/test
```

#### 部署 PV
把 nfs 封装为 PV

创建了 7 个 pv

+ server：NFS 服务器 IP
+ path：NFS 导出的目录
+ ReadWriteMany：多 Pod 可同时读写
+ ReadWriteOnce：单 Pod 读写
+ Retain：Pod 删除 PVC 后数据保留
+ Recycle：Pod 删除 PVC 回收

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfs/1
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv2
spec:
  capacity:
    storage: 0.9Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfs/2
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv3
spec:
  capacity:
    storage: 1.3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfs/3
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv4
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfs/4
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv5
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfs/5
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv6
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs1
  nfs:
    path: /nfs/6
    server: 192.168.66.11
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv7
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/7
    server: 192.168.66.11

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764176751291-ef3f2b1c-194d-4164-9911-b83cdfd3a1be.png)



#### StatefulSet 控制器


这里的 service 是 Headless Service（无头服务），它没有指定clusterIP，也就没有访问的能力，简单来说，无头服务 是一种没有集群IP（Cluster-IP）的Kubernetes服务，它的主要用途是直接返回后端Pod的IP地址，而不是进行负载均衡，一般搭配StatefulSet 控制器使用。

+ 普通Service：当你访问一个普通Service时，你会得到一个稳定的集群IP，流量会被随机转发到后端的某个Pod。
+ 无头Service：当你进行DNS查询时，DNS服务器会直接返回所有后端Pod的IP地址列表，客户端可以直接与这些Pod通信。

无头服务，没有绑定Cluster-ip，但是有 dns 解析

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764176874948-090378e2-04b8-4270-ab5b-9d031ab8e3be.png)

statefulset 副本控制器特点

StatefulSet 会根据 volumeClaimTemplates 自动生成 PVC

+ 有序创建
    - 场景：应用需要严格按节点顺序启动的时候
+ 有序回收
    - 场景：确保数据同步完成后再下线
+ 预选
    - 无论 Pod 被删、被漂移、被重新调度，pod 都有顺序
        * 意义：Kafka、Zookeeper、Redis、Etcd 都要求固定节点 ID、稳定 DNS
    - 满足期望的最低要求、存储空间等于大于期望值、读写模式要相同、类要相同
    - StatefulSet 结合 Headless Service 时会创建固定 DNS
        * web-0.myservice.default.svc.cluster.local
        * web-1.myservice.default.svc.cluster.local
+ 优选
    - StatefulSet 中最关键的“优选”体现在：每个 Pod 优先、并且永远使用自己的卷（PVC + PV）。
        * data-app-0 → PV-0  data-app-1 → PV-1  data-app-2 → PV-2
        * 防止数据混乱、跨 Pod 访问、不一致风险
        * Pod 安排在最合适的节点，优先访问自己的数据
        * 保留模式优先回收（全都一样随机选）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: docker.xuanyuan.run/nginx:alpine
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "nfs"
        resources:
          requests:
            storage: 1Gi

```

有序创建，第一 pod 创建后第二个 pod 才会创建

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764176803338-4890a892-e364-4894-ac9f-2bdc1ac9dd3c.png)



##### 预选
满足期望的最低要求、存储空间等于大于期望值、读写模式要相同、类要相同



将statefulset 的 pod 数量调整到 5 个，第五个 web-4 状态是 Pending 状态

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764177153510-e906d3c9-1013-4a3f-8187-97f6315524d6.png)

根据 describe 显示，没有根据 statefulset 的要求找到合适的 pv

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764177227371-3125dbb3-44f3-4966-8b55-54726269ffe2.png)

根据 pv 显示，有三个可用的 pv，但是它们都不符合StatefulSet 定义的期望要求，其中 nfspv2 大小不够、nfspv5 是多节点读写、nsfpv6 多 storageclass 命名是 nfs1，不是 nfs，所以没有可用的 PV 与 PVC 匹配，pod 就是Pending 状态了

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764177362886-adcab5d9-cada-4655-bc1e-b9cc97c0d1d2.png)

##### 优选-1
通过预选后如果有多个条件，保留模式大于回收模式，同样条件随机，所以 012 随机选了 pv，而到了 web-3，没有同样条件的了，选了一个比 1G 大小大的 pv

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764178120552-c57fecc7-2adf-4be0-bb34-eb5e01051ec3.png)



##### 优选-2
数据持久化 pod 级别，删除 pod，控制器会自动创建一个新的，且优先使用自己之前的卷

pod 绑定 pvc 绑定 pv 绑定 nfs

稳定的访问方式，IP 在重构后会变，但是域名不会

```yaml
<statefulset-pod>.<service-name>.<namespace>.svc.cluster.local

web-0.nginx.default.svc.cluster.local
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764180043220-909d49b2-6882-4d7d-843d-9934160912ab.png)



##### 命名
www 是 卷的名字，web-0 是 pod 的名字，而绑定的 pv 内容就是挂载到 nfs 的内容

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764178547005-7f079b69-95ef-4836-9ecb-689bf9bfe49c.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764179415882-6dbc03f3-2fb1-4558-aa5e-a78b3c3cee47.png)

##### 回收
删除statefulset

```yaml
kubectl delete -f statefulset3.yaml
# or
kubectl delete statefulset statefulsetname
```

删除后 pv/pvc 仍然存在，因为配置的 pv 是Retain（保留）

如果不知道项目是否保留，可以试着保留 pv/pvc，这样重新跑项目的时候，数据也可以保留

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764180526803-15a62fb2-ba53-4386-9f60-fb0814332b60.png)

删除 pvc

不建议在生产环境使用--all 这种参数

```yaml
delete pvc www-web-0 www-web-1 www-web-2 www-web-3 www-web-4
```

释放 pv

```yaml
kubectl delete -f pv.yaml
```



## storageClass
动态的申请存储机制

云供应商 / 存储系统

      ↓ 提供自动创建存储卷功能（通过 CSI 插件）

StorageClass（存储类）

      ↓ 根据 PVC 的需求动态创建 PV

PVC（声明需要多大、多快、什么模式的存储）

      ↓ 等待匹配 PV（动态或静态）

Pod（挂载 PVC）

StorageClass 是一种资源对象，用于定义持久卷，动态供给策略，允许管理员定义不同类型的存储，并指定如何动态创建持久卷使用



### 本地nfs-client-provisioner存储
nfs-client-provisioner 的创建，在本地创建一个 nfs-client 模拟云的存储工厂

nfs

首先，这个nfs-client-provisioner 的场景非常有限，且稳定性、性能、数据一致性、容灾能力、安全性、扩容能力都不及真正的云存储工厂，但是如果是轻量级项目，可以考虑使用。



nfs-client-provisioner 

+ NFS 服务器上创建一个子目录把这个子目录包装成 PV

云存储工厂（例如阿里云 disk plugin）：

+ 创建真实云盘（块存储）
+ 调用云 API
+ 挂载逻辑卷、格式化
+ 管理生命周期（删除/快照/扩容）
+ 具备很高性能、容错、原地扩容等能力



#### 部署nfs-client-provisioner 
在 nsf 服务器上配置一个共享存储地址

```yaml
mkdir /nfs/share
# /etc/exports
/nfs/share *(rw,sync,no_subtree_check)

# 权限限制
chown -R nobody /nfs/share

systemctl restart nfs-server && systemctl restart rpcbind

```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764219482101-81be1579-293a-4f73-ae74-7e5bb0a1d78d.png)



##### 创建命名空间
命名空间通常用来做项目隔离

```yaml
kubectl create ns nfs-storageclass
```



##### deployment
运行 nfs-client-provisioner

+ 监听 PVC 创建
+ 根据 PVC 动态创建子目录
+ 调用 NFS 服务创建目录
+ 动态创建 PV 绑定给 PVC
+ 配置 docker 镜像代理的 secret，名字为 dockerproxy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-storageclass
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      imagePullSecrets:
        - name: dockerproxy
      containers:
        - name: nfs-client-provisioner
          image: docker.xuanyuan.run/dyrnq/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: "192.168.66.11"
            - name: NFS_PATH
              value: "/nfs/share"
      volumes:
        - name: nfs-client-root
          nfs:
            server: "192.168.66.11"
            path: "/nfs/share"

```



##### rbac
赋权

+ 创建 / 删除 PV
+ 更新 PVC 状态
+ 监听 StorageClass、PVC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-storageclass
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-storageclass
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```



##### storageclass
存储类

+ provisioner: 指定谁是工厂  
`k8s-sigs.io/nfs-subdir-external-provisioner`
+ parameters: 指定创建逻辑，例如目录结构  
`pathPattern: ${.PVC.namespace}/${.PVC.name}`
+ reclaimPolicy: 保留 / 删除策略

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  pathPattern: "${.PVC.namespace}/${.PVC.name}"
  onDelete: delete
```

secret

私有镜像仓库的配置

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerproxy
  namespace: nfs-storageclass
type: kubernetes.io/dockerconfigjson
data:
   .dockerconfigjson: "这里填自己的私有镜像地址"
```

```yaml
kubectl apply -f ./
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764225774412-63bfdf46-a3d0-4e3d-98a5-bfd24482f791.png)

##### Pod 与 PVC
这里的 pod 和 pvc 没有创建命名空间（可创建可不创建），因为storageclass 也是没有创建命名空间的，storageclass 是全局可用资源，不属于任何 namespace

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  storageClassName: nfs-client
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  restartPolicy: Never
  containers:
    - name: test-pod
      image: docker.xuanyuan.run/nginx:alpine
      volumeMounts:
        - name: nfs-pvc
          mountPath: /usr/share/nginx/html
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim

```

成功挂载使用

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764226802927-54f5caf7-8701-4d99-90e8-962e51efd32a.png)

因为是动态的，所以 pv 会根据 pvc 的要求，创建对应的容量，动态存储是只提供这样的能力，但是具体的操作还是通过rbac 实现的

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764226845358-8bf4032b-e736-4229-9cc5-a71b98fd6b70.png)

这个目录是自动创建的，其中/nfs/share/是指定的根共享目录，存储工厂指定，default 是 Pod 的命名空间，没有指定命名空间的 Pod 就说 default，test-claim 是创建 pvc 时命名的元数据

```yaml
/nfs/share/default/test-claim/
```

这里的文件权限就是之前创建nfs-client-provisioner 服务时给到的根目录的权限，也就是/nfs/share，

这里的权限很重要，如果 Pod 没有读取写入的权限，那就会跑不起来，状态会在Pending 下

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1764227318591-81882dfb-f4bc-4956-bff1-b91fed438b5d.png)





