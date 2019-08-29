---
title: Android开发中使用ndk-build进行NDK构建
date: 2019-08-26 16:50:36
tags:
- Android
- NDK
categories:
- Android学习
mathjax: true
---

# 前言

&emsp;&emsp;NDK一般与JNI共同出现，此处先简单地介绍JNI与NDK。JNI全称Java Native Interface，JNI是Java的编程框架，可以先通俗地把其理解为Java开发中的一种技巧，使用这种技巧可以使得Java与C/C++交互，即可达到在Java里调用C/C++的目的。NDK全称为Native Development Kit，可以先通俗地将其理解为Android中实现JNI的工具。

# 构建NDK项目

&emsp;&emsp;简单地说，一般做Android开发，我们通常都使用的是SDK，而SDK的开发都是Java语言，但是在实际情况中，有些我们所需实现的功能是使用C/C++语言来开发的（如ffmpeg），若使用SDK，则需要使用Java语言将其再实现一遍，这样会比较耗费时间，所以有时候为了节省开发时间，通过构建NDK及使用JNI就可以实现对C/C++所编写生成的.so文件的调用。要达到这一目的，首先要做的第一步就是构建NDK，我们首先来构建一个NDK有关的简单项目。

## 1.下载安装NDK工具

&emsp;&emsp;首先第一步需要构建NDK环境，需要先下载并安装NDK工具，安装方式如图所示：

![][1]

![][2]

由于此处，我已安装好了NDK工具，所以显示状态为installed，若未安装则显示未Not installed，此时将NDK选项勾上点确定就会进入下载安装界面，进行安装即可。

## 2.定义调用C/C++实现功能函数的类

&emsp;&emsp;首先New一个普通的JavaClass文件,并定义一个新的类，类的内容如图所示：

![][3]

其中System.loadLibrary("main")作用为加载.so文件。而我们后面所生成的.so文件将以main来命名，此处指定main来调用相应的.so文件。而我们声明的getNDKPrint函数使用native关键字修饰，表明该方法使用非Java来实现。

## 3.生成相应的C/C++头文件

&emsp;&emsp;此处选择Android Studio下方的Terminal，进入命令行界面，并键入以下命令：

![][4]

```
cd app\src\main
javah -d jni -classpath ./java com.example.jnistaticregister.NDKHelper
```

&emsp;&emsp;解释以下第二条指令，javah为Java中用来生成相应C/C++头文件的一个指令，-d为生成目录的选项，-classpath为需要生成C/C++头文件的类的完整类名（即完整包名+类名）的所在路径，整条指令的意思就是在当前目录下创建一个jni目录，并在该目录下生成相应的头文件，而所需生成头文件的类在./java这个路径下。

&emsp;&emsp;命令执行完后，我们就可以在相应路径下看到jni目录及生成的头文件：

![][5]

&emsp;&emsp;实际上除了通过使用命令行的方式生成外，我们也可以手动编写头文件，只要按照JNI规则进行编写即可。

## 4.编写相应的C/C++函数实现

&emsp;&emsp;有了头文件及相应的函数声明，就需要对函数进行定义，我们在jni目录下New一个C/C++的源文件，并进行函数实现编写：

![][6]

可以看到实现的函数与生成的头文件中的函数声明是一样的，其中JNIEXPORT与JNICALL是JNI的相应关键字，jstring是JNI中的类型，JINEnv为JIN运行环境指针，jobject为JNI中的Object类。

## 5.将C/C++文件通过NDK配置关联起来

&emsp;&emsp;首先先在jni目录下新建Android.mk与Application.mk文件并进行编写：

![][7]

编写内容如下：

```
//Android.mk

LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := main
LOCAL_SRC_FILES := NDKHelper.c

include $(BUILD_SHARED_LIBRARY)
```

