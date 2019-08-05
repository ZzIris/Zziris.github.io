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
            "access_key": "04TL98GUDSH5ZVXM3SN8",
            "secret_key": "JskLgBlTbdwVUdLmLhk6C37CEMx4eQhO9D1eXNy9"
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

&emsp;&emsp;准备工作做好后我们就可以进行S3接口使用的测试代码了，此处分为三部分s3.h，s3.cpp以及s3_main.cpp：
```
//s3.h
#include <stdlib.h>
#include <iostream>
#include <fstream>
#include <cstring>
#include <sys/stat.h>
#include "libs3.h"

struct put_object_callback_data
{
    FILE *infile;
    uint64_t contentLength;
}; 

extern const char access_key[];
extern const char secret_key[];
extern const char host[];
extern const char sample_bucket[];
extern const char sample_key[];
extern const char sample_file[];
extern const char *security_token;
extern const char *auth_region;

extern S3Status responsePropertiesCallback(const S3ResponseProperties *properties, 
                void *callbackData);
extern void responseCompleteCallback(S3Status status, const S3ErrorDetails *error, 
                void *callbackData);
extern S3Status listServiceCallback(const char *ownerId, const char *ownerDisplayName,
                const char *bucketName,
                int64_t creationDate, void *callbackData);
extern S3Status listBucketCallback(int isTruncated, const char *nextMarker, int contentsCount,
                const S3ListBucketContent *contents, int commonPrefixesCount,
                const char **commonPrefixes, void *callbackData);
extern int putObjectDataCallback(int bufferSize, char *buffer, void *callbackData);
extern S3Status getObjectDataCallback(int bufferSize, const char *buffer, void *callbackData);

extern S3BucketContext bucketContext;

extern S3ResponseHandler responseHandler;

extern S3ListServiceHandler listServiceHandler;

extern S3ListBucketHandler listBucketHandler;

extern S3PutObjectHandler putObjectHandler;

extern S3GetObjectHandler getObjectHandler;

extern S3ResponseHandler deleteResponseHandler;

```

```
//s3.cpp
#include "s3.h"

const char access_key[] = "04TL98GUDSH5ZVXM3SN8";
const char secret_key[] = "JskLgBlTbdwVUdLmLhk6C37CEMx4eQhO9D1eXNy9";
const char host[] = "192.168.0.106:7480";
const char sample_bucket[] = "sample_bucket";
const char sample_key[] = "IMG_8208.MOV";
const char sample_file[] = "./IMG_8208.MOV";
const char *security_token = nullptr;
const char *auth_region = nullptr;

S3Status responsePropertiesCallback(
                const S3ResponseProperties *properties,
                void *callbackData)
{
        return S3StatusOK;
}

void responseCompleteCallback(
                S3Status status,
                const S3ErrorDetails *error,
                void *callbackData)
{
        return;
}

S3Status listServiceCallback(
                const char *ownerId,
                const char *ownerDisplayName,
                const char *bucketName,
                int64_t creationDate, void *callbackData)
{
        bool *header_printed = (bool*) callbackData;
        if (!*header_printed) {
                *header_printed = true;
                printf("%-22s", "       Bucket");
                printf("  %-20s  %-12s", "     Owner ID", "Display Name");
                printf("\n");
                printf("----------------------");
                printf("  --------------------" "  ------------");
                printf("\n");
        }

        printf("%-22s", bucketName);
        printf("  %-20s  %-12s", ownerId ? ownerId : "", ownerDisplayName ? ownerDisplayName : "");
        printf("\n");

        return S3StatusOK;
}


S3Status listBucketCallback(
                int isTruncated,
                const char *nextMarker,
                int contentsCount,
                const S3ListBucketContent *contents,
                int commonPrefixesCount,
                const char **commonPrefixes,
                void *callbackData)
{
        printf("%-22s", "      Object Name");
        printf("  %-5s  %-20s", "Size", "   Last Modified");
        printf("\n");
        printf("----------------------");
        printf("  -----" "  --------------------");
        printf("\n");

    for (int i = 0; i < contentsCount; i++) {
        char timebuf[256];
                char sizebuf[16];
        const S3ListBucketContent *content = &(contents[i]);
                time_t t = (time_t) content->lastModified;

                strftime(timebuf, sizeof(timebuf), "%Y-%m-%dT%H:%M:%SZ", gmtime(&t));
                sprintf(sizebuf, "%5llu", (unsigned long long) content->size);
                printf("%-22s  %s  %s\n", content->key, sizebuf, timebuf);
    }

    return S3StatusOK;
}

int putObjectDataCallback(int bufferSize, char *buffer, void *callbackData)
{
    put_object_callback_data *data = (put_object_callback_data *) callbackData;

    int ret = 0;

    if (data->contentLength) {
        int toRead = ((data->contentLength > (unsigned) bufferSize) ? (unsigned) bufferSize : data->contentLength);
                ret = fread(buffer, 1, toRead, data->infile);
    }
    data->contentLength -= ret;
    return ret;
}

S3Status getObjectDataCallback(int bufferSize, const char *buffer, void *callbackData)
{
        FILE *outfile = (FILE *) callbackData;
        size_t wrote = fwrite(buffer, 1, bufferSize, outfile);
        return ((wrote < (size_t) bufferSize) ? S3StatusAbortedByCallback : S3StatusOK);
}

S3BucketContext bucketContext =
{
        host,
        sample_bucket,
        S3ProtocolHTTP,
        S3UriStylePath,
        access_key,
        secret_key,
        security_token
};

S3ResponseHandler responseHandler =
{
        &responsePropertiesCallback,
        &responseCompleteCallback
};

S3ListServiceHandler listServiceHandler =
{
        responseHandler,
        &listServiceCallback
};

S3ListBucketHandler listBucketHandler =
{
        responseHandler,
        &listBucketCallback
};

S3PutObjectHandler putObjectHandler =
{
        responseHandler,
        &putObjectDataCallback
};

S3GetObjectHandler getObjectHandler =
{
        responseHandler,
        &getObjectDataCallback
};

S3ResponseHandler deleteResponseHandler =
{
        0,
        &responseCompleteCallback
};
```

