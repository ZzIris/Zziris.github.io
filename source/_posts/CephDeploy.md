---
title: Ceph集群搭建教程及一些学习搭建过程中遇到的相关操作与问题
date: 2019-07-23 21:19:14
tags:
- Ceph
- 分布式存储
categories:
- Ceph
---

# 一. 前言

&emsp;&emsp;由于近期开始接触学习Ceph分布式存储技术，且搭建了Ceph集群，本文将对个人搭建Ceph集群这一过程步骤进行梳理，并对这一过程中遇到的问题进行总结且给出解决方法。

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
$ sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

### 安装Ceph相关的包
&emsp;&emsp;首先我们需要在部署机上安装ceph-deploy部署工具，以在后续通过该工具对其余节点进行自动化部署。
```
$ sudo yum install ceph-deploy -y
```

### 安装openssh-server
&emsp;&emsp;在部署过程中我们需要通过部署机使用ssh连接到各个节点上进行操作，所以需要下载ssh服务，而在我在最小化安装了CentOs7.5后该服务都是已经装好的，无需再装，若发现没有装的话再进行安装的命令。
```
$ sudo yum install openssh-server -y
```

### 为部署机及每个集群节点机创建用户并进行sudo免密操作
&emsp;&emsp;在这里比较推荐在部署机以及集群节点机上新建用于集群工作的用户，官方文档申明在新建这些用户时不要使用"ceph"来作为用户名，也最好不要使用root用户来进行操作，总之，新建一个新的用户用于创建集群是最好的。而在我的搭建过程中部署机用户名设为deploy，其余三个集群节点用户名分别为node1、node2、node3。
```
$ sudo useradd -d /home/{username} -m {username}   #username为你需要创建的用户名
$ sudo passwd {username}   #设置用户密码
```
&emsp;&emsp;对每个节点进行sudo的免密设置，由于部署机在使用命令对所有节点进行部署的时候都是自动进行的，期间其会自动ssh连接到相应的节点机上进行sudo的操作来写入一些配置文件并且不会提示用户进行密码输入，所以需要提前设置好sudo的免密，以让部署机能够顺利对节点进行部署。（我自己将部署机也同时设置了sudo免密）。
```
$ sudo echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
$ sudo chmod 0440 /etc/sudoers.d/{username}
```

### 设置部署机与其余集群节点的免密ssh连接
&emsp;&emsp;通过部署机对其余集群节点进行部署需要让部署机与其余节点进行连接，因此需要用到ssh连接，我们需要在部署机上生成ssh密钥,输入命令后一直回车就好。
```
$ ssh-keygen
```
在执行命令后，将在我们的当前部署工作所在的用户家目录下的.ssh文件(其为隐藏文件)下生成一个id_rsa和id_rsa.pub两个文件，前者记录着私钥，后者记录着公钥，我们接下来需要将公钥拷贝到我们需要建立的集群节点机上。
&emsp;&emsp;此处我将我的deploy节点上的公钥分别拷贝到node1,node2和node3上,于是在deploy上执行：
```
$ ssh-copy-id node1@192.168.0.105
$ ssh-copy-id node2@192.168.0.106
$ ssh-copy-id node3@192.168.0.107
```
这样便将公钥分别拷贝到三个集群节点上了。
&emsp;&emsp;由于我们进行我们进行进行ssh连接时一般需要输入{username}@{hostip}的形式，为了简化ssh连接的形式让ceph-deploy工具顺利部署，我们需要修改一些配置文件，修改如下：
```
$ sudo vim /etc/hosts
```
在该文件中增加一些内容，将ip换成一个对应名字：
```
192.168.0.105 cpnode1
192.168.0.106 cpnode2
192.168.0.107 cpnode3
```
再修改.ssh文件夹下的config，若不存在该文件则创建它：
```
$ sudo vim ~/.ssh/config
```
增加内容为：
```
Host ceph1
   Hostname cpnode1
   User node1
Host ceph2
   Hostname cpnode2
   User node2
Host ceph3
   Hostname cpnode3
   User node3
```
修改完以上文件后，你就可以直接以如下形式进行ssh连接且需要再输入用户密码，方便了许多。
```
$ ssh ceph1
$ ssh ceph2
$ ssh ceph3
```

