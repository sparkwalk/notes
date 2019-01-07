# 交叉编译c++ boost库

## 1.下载源码[boost-1.68.0](https://dl.bintray.com/boostorg/release/1.68.0/source/boost_1_68_0.tar.gz)

 ``` shell
wget https://dl.bintray.com/boostorg/release/1.68.0/source/boost_1_68_0.tar.gz
 ```

## 2.解压文件

``` shell
tar -xvf boost_1_68_0.tar.gz
cd boost_1_68_0/
```

## 3.生成配置文件

``` shell
./bootstrap.sh
```

## 4. 修改配置文件

编辑配置文件 project-build.jam, 修改

``` shell
using gcc
```

为

``` shell
using gcc : arm : arm-linux-g++ ;
```

## 5.安装python开发包

``` shell
sudo apt-get install python-dev
```

## 6.编译, 安装

``` shell
./bjam install toolset=gcc-arm --prefix=/home/licg/boost_arm --disable-long-double -sNO_ZLIB=1 -sNO_BZIP2=1
```
