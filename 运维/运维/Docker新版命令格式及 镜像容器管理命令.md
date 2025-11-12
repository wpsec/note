**新版docker命令采用结构化格式，虽然老版命令依旧可以使用，但是还是推荐结构化的使用命令，更加规范有效的使用。**

# 官方技术文档：[https://docs.docker.com/engine/reference/commandline/container_attach/](https://docs.docker.com/engine/reference/commandline/container_attach/)
## docker命令格式
docker现采用 选项 + 命令参数  
选项命令比较少而且用得也不多，多的是后面的参数命令  
常用的选项命令是：

```bash
$ docker -v			//查看版本
```

![](https://i-blog.csdnimg.cn/blog_migrate/6d39f0301e966d1ead058252ef6879d9.png#pic_center)

#### 管理命令
管理命令把每个命令管理的范畴规范为一个子命令

```bash
config			//管理docker配置
container		//管理容器
image			//管理镜像
network			//管理网络
node			//管理Swarm节点
plugin			//管理插件
secret			//管理docker安全
service			//管理服务
swarm			//管理Swarm集群
system			//管理docker系统
trust			//管理镜像信任
volume			//管理卷
```



![](https://i-blog.csdnimg.cn/blog_migrate/c51fad32d483f80d6eaef594e02ef7d3.png#pic_center)

## 命令
```bash
命令:                                                                                                                          
  attach      //将标准输入和标准输出连接到正在运行的容器                                        
  build       //使用dockerfile文件创建镜像                                                                                     
  commit      //从容器的修改项中创建新的镜像
  cp          //将容器的目录或文件复制到本地文件系统中
  create      //创建一个新的镜像
  diff        //检查容器文件系统的修改
  events      //实时输出docker服务器中发生的事件
  exec        //从外部运行容器内部的命令
  export      //将容器的文件系统到处为tat文件包
  history     //显示镜像的历史
  images      //输出镜像列表
  import      //从压缩为tar文件的文件系统中创建镜像
  info        //显示当前系统信息、docker容器与镜像个数、设置信息等
  inspect     //使用JSON格式显示容器与镜像的详细信息
  kill        //向容器发送kill信号关闭容器
  load        //从tar文件或标准输入中加载镜像
  login       //登录docker注册服务器
  logout      //退出docker注册服务器
  logs        //输出容器日志信息
  pause       //暂停容器中正在运行的所有进程
  port        //查看容器的端口是否处于开放状态
  ps          //输出容器列表
  pull        //从注册服务器中拉取一个镜像或仓库
  push        //将镜像推送到docker注册服务器
  rename      //重命名一个容器
  restart     //重启一个或多个容器
  rm          //删除一个或多个容器，若没有指定标签则删除lastest标签。
  rmi         //删除一个或多个镜像，若没有指定标签则删除lastest标签。                                                
  run         //在一个新容器中中运行命令，用于指定镜像创建容器。
  save        //将一个或多个镜像保存为tar包             
  search      //从Docker Hub中搜索镜像
  start       //启动一个或多个已经停止的容器
  stats       //查看容器使用量                                                    
  stop        //停止一个或多个正在运行的容器
  tag         //设置镜像标签
  top         //显示容器中正在运行的进程信息
  unpause     //重启pause命令暂停的容器
  update      //更新一个或多个容器的配置
  version     //显示docker版本信息
  wait        //等待容器终止然后输出退出码             
```

![](https://i-blog.csdnimg.cn/blog_migrate/f653bb2a51cd09e29e93077c55896b21.png#pic_center)



## 镜像管理命令
```bash
build		//构建镜像
history		//镜像历史
import		//将容器打包
inspect		//显示容器详细
load		//加载镜像
ls			//查看当前镜像
prune		//移除未使用镜像
pull		//拉去镜像
push		//上传镜像
rm			//删除镜像
save		//保存镜像
tag			//标记
```

![](https://i-blog.csdnimg.cn/blog_migrate/93a4d6960c922255ac2ae712a7424cf5.png#pic_center)

## 管理镜像常用命令
```bash
$ docker image ls			//列出所有镜像
$ docker image build		//构建镜像来自Dockerfile
$ docker image history		//查看镜像历史
$ docker image inspect		//显示一个或多个镜像详细信息
$ docker image pull			//从镜像仓库拉去镜像
$ docker image push			//推送一个镜像到镜像仓库
$ docker image rm			//移除一个或多个镜像
$ docker image prune		//移除未使用，未标记的镜像
$ docker image tag			//创建一个引用源镜像标记的目标镜像
$ docker image save 		//保存一个或多个镜像到tar归档文件
$ docker image load 		//加载镜像
```



## 创建容器常用命令
```bash
-i（–interactive ）		//交互式
-t（-ttl）				//分配一个伪终端
-d（-detach）			//后台运行容器
-e（-env）				//设置环境变量
-p（-publish list）		//发布容器端口到主机
-P（-publish-all）		//发布容器所有EXPOSE的端口到宿主机随机端口
--name					//指定容器名
-h（-hostname）			//指定容器的主机名
--ip string				//指定容器IP，只能用于自定义网络
--network				//连接容器到一个网络
--mount mount			//将文件系统符加到容器
-v（-volume list）		//绑定挂载一个卷
--restart string		//容器退出时重启策略，默认no
```

## 容器资源限制命令
```bash
-m（memory）				//容器可以使用的最大内存量
--memory-swap				//允许交换到磁盘的内存量；-1表示不限制使用，如果跟内存一样大小表示不设置swap，docker默认交换区与内存相同，如果swap比内存大，及 （swap-m）= swap可以使用的swap大小
--memory-swappines=<0-100>	//容器使用SWAP分区交换的百分比
-oom-kill-disable			//禁用OOM Killer，如果服务过多，会自动kill掉服务，开启此服务不会存自动kill
--cpus						//可以使用的CPU数量
-cpuset-cpus				//限制容器使用特定的CPU核心，如（0-3，0，1）
-cpu-shares					//CPU共享（相对权重）
```

## 实例
```bash
内存限额：
允许容器最多使用500M内存和100M的Swap，并禁用 OOM Killer：
$ docker container run -d --name nginx01 --memory="500m" --memory-swap="600m" --oom-kill-disable nginx


CPU限额：
允许容器最多使用一个半的CPU：
$ docker container run -d --name nginx04 --cpus="1.5" nginx
允许容器最多使用50%的CPU：
$ docker container run -d --name nginx05 --cpus=".5" nginx


创建一个nginx并转发到宿主机8080端口
$ docker container run -itd --name web_01 -e te
st=123456 -p 8080:80 nginx
```

**按照新docker命令规范化可以极大提升效率**

![](https://i-blog.csdnimg.cn/blog_migrate/6a74982f0511363aa261582f58cfe647.png#pic_center)