### 关闭firewall服务和selinux服务
&emsp;&emsp;由于firewall和selinux会在部署过程中造成一些影响，所以建议直接关闭。
&emsp;&emsp;关闭firewall：
```
$ sudo systemctl disable firewalld
$ sudo systemctl stop firewalld
```
&emsp;&emsp;关闭selinux：
```
$ sudo vim /etc/selinux/config
```
修改文件内容如下：
```
SELINUX=disabled
```
然后执行以下命令：
```
$ sudo setenforce 0
```
&emsp;&emsp;这样便可以关闭firewall和selinux了。

### 安装Ceph
&emsp;&emsp;在这里我们打算使用13.2.5版本的Ceph，由于在写这篇文章时mimic大版本的ceph最新已经是13.2.6，于是这里就涉及到了指定版本的安装。在这里我们介绍两种安装方式：1.直接通过外网镜像源安装。2.通过自己搭建本地源进行安装。（注：再官方给出的文档中是通过ceph-deploy install来进行自动安装ceph，但该种安装方式存在多种弊端，如指定版本或指定下载源比较麻烦等，因此比较推荐在每个集群节点上直接使用yum来安装。）

#### 方式一：通过外网镜像源安装
&emsp;&emsp;这种方式的优点是操作步骤较为简单，但缺点是受外网镜像源的影响，安装速度慢且有时候会发生寻找不到有效源。该方式只需要在集群机上进行，无需在部署机上进行。
&emsp;&emsp;使用该方式进行安装，我们首先需要在本地的yum源仓库文件中增添Ceph的镜像源仓库文件，操作如下：
```
$ touch /etc/yum.repo.d/ceph.repo
$ sudo vim /etc/yum.repo.d/ceph.repo
```
我们在ceph.repo中增添如下内容：
```
[Ceph]
name=Ceph packages for $basearch
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
```
可以看到，在该增添的内容中，我们使用的是阿里云的源，而没有使用官方源，这是因为国内的源下载速度会比国外的源快一些。
&emsp;&emsp;当我们添加完Ceph有关的repo文件后，就可以进行安装了，命令如下：
```
$ sudo yum install ceph-13.2.5 ceph-radosgw-13.2.5 -y
```
这里注意，想要指定相应版本，需要连同ceph-radosgw一起指定安装，因为ceph-radosgw中会有ceph的依赖项，单独指定ceph-13.2.5是无法完成ceph rpm包的下载安装的。

#### 方式二：通过自己搭建本地源进行安装
&emsp;&emsp;为了更快地进行安装，我们可以先将ceph相关的rpm包下载到部署机上后，建立本地源仓库来进行安装，这种安装方式在安装速度上是绝对占优的且不受外网镜像源的影响，但前提是需要现把rpm包下载下来。比较推荐学会搭建本地源，因为在实际工作环境中，很多机房中服务器是无法进行直接通过yum外网镜像源安装的，所以需要搭建本地源来解决。

首先我们先下载安装一个创建yum源仓库的工具createrepo：
```
$ sudo yum install createrepo -y
```

