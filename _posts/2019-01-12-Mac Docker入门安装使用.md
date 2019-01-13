---
layout: post
title: Mac Docker入门安装使用
categories: docker
description: Mac Docker入门安装使用
keywords: Docker
---

最新mac系统千万不要用brew安装，推荐使用官方文档：https://docs.docker.com/docker-for-mac/#proxies

##### 安装镜像：

```shell
docker pull centos:latest
```

latest代表拉取最新的镜像，当然可以先搜索下 

```shell
docker search centos
```

##### 查看本地镜像库：

```shell
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              1e1148e4cc2c        9 days ago          202MB
```

> 在列出信息中，可以看到几个字段信息:
>
> 来自于哪个仓库，比如 ubuntu
> 镜像的标记，比如 16.04
> 它的 ID 号(唯一)，比如e4415b714b62
> 创建时间
> 镜像大小

##### 启动镜像：

```shell
tongkun@localhost java (master) $ docker run -it centos bash 
[root@fc68ad1849ef /]# 
```

-it 表示运行在交互模式，是-i -t的缩写，即-it是两个参数：-i和-t。前者表示打开并保持stdout，后者表示分配一个终端（pseudo-tty）一般这个模式就是可以启动bash，然后和容器有命令行的交互

启动镜像后，分配了一个新终端，命令行变为`[root@fc68ad1849ef /]# `说明启动成功，并且登陆到了根目录

在这里可以随意使用Linux命令了，但是有些命令是没有的，需要手动安装，比如vim，可以使用yum安装，命令：

```shell
yum install vim
```

##### 退出容器

如果使用exit，命令退出，则容器的状态处于Exit，而不是后台运行。如果想让容器一直运行，而不是停止，可以使用快捷键 ctrl p+q 退出，此时容器的状态为Up

使用exit，然后使用

查看正在运行的容器：docker ps

```shell
tongkun@localhost java (master) $ docker run -it centos bash 
[root@c06a8694d372 /]# 
[root@c06a8694d372 /]# exit
exit
tongkun@localhost java (master) $ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bd7181bd39ff        centos              "bash"              3 minutes ago       Up 3 minutes                            priceless_goldwasser
```

可以看到，当前有一个id为bd7181bd39ff的容器，image为centos，就是刚刚启动的，如果通过exit退出容器，这里就不会显示了。

##### 启动、停止、重启容器

```shell
tongkun@localhost java (master) $ docker stop bd7181bd39ff
bd7181bd39ff
tongkun@localhost java (master) $ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
tongkun@localhost java (master) $ docker start bd7181bd39ff
bd7181bd39ff
tongkun@localhost java (master) $ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bd7181bd39ff        centos              "bash"              6 minutes ago       Up 1 second                             priceless_goldwasser
tongkun@localhost java (master) $ docker restart bd7181bd39ff
bd7181bd39ff
tongkun@localhost java (master) $ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bd7181bd39ff        centos              "bash"              6 minutes ago       Up 4 seconds                            priceless_goldwasser
```

##### 进入容器attach

```shell
tongkun@localhost java (master) $ docker attach bd7181bd39ff
[root@bd7181bd39ff /]# 
```

##### 安装软件、保存环境

安装vim

```
[root@bd7181bd39ff /]# yum install vim 
.....
[root@bd7181bd39ff /]# vi  
vi        view      vigr      vim       vimdiff   vimtutor  vipw 
```

保存容器，先退出容器，然后commit

```shell
tongkun@localhost java (master) $ docker commit -m '安装vim' -a 'tongkun' bd7181bd39ff tongkun/centos:vim
sha256:1dab79502fbda22037e865b81882e073575af4e8a0bd8a0de16989b0ed244e2d
```

-m指定说明信息；-a指定用户信息；bd7181bd39ff代表容器的id；tongkun/centos:vim指定目标镜像的用户名、仓库名和 tag 信息

查看镜像库，就已经有刚提交的镜像了

```shell
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
tongkun/centos      vim                 1dab79502fbd        About a minute ago   327MB
centos              latest              1e1148e4cc2c        9 days ago           202MB
```

退出现有镜像，启动刚刚commit的镜像，查看安装的vim是否存在

```shell
tongkun@localhost java (master) $ docker stop bd7181bd39ff
bd7181bd39ff
tongkun@localhost java (master) $ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
tongkun@localhost java (master) $ docker run -it tongkun/centos bash 
Unable to find image 'tongkun/centos:latest' locally
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: Service Unavailable.
See 'docker run --help'.
tongkun@localhost java (master) $ docker run -it tongkun/centos:vim  bash 
[root@a7880e04c1d4 /]# vi 
vi        view      vigr      vim       vimdiff   vimtutor  vipw  
```

可以看到，这是我们刚刚commit的镜像，有vim命令工具

##### 删除容器或镜像

如果想删除容器或者镜像，可以使用rm命令，注意：删除镜像前必须先删除以此镜像为基础的容器（哪怕是已经停止的容器），否则无法删除该镜像，会报错**Failed to remove image (e4415b714b62): Error response from daemon: conflict: unable to delete e4415b714b62 (cannot be forced) - image has dependent child images**类似这种。

