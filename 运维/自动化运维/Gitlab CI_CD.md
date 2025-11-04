## 传统传统应用开发模式
开发本地开发、测试，提交到代码版本管理库

运维把应用部署到测试环境，供 QA 测试，测试通过部署到生产环境

QA 进行测试，通过后通知运维发布到生产



问题

错误发现不及时

人工错误发生

团队沟通效率

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762240397667-3e01d4f4-03f1-4666-8d36-8ea239568f10.png)



## 自动化运维环境开发模式
持续集成（Continuous Integration，CI）：代码合并、部署、自动化测试都在一起，不断执行过程，反馈结果



持续交付（Continuous Delivery，CD）：一种软件工程方法，让软件的产出过程在一个短周期内完成，保持随时可发布状态。与持续集成相比，持续交付偏向可交付的产物



持续部署（Continuous Deployment，CD）：通过自动化部署的手段将软件频繁的交付，部署到期望的环境。

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762261118053-7bad7a07-0e99-4460-9440-ded33f9576d1.png)

## Gitlab CI/CD
代码审查、问题跟踪、项目 wiki、多角色、CI 工具集成，有开源版本



Gitlab CI/CD

+ Gitlab 的一部分
+ Gitlab Runner，一个处理构建的应用程序，可单独部署，可通过 API 与 Gitlab CI/CD 一起使用
+ .gitlab-ci.yml





将代码托管到 Git 仓库

在项目根目录创建 ci 文件.gitlab-ci.yml，在文件中指定构建，测试和部署脚本

Gitlab 将检测到它并使用名为 Gitlab Runner 的工具运行脚本

脚本被分组作业，它们共同组成一个管道

![](https://cdn.nlark.com/yuque/__mermaid_v3/9ee3452fc5a4083b294c8f402a434083.svg)





## 安装 Gitlab
rpm 方式

kubernetes 方式

docker 方式

```graphql
docker pull docker.xuanyuan.run/gitlab/gitlab-ce:latest
docker image ls
docker tag fef3e02c1cc7 gitlab/gitlab-ce
docker image rm docker.xuanyuan.run/gitlab/gitlab-ce

vim /etc/hosts
mkdir -p ~/data/gitlab/config ~/data/gitlab/logs ~/data/gitlab/data
docker run -d  -p 443:443 -p 80:80 -p 2222:22 --name gitlab --memory 4g --cpus 2 --restart always -v /root/data/gitlab/config:/etc/gitlab -v /root/data/gitlab/logs:/var/log/gitlab -v /root/data/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce:latest

vim /root/data/gitlab/config/gitlab.rb 
external_url 'http://192.168.1.28'
gitlab_rails['gitlab_shell_ssh_port'] = 2222

docker exec -it gitlab /bin/bash
gitlab-ctl reconfigure

docker exec -it gitlab cat /etc/gitlab/initial_root_password
# WARNING: This password is only valid if ALL of the following are true:
#          • You set it manually via the GITLAB_ROOT_PASSWORD environment variable
#            OR the gitlab_rails['initial_root_password'] setting in /etc/gitlab/gitlab.rb
#          • You set it BEFORE the initial database setup (typically during first installation)
#          • You have NOT changed the password since then (via web UI or command line)
#
#          If this password doesn't work, reset the admin password using:
#          https://docs.gitlab.com/security/reset_user_password/#reset-the-root-password

Password: 85yB+P0RKPxDFAi4LIlXRb6ZmLwNrXiCZqkjJEgt+ag=

# NOTE: This file is automatically deleted after 24 hours on the next reconfigure run.
```



user:root

pass:85yB+P0RKPxDFAi4LIlXRb6ZmLwNrXiCZqkjJEgt+ag=

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762249306669-aa73ffcc-70a8-4921-9994-8da7c758bed0.png)



账户密码的修改

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762257924548-4dd5a9ef-2bb8-429e-936d-d1a41842569f.png)



