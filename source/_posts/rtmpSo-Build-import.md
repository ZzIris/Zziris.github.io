---
title: Android NDK开发例程-rtmp的动态库封装以及导入使用
date: 2019-08-29 16:16:06
tags:
- Android
- NDK
- rtmp
categories:
- Android学习
---

# 一. 前言

&emsp;&emsp;本文的主要内容是介绍在Windows平台下Android Studio中使用ndk-build方式和CMake方式进行rtmp动态库的封装并进行调用，并对在这一实践操作历程中所踩的坑进行归纳总结。

# 二. 使用ndk-build方式封装rtmp动态库并进行调用

## 1. 封装rtmp动态库-librtmp.so

### 下载rtmp源码

&emsp;&emsp;为了对rtmp进行封装，我们首先需要获取其源码，源码下载直接到[官网下载](http://rtmpdump.mplayerhq.hu/download/)即可。此处本例程所下载的是rtmpdump-2.3.tgz。下载后进行解压，在得到的解压文件夹中的librtmp文件夹里便是源码，我们只需将其.h头文件和.c源文件取出即可。

### 创建封装rtmp动态库的工程

&emsp;&emsp;在Android Studio中新建名为rtmpNDKBuild的工程，并在app/src/main路径下创建两个文件夹分别为rtmp_inc和rtmp_src，其分别用来存放rtmp源码的头文件和源文件，如图所示：

![][1]

### 编写Android.mk文件

&emsp;&emsp;在app/src/main/cpp下新建Android.mk文件，编写内容如下：

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := rtmp

LOCAL_SRC_FILES := \
        rtmp_src/amf.c \
        rtmp_src/hashswf.c \
        rtmp_src/log.c \
        rtmp_src/parseurl.c \
        rtmp_src/rtmp.c \

LOCAL_C_INCLUDES += $(LOCAL_PATH)/rtmp_inc

include $(BUILD_SHARED_LIBRARY)
```

此处需要注意该文件中的关键字不要打错，例如我自己就在编写Android.mk文件时把其中的LOCAL_SRC_FILES打成了LOCAL_SRC_FILE，少了个S，虽然在该封装rtmp动态库的工程中可能会顺利通过编译生成.so文件，但这样编译出来的.so文件是存在问题的。并且该失误若出现在调用so的过程中则会一直报错，后面我们介绍到调用封装好的so的部分时会再次提到。

&emsp;&emsp;此处另一个注意的地方是关于头文件，由于在rtmp的源文件中对头文件是直接进行include的，如#include<rtmp_sys.h>，这意味着头文件和源文件是在一个目录下，但由于此处我们将它们分开存放于两个文件夹，因此需要在此处指定LOCAL_C_INCLUDES，并且在指定形式上不能写成相对路径，如LOCAL_C_INCLUDES += rtmp_inc，这样依然无法找到头文件，因此需要把我们的LOCAL_PATH加上。

### 进行编译

&emsp;&emsp;在完成了Android.mk后，我们就可以进行编译了，我们首先Link C++ Project with Gradle，该步具体流程可见另一篇文章《Android开发中使用ndk-build进行NDK构建》。完成之后再点击Build->Make Project进行编译。但此时我们发现，编译报错了，报错如下图所示：

![][2]

错误意思就是 找不到openssl这个相应文件目录下的头文件，至于openssl，其作用大致是用于对rtmp推流进行加密的，然而我们目前还不需要进行加密操作，因此可以先不使用它，具体的操作方法是修改rtmp的相应头文件rtmp_sys.h文件：

```
...
#define NO_CRYPTO   //在#ifdef _WIN32前增加该宏定义
#ifdef _WIN32

#ifdef _XBOX
#include <xtl.h>
#include <winsockx.h>
#define snprintf _snprintf
#define strcasecmp stricmp
#define strncasecmp strnicmp
#define vsnprintf _vsnprintf
...
```

修改之处便是在rtmp_sys.h中的#ifdef _WIN32之前增加宏定义#define NO_CRYPTO即可。

&emsp;&emsp;再重新进行编译，就可以通过了，查看app/build/intermediates/ndkBuild/debug/obj/local目录下，就可以找到相应的对应的ABI文件夹，每个文件夹下都有一个librtmp.so和objs-debug文件夹，该文件目录的子目录下是相应的rtmp源码生成的.o文件。至此librtmp.so就封装成功了。

## 2. 调用librtmp.so

&emsp;&emsp;在得到librtmp.so后，便是对其中的接口方法进行调用以测试是否真正封装成功。

### 创建调用rtmp动态库的工程

&emsp;&emsp;首先新建一个名为UseRtmpLib的工程，并在app/src/main路径下创建rtmp_lib文件夹，并将之前生成的rtmp动态库移到该目录下，如下图所示：

![][3]

### 新增带有native方法的Java类

&emsp;&emsp;接着再工程下新增一个名为UseRtmpLib的Java类，并完成库加载和native方法声明，如图所示：

![][4]

### 创建native方法的实现

&emsp;&emsp;在app/src/main路径下创建cpp文件夹，并在该文件夹下创建include和source两个子文件夹，分别用来存放native方法实现的头文件和源码文件，同时在include文件夹中还需要把rtmp相关的头文件夹也放进去，注意此处放入的rtmp头文件中的rtmp_sys.h一定是我们之前进行修改过的。如图所示：

![][5]



&emsp;&emsp;native方法实现的头文件和源文件内容为：

```
// useRtmpLib.h

#include <jni.h>
#include "rtmp_sys.h"   //调用rtmp的相应头文件
#ifndef USERTMPLIB_USERTMPLIB_H
#define USERTMPLIB_USERTMPLIB_H

#ifdef __cplusplus
extern "C"{
#endif

JNIEXPORT jstring JNICALL Java_com_example_usertmplib_UseRtmpLib_rtmpLibTest(JNIEnv *env, jobject obj);

#ifdef __cplusplus
}
#endif

#endif //USERTMPLIB_USERTMPLIB_H
```

```
// useRtmpLib.c

#include "useRtmpLib.h"
#ifdef __cplusplus
extern "C"{
#endif

JNIEXPORT jstring JNICALL Java_com_example_usertmplib_UseRtmpLib_rtmpLibTest(JNIEnv *env, jobject obj){

    RTMP *rtmp = RTMP_Alloc();  //使用rtmp.so中的接口方法
    RTMP_Init(rtmp);   //使用rtmp.so中的接口方法
    return (*env) -> NewStringUTF(env, "rtmpLib test successfully");

}

#ifdef __cplusplus
}
#endif
```

在编写native实现时，要注意遵守JNI规则，否则无法调用。

### 编写Android.mk文件

在app/src/main/cpp下新建Android.mk文件，编写内容如下：

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE    := librtmp
LOCAL_SRC_FILES := ../rtmp_lib/x86_64/librtmp.so
include $(PREBUILT_SHARED_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := rtmpLibTest
LOCAL_SHARED_LIBRARIES := rtmp
LOCAL_SRC_FILES := source/useRtmpLib.c
LOCAL_C_INCLUDES += $(LOCAL_PATH)/include/
include $(BUILD_SHARED_LIBRARY)
```

此处注意若把上述内容中下半部分的LOCAL_SRC_FILES少打一个S即变成LOCAL_SRC_FILE，则会引发错误，所以需要注意关键字的拼写。

![][6]

![][7]

&emsp;&emsp;此处的Android.mk文件编写很关键，为了调用我们之前封装好的so，就需要这么编写Android.mk。在文件的上半部分，其指定的是我们所需调用的.so文件的具体模块名和.so文件的文件位置，可以使用相对路径，并且后面的include中使用的是PREBUILT_SHARED_LIBRARY关键字，表示这是已经编译好的动态库。在文件的下半部分，给出的是我们当前调用rtmp动态库的native方法实现的源文件信息，给出源文件的位置同时还需指定我们所需的头文件的位置，头文件的指定方式如前面所述。其也相应要封装成一个.so文件，因此也需要指定其生成动态库的模块名。而这一部分后面的include中使用的是BUILD_SHARED_LIBRARY关键字

&emsp;&emsp;总体而言就是我们编译一个符合JNI规范的.so文件以在Java实现native方法，而该符合JNI规范的.so文件中又相应地调用了不符合JNI规范的.so文件（仔细回想，我们之前封装librtmp.so时并没有对其源码的方法形式进行修改使其符合JNI规范，而是直接进行封装，这样的.so时无法直接再Java层通过native方法来使用的，要想调用它只能在其上面再进行一层符合JNI规范的封装来实现相应的功能以进行调用）。

### 调用native方法

&emsp;&emsp;我们实例化含有native方法的类，然后再调用native方法看是否出现报错，实现如图所示：

![][8]

### 进行编译
&emsp;&emsp;进行Link C++ Project with Gradle，然后点击运行即可，此处发现可以正常运行没有报错，说明在native方法的实现中，对rtmp接口的调用是正常的。即librtmp调用正常。

&emsp;&emsp;但此处有一个问题，当你点击的是Build->Make Project的话，会发现竟然报错了，报错如下：

![][9]

错误意思就是没有找到相关librtmp中的接口函数的实现。这就很奇怪了，为什么直接run是正常的，但Make Project就发生错误了呢？

&emsp;&emsp;在经历了一番折腾之后终于发现了问题。我们查看报错信息的第二行的中间部分，发现如图中所示的信息：

![][10]

&emsp;&emsp;此处发现ndk-build的APP_ABI选项为X86，而我们在Android.mk中指定librtmp.so时只指定了x86_64下的动态库.so的路径，这也就是说该报错是由于编译x86ABI的librtmpLibTest.so时报的错，为了证实这个猜想，我们到app/build/intermediates/ndkBuild/debug/obj/local底下进行查看，我们发现一下情况：

![][11]

&emsp;&emsp;从图中我们可以看出x86_64下librtmpLibTest.so已经成功编译，并且引用了librtmp.so，而在x86下只有librtmp.so而没有librtmpLibTest.so，因为此时librtmpLibTest.so编译报错了所以不会出现，而出现在该目录下的librtmp.so其实是x86_64版本的，因此发生了链接错误。而其他ABI文件夹下都还没来得及进行编译所以不存在任何.so文件。问题终于搞清楚了，原来在进行Make Project时，若没有对APP_ABI进行指定，则会对所有的ABI版本都进行编译，而run的时候只会运行我们虚拟机所使用的系统的ABI版本的.so（本例程中所使用的虚拟机的系统是x86_64的），因此导致了run没有报错而Make Project会报错的情况。

&emsp;&emsp;为了解决该问题，只需修改build.gradle即可，修改如下图所示：

![][12]

我们只需指定abiFilter，让Make Project只对X86_64版本的.so进行编译即可，这样就不会出错了，打开app/build/intermediates/ndkBuild/debug/obj/local进行查看：

![][13]

这时就只生成了x86_64的文件夹，其他的ABI文件夹都没又出现了。

# 三. 使用CMake方式封装rtmp动态库并进行调用

&emsp;&emsp;在上一节中，我们使用了ndk-build的方式对rtmp进行了动态库封装以及调用，而本节将改用Cmake的方式来进行对rtmp动态库封装以及调用。

## 1. 封装rtmp动态库-librtmp.so

### 创建封装rtmp动态库的工程

&emsp;&emsp;rtmp的源码我们在上一节中已经获取到了。我们新建一个名为RtmpCmakeTest的工程，不过注意的是，此时创建的工程和我们平时创建的工程不太一样，不再是选择xxx Activity，而是需要选择Native C++。

![][14]

&emsp;&emsp;在创建了这个工程后，进去后会发现，在我们平时的app/src/main路径下多了一个cpp的目录，且在该目录下还有一个CMakeLists.txt文件和IDE为我们自动生成的native-lib.cpp（这是一个IDE为我们自动生成的native样例，其Java层的native函数声明在MainActivity类中）。然后我们在该cpp目录下创建两个目录分别为include和src，分别用来存放rtmp的头文件和源文件。

![][15]

&emsp;&emsp;接着我们修改CMakeLists.txt文件，内容为：
```
cmake_minimum_required(VERSION 3.4.1)

add_library( # Sets the name of the library.
             rtmp

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/rtmp.c src/parseurl.c src/log.c src/hashswf.c src/amf.c)

target_include_directories(rtmp PRIVATE
        ${CMAKE_SOURCE_DIR}/include)


find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

#target_link_libraries( # Specifies the target library.
#                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
#                       ${log-lib} )
```

由于我们是生成不满足JNI规范的.so文件，并非直接供给Java层调用的.so，因此我们将最后一部分target_link_libraries的内容注释掉。

&emsp;&emsp;此时我们同样不需要openssl，所以还是需要对rtmp_sys.h进行处理，处理方式如上一节所述。

&emsp;&emsp;接着，我们点击Make Project即可，这样就可以生成相应的librtmp.so了。

## 2. 调用librtmp.so

&emsp;&emsp;我们重新创建一个工程并命名为RtmpUseCmakeTest，此处使用的同样是native C++工程，然后我们在cpp目录下创建两个文件夹分别为include和libs，一个用来存放rtmp的头文件，另一个用来存放我们前面生成的.so文件。

![][16]

此处我将ABI为x86_64的librtmp.so文件移到了libs目录下，注意要同时带有x86_64的文件夹（IDE进行了优化，将目录整合成了libs.x86_64）

&emsp;&emsp;然后我们修改CMakeLists.txt文件，内容为：
```
cmake_minimum_required(VERSION 3.4.1)
set(LIB_DIR ${CMAKE_SOURCE_DIR}/libs)

add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        native-lib.cpp)

target_include_directories(native-lib PRIVATE
        ${CMAKE_SOURCE_DIR}/include)

add_library(rtmp SHARED IMPORTED)
set_target_properties(rtmp
        PROPERTIES IMPORTED_LOCATION
        ${LIB_DIR}/${ANDROID_ABI}/librtmp.so)

find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log)

target_link_libraries( # Specifies the target library.
        native-lib
        rtmp
        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})
```

此处由于我们是要在符合JNI规范的native-lib.cpp中调用我们不符合JNI规范的librtmp.so，因此我们增加了第二个add_library，并且第一个参数指定了需要调用的动态库的名称，第三个参数指定为IMPORTED，同时设置了set_target_properties，指出了librtmp.so的路径（文件前面调用set设置了LIB_DIR的值），最后在target_link_libraries中给出需要link的所有动态库的名称。

&emsp;&emsp;在修改完CMakeLists.txt文件，我们对native-lib.cpp修改一下，让其调用相关的rtmp的函数，内容如下：

```
//native-lib.cpp

#include <jni.h>
#include <string>
#include "rtmp_sys.h"   //引入rtmp头文件

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_rtmpusecmaketest_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "RTMP Cmake Test";
    RTMP *rtmp = RTMP_Alloc();   //调用rtmp的函数
    RTMP_Init(rtmp);   //调用rtmp的函数

    return env->NewStringUTF(hello.c_str());
}
```

由于在MainActivity中已经帮我们调用了声明的native函数，所以我们不需要修改

&emsp;&emsp;而为了避免出现上一节中Make Project会编译全部ABI类型的.so文件（我们只导入了x86_64的动态库文件）导致Make Project不通过而Run能通过的情况，我们需要在build.gradle中指定一下ABI，如下所示：

![][17]

&emsp;&emsp;然后我们点击Make Project，发现通过了，然后点击Run，这时候问题来了，发生了报错，报错内容如下：

![][18]

&emsp;&emsp;问题是找不到librtmp.so这个动态库文件，后来经过一系列的折腾及网上答案的查找后发现，在运行程序之前还需要在build.gradle中指定要调用的动态库的路径，这针对的是这种不放在默认目录下而是放在自定义位置的动态库文件。我们修改build.gradle文件，增加的部分如图所示：

![][19]

其中jniLibs.srcDirs的默认路径为app/libs，将其改为我们自定义存放动态库文件的目录路径即可。

&emsp;&emsp;以上设置完成点击Run,即可运行成功。

# 四. 末言

&emsp;&emsp;总结一下本文所做的事情。本文的例程使用了ndk-build和CMake两种方式进行对rtmp的封装并调用。即将rtmp源码进行直接封装，其方法形式没有符合JNI规范，为了对其进行调用，就在其基础上，又封装了一层符合JNI规范的.so库文件，在其中调用了rtmp的.so库文件，然后在Java层使用native方法来调用符合JNI规范的.so库文件来达到使用rtmp的效果。

[1]: /images/rtmp_build_import/1.png "图1"
[2]: /images/rtmp_build_import/2.png "图2"
[3]: /images/rtmp_build_import/3.png "图3"
[4]: /images/rtmp_build_import/4.png "图4"
[5]: /images/rtmp_build_import/5.png "图5"
[6]: /images/rtmp_build_import/6.png "图6 错误显示左半部分"
[7]: /images/rtmp_build_import/7.png "图7 错误显示右半部分"
[8]: /images/rtmp_build_import/8.png "图8"
[9]: /images/rtmp_build_import/9.png "图9"
[10]: /images/rtmp_build_import/10.png "图10"
[11]: /images/rtmp_build_import/11.png "图11"
[12]: /images/rtmp_build_import/12.png "图12"
[13]: /images/rtmp_build_import/13.png "图13"
[14]: /images/rtmp_build_import/14.png "图14"
[15]: /images/rtmp_build_import/15.png "图15"
[16]: /images/rtmp_build_import/16.png "图16"
[17]: /images/rtmp_build_import/17.png "图17"
[18]: /images/rtmp_build_import/18.png "图18"
[19]: /images/rtmp_build_import/19.png "图19"