&emsp;&emsp;由于需要在部署机上将ceph相关的rpm包下载下来，我们需要在部署机上的/etc/yum.repo.d/路径下添加ceph相关的.repo文件，步骤在方式一中已给出不再赘述。当.repo文件添加完后就进行下载，这里注意，我只需要将rpm包下载下来即可，不需要安装，因此命令为：
```
$ sudo yum install --downloadonly --downloaddir=/mnt/myrepo/ceph/ ceph-13.2.5 ceph-radosgw-13.2.5
```
此处我将相关的包下载到/mnt/myrepo/ceph/路径下，这个路径根据自己喜好来进行即可。
&emsp;&emsp;下载完毕后，我们将myrepo文件夹建立成仓库：
```
$ sudo createrepo /mnt/myrepo/
```
命令执行结束后，便会在myrepo文件夹下生成repodata文件夹，此时代表仓库建立成功，此时通过createrepo后，仓库里的各个包之间的依赖关系就建立起来了，这样就可以通过yum命令来进行安装下载下来的包而不需要考虑rpm包之间的依赖关系了。
&emsp;&emsp;由于我们的ceph仓库是建立在部署机上的，为了能让其他集群节点能访问到且进行安装，我们需要在部署机上打开http服务，将仓库文件在局域网内共享。首先我们需下载http相关的包并进行安装：
```
$ sudo yum install httpd -y
```
&emsp;&emsp;然后打开httpd且设为开机启动，然后查看其是否已打开。
```
$ sudo systemctl enable httpd
$ sudo systemctl start httpd
$ systemctl status httpd   #状态为active即为打开
```
&emsp;&emsp;打开服务后，我们将我们创建好的仓库文件夹做一个软连接映射到httpd的默认共享路径下：
```
$ sudo ln -s /mnt/myrepo /var/www/html/
```
或者也可以不做软链接，直接到httpd的配置文件中去修改其默认的共享文件夹为我们的仓库目录：
```
$ sudo vim /etc/httpd/conf/httpd.conf
```
修改内容为：
```
DocumentRoot "/mnt/myrepo"

#
# Relax access to content within /var/www.
#
<Directory "/mnt/myrepo">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>

# Further relax access to the default document root:
<Directory "/mnt/myrepo">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    #Options Indexes FollowSymLinks
     Options ALL


    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   Options FileInfo AuthConfig Limit
    #
    AllowOverride None

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>

```
修改几处与路径相关的地方，同时将Option后面改为ALL。然后我们再到将httpd的一个欢迎页面配置文件给删除掉，这里为了防止文件丢失，我们进行改名让其无法生效即可：
```
$ sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.bak
```
&emsp;&emsp;最后我们重启一下httpd服务即可：
```
$ sudo systemctl restart network
```
&emsp;&emsp;接下来我们来验证在部署机上搭建的本地源是否已将文件共享出来，我们打开浏览器在地址栏输入以下内容（192.168.0.223为我建立本地仓库的部署机的IP，此处应相应换成你们所搭建源的主机IP，下同）：
```
http://192.168.0.223/myrepo   #若是做了软连接则输入该形式
http://192.168.0.223   #若是修改了httpd的配置文件的默认共享地址则输入该形式
```
如果能在浏览器中看到我们的仓库文件内容即表明仓库文件共享成功。
&emsp;&emsp;由于使用的是本地搭建的源而不是外网源，所以本方式集群节点机上的ceph相关的.repo文件应该这么修改：
```
[Ceph]
name=Ceph packages for $basearch
baseurl=http://192.168.0.223/myrepo
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=//192.168.0.223/myrepo
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=//192.168.0.223/myrepo
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
```
由于是本地源，我们可以选择把gpgcheck给关掉。
&emsp;&emsp;为了避免其他源对我们所建的本地源的影响，我们可以将其他的源移到别的路径下备份起来：
```
$ sudo mkdir /etc/yum.repo.d/backup
$ sudo mv /etc/yum.repo.d/*.repo /etc/yum.repo.d/backup   #此时ceph.repo也被移进了backup
$ sudo mv /etc/yum.repo.d/backup/ceph.repo /etc/yum.repo.d/   #将已创建的ceph.repo移回原位
```
&emsp;&emsp;然后直接进行yum安装指令即可，注意此时由于我们下载下来搭建本地源的ceph已经是13.2.5版本了，我们无需再进行版本指定。因为仓库里只有13.2.5版本的ceph。
```
$ sudo yum install ceph ceph-radosgw -y
```
注意：如果在httpd共享文件后，别的主机无法访问到你所搭建的仓库，则有可能是firewall服务或者selinux服务没关，导致搭建源的主机拒绝任何访问从而无法共享文件，此时应查看相应的两个服务是否关掉，如果这两个服务都关掉了还是无法访问则需查看搭建源的每一级目录是否有执行权限，如果没有执行权限也会导致无法访问路径下的内容。

