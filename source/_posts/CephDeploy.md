---
title: Ceph集群搭建教程及一些学习搭建过程中遇到的相关操作与问题
date: 2019-07-23 21:19:14
tags:
- Ceph
- 分布式集群
categories:
- Ceph
---

# 一. 前言

&emsp;&emsp;作为一个工作方向为媒体存储的人，也作为一个初次接触存储这一领域的菜鸟，在入职这几天也在照着组长培训安排下开始不断地接触相关性的东西。由于所在的媒体存储组内其所采用的底层存储技术是基于Ceph的，所以这两周来也在尝试着去接触Ceph，从搭建Ceph底层rados集群开始，慢慢地对Ceph这个东西有了一定的认识，由于也是刚开始接触其相关性的原理还不太熟知，而这两周来也是一直在虚拟机上进行集群的搭建，也是在今天下总算搭建出了一个HEALTH_OK的集群，这期间也摸索了一下如何在集群中进行一些对osd、monitor部件的增删操作，大体都有了一定的了解，先将这段时间的所积累的关于Ceph集群搭建的过程记录下来，同时也把这期间碰到的坑也一并记录，给自己也给同为刚接触Ceph这一技术的新手们一个教程参考，接下来，让我们进入主题吧！

# 二. Ceph集群搭建操作步骤

&emsp;&emsp;首先我们的Ceph集群搭建是在虚拟机上进行的，使用系统为CentOs 7.5，总共开了四台虚拟机，一台作为部署机，其余三台作为集群的节点，其IP地址如下所示：
```
deploy 192.168.0.223
node1 192.168.0.105
node2 192.168.0.106
node3 192.168.0.107
```

## 1. 前期准备工作

### 安装epel-release
&emsp;&emsp;由于RHEL以及其衍生发行版如CentOs等Linux为了稳定，其官方的rpm repository提供的rpm包往往很滞后，当然这是理所应当的，毕竟是服务器版本，服务器的安全稳定才是首要的。而epel的出现就是为了解决官方rpm仓库内容不够丰富且版本滞后的问题，epel全称为Extra Packages for Enterprise Linux。是由Fedora社区打造的，为RHEL及其衍生发行版本提供高质量软件包的项目。装了epel就相当于添加了一个第三方源。
&emsp;&emsp;我们可以通过以下命令来对epel-release来进行安装：
```
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

### 安装Ceph相关的包
&emsp;&emsp;首先我们需要在部署机上安装ceph-deploy部署工具，以在后续通过该工具对其余节点进行自动化部署。
```
sudo yum install ceph-deploy -y
```

### 安装openssh-server
&emsp;&emsp;在部署过程中我们需要通过部署机使用ssh连接到各个节点上进行操作，所以需要下载ssh服务，而在我在最小化安装了CentOs7.5后该服务都是已经装好的，无需再装，若发现没有装的话再进行安装的命令。
```
sudo yum install openssh-server -y
```

### 为部署机及每个集群节点机创建用户并进行ssh免密操作
&emsp;&emsp;在这里比较推荐在部署机以及集群节点机上新建用于集群工作的用户，官方文档申明在新建这些用户时不要使用"ceph"来作为用户名，也最好不要使用root用户来进行操作，总之，新建一个新的用户用于创建集群是最好的。而在我的搭建过程中部署机用户名设为deploy，其余三个集群节点用户名分别为node1、node2、node3。
```
ssh user@ceph-server   #此处user为你需要连接服务器上的一个早先就创建好的用户名，ceph-server为需连接的服务器ip
sudo useradd -d /home/{username} -m {username}   #username为你需要创建的用户名
sudo passwd {username}   #设置用户密码
```
&emsp;&emsp;对每个节点进行sudo的免密设置，由于部署机在使用命令对所有节点进行部署的时候都是自动进行的，期间其会自动ssh连接到相应的节点机上进行sudo的操作来写入一些配置文件并且不会提示用户进行密码输入，所以需要提前设置好sudo的免密，以让部署机能够顺利对节点进行部署。（我自己将部署机也同时设置了sudo免密）。
```
sudo echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
```