其中重要的为LOCAL_MODULE和LOCAL_SRC_FILES关键字，其余部分都是固定的，LOCAL_MODULE为我们需要生成的.so文件的命名，LOCAL_SRC_FILES为我们C/C++的实现源文件，将这两项指定好即可

```
//Application

APP_ABI := x86_64
APP_STL := c++_static
```
其中，APP_ABI为相应支持的CPU架构，目前Android支持的CPU架构为下图所示：

![][8]

APP_STL为标准库选择。

&emsp;&emsp;在编写完.mk文件后就可以进行NDK构建了。

### 使用ndk-build进行构建

&emsp;&emsp;构建方式如图所示：

![][9]

![][10]

选择Build-System时选择ndk-build并且在Project Path中将前面所编写好的Android.mk导入，然后选择OK。

&emsp;&emsp;在执行完上述步骤后，我们打开app下的build.gradle文件，我们发现比原来多了一部分内容：

![][11]

观察内容，其中就是给出了我们的Android.mk文件，因此我们发现，其实前面的操作和直接手动在build.gradle文件中增添这部分内容效果是一样的，只不过前面是通过IDE给我们的可视化流程来进行的。

&emsp;&emsp;前面的步骤操作完后，接下来就来配置Application.mk文件了，配置方法如下图所示：

![][12]

我们在build.gradle文件中继续增加上图所示的内容，在其中指定了Application.mk，并增加了C/C++编译所需的相应参数选项。

&emsp;&emsp;在完成前面所有的步骤后，我们build以下就基本大功告成了：

![][13]

&emsp;&emsp;build完后，我们打开相应目录app/build/即可看到生成的.so文件:

![][14]

![][15]

但我们会发现在所有的CPU架构文件下都生成了.so文件，但是我们在Application.mk文件中APP_ABI中只指定了x86_64，说明Application.mk中的该选项没有生效，这是一个bug现象，所以为了完成在指定的相应架构目录下生成.so文件的操作，需要以下操作：

![][16]

我们还是在build.gradle增加该部分内容，以指定生成相应ABI的.so文件。将刚才所生成的.so文件全部删掉再重新build一下，就发现只有在指定的架构目录才生成.so文件了。

## 6.检验成功

&emsp;&emsp;前面的步骤我们完成了一个工作，即NDKHelper类中的getNDKPrint函数其实现是使用C/C++来实现的，我们通过调用该函数来直观看看是否实现成功，实现如下：

![][17]

可以看到我们在button的监听中实例化了NDKHelper对象，并调用了getNDKPrint函数获取相应的String并重新设置TextView的文字显示。在虚拟调试机上，我们点击按钮后，发现显示的字变了即表明我们的函数已调用成功。

![][18]

![][19]

# 末言

&emsp;&emsp;本文大致地总结了NDK的简单构建流程，但对于其中的原理及涉及到东西如文中提到的JINEXPORT和JINCALL等内容并未详细解释，这部分内容在后续进行了更深入的学习后再进行总结。

[1]: /images/NDK_Build/1.png "图1"
[2]: /images/NDK_Build/2.png "图2"
[3]: /images/NDK_Build/3.png "图3"
[4]: /images/NDK_Build/4.png "图4"
[5]: /images/NDK_Build/5.png "图5"
[6]: /images/NDK_Build/6.png "图6"
[7]: /images/NDK_Build/7.png "图7"
[8]: /images/NDK_Build/8.png "图8"
[9]: /images/NDK_Build/9.png "图9"
[10]: /images/NDK_Build/10.png "图10"
[11]: /images/NDK_Build/11.png "图11"
[12]: /images/NDK_Build/12.png "图12"
[13]: /images/NDK_Build/13.png "图13"
[14]: /images/NDK_Build/14.png "图14"
[15]: /images/NDK_Build/15.png "图15"
[16]: /images/NDK_Build/16.png "图16"
[17]: /images/NDK_Build/17.png "图17"
[18]: /images/NDK_Build/18.png "图18"
[19]: /images/NDK_Build/19.png "图19"