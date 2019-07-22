---
title: 使用GitHubPage+Hexo搭建个人博客教程
date: 2019-07-20 15:34:52
tags:
- GitHubPage
- Hexo
- 博客
categories:
- 教程
---

# 一. 前言

&emsp;&emsp;2019年的夏天，我毕业了，离开了成电，离开了成都，在广州开始了新的生活。读了20年的书如今不再是一个学生，入了职变成了一个社会人。回顾本科加研究生这七年的成电生活，学过单片机，摸过Java，熟读C++，搞过深度学习...不得不说这么些年还是学了那么点皮毛，但真让我说我会什么，我可能便是哑口无言。在校期间没有什么实习经历。没有真正意义上的项目经历，有过点参赛经历却又做得太浅...现在想想这些，只能觉得自己太失败，现在总结这七年的大学生活，也都怪自己没有养成认真钻研一门技术的习惯，总是浪费大把的时间在于一些毫无意义的事情之上，并且自己也没有能够认真总结一些自己所学到的知识，更很少将其运用至实际中去，所以导致如今的自己感到如此地挫败。而如今的自己也不再是一个学生了，步入职场我也意识到，这一切都需要个改变，毕竟在这个社会中讲究的是不进则退弱肉强食的森林法则，很多东西都太现实像车子房子等等，若自己没有能力那又如何能得到又如何能让自己更好地立足于社会之中。思安则危，识危则稳这个道理才是往后生存的真正道理。
&emsp;&emsp;因此，为了让自己能够更好地钻研技术积累知识增长自己的能力，我也就打算自己弄一个自己的博客，记录下自己在往后生活与工作中的一些知识与经验，让自己能够更好地成长为一个有技术的人。同时也可以将自己生活中的一些有趣的经历记录下来，让自己也有个回忆的承载之地。而我的博客文章也就以这篇搭建个人博客的教程为一个开端正式拉开帷幕。
&emsp;&emsp;废话了那么多，接下来就进入正题吧！

# 二. 使用Github Page + Hexo搭建个人博客

## 1. GitHub Page
&emsp;&emsp;一般来说搞技术的人大部分都会知道github，其是一个面向开源及私有软件项目的托管平台，因为其支持的是Linux之父Linus所创造的git分布式版本控制，故名github。而在github上有一个叫做github page的项目，该项目目的就是为了方便开发者快速建立个人页面来记录分享自己的想法于代码。而本教程就是基于该github page项目来进行。所以首先我们需要拥有属于自己的github个人页面。