### Runner
[https://docs.gitlab.com/runner/install/linux-repository/](https://docs.gitlab.com/runner/install/linux-repository/)

[https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/](https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/)

```graphql
根据自己的发行版安装对应版本

lsb_release -a
```

GitLab Runner 是一个与 GitLab CI/CD 配合使用的应用程序，用于在流水线中运行作业。可以理解为 agent



Runner 需要独立部署且版本最好跟 gitlab 同步版本



runner 支持本地、docker、windows 等

支持热加载，修改配置，无需重启



```graphql
docker 部署
mkdir -p ~/data/devops/gitlab-runner/config

docker pull docker.xuanyuan.run/gitlab/gitlab-runner:bleeding
docker image ls
docker tag e7e660656e98 gitlab/gitlab-runner
docker image rm docker.xuanyuan.run/gitlab/gitlab-runner:bleeding
docker run --rm -t -id -v ~/data/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest


注册
gitlab-runner register


```

Runner 类型与状态（旧版）

+ shared，共享类型，运行整个平台的作业（gitlab）
+ group ，项目组类型，运行特定 group 下的所有项目作业（group）
+ specific，项目类型，运行指定的项目作业（project）

状态

+ locked，锁定状态，无法运行项目作业
+ paused，暂停状态，暂时不会接受新的作业





Runner 类型与状态（新版）

+ Instance，整个实例，对应shared
+ group，项目组类型，对应group
+ project，某个项目，对应specific

状态

+ locked，锁定状态，无法运行项目作业
+ paused，暂停状态，暂时不会接受新的作业



其实没有区别，基本上是继承状态，只是名字换了

### Runner 注册
获取 runner token ——》注册



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762252279014-c846308d-003d-4063-9138-4e306ccfe659.png)

旧版：gitlab-runner register

新版：gitlab-runner register 或 gitlab-runner register --url ... --token ...



创建 runner

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762252891618-3266c45a-9a03-4222-9941-78a260cd22ba.png)

复制命令执行就行了



执行器 

+ Docker 执行器 → 在容器里执行 Job；
+ Shell 执行器 → 直接在宿主机上执行；
+ Kubernetes 执行器 → 在 K8s Pod 中执行；



一台机器上可以做多台 runner，只要资源允许

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762253294202-f4649ee5-29d8-4833-851a-9649d48dbdff.png)



### Runner 命令
启动命令

```graphql
gitlab-runner --debug <command>   #调试模式排查错误特别有用。
gitlab-runner <command> --help    #获取帮助信息
gitlab-runner run       #普通用户模式  配置文件位置 ~/.gitlab-runner/config.toml
sudo gitlab-runner run  # 超级用户模式  配置文件位置/etc/gitlab-runner/config.toml
```

注册命令

```graphql
gitlab-runner register  #默认交互模式下使用，非交互模式添加 --non-interactive
gitlab-runner list      #此命令列出了保存在配置文件中的所有运行程序
gitlab-runner verify    #此命令检查注册的runner是否可以连接，但不验证GitLab服务是否正在使用runner。 --delete 删除
gitlab-runner unregister   #该命令使用GitLab取消已注册的runner。


#使用令牌注销
gitlab-runner unregister --url http://gitlab.example.com/ --token t0k3n

#使用名称注销（同名删除第一个）
gitlab-runner   --name test-runner

#注销所有
gitlab-runner unregister --all-runners

```

服务管理

```graphql
gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner

# --user指定将用于执行构建的用户
#`--working-directory  指定将使用**Shell** executor 运行构建时所有数据将存储在其中的根目录

gitlab-runner uninstall #该命令停止运行并从服务中卸载GitLab Runner。

gitlab-runner start     #该命令启动GitLab Runner服务。

gitlab-runner stop      #该命令停止GitLab Runner服务。

gitlab-runner restart   #该命令将停止，然后启动GitLab Runner服务。

gitlab-runner status #此命令显示GitLab Runner服务的状态。当服务正在运行时，退出代码为零；而当服务未运行时，退出代码为非零。

```



### 测试 Runner
测试流水线





## 一个项目的学习
基础环境：

两台 ubuntu server 2404

+ 192.168.1.28 - Gitlab 服务器（已安装 Docker）
+ 192.168.1.27 - Gitlab Runner （已安装 Docker）



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762258566626-8724ce41-5d37-47dc-83b7-484da467db13.png)



### 部署 Gitlab 容器
```graphql
#docker volume rm gitlab-etc
#docker volume rm gitlab-log
#docker volume rm gitlab-opt