```
//s3_main.cpp
#include "s3.h"

int main(int argc, char **argv){
    //初始化，创建连接
    S3_initialize("s3", S3_INIT_ALL, host);
    S3_deinitialize();
    bool header_printed = false;

    for(int i = 1; i < argc;){
        //列出服务器里的桶 -lb
        if(0 == strcmp(argv[i], "-lb")){
            S3_list_service(S3ProtocolHTTP, access_key, secret_key, security_token, host,
            nullptr, &listServiceHandler, &header_printed);
            std::cout << std::endl;
            ++i;
            std::cout << "1" <<std::endl;
            continue;
        }
        //列出已有的桶里的文件 -lf bucket_name
        if(0 == strcmp(argv[i], "-lf")){
            ++i;
            if('-' == argv[i][0]){
                std::cout << "Usage: -lf bucket_name" << std::endl;
                std::cout << "3" <<std::endl;
                exit(-1);
            }
            const char *s_bucket = argv[i];
            S3BucketContext bucketContext =
            {
                host,
                s_bucket,
                S3ProtocolHTTP,
                S3UriStylePath,
                access_key,
                secret_key,
                security_token
            };
            S3_list_bucket(&bucketContext, nullptr, nullptr, nullptr, 0, nullptr, 
            &listBucketHandler, nullptr);
            std::cout << std::endl;
            ++i;
            std::cout << "2" <<std::endl;
            continue;
        }
        //创建新的桶 -c bucket_name
        if(0 == strcmp(argv[i], "-c")){
            ++i;
            if('-' == argv[i][0]){
                std::cout << "Usage: -c bucket_name" << std::endl;
                std::cout << "3" <<std::endl;
                exit(-1);
            }
            else{
                const char *s_bucket = argv[i];
                S3_create_bucket(S3ProtocolHTTP, access_key, secret_key, nullptr, host, s_bucket, 
                        S3CannedAclPrivate, nullptr, nullptr, &responseHandler, nullptr);
                std::cout << "4" <<std::endl;
            }
            ++i;
            continue;
        }
        //删除桶 -db bucket_name
        if(0 == strcmp(argv[i], "-db")){
            ++i;
            if('-' == argv[i][0]){
                std::cout << "Usage: -db bucket_name" << std::endl;
                std::cout << "7" <<std::endl;
                exit(-1);
            }
            else{
                const char *s_bucket = argv[i];
                S3_delete_bucket(S3ProtocolHTTP, S3UriStylePath, access_key, secret_key, nullptr, 
                        host, s_bucket, nullptr, &responseHandler, nullptr);
                std::cout << "8" <<std::endl;
            }
            ++i;
            continue;
        }
        //存储文件对象 -s bucket_name file_name
        if(0 == strcmp(argv[i], "-s")){
            if('-' == argv[i + 1][0] || '-' == argv[i + 2][0]){
                std::cout << "Usage: -s bucket_name file_name" << std::endl;
                std::cout << "9" <<std::endl;
                exit(-1);
            }
            else{
                const char *s_bucket = argv[i + 1];
                const char *s_file = argv[i + 2];
                put_object_callback_data data;
                struct stat statbuf;
                if (stat(s_file, &statbuf) == -1) {
                    fprintf(stderr, "\nERROR: Failed to stat file %s: ", s_file);
                    perror(0);
                    exit(-1);
                }

                int contentLength = statbuf.st_size;
                data.contentLength = contentLength;

                if (!(data.infile = fopen(s_file, "r"))) {
                    fprintf(stderr, "\nERROR: Failed to open input file %s: ", s_file);
                    perror(0);
                    exit(-1);
                }
                S3BucketContext bucketContext =
                {
                    host,
                    s_bucket,
                    S3ProtocolHTTP,
                    S3UriStylePath,
                    access_key,
                    secret_key,
                    security_token
                };
                S3_put_object(&bucketContext, s_file, contentLength, nullptr, nullptr, 
                        &putObjectHandler, &data);
                fclose(data.infile);
                std::cout << "10" <<std::endl;
            }
            i += 3;
            continue;
        }
        //从桶里下载文件 -download bucket_name dl_filename
        if(0 == strcmp(argv[i], "-download")){
            if('-' == argv[i + 1][0] || '-' == argv[i + 2][0]){
                std::cout << "Usage: -download  bucket_name file_name" << std::endl;
                std::cout << "11" <<std::endl;
                exit(-1);
            }
            else{
                const char *s_bucket = argv[i + 1];
                const char *s_file = argv[i + 2];
                FILE *outfile = fopen(s_file, "w");
                S3BucketContext bucketContext =
                {
                    host,
                    s_bucket,
                    S3ProtocolHTTP,
                    S3UriStylePath,
                    access_key,
                    secret_key,
                    security_token
                };
                S3_get_object(&bucketContext, s_file, nullptr, 0, 0, nullptr, &getObjectHandler, outfile);
                fclose(outfile);
                std::cout << "12" <<std::endl;
            }
            i += 3;
            continue;
        }
        //删除桶中的文件对象 -df bucket_name file_name
        if(0 == strcmp(argv[i], "-df")){
            if('-' == argv[i + 1][0] || '-' == argv[i + 2][0]){
                std::cout << "Usage: -df bucket_name file_name" << std::endl;
                std::cout << "13" <<std::endl;
                exit(-1);
            }
            else{
                const char *s_bucket = argv[i + 1];
                const char *s_file = argv[i + 2];
                S3BucketContext bucketContext =
                {
                    host,
                    s_bucket,
                    S3ProtocolHTTP,
                    S3UriStylePath,
                    access_key,
                    secret_key,
                    security_token
                };
                S3_delete_object(&bucketContext, s_file, nullptr, &deleteResponseHandler, nullptr);
                std::cout << "14" <<std::endl;
            }
            i += 3;
            continue;
        }
    }
    
    return 0;
}
```
&emsp;&emsp;代码编写好后进行编译：
```
$ sudo g++ --std=c++11 -pthread -l s3 s3.cpp s3_main.cpp -o s3_main
```
&emsp;&emsp;编译好后，运行生成的执行文件并按照代码里给的指令格式对集群进行相应的操作：
```
$ ./s3_main 选项 [参数]
```

## 2.使用S3Browser来查看操作效果

&emsp;&emsp;我在命令行上对集群来进行操作显得不够直观，我们可以通过前面在Windows上安装好的S3Browser来直接观察效果。

### S3Browser连接集群
&emsp;&emsp;点击Account后，再选择Add new account选项：
![](/images/S3Example/1.png)
&emsp;&emsp;然后相应填入信息，其中Account Name随意填，Account Type注意选择S3 Compatible Storage，REST Endpoint填入的是创建了对象网关的节点地址，并且给出其对象网关的端口号，默认7480，然后输入Access Key ID和Secret Access Key，最后把(SSL/TLS)的勾给去掉（默认是勾上的）。
![](/images/S3Example/2.png)
这样便可以让S3Browser连上我们的ceph集群了，对桶的创建删除，对文件的上传下载删除都可以通过S3Browser来直观看到效果。

# 三. 末语
&emsp;&emsp;通过对自己搭好的Ceph集群来进行操作，可以更直观地对Ceph有进一步的了解，知道了如何使用相应的S3协议来往Ceph集群中进行文件的存储下载等。