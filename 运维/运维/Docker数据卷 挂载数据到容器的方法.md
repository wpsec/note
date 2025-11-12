@[TOC](目录)

 **PS：关于卷的官方文档：**[https://docs.docker.com/storage/](https://docs.docker.com/storage/)

## 1. 三种挂载数据的方法
### - volumes
docker管理宿主机文件系统的一部分（/var/lib/docker/volumes）。  
保存数据的最佳方式。

### - bind mounts
将宿主机上的任意位置文件或者目录挂载到容器中。

### - tmpfs
挂载存储在主机的内存中，不持久存储。

常用为	volumes，bind mounts，tmpfs并不常用

![](https://i-blog.csdnimg.cn/blog_migrate/ab19a8cb7836a5f060fd243ad01575c6.png#pic_center)

## 2. 管理卷
### Volume：
volume属于管理命令  
![](https://i-blog.csdnimg.cn/blog_migrate/73ba3e249f56d4125c66ed3ffe20418f.png#pic_center)  
命令下的参数比较少，也比较好理解

```bash
create		//创建
inspect		//查看卷详情
ls			//列出卷
rm			//删除卷
prune		//删除所有未使用的卷
```

![](https://i-blog.csdnimg.cn/blog_migrate/b9043c0d6785980841ad3e993a662e85.png#pic_center)

#### 创建一个卷
```bash
$ docker volume create web_01_vol
```

创建的卷在这个目录下：  
**/var/lib/docker/volumes**  
![](https://i-blog.csdnimg.cn/blog_migrate/cc5eefb1cfb8bf60a2ebe5a2845e45dd.png#pic_center)

#### 如何挂载
如何挂载这个卷到容器中

```bash
$ docker container run -itd --name=web_01 -p 8080:80 --mount src=web_01_vol,dst=/usr/share/nginx/html nginx
```

![](https://i-blog.csdnimg.cn/blog_migrate/e9c3eb1e8a2d9cdf9622585c9fa2e31c.png#pic_center)

容器创建成功后，该目录也会挂载到   **/var/lib/docker/volumes/web_01_vol/_data**   
![](https://i-blog.csdnimg.cn/blog_migrate/101db9a457c9ae66f2048156611a61a9.png#pic_center)  
修改index.html文件 验证

![](https://i-blog.csdnimg.cn/blog_migrate/84ca2b6242f25ee12afe57a3b34bb663.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/313525b0ae1a800ddd3cea70bbc2af79.png#pic_center)  
如果没有指定挂载卷，会自动挂载一个卷  
![](https://i-blog.csdnimg.cn/blog_migrate/b776811657b5ae0c35f0cdb0b2e126c5.png#pic_center)  
重新创建一个卷积，共用一个卷，卷积的文件通用，数据卷不会因为容器的删除而丢失，它会长期保留。

```bash
$ docker container run -d --name=test_01 --mount src=web_01_vol,dst=/usr/share/nginx/html nginx
```



![](https://i-blog.csdnimg.cn/blog_migrate/6cd91e30ce34a18ae88accf6ef1cdbcd.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/181dfecfc54ed7f5f606e41b7cbd21ab.png#pic_center)

#### 删除卷
```bash
$ docker container stop 7740ca246029 f11f0c5fbcf5		//关机
$ docker container rm f11f0c5fbcf5 7740ca246029			//删除容器
$ docker volume rm web_01_vol							//删除卷
```

## Bind Mounts
它是将宿主机的某个文件挂载到容器中  
其命令格式都一样

实例：

#### 创建一个Bind 卷
假如宿主机 /home/目录下有个1.txt文件需要挂载到容器中  
目录跟文件均可挂载，挂载到容器的文件和目录跟宿主机其实差不多是一个软链接状态，容器中改变文件或宿主机改变文件都会受影响。

```bash
$ docker container run -itd --name=test_01 --mount type=bind,src=/home/test/123.txt,dst=/home/1.txt centos
```

![](https://i-blog.csdnimg.cn/blog_migrate/4c9696fcb0eb3aa3b050e626390cd918.png)

验证  
如果没有指定卷，会报错。

```bash
$ docker container inspect c4a1deeb58d2
```

![](https://i-blog.csdnimg.cn/blog_migrate/f30595e589a4caeaa4869aa5b8f4b233.png)

#### 删除Bind卷
跟Volume一样

```bash
$ docker container stop 7740ca246029 f11f0c5fbcf5		//关机
$ docker container rm f11f0c5fbcf5 7740ca246029			//删除容器
$ docker volume rm web_01_vol							//删除卷
```

## Volume 与 Bind 特点
**Volume**

+ 多个运行容器之间共享数据，多个容器可以同时挂载相同的卷。
+ 当容器停止或被移除时，该卷依然存在。
+ 当明确删除卷时，卷才会被删除。
+ 将容器的数据存储在远程主机或其他存储上（间接）
+ 将数据从一台Docker主机迁移到另一台时，先停止容器，然后备份卷的目（/var/lib/docker/volumes/）

**Bind**

+ 从主机共享配置文件到容器。默认情况下，挂载主机/etc/resolv.conf到每个容器，提供DNS解析。
+ 在Docker主机上的开发环境和容器之间共享源代码。例如，可以将Maven target目录挂载到容器中，每次在Docker主机
+ 上构建Maven项目时，容器都可以访问构建的项目包。
+ 当Docker主机的文件或目录结构保证与容器所需的绑定挂载一致时