&emsp;&emsp;通过以上步骤，我们就完成了搭建Ceph集群的准备工作，接下来便进入正式部署集群阶段。

## 2. 部署Ceph集群

&emsp;&emsp;此部分的部署工作都在部署机上进行，涉及到ceph-deploy这一命令的都是在部署机上执行的，在该实验中，我的部署机上没有装ceph，单纯只为用于部署工作，当然也可以将部署机作为一个集群节点。

### 新建部署所用目录
&emsp;&emsp;首先在进行部署集群之前，我们在部署机的部署用户下创建一个用于存放部署文件的目录（目录名称任取）：
```
$ mkdir ~/my-cluster
$ cd ~/my-cluster
```

### 创建monitor
&emsp;&emsp;首先创建集群，初始化将要指定为monitor的节点,此处我先将node1和node2指定为初始化的monitor节点：
```
$ ceph-deploy new node1 node2
```
在执行完命令之后，你之前在部署机上所创建的所用于部署的目录下将新增以下几个文件：
``` 
ceph.conf  
ceph-deploy-ceph.log  
ceph.mon.keyring
```
&emsp;&emsp;接下来，我们需要创建前面已初始化好的monitor节点：
```
$ ceph-deploy mon create-initial
```
则所用于部署的目录下又再新增了几个文件：
```
ceph.bootstrap-mds.keyring                 
ceph.client.admin.keyring
ceph.bootstrap-mgr.keyring                 
ceph.bootstrap-osd.keyring 
ceph.bootstrap-rgw.keyring
```

### 分发命令权限
&emsp;&emsp;然后我们将为我们想要其能执行ceph命令的节点分发admin.keyring:
```
$ ceph-deploy admin node1 node2
```
没有被分发admin.keyring的节点是无法使用ceph命令的，而这个admin.keyring的文件将出现再被分发节点的/etc/ceph目录下，再该目录下，还会看到与部署机一致的ceph.conf文件，在整个集群配置时，该文件都要保持一致，一旦部署机上的conf文件修改，都要推送覆盖到其余节点上，这样才能保证配置不会出错。
&emsp;&emsp;在该命令执行完后，我们可以在任意一台本分发了admin.keyring的节点机输入以下命令测试一下：
```
$ sudo ceph -s
```
该命令可以查看集群的整体情况。

### 创建mgr
&emsp;&emsp;接下来再创建mgr:
```
$ ceph-deploy mgr create node1 node2
```
mgr可以在两三个节点上创建，但也没必要太多。

### 创建osd
&emsp;&emsp;最后我们便可以创建osd了，在这里需注意，创建使用lvm分区的osd和创建使用块设备的osd命令略有不同，而我的机子上使用的是lvm分区，所以命令如下：
```
$ ceph-deploy osd create --data vg1/lv1 node1
$ ceph-deploy osd create --data vg2/lv2 node2
```
如果需要创建使用块设备的osd，则需要输入：
```
$ ceph-deploy osd create --data /dev/sdb node1
$ ceph-deploy osd create --data /dev/sdc node2
```
可以看出，这两种方式在命令上不同的地方为--data后的形式不一样，在使用lvm分区的情况，格式为 卷组名/逻辑卷名，卷组名和逻辑卷名是在创建lvm分区时就已确定。

### 查看集群部署情况
&emsp;&emsp;在执行完创建osd的命令后，Ceph集群的几个基本组件就都部署好了，我们可以在集群节点上通过以下命令来查看当前的集群情况：
```
$ sudo ceph health
```
如以下命令执行完后出现：
```
HEALTH_OK
```
则表明集群状况良好。除此之外，我们还可以通过另一指令来查看更详细的集群情况：
```
$ sudo ceph -s
```
命令执行后将得到以下信息：
```
cluster:
    id:     85313a2d-104f-4425-a033-7f17888f8021
    health: HEALTH_OK
 
  services:
    mon: 2 daemons, quorum node3,node2
    mgr: node2(active), standbys: node3
    osd: 2 osds: 2 up, 2 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   2.0 GiB used, 28 GiB / 30 GiB avail
    pgs:  
```
在这里面我们将可以看到monitor的守护进程已存在，mgr已激活，osd也成功创建且状态都处于up和in，集群整体情况很健康。