docker volume create gitlab-etc
docker volume create gitlab-log
docker volume create gitlab-opt

docker run --name gitlab \
--hostname 192.168.1.28 \
--restart=always \
-p 80:80 \
-p 443:443 \
-v gitlab-etc:/etc/gitlab \
-v gitlab-log:/var/log/gitlab \
-v gitlab-opt:/var/opt/gitlab \
-d gitlab/gitlab-ce:latest


docker logs -f gitlab

sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```



### 部署 Runner
安装 JDK21

```graphql
cd /usr/local
wget https://download.java.net/java/GA/jdk21.0.1/415e3f918a1f4062a0074a2794853d0d/12/GPL/openjdk-21.0.1_linux-x64_bin.tar.gz -O openjdk-21.0.1_linux-x64_bin.tar.gz
tar -xzvf openjdk-21.0.1_linux-x64_bin.tar.gz

cat >> /etc/profile <<-'EOF'
export JAVA_HOME=/usr/local/jdk-21.0.1
export PATH=$PATH:$JAVA_HOME/bin
EOF

source /etc/profile
java -version

```

安装 Apache Maven

```graphql
cd /usr/local
wget --no-check-certificate wget  https://manongbiji.oss-cn-beijing.aliyuncs.com/ittailkshow/devops/download/apache-maven-3.8.6-bin.tar.gz
tar -xzvf apache-maven-3.8.6-bin.tar.gz

cat >> /etc/profile <<-'EOF'
export PATH=$PATH:/usr/local/apache-maven-3.8.6/bin
EOF

source /etc/profile
mvn -v

```



### 安装 Runner
```graphql
curl -L "https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/ubuntu/pool/noble/main/g/gitlab-runner/gitlab-runner_18.5.0-1_amd64.deb" -o /tmp/gitlab-runner.deb && sudo dpkg -i /tmp/gitlab-runner.deb && sudo apt -f install
```



### 注册 Runner
![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762262150843-dec20513-2829-464e-9397-bccf844671f6.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762262178583-a8a02a7e-bb26-40c7-b030-174d9a66fe1a.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762262207639-0001b3ca-7362-46ff-97dd-19b22de2c730.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762262240809-2ab14c6a-0402-42b7-bc65-aeacec5f7b13.png)









### 创建仓库


![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762263038450-48c91fb3-8318-4784-b948-b7b039275ef1.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762263067006-f492718d-e8af-4ed5-87ea-65ec930a5e2f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762263080720-01861c49-f0ae-4fba-ab3b-6b07addad2fb.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762263112876-a708fb09-6f6a-41c3-86a2-289653760c64.png)



git clone my-project

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762264960650-2bd0cd43-4ee8-44e1-ae77-d89d5a1dc45b.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265037437-e4754bfb-b1fa-4a0a-bd89-2bbde0026893.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265165180-ab096269-2470-448c-b3fb-07173c903bcb.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265150927-78695faa-7343-4dc3-85ec-4a411656c97e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265307426-79e37de7-0abe-410f-acf4-49dd1f16b3b1.png)



![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265410574-f0b9b0ad-7227-48ec-9cbd-c0dc2a19f041.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265427874-56db579b-699b-4830-b23b-db555c2ca2e1.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265441285-e381a800-7d0b-4ab7-905c-1442eb27f52e.png)

推送代码

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762265999459-666ebf4a-5ab0-45d4-910a-fc002afbdd0f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762266045954-18d6bd16-244c-47cc-90e0-d224f4a096e9.png)



根据触发条件，打一个 tag，触发流水线

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762266225074-6afd3507-ba0e-4a3d-a763-310dc9fffc81.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762266185589-c0e04fb7-fb93-4777-8cea-794454695836.png)

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762266249551-42c21387-69d1-45e1-ab4a-cb8d75c8d466.png)

查看 pipelines 流水线

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762266262688-ba90e716-2d54-4329-b16f-9c41259b5cd0.png)





![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762269706240-b364b96e-6f99-4004-a596-2b24f8c95142.png)

第一步 编译 jar，第二步打包成 image，第三步 push 到 dockerhub

![](https://cdn.nlark.com/yuque/0/2025/png/27875807/1762270025137-963b515c-37fb-49ee-9ed0-f8533b79241e.png)



