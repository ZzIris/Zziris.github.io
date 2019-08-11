---
title: SMB+CephFs实现文件存储
date: 2019-08-11 11:27:59
tags:
- Ceph
- 分布式存储
- SMB协议
categories:
- Ceph
---

# 一. 前言
&emsp;&emsp;我们除了使用S3协议通过对象存储的方式来在ceph集群中存储文件外，我们还可以使用smb协议并通过ceph文件系统的存储方式来对文件进行存储。

# 二. SMB服务与CephFs

## 1.创建CephFs

&emsp;&emsp;在使用SMB协议进行文件存储之前，我们需要启用ceph的文件系统，首先需要创建元文件服务器mds，此处选在ceph集群的一个mon节点上创建:
```
$ ceph-deploy --overwrite-conf mds create  node2
```
&emsp;&emsp;接着创建相应的pool：
```
$ ceph osd pool create cephfs_data 16 16
$ ceph osd pool create cephfs_metadata 16 16
$ ceph fs new cephfs cephfs_metadata cephfs_data
```
此处先为文件的metadata创建一个pool叫cephfs_metadata，再为文件的data创建一个pool叫cephfs_data，最后将这两个pool共同整合成文件系统叫cephfs，创建完后可以通过以下命令进行查看：
```
$ ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
$ ceph mds stat
cephfs-1/1/1 up  {0=cephfs-master1=up:active}
```
通过返回数据可以看到相应的信息，只要看到mds stat中显示up和active即表示文件系统准备就绪。

## 2.CephFs物理机挂载

&emsp;&emsp;在我们将cephfs准备就绪后，要想使用它就需要将其挂载到某一目录下，就像我们平时挂载格式化好的物理设备存储设备是一样的，挂载这一步可以选择在ceph集群的任何节点上进行也可以选在这些相应节点之外的另一台机器上进行，如果是后者，则也需要在后者上装上对应版本的ceph：
```
$ ceph-deploy admin <Linux-client-IP>   //在部署机上执行，如果选择在另一台不属于ceph集群节点的机器上进行挂载则需要此命令，若是在ceph集群的节点上进行挂载则忽略此命令
$ ceph auth get client.admin
exported keyring for client.admin
[client.admin]
	key = AQCy70RdRRatCBAAp11vjAzQDBioKFvknCx9oQ==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"

```
可以看到执行完第二条命令后，就会给出相应的key，将其复制出来。
&emsp;&emsp;接下来我们创建一个目录并将cephfs挂载上去：
```
$ sudo mkdir /mycephfs
$ sudo chmod 777 /mycephfs
$ sudo mount -t ceph <monitor-ip>:6789:/ /mnt/cephfs/ -o name=admin,secret=AQCy70RdRRatCBAAp11vjAzQDBioKFvknCx9oQ==
```
可以看到我们在根目录下创建目录mycephfs并将其权限更改为777,这是为了后面将文件共享出去后其他使用共享的用户可以对目录进行访问读取写入。然后我们通过指定集群的任意一个mon节点的ip并使用6789端口将其挂载，同时还要输入用户名以及前面我们复制了的key，用户名默认为admin。至此CephFs就创建并挂载完成。

## 3.搭建samba

&emsp;&emsp;搭建好并挂载好后的cephfs要想共享出去就可以使用samba来实现。首先需要安装samba：
```
$ sudo yum install samba
```
&emsp;&emsp;然后创建一个samba用户：
```
$ smbpasswd -a sambauser -c /etc/samba/smb.conf
```
此处的sambauser用户名必须为在你Linux上存在的用户。
&emsp;&emsp;然后修改smb.conf文件：
```
$ sudo vim /etc/samba/smb.conf
```
增加以下内容：
```
[cephcc]
        comment = ceph test
        browseable = yes
        writeable = yes
        path = /mycephfs

```
其中writeable一定要为yes,否则无法更改共享路径下的内容，path为需要共享的路径。
&emsp;&emsp;修改完文件后启动服务即可：
```
$ sudo systemctl start smb
```
这样cephfs所挂载在的目录就共享出去了，我们在其他机器上往共享目录里写入或读取文件就都是通过smb协议来进行。

## 4.在其他Windows或者Linux系统机器上访问共享的cephfs文件系统目录

### Windows

&emsp;&emsp;在Windows上，我们只需要通过打开文件资源管理器（即我们平时点开的我的电脑），在地址栏中输入你打开了smb服务并共享了cephfs挂载目录的机器的ip，如我打开了smb服务并共享了cephfs挂载目录的机器ip为192.168.0.106就可输入：
```
\\192.168.0.106
```
这样便可以访问之前共享出来的目录了。

### Linux
&emsp;&emsp;在Linux上，要想访问共享出来的cephfs，需要先安装smbclient
```
$ sudo yum install samba-client
```
&emsp;&emsp;后燃输入如下命令，进入到共享的目录下：
```
$ smbclient //smbseverIP/dir -U smbuser
Enter SAMBA\smbuser's password:
```
其中smbseverIP为打开了smb服务并共享了cephfs挂载目录的机器的ip，dir为共享的目录的名称，smbuser为前面所创建的samba用户名，最后输入之前设置的用户密码即可，这样就能在Linux下访问共享的cephfs文件系统挂载目录了。

# 末语
&emsp;&emsp;本文章总结归纳了ceph文件系统存储方式的使用，明确给出了cephfs的搭建以及如何使用smb服务将cephfs进行共享的操作步骤，并在其他Windows和Linux上对共享的cephfs挂载目录进行访问。