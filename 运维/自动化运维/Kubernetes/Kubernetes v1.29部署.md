# K8s 安装
## kubeadm
容器化

1. 简单
2. 装一台机器上性能上可能不理想

## 二进制安装
本地宿主机安装

1. 安装复杂，组件都需要配置
2. 可装在不同主机上，性能更好

## 一主两从-vm 虚拟机部署
根据自己的情况部署以下配置

自定义网络，出网，通过 ikuai 充当路由器

192.168.66.1/24

桥-不配置网关，不出网，内网 ssh 使用（vm 在 windows 机器上，mac 通过同局域网下 ssh 连）

192.168.1.1/24



创建虚拟网卡

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762937384261-447c0272-d0d3-457b-9ac4-fd1cde7c1dec.png)

![](https://cdn.nlark.com/yuque/__mermaid_v3/13e4355039a6b8c7014d0087e381e0a6.svg)





#### ikuai
新装一个 ikuai 做为路由器为自定义网络的 k8s 访问互联网使用

eth0-wan 口-桥接- 互联网  192.168.1.19/24

eth1-lan 口-仅主机-内网路由 192.168.66.11/24

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762937502121-3b45a386-e38c-49d1-985a-a50d0b633021.png)

配置 lan 口

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762941821537-4d0b5b63-a912-4897-82d8-ffcf5451c395.png)

配置 wan 口

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762941865146-48095aff-1e47-47fe-90cd-7e5353a50b47.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762941847831-39e9ce8f-5194-493b-9c1a-92eed1f14a83.png)

#### master、node1、node2
Rocky Linux9

4c4g

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762860016356-ad443984-8039-4c6c-baaf-cd4844d639f8.png)

两张网卡，第一张连接 ikuai 的路由器，第二张桥接，不出网，模拟的我的内网

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762939205008-460e2fed-fde8-41e9-9393-75ea37192b25.png)



防火墙替换

```yaml
systemctl stop firewalld
systemctl disable firewalld
yum install iptables-services -y
systemctl start iptables
iptables -F
systemctl enable iptables
service iptables save
```



IP 配置

```bash
[root@localhost ~]# cat /etc/NetworkManager/system-connections/ens160.nmconnection 
[connection]
id=ens160
uuid=e77fd435-1784-39fe-aa64-e3adf26c2adc
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1762887704

[ethernet]

[ipv4]
#method=auto
method=manual
address=192.168.66.11/24,192.168.66.200
dns=114.114.114.114

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
[root@localhost ~]# cat /etc/NetworkManager/system-connections/ens192.nmconnection 
[connection]
id=ens192
uuid=454add40-c04b-30b9-88eb-5fb75f3cb530
type=ethernet
autoconnect-priority=-999
interface-name=ens192
timestamp=1762938245

[ethernet]

[ipv4]
address1=192.168.1.6/24
method=manual
never-default=true

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]


systemctl restart NetworkManager
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762942290782-64385747-2271-4324-be53-c0d7e0b8a9b2.png)

初始化

```yaml
# 源
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-*.repo

dnf makecache

# SElinux关闭
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
grubby --update-kernel ALL --args selinux=0

# 时区
timedatectl set-timezone Asia/Shanghai
```

docker、ipvsadm 管理工具、开启路由转发、开启所有流量经过 iptables 规则

```yaml
yum install -y bridge-utils
yum install -y epel-release
yum install -y yum-utils
yum install -y ipvsadm

# 加载 br_netfilter 模块并设置为开机自动加载
cat <<EOF | sudo tee /etc/modules-load.d/bridge.conf > /dev/null
br_netfilter
EOF
modprobe br_netfilter

# 写入 Kubernetes 网络所需的 sysctl 参数
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf > /dev/null
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

# 应用配置
sysctl --system


yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

指定 Docker 存储所有数据的根目录，设置 Docker 使用的 cgroup 驱动为 systemd，定义 Docker 容器日志的存储驱动，设置单日志大小 100 m，旧日志文件数量 100

```yaml
cat > /etc/docker/daemon.json <<EOF
{
    "data-root": "/data/docker",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "100"
    }
}
EOF
```

docker 启动目录

```yaml
mkdir -p /etc/systemd/system/docker.service.d
systemctl enable docker
```



关闭 swap 分区，不是必须的，可跳过安装时的警告

```yaml
sed -i 's:/dev/mapper/rl-swap:#/dev/mapper/rl-swap:g' /etc/fstab
```

主机名

```yaml
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02
```

```yaml
192.168.66.11   k8s-master01 m1
192.168.66.12   k8s-node01 n1
192.168.66.13   k8s-node02 n2
192.168.66.14		harbor
```