## 3. 新增monitor节点：
&emsp;&emsp;通过上述步骤我们知道，创建Ceph集群的第一步就是初始化需要指定为monitor的节点，而这里的新增monitor节点指的是没有经过第一步new过的节点，此时我们需要先修改ceph.conf文件，我们在部署机上进行修改，并把修改后的ceph.conf文件推送覆盖其他节点的ceph.conf文件，使它们的这个设置文件保持一致：
```
$ sudo vim ~/ceph.conf
```
修改的文件内容为：
```
[global]
fsid = 85313a2d-104f-4425-a033-7f17888f8021
mon_initial_members = node1, node2
mon_host = 192.168.0.105,192.168.0.106
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

public_network=192.168.0.0/24
```
此处public_network的设置要使得你所增加的monitor节点和已存在的节点的IP同处一个网段。
然后增加你想要增加的monitor节点，此处我们将node3新增为monitor节点：
```
$ ceph-deploy mon add node3
```
这样一个新的monitor节点就添加完成了。

## 4. 新增osd节点：
&emsp;&emsp;新增osd节点比较简单，直接创建即可：
```
$ ceph-deploy osd create --data  vg3/lv1 node3
```

## 5. 删除osd节点：
&emsp;&emsp;首先我们先明确需要剔除的osd节点是几号节点：
```
$ sudo ceph osd df tree
```
可以获得以下信息：
```
ID CLASS WEIGHT  REWEIGHT SIZE   USE     AVAIL   %USE  VAR  PGS TYPE NAME      
-1       0.03908        - 40 GiB 3.0 GiB  37 GiB  7.52 1.00   - root default   
-3       0.01949        - 20 GiB 1.0 GiB  19 GiB  5.02 0.67   -     host node1 
 0   hdd 0.01949  1.00000 20 GiB 1.0 GiB  19 GiB  5.02 0.67   0         osd.0  
-5       0.00980        - 10 GiB 1.0 GiB 9.0 GiB 10.03 1.33   -     host node2 
 1   hdd 0.00980  1.00000 10 GiB 1.0 GiB 9.0 GiB 10.03 1.33   0         osd.1  
-7       0.00980        - 10 GiB 1.0 GiB 9.0 GiB 10.03 1.33   -     host node3 
 2   hdd 0.00980  1.00000 10 GiB 1.0 GiB 9.0 GiB 10.03 1.33   0         osd.2  
                    TOTAL 40 GiB 3.0 GiB  37 GiB  7.52 
```
可以看到主机名和在其上创建的osd编号，我们此处将删除node3上的osd.2，删除操作我们在相应的节点机上执行。
&emsp;&emsp;我们先将需要删除的节点的权重置为0，然后将其踢出集群，接着停止该osd的服务，再然后从crush map中将其移除,再接着删除其的认证密钥，最后把其进行删除：
```
$ sudo ceph osd crush reweight osd.2 0
$ sudo ceph osd out 2
$ sudo systemctl stop ceph-osd@2
$ sudo ceph osd crush remove osd.2
$ sudo ceph auth del osd.2
$ sudo ceph osd rm 2
```

# 三. 在搭建Ceph集群中碰到的一些情况

&emsp;&emsp;由于自己是新手，在搭建Ceph集群的过程中也碰到了较多的麻烦与问题，因此，在这一部分内容中，将对自己在这一过程中碰到的一些问题与情况做一个总结与梳理，也让更多的人在碰到这样的问题时能够作为解决方法的参考。

