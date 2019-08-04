---
title: 使用Amazon的S3接口来与Ceph集群进行交互
date: 2019-08-03 23:08:18
tags:
- Ceph
- 分布式存储
- Amazon S3
categories:
- Ceph
---

# 一. 前言

&emsp;&emsp;当我们搭完Ceph集群后，接下来就是想着该如何去搭好的集群进行交互，我们尝试着往集群中存储一些东西，以起到其作为分布式存储的最基本功能。由于Ceph支持Amazon的S3接口来与集群进行交互，本文将简单介绍使用S3的接口来在Ceph集群上进行一些基本操作。本文中所涉及到的操作都是在centos7.5系统下进行的操作。

# 二. C++ S3接口使用

## 1. 准备工作

### 搭建好的Ceph集群
&emsp;&emsp;这是最重要的部分，Ceph集群搭建不成功也无法进行后面的操作，Ceph集群的搭建请参考我的博客文章《Ceph集群搭建教程及一些学习搭建过程中遇到的相关操作与问题》，此处不再赘述。

### gcc、g++等编译器
&emsp;&emsp;可以先查看是否装了这两个编译器：
```
$ which gcc
$ which g++
```
如果没安装则需要进行安装：
```
$ sudo yum install gcc gcc-c++
```

### 安装S3lib
&emsp;&emsp;要使用S3的接口，则需要将其相应的库下载并安装：
```
$ sudo yum install libs3-devel
```

### S3Browser下载安装
&emsp;&emsp;由于我的使用S3接口的C++代码是在虚拟机上运行的，因此需要在Windows上通过直观地查看代码效果是否达到，这里推荐使用Windows的客户端S3Browser来进行查看，此处给出[S3Browser的下载地址](https://s3browser.com/)。

### 在Ceph集群上创建对象存储网关
&emsp;&emsp;由于S3接口是针对对象存储的，因此需要在搭建好的集群上创建对象存储网关，在部署机上执行一下操作：

```
$ ceph-deploy rgw create node1
```
此处我们选择在node1节点上创建网关。

### 创建对象网关用户
&emsp;&emsp;连接到我们创建了对象网关的节点上，进行对象网关用户的创建：
```
sudo radosgw-admin user create --display-name=ching --uid=ching
```
此处的ching是我使用的用户名，可以根据自己的喜好给定自己想要的用户名。命令执行完后会出现以下内容：
```
{
    "user_id": "ching",
    "display_name": "ching",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "ching",
            "access_key": "EALMXG238PNS29UG3LGH",
            "secret_key": "Gz3bg8bOqY5ighJcpo6KzGmhkp4s41S7ZPVAwuF5"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```
其中我们需要的是access_key和secret_key。

## 2.实例代码

&emsp;&emsp;准备工作做好后我们就可以进行S3接口使用的测试代码了