重启

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762943316588-b6a81e72-94f0-4978-b67c-09cc7b93717c.png)

安装 cri-docker

k8s 在 1.20 版本抛弃了 docker，因为一些历史原因，docker 持续不兼容 CRI 格式，最开始 k8s 负责兼容 docker，用containerd-shim，k8s 最终选择不兼容 docker，但是因为大家系统使用 docker，所以有了一个 cri 的一个插件，让 k8s 还能继续支持 docker

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762948912355-a28d3bd8-c517-4b3a-bae0-450205b41797.png)

[https://github.com/mirantis/cri-dockerd](https://github.com/mirantis/cri-dockerd)

```yaml
export https_proxy=http://127.0.0.1:7897 http_proxy=http://127.0.0.1:7897 all_proxy=socks5://127.0.0.1:7897
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.21/cri-dockerd-0.3.21.amd64.tgz
tar -zxvf cri-dockerd-0.3.21.amd64.tgz
cp cri-dockerd/cri-dockerd /usr/bin/
chmod +x /usr/bin/cri-dockerd
```



```yaml
sudo tee /usr/lib/systemd/system/cri-docker.service >/dev/null <<"EOF"
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=60
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

```



```yaml
cat > /usr/lib/systemd/system/cri-docker.socket <<"EOF"
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock  # 监听路径：/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```

```yaml
systemctl daemon-reload
systemctl start cri-docker.socket
systemctl restart cri-docker.service
systemctl is-active cri-docker.service
```



#### k8s 的安装
m1，n1，n2

```yaml
[root@k8s-master01 ~]# ls
anaconda-ks.cfg  cri-dockerd-0.3.21.amd64.tgz  kubernetes-1.29.2-150500.1.1.tar.gz
cri-dockerd      kubernetes-1.29.2-150500.1.1
[root@k8s-master01 ~]# cd kubernetes-1.29.2-150500.1.1/
[root@k8s-master01 kubernetes-1.29.2-150500.1.1]# ls
conntrack-tools-1.4.7-2.el9.x86_64.rpm  kubernetes-cni-1.3.0-150500.1.1.x86_64.rpm
cri-tools-1.29.0-150500.1.1.x86_64.rpm  libnetfilter_cthelper-1.0.0-22.el9.x86_64.rpm
kubeadm-1.29.2-150500.1.1.x86_64.rpm    libnetfilter_cttimeout-1.0.0-19.el9.x86_64.rpm
kubectl-1.29.2-150500.1.1.x86_64.rpm    libnetfilter_queue-1.0.5-1.el9.x86_64.rpm
kubelet-1.29.2-150500.1.1.x86_64.rpm    socat-1.7.4.1-5.el9.x86_64.rpm
[root@k8s-master01 kubernetes-1.29.2-150500.1.1]# yum install *
Last metadata expiration check: 1:38:17 ago on Wed 12 Nov 2025 08:25:46 PM CST.
Dependencies resolved.
======================================================================================================================
 Package                            Architecture       Version                         Repository                Size
======================================================================================================================
Installing:
 conntrack-tools                    x86_64             1.4.7-2.el9                     @commandline             221 k
 cri-tools                          x86_64             1.29.0-150500.1.1               @commandline             8.2 M
 kubeadm                            x86_64             1.29.2-150500.1.1               @commandline             9.7 M
 kubectl                            x86_64             1.29.2-150500.1.1               @commandline              10 M
 kubelet                            x86_64             1.29.2-150500.1.1               @commandline              19 M
 kubernetes-cni                     x86_64             1.3.0-150500.1.1                @commandline             6.7 M
 libnetfilter_cthelper              x86_64             1.0.0-22.el9                    @commandline              23 k
 libnetfilter_cttimeout             x86_64             1.0.0-19.el9                    @commandline              23 k
 libnetfilter_queue                 x86_64             1.0.5-1.el9                     @commandline              28 k
 socat                              x86_64             1.7.4.1-5.el9                   @commandline             300 k

Transaction Summary
======================================================================================================================
Install  10 Packages

Total size: 54 M
Installed size: 297 M
Is this ok [y/N]: 
```

自启动

Kubelet 是 Kubernetes 工作节点上的核心代理组件，负责管理 Pod 和容器的生命周期，确保它们按照预期运行，并与控制平面通信。

```yaml
systemctl enable kubelet.service 
```



初始化主节点

```yaml
# 指定 API 服务器对外通告的 IP 地址（这里是 192.168.66.11）
# 其他节点将使用这个地址连接到控制平面
--apiserver-advertise-address=192.168.66.11

# 指定从阿里云镜像仓库拉取 Kubernetes 所需镜像
#替代默认的 k8s.gcr.io
--image-repository registry.aliyuncs.com/google_containers

# 指定要安装的 Kubernetes 版本（这里是 1.29.2）
--kubernetes-version 1.29.2

# 指定 Service 的 IP 地址范围（CIDR）
# 这个范围应该与任何网络组件不冲突
--service-cidr=10.10.0.0/12

# 指定 Pod 网络的 IP 地址范围（CIDR）
# 这个范围通常由 CNI 插件使用（如 Flannel）
--pod-network-cidr=10.244.0.0/16

# 忽略所有预检错误
--ignore-preflight-errors=all

# 指定 CRI（容器运行时接口）的套接字路径，docker的cri
--cri-socket unix:///var/run/cri-dockerd.sock
```

```yaml
kubeadm init --apiserver-advertise-address=192.168.66.11 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.29.2 --service-cidr=10.10.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all --cri-socket unix:///var/run/cri-dockerd.sock
```

配置连接密钥

```yaml
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762958206624-1d658fb4-23ef-4046-8f4d-83a2acf49813.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762958250725-ae6ebe53-c9db-42ac-ac8a-628533d84cb6.png)

node1 和 node2 加入集群

复制 master 给出的命令，加上 cri-docker 套接字

```yaml
# work token 过期后，重新申请
# kubeadm token create --print-join-command

kubeadm join 192.168.66.11:6443 --token q6o23u.0yr1i1rk570s9bu7 \
        --discovery-token-ca-cert-hash sha256:8fb7c4286b77a8a953d02cfedf65dd6022e81a039b83f4c8a05d4980cde28ae5 --cri-socket unix:///var/run/cri-dockerd.sock
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762958561980-de2e44a3-bc5f-4883-b1d8-26e45826c0f3.png)