```shell
tongkun@localhost java (master) $ docker ps
CONTAINER ID        IMAGE                COMMAND             CREATED             STATUS              PORTS               NAMES
a7880e04c1d4        tongkun/centos:vim   "bash"              5 minutes ago       Up 5 minutes                            blissful_volhard
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
tongkun/centos      vim                 1dab79502fbd        10 minutes ago      327MB
centos              latest              1e1148e4cc2c        9 days ago          202MB
tongkun@localhost java (master) $ docker rmi 1dab79502fbd
Error response from daemon: conflict: unable to delete 1dab79502fbd (cannot be forced) - image is being used by running container a7880e04c1d4
```

删除镜像 `docker rmi 容器id`, 因为此镜像有容器在使用，所以不能被删除，需要先删除容器，删除容器命令`docker rm 镜像id`，删除之前需要先stop容器，否则也会报错，如下：**Error response from daemon: You cannot remove a running container a7880e04c1d42f6d1f672ac920dd33df552a409cc19029314672643ee18e5836. Stop the container before **

```shell
tongkun@localhost java (master) $ docker rm a7880e04c1d4
Error response from daemon: You cannot remove a running container a7880e04c1d42f6d1f672ac920dd33df552a409cc19029314672643ee18e5836. Stop the container before attempting removal or force remove
tongkun@localhost java (master) $ docker stop a7880e04c1d4
a7880e04c1d4
tongkun@localhost java (master) $ docker rm a7880e04c1d4
a7880e04c1d4
tongkun@localhost java (master) $ docker rmi 1dab79502fbd
Untagged: tongkun/centos:vim
Deleted: sha256:1dab79502fbda22037e865b81882e073575af4e8a0bd8a0de16989b0ed244e2d
Deleted: sha256:f6def596fa2f515b28700f4cd3241e0ea78743abe6dad6d2f65bbf945f6dbf15
```

#### Docker push

正所谓“一次提交，到处使用”，我们可以把配置好的Docker push到仓库中，比如 docker hub

先把刚删除的镜像重新弄回来一遍。。。。。

首先到https://hub.docker.com/注册个账号，然后登陆

```shell
tongkun@localhost java (master) $ docker login 
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: tongkun
Password: 
Login Succeeded
```

push镜像：

```shell
tongkun@localhost java (master) $ docker push tongkun/centos:vim
The push refers to repository [docker.io/tongkun/centos]
15f896816a9b: Pushed 
071d8bd76517: Mounted from library/centos 
vim: digest: sha256:41bdaf55a709080577ccb40f61c0f91275e4a1cb62827a3893b6a5269a619d67 size: 741
```

push成功之后，到docker hub的仓库中，我们就可以看到自己push上去的镜像了，跟github类似，如图：

![屏幕快照 2018-12-16 上午12.35.14](http://upload-images.jianshu.io/upload_images/1272254-e4ef3ca53736d6f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### pull镜像

当我们在其他地方也需要使用此镜像是，只需要配置好docker，并登陆docker就可以pull已有的镜像了，为了模拟我们先把本地镜像和容器删掉，从仓库中拉取

```shell
tongkun@localhost java (master) $ docker stop 133b1f45876f
133b1f45876f
tongkun@localhost java (master) $ docker rm 133b1f45876f
133b1f45876f
tongkun@localhost java (master) $ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
tongkun/centos      vim                 78c5c9cce361        8 minutes ago       327MB
centos              latest              1e1148e4cc2c        9 days ago          202MB
#删除镜像
tongkun@localhost java (master) $ docker rmi 78c5c9cce361
Untagged: tongkun/centos:vim
Untagged: tongkun/centos@sha256:41bdaf55a709080577ccb40f61c0f91275e4a1cb62827a3893b6a5269a619d67
Deleted: sha256:78c5c9cce361122999251ef6ed00d286e4a1af70124a53583e46dbaeb3517879
Deleted: sha256:4b40340dc18b3ef39430f7892e0021af55dcae7f5d76e3b73e0087392f6ca353
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              1e1148e4cc2c        9 days ago          202MB
#从仓库中拉取镜像
tongkun@localhost java (master) $ docker pull tongkun/centos:vim 
vim: Pulling from tongkun/centos
a02a4930cb5d: Already exists 
260974091ff8: Pull complete 
Digest: sha256:d57b9eb7123569c3b49279e8211d145b1070656be8aede5f97a0d025fc6ec6ee
Status: Downloaded newer image for tongkun/centos:vim
tongkun@localhost java (master) $ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
tongkun/centos      vim                 defbd9f314c9        11 minutes ago      327MB
centos              latest              1e1148e4cc2c        9 days ago          202MB
tongkun@localhost java (master) $ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
#启动镜像
tongkun@localhost java (master) $ docker run -it tongkun/centos:vim bash 
[root@c42f9f275474 /]# tongkun@localhost java (master) $ 
tongkun@localhost java (master) $ docker ps 
CONTAINER ID        IMAGE                COMMAND             CREATED             STATUS              PORTS               NAMES
c42f9f275474        tongkun/centos:vim   "bash"              11 seconds ago      Up 10 seconds                           frosty_williams
```

##### 最后来张docker命令图收尾

![20171005132826220](http://upload-images.jianshu.io/upload_images/1272254-8b641689bce503ad.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Mac入门就到这里了，深层次的使用和控制，后面学习再补充~

参考：http://lihuia.com/2018/03/09/docker/