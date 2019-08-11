---
title: 使用iSCSI协议实现Ceph块存储
date: 2019-08-11 16:09:08
tags:
- Ceph
- 分布式存储
- iSCSI
categories:
- Ceph
---

# 一. 前言

&emsp;&emsp;在前面的几篇文章里分别介绍了使用Amazon S3协议来使用ceph的对象存储以及使用smb协议+ceph的文件系统来进行文件存储，而对于ceph而言，其还支持块存储，因此本篇文章将对块存储方式的使用进行归纳总结，给出使用ceph快存储的操作步骤。

# 二. 使用iSCSI协议在ceph集群上进行块存储

## 1.前期准备

&emsp;&emsp;为了能够及时检测down掉的osd节点尽可能减少iSCSI initiator的超时，需要对每个osd节点做一下设置：
```
$ ceph tell osd.* config set osd_heartbeat_grace 20
$ ceph tell osd.* config set osd_heartbeat_interval 5
```
此处osd.*代表对所有osd节点都进行相同设置，若想单独对某个节点设置须将*换成具体编号（如0号osd则需改成osd.0）。

## 2.配置iSCSI Target

### 安装所需的包
&emsp;&emsp;首先需要下载安装相应的包，主要所需包如下：
```
tcmu-runner-1.3.0 or      newer package
ceph-iscsi-config-2.4 or      newer package
ceph-iscsi-cli-2.5 or      newer package
```
这几个包若直接使用yum源安装是找不到的，所以需要自行下载rpm包并置于自建本地源来进行安装，自建本地源的流程已在搭建ceph集群教程中给出，此处不在赘述，而所需的软件包点击[这里](https://blog.51cto.com/devingeng/2125656)即可从该博客博主所提供的百度网盘中下载得到。一切准备好后只要安装以上三个主要包即可将其他所需的依赖包都安装上。由于后续需要至少两个iSCSI网关，因此需要在至少两台机器上装这些包，可以是ceph集群上的节点，也可以不是，若不是ceph集群中的节点，则该机器上也许要装上ceph，并由部署机给其分配相应ceph集群的admin权力，使其可用ceph命令对集群进行操作。

### 创建rbd所需的pool
&emsp;&emsp;由于后续步骤中在gwcli的设置中需要一个pool所以此处需要先创建一个pool：
```
$ sudo ceph osd pool create rbd 16 16 3
```
此命令中rbd为新建pool的名字，两个16依次为pg数量和pgp数量，3为副本数量，其中pg数必须指定，而pgp数量和副本数可不显示指定，若不指定则使用默认值。创建完pool后可以使用命令来进行查看：
```
$ sudo ceph osd lspools
1 .rgw.root
2 default.rgw.control
3 default.rgw.meta
4 default.rgw.log
5 default.rgw.buckets.index
6 default.rgw.buckets.data
7 rbd
8 cephfs_data
9 cephfs_metadata
```
可以看到我们刚创建rbd pool就在其中。

### 配置iSCSI网关
&emsp;&emsp;然后我们需要设置iSCSI网关的配置文件：
```
$ sudo touch /etc/ceph/iscsi-gateway.cfg
$ sudo vim /etc/ceph/iscsi.cfg
```
写入内容如下：
```
[config]
# Name of the Ceph storage cluster. A suitable Ceph configuration file allowing
# access to the Ceph storage cluster from the gateway node is required, if not
# colocated on an OSD node.
cluster_name = ceph

# Place a copy of the ceph cluster's admin keyring in the gateway's /etc/ceph
# drectory and reference the filename here
gateway_keyring = ceph.client.admin.keyring


# API settings.
# The API supports a number of options that allow you to tailor it to your
# local environment. If you want to run the API under https, you will need to
# create cert/key files that are compatible for each iSCSI gateway node, that is
# not locked to a specific node. SSL cert and key files *must* be called
# 'iscsi-gateway.crt' and 'iscsi-gateway.key' and placed in the '/etc/ceph/' directory
# on *each* gateway node. With the SSL files in place, you can use 'api_secure = true'
# to switch to https mode.

# To support the API, the bear minimum settings are:
api_secure = false

# Additional API configuration options are as follows, defaults shown.
# api_user = admin
# api_password = admin
# api_port = 5001
trusted_ip_list = 192.168.0.106,192.168.0.107

```
此处需要在trusted_ip_list出给出你所设置iSCSI网关的机器的ip（设置了几个写几个），此处我们只设置两个网关。
&emsp;&emsp;然后打开相应的rbd-target-api服务：
```
$ systemctl daemon-reload
$ systemctl enable rbd-target-api
$ systemctl start rbd-target-api
```

### 使用gwcli来进行配置
&emsp;&emsp;此处我们需要使用gwcli工具来配置iSCSI网关和RBD images，并在网关节点间同步配置。底层的工具，如 targetcli和rbd，可以被用来查询当前的配置，但不应该被用来改变配置。所有的配置改动都必须通过gwcli来做。
```
$ sudo gwcli
```
&emsp;&emsp;创建一个iscsi-target：
```
/> cd /iscsi-target
/iscsi-target>  create iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw
```
&emsp;&emsp;创建一个iscsi-gateway，前面我们已经在两个机器上装了iscsi相关的包，准备将其作为iscsi网关：
```
/iscsi-target> cd iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/gateways
/iscsi-target...-igw/gateways>  create cpnode2 192.168.0.106
/iscsi-target...-igw/gateways>  create cpnode3 192.168.0.107
```
此处要注意两点,一是注意，如果你操作系统不是 RHEL或CentOS，又或者你使用的是测试版内核，那么在上述命令末尾还要加上 skipchecks=true；二是cpnode2其实对应的就是192.168.0.106这个ip，而cpnode3对应的就是192.168.0.107这个ip，这个对应关系需要在/etc/hosts中就给出：
```
$ sudo cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.0.106 cpnode2
192.168.0.107 cpnode3

```
如果你的hosts文件中没有相应给出，就需要提前去修改增加，将ip对应成你想要的名字。
&emsp;&emsp;在前面创建的rbd pool中创建rbd image名字叫disk_1（任意取）:
```
/iscsi-target...-igw/gateways> cd /disks
/disks> create pool=rbd image=disk_1 size=20G
```
&emsp;&emsp;创建一个initiator client名字：
```
/disks> cd /iscsi-target/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/hosts
/iscsi-target...eph-igw/hosts>  create iqn.1994-05.com.redhat:rh7-client
```
&emsp;&emsp;设置客户端chap用户名和密码：
```
/iscsi-target...at:rh7-client>  auth ching226210/ching226210@   //有的版本命令形式为 auth username=ching226210 password=ching226210@
```
此处要注意，chap的用户名和密码有格式要求，具体可以使用help auth来查看要求。
&emsp;&emsp;最后增加新建的rbd image到客户端：
```
/iscsi-target...at:rh7-client> disk add rbd.disk_1   //有的版本命令形式为disk add rbd/disk_1
```
&emsp;&emsp;至此iSCSI网关和RBD images的配置就结束了。

## 3.配置initiator

&emsp;&emsp;配置initiator可以在随意的一台机器下进行并不一定要在ceph集群节点或者是iscsi网关节点上进行，此处我们在Linux下进行配置（也可以在Windows下进行）。

### 安装initiator工具包
&emsp;&emsp;首先安装相应的包：
```
$ yum install iscsi-initiator-utils
$ yum install device-mapper-multipath
```

###进行配置
&emsp;&emsp;先启动multipath服务，然后修改multipath.conf文件：
```
$ sudo mpathconf --enable --with_multipathd y
$ sudo vim /etc/multipath.conf
```
在multipath.conf文件中增加以下内容：
```
devices {
        device {
                vendor                 "LIO-ORG"
                hardware_handler       "1 alua"
                path_grouping_policy   "failover"
                path_selector          "queue-length 0"
                failback               60
                path_checker           tur
                prio                   alua
                prio_args              exclusive_pref_bit
                fast_io_fail_tmo       25
                no_path_retry          queue
        }
}
```
然后重启multipath服务：
```
$ sudo systemctl reload multipathd
```
&emsp;&emsp;修改iscsid.conf文件
```
$ sudo vim /etc/iscsi/iscsid.conf
```
打开相应部分的注释，并修改其chap用户名和密码为之前所设置的chap用户名和密码：
```
# *************
# CHAP Settings
# *************

# To enable CHAP authentication set node.session.auth.authmethod
# to CHAP. The default is None.
node.session.auth.authmethod = CHAP   //打开此处注释

# To set a CHAP username and password for initiator
# authentication by the target(s), uncomment the following lines:
node.session.auth.username = ching226210   //打开此处注释并修改为你所设置的chap用户名
node.session.auth.password = ching226210@   //打开此处注释并修改为你所设置的chap密码
```
&emsp;&emsp;修改initiatorname.iscsi文件
```
$ sudo vim /etc/iscsi/initiatorname.iscsi
```
修改内容如下，此处将InitiatorName修改为之前所创建的initiator client名字：
```
InitiatorName=iqn.1994-05.com.redhat:rh7-client
```
&emsp;&emsp;重启iscsi服务：
```
$ sudo systemctl restart iscsid
$ sudo systemctl restart iscsi
```
这里要注意要先重启上层服务iscsid，再去重启服务iscsi。

### iSCSI发现与登录
&emsp;&emsp;首先initiator需要先发现target:
```
$ iscsiadm -m discovery -t st -p 192.168.0.106
192.168.0.106:3260,1 iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw
192.168.0.107:3260,2 iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw
```
该命令的结果显示为之前所创建的两个iscsi网关，该命令中的最后的ip只要是iscsi网关的ip地址皆可。
&emsp;&emsp;发现成功后就进行登录：
```
$ sudo iscsiadm -m node -T iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw -l
```
&emsp;&emsp;最后可以使用以下命令来进行验证是否成功：
```
$ sudo iscsiadm -m session
tcp: [19] 192.168.0.106:3260,1 iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw (non-flash)
tcp: [20] 192.168.0.107:3260,2 iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw (non-flash)
$ sudo multipath -ll
mpatha (36001405c1c044ce0113403ab904abef5) dm-3 LIO-ORG ,TCMU device     
size=20G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='queue-length 0' prio=50 status=enabled
| `- 21:0:0:0 sdd 8:48 failed ready running
`-+- policy='queue-length 0' prio=10 status=active
  `- 22:0:0:0 sdc 8:32 failed ready running

```
出现以上信息则表明成功。
