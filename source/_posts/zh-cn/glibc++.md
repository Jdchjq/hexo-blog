---
title: Centos7 升级 glibc2.31
date: 2024-08-10 15:03:25
updated:
keywords:
slug: centos7+glibc
cover: /image/glibc++.png
top_image:
comments: false
maincolor:
categories:
  - 后端开发
tags:
  - glibc
typora-root-url: ./glibc++
---

## 背景

centos7 是常见的 linux 开源系统，虽然官方已经停止维护，但仍占有较高的市场。系统自带的 glibc 版本是 2.17，当开发需要用到 node.js18 时，就会报错需要 GLIBC_2.28。
报错如下：

```bash
[root@root ~]# node -v
node: /lib64/libm.so.6: version `GLIBC_2.27' not found (required by node)
node: /lib64/libc.so.6: version `GLIBC_2.25' not found (required by node)
node: /lib64/libc.so.6: version `GLIBC_2.28' not found (required by node)
node: /lib64/libstdc++.so.6: version `CXXABI_1.3.9' not found (required by node)
node: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by node)
node: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found (required by node)
```

下面介绍如何在 centos7 系统升级 glibc。

{% note warning modern %}
升级 glibc 版本有风险导致系统崩溃，升级前请先备份好服务器重要资料。
glibc 有些版本是不带前面连续的版本安装包的，这样有可能导致在使用的时候引用前面版本的代码直接报错，因此跨版本升级有风险。
glibc 2.31 亲测是携带了连续的版本安装包，因此直接从 2.17 升级到 2.31 没有什么风险，但建议还是自己先找开发机测试一下，同时也要注意备份。
{% endnote %}

## 安装过程

### 1、检查当前系统

```bash
cat /etc/redhat-release
strings /lib64/libc.so.6 | grep GLIBC
ll /lib64/libc.so*
```

查看当前系统版本，是 Centos7
CentOS Linux release 7.2.1511 (Core)

并且可以看到，目前 GLIBC 最新版本是 2.17
![](img1.png)

同时可以发现，当前 libc.so 是指向版本 2.17
![](img3.png)

### 2、源码编译升级 gcc9.3.0

> 前提：进入 root 模式执行以下命令

1）安装编译

```bash
wget https://mirrors.aliyun.com/gnu/gcc/gcc-9.3.0/gcc-9.3.0.tar.gz
cp gcc-9.3.0.tar.gz /opt
cd /opt
tar -zxf gcc-9.3.0.tar.gz
cd gcc-9.3.0/

./contrib/download_prerequisites
cat /proc/cpuinfo| grep "processor"| wc -l

mkdir build
cd build
../configure --enable-checking=release --enable-language=c,c++ --disable-multilib --prefix=/usr
make -j6
make install
```

下载 gcc-9.0.3 安装包，然后使用命令 `./contrib/download_prerequisites` 安装升级 gcc 所需的依赖。

如果无法升级依赖，可以到下载地址 https://gcc.gnu.org/pub/gcc/infrastructure/ 手动下载这四个文件到目录/opt/gcc-9.3.0 下
![](img5.png)

使用命令 `cat /proc/cpuinfo| grep "processor"| wc -l` 查看服务器有多少内核，等下用于编译。多内核意味着可以开多线程编译，速度会更快。

开始配置并编译新版 gcc
![](img7.png)

前面查看服务器有 24 个内核，由于服务器还有其他进程在运行，这里只使用 6 个线程进行编译，即 `make -j6`，等待编译。执行完成结果如下：
![](img8.png)

编译完成后，执行`make install`安装
![](img9.png)

2）升级成功后检查 gcc 版本

```bash
cd /usr/lib64
ll libstdc++*
gcc -v
```

执行结果，显示 gcc 已经升级到 9.3.0
![](img10.png)

### 3、源码编译 make

1)、下载源码编译

```bash
wget https://mirrors.aliyun.com/gnu/make/make-4.3.tar.gz
cp make-4.3.tar.gz /opt
cd /opt/
tar -zxf make-4.3.tar.gz
cd make-4.3/
mkdir build
cd build
../configure --prefix=/usr
make
make install
```

下载解压，并配置编译
![](img11.png)

在 root 下执行 `make && make install`
![](img12.png)

2)、检查安装版本

```bash
make -v
```

![](img13.png)

### 4、升级 glibc2.31

```bash
cd /opt
wget https://mirrors.aliyun.com/gnu/glibc/glibc-2.31.tar.gz
tar -zxf glibc-2.31.tar.gz
cd glibc-2.31/

mkdir build
cd build
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin --disable-sanity-checks --disable-werror

make -j6
make install
```

下载 glibc2.31
![](img14.png)

配置 glibc2.31，如果没有报依赖错误，则可以继续。有些机器可能 python 版本过低，会报错，需要安装 python3.4 以上版本。
用 `cat INSTALL | grep -E "newer|later"` 可以查看依赖所需的版本
![](img15.png)

接下来是编译和安装，这个过程等待时间比较久。

> 同样的，`make -j` 表示指定 job 数量，如果你的服务器核心够多，可以适当增加，以减少编译时间

![](img16.png)

可以发现 make install 报了一个 error，提到`libnss_test2.so`这个测试库找不到，不影响安装。
继续查看 libc.so.6 的软连接，已经链接到 2.31 版本文件了
同时可以看到，2.31 前面的缺失版本也都安装上了
![](img17.png)
![](img18.png)

升级后，再次登录服务器可能会报系统语言无法设置，需要继续在 glibc build 目录下执行：

```bash
make localedata/install-locales
```

![](img19.png)

至此，升级 glibc 版本的流程全部完成

## 总结

我们通过升级 glibc 版本后，终于能满足 node.js 18 的运行要求了。如果需要升级到更高版本，最好是先在一个虚拟机或者没有生产环境程序的机器上实践。

接下来又能愉快编码了 ~ 🥳
