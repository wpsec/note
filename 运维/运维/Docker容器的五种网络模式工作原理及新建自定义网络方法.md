@[TOC](目录)



PS：关于网络的官方文档：[https://docs.docker.com/network/](https://docs.docker.com/network/)



## 一. 网络模式
docker提供五种网络模式

+ bridge
+ host
+ none
+ container
+ 自定义网络

### 1. bridge
–net=bridge  
**默认网络，Docker启动后创建一个docker0网桥，默认创建的容器也是添加到这个网桥中。**

![](https://i-blog.csdnimg.cn/blog_migrate/0abeb36e320b36aaf1c1290ef51dabca.png#pic_center)

### 2. host
–net=host  
**容器不会获得一个独立的network namespace，而是与宿主机共用一个。这就意味着容器不会有自己的网卡信息，而是使用宿主机的。容器除了网络，其他都是隔离的。**

```bash
$ docker image pull busybox											//拉取一个busybox测试
$ docker container run -it --name=test_bus --network=host busybox	//新建容器
```

![](https://i-blog.csdnimg.cn/blog_migrate/c915eacb802b9b15daf01695badb6f82.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/0f51ad7300ccd447fd73e66040be3ed1.png#pic_center)



### 3. none
–net=none  
**获取独立的network namespace，但不为容器进行任何网络配置，需要我们手动配置。**

```bash
$ docker container run -it --network=none busybox
```

![](https://i-blog.csdnimg.cn/blog_migrate/54eb6ad8e9256d7c75d922a1b4735688.png#pic_center)

### 4. container
–net=container:Name/ID  
**与指定的容器使用同一个network namespace，具有同样的网络配置信息，两个容器除了网络，其他都还是隔离的。**

```bash
$ docker container run -itd -p 8080:80 busybox			//创建一个box映射80端口
$ docker container run -itd  --name nginx_01 --network=container:3f623edc41e5 nginx			//创建一个nginx并把网络指定到box的空间中
$ docker container inspect 3f623edc41e5		//查看box的网络
```

最后能够访问box中的nginx  
![](https://i-blog.csdnimg.cn/blog_migrate/edfbd9cd2110641524fb924709343509.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/33a11fb6e640d1f0ea64006b6e9119f9.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/7bee03e845f8c17087b9c471a0e2b652.png#pic_center)

### 自定义网络
**与默认的bridge原理一样，但自定义网络具备内部DNS发现，可以通过容器名容器之间网络通信。**

```bash
$ docker network create net_test		//创建一个自定义网络
$ docker network ls						//ls命令可以查看当前网络
$ docker network inspect 090e7585eee3	//查看详情等  跟 vol数据卷操作几乎相同
```

创建好一个网络后在创建两个容器 test

```bash
$ docker container run -itd --network=net_test busybox		
$ docker container run -itd --network=net_test busybox		
```

![](https://i-blog.csdnimg.cn/blog_migrate/e328d92912473a1bd0bef26f126c465d.png#pic_center)



## 二. 容器网络访问原理
一台宿主机，网卡eth0 192段，宿主机内的容器默认172段，其它主机访问容器是通过宿主机转发完成，  
下图的veth类似一个管道，提供宿主机与容器的通信（宿主机一个容器一个），创建容器时就会自动创建veth虚拟设备

![](https://i-blog.csdnimg.cn/blog_migrate/081cfec08fbcf650c7e9ea8849a4ed7f.png#pic_center)



![](https://i-blog.csdnimg.cn/blog_migrate/180e113a09891101a281b2a0c44dbcf5.png#pic_center)  
上图的网关最终到了宿主机的一个模拟网卡上，然后通过ens33出去  
![](https://i-blog.csdnimg.cn/blog_migrate/d28d28db04f6d696de0dfb9f93d4998e.png#pic_center)  
![](https://i-blog.csdnimg.cn/blog_migrate/9316c1beff1d2550d93729b199973acf.png#pic_center)

![](https://i-blog.csdnimg.cn/blog_migrate/6a16b68a6952d3ebba25e77cfd4ba292.png#pic_center)  
所以veth有极大的作用  
![](https://i-blog.csdnimg.cn/blog_migrate/ed6998767e12defe86068689dca7e385.png#pic_center)

