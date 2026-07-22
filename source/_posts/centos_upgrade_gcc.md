---
title: centos7 升级gcc
date: 2024-12-15 22:26:08
tags:
---
# centos 升级gcc

## 方法1

```
scl enable devtoolset-7 bash

vim /etc/profile

source /opt/rh/devtoolset-7/enable
```

## 方法2 

源码编译

GCC源码地址为http://ftp.gnu.org/gnu/gcc

```
wget http://ftp.gnu.org/gnu/gcc/gcc-8.3.0/gcc-8.3.0.tar.gz
tar -zxvf gcc-8.3.0.tar.gz

cd gcc-8.3.0
```

利用源码包里自带的工具下载所需要的依赖项，确保系统可以联网

```
./contrib/download_prerequisites
```

## 安装

```
mkdir build
cd build
../configure --enable-checking=release --enable-languages=c,c++ --disable-multilib
make
make install
```

引用

https://blog.csdn.net/jiangshuanshuan/article/details/103890748