## 1. 时钟漂移问题：
&emsp;&emsp;由于Cpeh集群是由多个主机来一起构建的，而每个主机都可能会存在时间差，这就会导致集群中不同节点之间存在时间漂移。有时候我们在查看Ceph集群状态时就会发现这个问题：
```
health: HEALTH_WARN
            clock skew detected on mon.node2
```
&emsp;&emsp;因此这时候我们就需要依靠ntp服务来解决这个问题了，ntp全称为Network Time Protocol，ntp提供准确的网络时间源，我们打开该服务，并设置好网络时间源，且将所有节点的ntp都指向同一时间源即可解决时钟漂移问题。
&emsp;&emsp;首先给所有集群节点安装ntp服务包并打开服务且设置为开机启动：
```
$ sudo yum install ntpd -y
$ sudo systemctl start ntpd
$ sudo systemctl enable ntpd
```
&emsp;&emsp;然后修改配置文件，我们在其中一个节点上设置网络时间源，我在我自己的node1节点上进行设置：
```
$ sudo vim /etc/ntp.conf
```
修改内容为：
```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

server time1.aliyun.com minpoll 3 maxpoll 4 iburst
server time2.aliyun.com minpoll 3 maxpoll 4 iburst
server time3.aliyun.com minpoll 3 maxpoll 4 iburst

```
将原有的server部分注释掉，并使用阿里云的时间源。而我们在其他节点上的相同的文件内修改的内容为：
```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

server node1 minpoll 3 maxpoll 4 iburst
```
同样也是将之前的server部分注释掉，但这里新增的语句是指向设置了网络时间源的节点，然后再事件源指向node1，最后重启一下ntp服务：
```
$ sudo systemctl restart ntpd
```
可能在所有ntp服务设置好之后，时钟漂移问题还会存在，这需要等待一小段时间，让时钟真正同步后才OK。

## 2. monotor节点根目录占用量太高：
&emsp;&emsp;有时候我们会看到这样一个Ceph集群状态健康警报：
```
health: HEALTH_WARN
            mon node1 is low on available space monitor
```
&emsp;&emsp;该问题并不是由于monitor所在节点的osd的可用容量不足而是由于monitor所在的节点的根目录使用率太高了，剩余空间较小，因此解决方法是对根目录进行扩容，这样即可解决问题。

## 3. osd usage为0：
&emsp;&emsp;有时候我们发现会出现在以下情况，集群状态里提示我们没有激活的mgr，此时的data字段中，所有数字都显示为0，这正是由于我们没有激活mgr所导致的。
```
  cluster:
    id:     85313a2d-104f-4425-a033-7f17888f8021
    health: HEALTH_WARN
                no active mgr
 
  services:
    mon: 2 daemons, quorum node3,node2
    mgr: no active mgr
    osd: 2 osds: 2 up, 2 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 GiB used, 0 GiB / 0 GiB avail
    pgs: 
```
针对此情况，我们激活mgr就好，我们在部署机上输入以下命令，此处我们选择在node1上激活mgr：
```
$ ceph-deploy mgr create node1
```
## 4. 将删除过的osd所在的lvm分区重新来创建osd：
&emsp;&emsp;如果我们想使用一个lvm分区来创建osd，在创建命令输入后报如下的错：
```
[node3][DEBUG ]  stderr: purged osd.1
[node3][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy.osd][ERROR ] Failed to execute command: /usr/sbin/ceph-volume --cluster ceph lvm create --bluestore --data vg4/lv1
[ceph_deploy][ERROR ] GenericError: Failed to create 1 OSDs
```
这是因为你所使用的这个lvm分区之前用来创建过osd，因此我们需要在重新创建osd之前做一个lvm分区的擦除操作：
```
sudo ceph-volume lvm zap volumename/logicalname
```

# 四. 末语

&emsp;&emsp;以上都是自己在实验中一步一步尝试然后总结下来的，在整个搭建的过程中也是经历坎坷，网上虽然也有很多对应的教程，但都比较杂乱，有的教程给的命令甚至无法运行。以上的个人总结也非完美，可能还是会存在纰漏之处，若有问题欢迎交流讨论。