### 注册github账号
&emsp;&emsp;首先需要到[github](https://github.com/)站点注册属于自己的github账号。

### 创建github仓库
&emsp;&emsp;首先我们需要在github的个人页面上找到Repositories选项，然后点击new
![](/images/GitHub-Hexo/1.jpg)
&emsp;&emsp;在进入创建仓库的界面后，需要在Repository name中填入xxx.github.io，其中xxx对应的是你的github名，一定要记住一定得是自己github账号的github名，写入其他的词汇会导致你的页面无法访问。
![](/images/GitHub-Hexo/2.jpg)
&emsp;&emsp;这样属于你自己的个人github博客主页就创好了，此时点击进入你刚创建的xxx.github.io仓库点击Settings并下拉找到GitHub Pages这一块内容，其中上面给出的http://xxx.github.io/ 就是属于你的个人页面。
![](/images/GitHub-Hexo/3.jpg)
![](/images/GitHub-Hexo/4.jpg)

## 2. Hexo
&emsp;&emsp;在安装Hexo之前需要安装Node.js和Git。

### 安装Node.js
&emsp;&emsp;下载地址：[Node.js](https://nodejs.org/en/download/)，安装正常情况下一直下一步就行。
### 安装Git
&emsp;&emsp;下载地址：[Git](https://git-scm.com/download/)，安装正常情况下也是一直下一步就好。
### 安装Hexo
&emsp;&emsp; 接下来就是最重要的一步，我们对着需要安装Hexo的文件夹点击鼠标右键，选择Git Bash Here即可进入Git的命令行界面，然后输入以下命令：
```
$ npm install -g hexo   #安装hexo
$ hexo -v   #可查看hexo是否安装成功，安装成功将显示版本号
```
&emsp;&emsp;在安装成功后便可以开始初始化博客网站了，输入如下命令：
```
$ hexo init   #初始化网站     
$ npm install   #用于安装hexo所需的一些依赖
$ hexo g   #或 hexo generate  用于生成相应的文件
$ hexo s   #或者 hexo server, 用于启动本地服务器，这一步之后就可以通过http://localhost:4000查看
```
&emsp;&emsp;若通过http://localhost:4000 可以在浏览器中打开博客界面即证明博客在本地计算机部署成功，而默认的hexo部署得到的博客使用的是landscape主题。
&emsp;&emsp;当然我们创建博客并不只是想要其只能在本地计算机上浏览，我们需要在互联网上能够对其进行访问，因此我们需要进行下一步操作将其与github page结合，从而成功部署到互联网上。

## 3. Gihub page + Hexo结合
&emsp;&emsp;由于需要对远程Github的xxx.Githb.io仓库进行操作，此时需要将本地计算机与远程仓库建立连接，我们将通过ssh连接的方式来连接远程仓库。

### 本地生成SSH密钥
&emsp;&emsp;在Git Bash里键入如下指令以生成SSH密钥：
```
$ cd ~   #Windows下~/相当于C:\Users\xxx   xxx为你的Windows登陆用户目录
$ mkdir .ssh   #在Windows当前用户目录下一般不存在.ssh文件，一般需要手动创建，若已有则忽略
$ ssh-keygen -t rsa -C "xxx@youremail.com"   #生成相关的密钥文件，邮箱地址写你的github关联的邮箱地址
```
&emsp;&emsp;这时候你的.ssh文件夹下就生成了一些文件，其中id_rsa保存的是你生成的私钥，而id_rsa.pub保存的是你生成的公钥，而我们接下需要打开id_rsa.pub将其中的内容复制出来。

### 将生成的SSH公钥添加到Github中
&emsp;&emsp;进入你的github主页，点击github右上角的头像，选择Settings。
![](/images/GitHub-Hexo/5.jpg)
&emsp;&emsp;然后点击SSH and GPG keys,再点击New SSH key。
![](/images/GitHub-Hexo/6.jpg)
&emsp;&emsp;于是便进入了添加SSH key的界面，其中Title可以随意，而Key部分把我们上一步中复制出来的公钥粘贴进去，然后点击Add SSH key即可，这样我们的公钥就添加成功了。
![](/images/GitHub-Hexo/7.jpg)
我们可以通过一下命令来测试连接是否成功：
```
$ ssh -T git@github.com   #接下来会让你选择yes/no，输入yes就OK
```
当显示一下信息则表示连接成功：
```
Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```
然后设置一下你的github账号信息：
```
$ git config --global user.name "name"   #此处不是github用户名
$ git config --global user.email "xxx@youremail.com"   #与github关联的邮箱
```

### 使用Hexo deployer部署github个人页面
&emsp;&emsp;在hexo安装的目录下找到_config.yml文件，打开找到以下代码段进行编辑：
```
deploy:
  type: git   #指定版本控制类型
  repo: git@github.com:xxx/xxx.github.io.git   #此处xxx填写你的github名
  branch: master   #指定每次部署hexo时文件的push分支
```
此处特别要注意代码的缩进（特别是冒号后面一定要有个空格）。
&emsp;&emsp;保存上诉修改后需要安装一个插件：
```
$ npm install hexo-deployer-git --save
```
&emsp;&emsp;插件安装好后输入以下下命令即可在互联网上部署我们的个人页面了：
```
$ hexo d
```
等命令执行结束后，浏览器输入xxx.github.io即可看到你的个人页面了（xxx为你的github名）。

# 三. 关于Hexo主题设置以及一些博客操作

## 1.Hexo主题设置
&emsp;&emsp;在完成了GitHub Page + Hexo的个人博客部署后，我们可以根据自己喜好来进行主题的设置，不同的主题操作会有不同的地方，具体需根据所选用的主题使用文档来进行设置，此处以我所用的next主题来进行介绍，让大家有一定的大致了解，更多主题选择请移步[官方主题库](https://hexo.io/themes/)。
&emsp;&emsp;首先我们先到Hexo安装目录下执行以下命令（依旧在Git Bash命令行界面）：
```
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
命令执行完后我们可以在themes文件夹下发现多了一个next文件夹，这便是主题包。
&emsp;&emsp;然后我们需要对配置文件进行修改，由于把next主题包下载下来后，该包中有一个与Hexo安装目录下的配置文件同名的文件_config.yml，其用于对主题进行配置，为了区分这两个配置文件，我们称Hexo安装目录下的配置文件为站点配置文件，next主题包下的配置文件为主题配置文件。
&emsp;&emsp;我们打开站点配置文件来进行主题设置，找到theme字段，将其修改成我们所下载的主题包名。
```
theme: next
```
&emsp;&emsp;保存修改后，我们对主题更换进行验证：
```
$ hexo clean   #当我们对工程文件做了修改后，最好都需要此命令清理以下Hexo的缓存
$ hexo s   #本地部署  命令执行后可通过http://localhost:4000来查看
```
能看到相应的主题界面说明Hexo主题配置成功。

## 2.设置博客菜单选项及页面

### 修改页面显示的菜单选项
&emsp;&emsp;编辑主题配置文件，修改以下内容：
```
menu:
  home: / || home   #主页
  about: /about/ || user   #关于
  tags: /tags/ || tags   #标签
  categories: /categories/ || th   #分类
  archives: /archives/ || archive   #归档
  #schedule: /schedule/ || calendar   #计划
  #sitemap: /sitemap.xml || sitemap   #站图
  #commonweal: /404/ || heartbeat   #公益404
```
把你想在博客主页显示的菜单选项去掉注释（||后面跟的是菜单对应选项的图标）。

### 创建需要的菜单选项页面
&emsp;&emsp;当把相应菜单选项去掉注释后，有的选项需要创建页面，否则点击选项后会显示找不到页面。设置页面命令如下：
```
$ hexo new page xxx
```
xxx为你要创建的相应页面（上面代码中的几种页面），执行命令后会在Hexo安装目录下的source文件夹中生成相应名字的文件夹，里面会有一个index.md文件，打开文件进行设置（此处以about菜单选项为例），在两行分界线之间的部分是菜单选项页面信息，此处重要是设置type，生成的源文件没有此项，需自己增加并写上相应类型，而第二行分界线下面的部分有的页面可依据需要增加页面的显示内容：
```
---
title: 关于我
date: 2019-07-19 23:31:10
type: "about"
---
<center>此处可以增加需显示的的内容</center>   #<centor>...</center>用于居中内容
```
对于分类和标签页面等基本使用同上的操作，只需要将type分别改为categories与tags等就OK。

### 更详细的主题设置
&emsp;&emsp;更详细的一些细节修改如字体、头像及签名等更详细的设置请参考[next主题官方博客](http://theme-next.iissnan.com/advanced-settings.html)

## 3.书写博客文章
&emsp;&emsp;在博客搭好后最重要的当然是写在自己的博客主页写文章了，这是一件很酷的事情，下面就教大家一些书写博客文章的基本操作。

### markdown
&emsp;&emsp;首先我们在搭建好博客后可以在个人页面上看到一篇Hello World的文章（真是标准的程序员学习新技术的打招呼方式~），这一篇就是Hexo初始化的一篇博客文章范文，而在本地的Hexo安装路径下我们可以在source目录下的_post子目录中看到一个hello-world的.md文件，这就是我们所需要书写文章的源文件，其中.md代表markdown文件，博客文章的书写需要使用markdown语言语言来编写，markdown语法可以百度一下，都有很多介绍，大部分最简单的基础语法就已经足够我们普通人使用，想要进行复杂的表达可以进一步地进行学习，这里给出一个我参考的markdown基本语法的[学习链接](https://www.jianshu.com/p/7771794c88a1)。

### 新建文章.md源文件
&emsp;&emsp;接下来我们便开始创建属于我们自己的博客文章了，首先输入如下命令：
```
$ hexo new article_name  #这里的article_name即你.md文件的文件名。
```
这里的new后面也可以跟上post选项，相对于前面创建菜单页面使用的是page，post表示你创建的是文章.md文件，其会出现在_post文件夹下。这便是我们要新写的文章源文件了。

### 编辑.md源文件并部署发布
&emsp;&emsp;我们打开新建的这个.md文件，有如下内容：
```
---
title: test
date: 2019-07-21 10:47:33
tags:
---
```
这是一个文章信息，可以指定你显示在个人博客页面的文章题目、创建时间、标签以及分类等，我们做具体如下的更改：
```
---
title: 使用GitHubPage+Hexo搭建个人博客教程
date: 2019-07-21 10:47:33
tags:
- GitHubPage
- Hexo
- 博客
categories:
- 其他
---
```
我们可以照着以上的形式进行修改，增加了categories选项，并且在tags和categories底下给出他们的标签、分类的具体指定，这样就可以在个人页面的标签、分类等菜单页面里看到给出的具体标签、分类内容了。接着我们就可以在这段抬头信息后面（第二行分割线下面）按照markdown语法编写博客文章内容了！在写完之后可以先部署到本地计算机上进行查看效果并做相应的修改
```
$ hexo s   #然后浏览器输入http://localhost:4000查看文章效果
```
在最终确定文章可以发布后按照部署的流程走一遍就可以发布到互联网页面上了，这里再次给出部署的几个命令：
```
$ hexo clean
$ hexo g
$ hexo d
```
基本上每次做出更改就可以使用这三步来进行重新部署。

### 关于图片插入
&emsp;&emsp;这个问题也是折腾了我一下午，按照网上的很多教程来做都失败了，后来在咨询了我的同事后，得到了一个有效的解决方案。
&emsp;&emsp;首先我们在Hexo的source目录下新建你存放图片的目录，比如像我是在source目录下新建了images文件夹，在该images文件夹下在新建相应文章的文件夹放置相应文章所需的图片，目录结构如图：
![](/images/GitHub-Hexo/10.jpg)
&emsp;&emsp;然后在文章源文件中使用markdown图片引用方式
```
![](/images/GitHub-Hexo/xxx.jpg)   #直接以source路径为当前路径然后写相对路径即可
```

# 四. 在不同的电脑上对博客进行维护及更新

&emsp;&emsp;在部署完自己的博客后，突然想到我的博客是在家里电脑上设置完成的，如果到了公司电脑上要怎么做才能按照已设置好的这些工作来继续维护和更新我的博客呢？于是我在网上找了解决方法，大部分的主要思路就是利用git分支来解决，于是做了如下的总结。

## 1.创建你自己的xxx.github.io仓库的分支

### 创建分支并将配置的Hexo上传到分支
&emsp;&emsp;首先进入到你的xxx.github.io仓库页面点击左边的Branch，然后输入搜索选择你想要的分支，若不存在你输入名字的分支，它会出现一个Create banch: xxx的选项，点击它就能创建一个新分支了。
![](/images/GitHub-Hexo/8.jpg)
&emsp;&emsp;然后我们需要设置推送及克隆的默认分支为我们创建的分支，点击右边的Settings再选择左边的Branches选项，然后选择我们创建的分支，最后再点击Update就可以把推送的默认分支设置为我们创建的分支了。这一步的目的是需要把我们做了修改的Hexo工程文件传到仓库中进行保存而不影响到我们用来部署发布的master分支。
![](/images/GitHub-Hexo/9.jpg)
&emsp;&emsp;然后选择一个想要的路径将分支克隆到本地计算机：
```
$ git clone https://github.com/xxx/xxx.github.io 
```
然后我们进入这个路径输入以下命令就可以查看到我们该本地仓库所关联的默认分支为我们所新建的分支：
```
$ git branch
```
然后将我们在家里配置好的Hexo安装目录下的所有内容复制到我们克隆操作所得到的仓库目录下，此时需要注意的是在我们完成复制操作后，我们到我们克隆所得到的仓库中的theme文件夹下的主题文件夹中去把它里面的.git隐藏文件夹给删除，否则在我们的克隆仓库中出现第二个.git文件夹会造成仓库冲突从而导致主题文件夹提交失败。
&emsp;&emsp;然后提交分支并把分支推送到远程仓库分支上：
```
$ git add .
$ git commit -m "back up hexo files"
$ git push 
```
然后到github主页仓库中去查看你创建的分支，发现分支仓库中相对于master分支多了很多文件即表明推送成功。

## 2.在其他电脑上进行博客维护和更新
&emsp;&emsp;完成上述操作后，你换了台电脑也可以对博客进行同步的维护和更新了，具体操作如下所述。

### 新电脑上进行博客维护和更新的前序工作
&emsp;&emsp;在新电脑上需要进行SSH连接远程仓库，具体步骤在本文章前面已提及此处不再赘述。在做完SSH准备工作后，我们需将上一步上传到新建分支的内容克隆到你的新电脑上，克隆指令同上。然后到克隆下来的目录下在Git Bash命令行界面中键入如下命令：
```
$ npm install
```
这是由于我们前面在推送文件到远程分支仓库时，文件夹里有一个.gitignore文件，该文件记录了推送时所忽略的文件及文件夹，其中就把一个叫node_modules的文件夹给忽略掉了，这其中包含了hexo需要的模块，因此需要我们在新电脑上克隆分支仓库文件以后再install以下以得到一些推送时忽略掉的内容。（此处可以说一说，之前的操作可能有的朋友会担心推送到远程分支仓库的文件夹里会有重复的文件，其实不会，大家可以观察一下.gitignore文件再对比一下本地的文件就可以得到一些信息了）。
&emsp;&emsp;经过以上的操作就可以在其他新电脑上同步维护和更新我们的博客了，但要记住无论我们在哪一台机子上都要在我们克隆下来的文件夹中进行操作（包括最初建立博客的机子，在做了远程分支后就不要在原Hexo安装路径下进行修改了，否则无法同步分支仓库的修改）。在我们对克隆仓库做了改动后要记住使用上述的提交及推送命令git add、 git commit、git push这三个操作。

# 五. 末语
&emsp;&emsp;本博客文章是我的第一篇博客文章，很多地方可能表达得不够好也不够仔细，更可能存在一些细节的遗漏，如果大家在搭建过程中按照步骤来却遇到了什么问题也可以与我联系交流，联系方式已在关于页面给出。
等后续开通了留言模块后也可以通过留言的方式来交流。
&emsp;&emsp;最后也希望自己能够通过多写微博来积累及分享知识，让自己能够更好更快地成长。