所有 node 已经加入了，但是是为就绪状态，因为它们没有工作在一个扁平化的网络空间下。

[https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes)

Calico 网络的安装 、master 所有 ndoe 都需要安装

```yaml
# calicoctl 执行文件
calicoctl-linux-amd64

# 配置文件，通常需根据集群环境修改 datastore 和 API 端点配置
calicoctl.yaml

# Typha 组件的部署清单，大规模集群（>50节点）建议启用
calico-typha.yaml	

# calico 离线镜像包
calico-images.tar.gz	

tar xvf calico-images.tar.gz
docker load -i calico-cni-v3.26.3.tars
docker load -i calico-kube-controllers-v3.26.3.tar
docker load -i calico-node-v3.26.3.tar
docker load -i calico-typha-v3.26.3.tar
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762960320474-0a15d771-4abe-435d-bc52-eebefdccc5d8.png)



在 master 中修改calico-typha.yaml

```yaml
CALICO_IPV4POOL_CIDR	指定为 pod 地址
	
# 修改为 BGP 模式
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
  value: "Always"  #改成Off

固定网卡(可选)
# 目标 IP 或域名可达
        - name: calico-node
          image: registry.geoway.com/calico/node:v3.19.1
          env:
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            - name: IP_AUTODETECTION_METHOD
              value: "can-reach=www.google.com"
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=can-reach=www.google.com
# 匹配目标网卡
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "interface=eth.*"
# 排除匹配网卡
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "skip-interface=eth.*"
# CIDR
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "cidr=192.168.200.0/24,172.15.0.0/24"
```

应用 calico

```yaml
kubectl apply -f calico-typha.yaml
kubectl get pod -A
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762961391570-d593f3ad-b31d-4e45-93bd-9bd4e843cc90.png)

如果遇到问题，可以依次检查，这些是否启起来

```yaml
docker
cri-docker
kubelet
```



k8s 在部署的时候，生成了一些证书，因为组件与组件之间都采用 https 的方式进行通信

```yaml
[certs] Using certificateDir folder "/etc/kubernetes/pki"
```

如果 k8s 的镜像拉不起来，可以预先 pull，排查问题

```yaml
kubeadm config images pull
```

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1763004718618-bdb0e14d-e56a-4c45-9e5b-c6a835baee52.png)

配置文件都在/etc/kubernetes 下

admin.conf 文件

是集群管理员使用的 kubeconfig 文件，包含访问 Kubernetes API 的凭据（证书、密钥、API Server 地址）

controller-manager.conf 

是Kubernetes Controller Manager 组件的 kubeconfig 文件，用于与 API Server 通信

scheduler.conf 

是Kubernetes Scheduler 组件的 kubeconfig 文件，用于与 API Server 通信

kubelet.conf

Kubelet 服务的 kubeconfig 文件，用于节点（Node）与 API Server 的认证和通信  



linux->docker->cri-docker->kubelet->AS/CM/SCH







